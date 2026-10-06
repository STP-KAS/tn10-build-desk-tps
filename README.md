# Grok Build on TN10: throughput and monitoring

Grok Build only. This is the desk, sending through public Testnet-10 nodes. It is not the Grok bot, and it is not the box node n0.

The bot's own effort, and the joint report after the 9 or 13 Oct run, stay in [STP-KAS/tn10-storm-throughput-questions](https://github.com/STP-KAS/tn10-storm-throughput-questions). This repo is the desk's findings: how fast it could submit, what the five Kaspa Pulse questions looked like on a monitored hold, and how the public API relates to Kaspa.stream.

Kaspa Testnet-10 only. Times below are UTC unless they say CEST. No transaction ids are in this repo.

## What the desk measured

Signed one-input one-output self-transfers. Fees are sompi per gram. 1× and 1.5× run in the same process, on even and odd lanes. Submit-OK means the node accepted the transaction into its mempool. Seen-accepted means the virtual chain of the node we were watching listed that transaction. Those are different clocks.

The desk has 16 cores and 24 threads. One Node process signs on one core. More processes are how the desk went past about 2,250 tx/s. Desk miners were off. The desk's own node was still syncing, so it took no traffic.

## Throughput

| Run | Fee, sompi/gram | Window | Submit-OK | Rejects | Mean submit | Seen accepted in the window |
|---|---|---|---:|---:|---:|---:|
| 1 process, vector-10 | 100 and 150 | 20 s | 44,996 | 0 | 2,250 tx/s | 32,344 |
| 6 processes, one per public node | 100 and 150 | 20 s | 126,426 | 0 | 6,321 tx/s | 38,822 |
| 10 processes, two on each of the five faster nodes | 100 and 150 | 12 s | 109,991 | 28,617 orphans | about 9,100 tx/s | about 14,500 |
| 6 processes plus a 4 tx/s ordered stream | 158 and 237 | 45 s | 282,651 | about 3,200 orphans | 6,281 tx/s | about 1,500 tx/s during the send |

The 45-second mean was 6,281 tx/s. The slowest second was 2,952 and the fastest was 10,404. Muon-10 added about 270 tx/s in that shape and was left out of the ten-process run.

Past about 9,000 tx/s the public nodes were the limit. The desk still had free CPU and free RAM. The clean zero-reject rate was **6,321 tx/s**. The higher 9,100 figure includes a large orphan count beside it.

A child is an orphan when it arrives before that node will take its parent. Short chains and one process per node kept orphans near zero. Long chains and two processes on the same node created them.

## Monitoring, 6 Oct 2026, 20:57–21:00 UTC

This is the 45-second hold in the table above. Acceptance was vector-10's virtual chain. The box still matches the same transactions on n0. The desk clock was +60 ms against time.windows.com. Miners on the desk: 0.

| Question | Result |
|---|---|
| Submit versus accept | 282,651 submit-OK, mean 6,281 tx/s. Listeners saw about 1,500 of ours accepted per second during the send. By two minutes after the stop, 55.8% had been accepted (157,738 of 282,651). The rest were still outstanding. Accept was well under 95% of submit, so this window was saturated. The written saturation rule is 60 seconds; this send was 45. |
| Confirmation, 1× versus 1.5× | Among transactions that landed before the watch stopped: 1× median 23.0 s, p95 140 s, n=79,448. 1.5× median 20.9 s, p95 139 s, n=78,290. The extra fee bought about 2 seconds at the median. The tail did not move. 44% were still unaccepted when the watch stopped, so these percentiles are only the ones that made it in. |
| Indexer | `https://api-tn10.kaspa.org/info/health` every 30 s returned HTTP 200. The database stayed synced. Accepted-tx lag was 0–3 seconds. It did not freeze during this short hold. |
| Mempool and fee | Four public nodes climbed together to a maximum of about 181,000. vector-10 peaked near 51,000. muon-10 stayed near a median of 2,100. The normal fee estimate on the watched node moved from about 148 to about 192 sompi/gram while our tiers stayed frozen at 158 and 237. |
| Order | The ordered stream, 4 tx/s, had 4 reversals in 66 consecutive pairs sent at least 1 second apart. That is 6.1%, with a 95% interval of 2.4–14.6%. One same-event tie. No 30-second stall on that stream. On the flood, 46% of 1.5× transactions passed an earlier 1× from the same process. |

Blocks kept arriving at about 8–10 per second, with one second at 21. The network kept producing blocks while those mempools grew.

## API up, Kaspa.stream is a different path

Checked 6 Oct 2026, about 23:12 CEST.

**api-tn10.kaspa.org is up, and it is following the chain.** A cache-busting `GET /info/health` returned HTTP 200. The database was synced, `acceptedTxBlockTimeDiff` was 2 seconds, and `blueScoreDiff` was 16. The kaspad behind that health check was 2.1.0, synced, with a UTXO index. `GET /info/blockdag` answered for `kaspa-testnet-10`. `GET /info/hashrate` answered. `GET /info/kaspad` answered: synced, UTXO index on, mempool size 4, server version 2.0.1. That mempool is the API's own node. It is not the mempool of the public nodes the desk was sending to. At the same time those nodes still showed mempools from about 16,000 (vector-10) to about 72,000 (proton-10, electron-10, quark-10, neutrino-10), and their wRPC fee estimate was about 857 sompi/gram.

The REST fee view moved during the same check. One read gave a priority bucket of 4,447 sompi/gram and a normal bucket of 526. A later read, still HTTP 200, gave 100 sompi/gram in every bucket. The API is answering. Its fee and mempool are one node's view, and they are not a live picture of every public mempool.

**Kaspa.stream does not get those numbers from the page, and it does not get them from submit acknowledgements.**

- `https://tn10.kaspa.stream/` is a Vue shell. The HTML has the title and no live TPS.
- `https://tn10.kaspa.stream/api/info/network` returns that same HTML, not JSON. The explorer does not proxy api-tn10 on that path.
- The counters in the browser come from socket.io on `wss://t-1.kaspa.ws` through `wss://t-4.kaspa.ws`. The homepage cards depend on `wss://t-2.kaspa.ws`.
- From this desk, at the same minute, t-1, t-2, and t-4 returned HTTP 403 and t-3 returned HTTP 503. A browser may still be allowed where this client is forbidden. A dead or refused socket shows the charts at zero, or stuck, while api-tn10 is fine.
- Chart TPS is accepted-transaction indexing over those sockets. It lags mempool acknowledgements. A backlog can leave the chart low while the desk is submitting thousands per second.

So a frozen Kaspa.stream chart, with api-tn10 health near a 2-second lag, is the explorer socket and the chart definition. It is not an API outage. The 2 Oct 2026 freeze was the other failure: health itself went to HTTP 503 and accepted-tx indexing stuck while blocks kept moving. That is not what this check shows.

## High-fee run

Raising the fee above the 600 sompi/gram cap did not raise TPS.

The first attempt put 2,000 and 3,000 sompi/gram on chains 16 deep. On vector-10 one process submitted about 1,100–1,700 tx/s and took a similar number of orphan rejects, then the chains filled and submit fell to about zero. That attempt was stopped.

A clean 20-second repeat, one process, vector-10 only, same 2,000 and 3,000 sompi/gram, chains 6 deep:

| | |
|---|---:|
| Submit-OK | 15,707 |
| Mean submit | 785 tx/s |
| Orphan rejects | 7,667 |
| Seen accepted before stop | 13,416 |
| Still in flight | 2,291 |

785 tx/s is below the 2,250 tx/s that one process held at 100 and 150 sompi/gram with no rejects. The higher fee did not buy a higher submit rate. About a third of attempts were orphans, and the node spent time refusing them.

At the low fee, six processes held 6,321 tx/s with zero rejects for 20 seconds, and 6,281 tx/s for 45 seconds. That remains the desk's highest clean rate. The 9,100 tx/s figure is a 12-second burst with 28,617 orphans beside it.

## What this is not

This desk run is not the 9 or 13 Oct combo. The bot and Build stay separate until that test. After it, the joint note (what each side did, why the pair is the measurement, and how the two logs are matched on n0) belongs in the questions repo, not here.

## Long holds, 6 Oct 2026 evening

The 20-second and 45-second runs are the burst record. A hold has to keep submitting after the first depth of the pipe fills. These two runs are that test. Same signed one-input one-output self-transfers. One process per public node. Fees frozen at the start of each process. Desk miners: 0.

### Ninety minutes at depth 8, 21:22–22:53 UTC

Six processes, one per public node, chains 8 deep, in-flight cap 96, target 8,000 tx/s, fees 100 and 150 sompi/gram. The process was armed for 7,200 seconds. The logs stop at 22:53 UTC, about 90 minutes in, with no summary line.

| Node | Seconds | Submit-OK | Seen accepted | Rejects | Mean submit | Seconds at 0 |
|---|---:|---:|---:|---:|---:|---:|
| vector-10 | 5,398 | 328,646 | 316,542 | 0 | 61 tx/s | 4,254 |
| proton-10 | 5,398 | 329,615 | 316,199 | 0 | 61 tx/s | 4,111 |
| electron-10 | 5,398 | 349,645 | 335,445 | 0 | 65 tx/s | 4,139 |
| muon-10 | 5,407 | 1,242,943 | 1,210,943 | 6,996 | 230 tx/s | 2,726 |
| quark-10 | 5,398 | 352,104 | 337,784 | 5,393 | 65 tx/s | 4,130 |
| neutrino-10 | 5,398 | 333,140 | 319,596 | 0 | 62 tx/s | 4,172 |
| All six | | 2,936,093 | 2,836,509 | 12,389 | | |

Each of the smaller nodes filled its pipe in the first seconds (about 1,500–1,800 lanes, depth 8) and then sat at zero for about four fifths of the run. Muon-10 had 4,000 lanes, the cap in that build, and was the only node that kept moving. The combined mean is about 540 tx/s. That is a long run with a low duty cycle. It is not a high-rate hold.

### Depth 2, in-flight 48, from 23:12 UTC

The next shape keeps the chain two deep, caps in-flight submits at 48 per process, and uses every mature coin on that node. A stampeded first second had been filling depth and then the virtual-chain feed on that socket stopped freeing lanes, so submit stayed at zero. Slowing the fill left the feed up: each of these five processes was still receiving virtual-chain events (the payload includes `acceptedTransactionIds`), about 35 events per second over the sample.

Muon-10 from the depth-8 run was left up. The other five were started again at 23:12:21 UTC. Armed window: through 09:02 UTC on 7 Oct (11:02 CEST). If one process exits before then, that process is started again. This section is the opening sample, not the 10-hour total.

Last 20 seconds ending 23:13:29 UTC. Rejects in this window: 0.

| Node | Fee, sompi/gram | Lanes | Mean submit | Seen accepted |
|---|---|---:|---:|---:|
| vector-10 | 136 and 204 | 2,130 | 669 tx/s | 661 tx/s |
| proton-10 | 100 and 150 | 2,094 | 315 tx/s | 314 tx/s |
| electron-10 | 100 and 150 | 2,275 | 331 tx/s | 329 tx/s |
| quark-10 | 100 and 150 | 2,184 | 327 tx/s | 328 tx/s |
| neutrino-10 | 100 and 150 | 2,070 | 313 tx/s | 316 tx/s |
| muon-10, still the depth-8 process | 100 and 150 | 4,497 live | 440 tx/s | 395 tx/s |

Combined submit in that window is about 2,400 tx/s, and seen-accepted is within a few percent of submit on each of the depth-2 nodes. A signed one-input one-output is about 1,624 grams, so a full 500,000-gram block at 10 blocks per second holds about 3,080 of them. This window is in that range. It is the first sample where submit and seen-accepted stayed together after the pipe filled.

The ordered stream, 4 tx/s at 100 and 150 sompi/gram on proton-10, was submitting in the same window with 0 rejects. Its 1× half averaged 2 tx/s over those 20 seconds.

At 23:13 UTC the six public mempools were about 16,000 to 18,000. `https://api-tn10.kaspa.org/info/health` was HTTP 200, database synced, accepted-tx lag 3 seconds, blue-score gap 31. The indexer did not freeze in this opening minute.

Desk clock against time.windows.com at 23:00:16 UTC: +73 ms. No resync.

This leg was stopped at 23:51 UTC. The totals are in the next section.

### Depth-2 leg, closed at 23:51 UTC

The five depth-2 processes ran from 23:12:22 to 23:51:44 UTC. Muon-10 was the older process, from 23:08:03 to the same stop. Seen-accepted stayed with submit. The desk CPU was about 5% because every lane sat at depth 2: the signers were waiting on inclusion, not on the CPU.

| Node | Seconds | Submit | Seen accepted | Rejects | Mean submit | Mean accepted |
|---|---:|---:|---:|---:|---:|---:|
| vector-10 | 2,346 | 1,359,208 | 1,356,531 | 6,029 | 579 tx/s | 578 tx/s |
| proton-10 | 2,345 | 618,661 | 615,239 | 2,704 | 264 tx/s | 262 tx/s |
| electron-10 | 2,345 | 660,831 | 657,101 | 2,976 | 282 tx/s | 280 tx/s |
| quark-10 | 2,345 | 643,439 | 639,841 | 2,976 | 274 tx/s | 273 tx/s |
| neutrino-10 | 2,345 | 617,138 | 613,685 | 2,640 | 263 tx/s | 262 tx/s |
| muon-10 | 2,603 | 1,218,343 | 1,213,183 | 18,120 | 468 tx/s | 466 tx/s |

The closing 3 minutes, 23:44 to 23:47 UTC, were the held rate: about 2,050 tx/s submit and the same seen-accepted, with 0 rejects. The rejects in the table are from earlier in the leg. Over those 3 minutes the pipes were full (depth equal to two hops on every live lane) and each signer used about 10–20% of one core.

The normal fee estimate on vector-10 had moved from 100 at 23:08 to about 186 by 23:47, with the priority bucket near 480, while these signers were still frozen at 100 and 150 (vector-10 at 136 and 204). Mempools were about 18,000. Network accepts on vector-10's virtual chain averaged about 4,270 per second over the two minutes ending 23:47, so a large part of each block was not ours. That is why the leg was stopped. It was a restart, not a stall.

### Two signers per node, fee 200 and 300, from 23:53 UTC

Same depth 2, fee cap still 600. Each node's coins are split across two signers, 12 signers plus the ordered stream. In-flight cap 64 on each signer. Fee frozen at the arm: 200 and 300 sompi/gram. Armed at 23:53:26 UTC, through the same 09:02 UTC deadline. A signer that exits is started again on its own coins. This leg is left running while it keeps including.

Last 20 seconds ending 23:57:25 UTC, with mempools already back near 21,000–24,000. Rejects in this window: 0. The normal estimate was 194 and the priority bucket was 876, so the 200 and 300 tiers sit on either side of normal.

| Node | Lanes | Mean submit | Seen accepted |
|---|---:|---:|---:|
| vector-10 | 1,874 | 370 tx/s | 371 tx/s |
| proton-10 | 1,784 | 393 tx/s | 395 tx/s |
| electron-10 | 1,852 | 426 tx/s | 428 tx/s |
| quark-10 | 1,823 | 392 tx/s | 391 tx/s |
| neutrino-10 | 1,796 | 381 tx/s | 383 tx/s |
| muon-10 | 3,512 | 453 tx/s | 482 tx/s |

Combined submit is about 2,420 tx/s and seen-accepted is about 2,450 tx/s. The first minute of this leg was about 2,530 submit and 2,500 seen-accepted, also with 0 rejects. Against the previous leg's closing 2,050 tx/s, the gain is about 400 tx/s, and it is on the five slower nodes. Vector-10 came down. The pipes are full again. The twelve signers together used about one core, and the machine load was still about 5%. More signing has no empty lane to fill. This is not the 10-hour total.

## Tasks for stp, from this desk

These are the tasks in [tn10-storm-throughput-questions](https://github.com/STP-KAS/tn10-storm-throughput-questions). Scored at 23:20 UTC on 6 Oct, while the pre-run was still inside its window (armed through 09:02 UTC on 7 Oct). The storm itself is still the later GO. This pre-run does not start it, and this section is not the 10-hour total.

1. **Dry run.** Done on the desk for the planned share. Four fixed processes held 734 tx/s, 97.9% of 750, over 3 minutes, with 0 rejects and 0 seconds at 0. The lower steps of 25, 100, 225 and 475 tx/s each hit the target on every second. The per-transaction logs for that run are on the desk. Matching those txids to n0 is the box's check, and that match is still open.
2. **Enough tKAS.** OK for both wallets. [Grok Build](https://github.com/STP-KAS/groks-wallet#grok-build) ([TN10 page](https://tn10.kaspa.stream/addresses/kaspatest:qp4jge54eztxewf8r53rtjdvxakmatsu6tjd0nn9sjhgvzxknsfvjvmwurqhd)) and [Grok Bot](https://github.com/STP-KAS/groks-wallet#grok-bot) ([TN10 page](https://tn10.kaspa.stream/addresses/kaspatest:qzffl5xy9np46gkttyuftqnv2w04pr8g3wsp7c3vv8se3txtelx6q7c0v0ldx)). At 23:00 UTC the desk read the Build wallet at 3,612,867 tKAS and 15,944 coins of at least 2 tKAS. The Bot wallet is OK on stp's word. The box balance was not read from this desk.
3. **Usage resets.** OK for the bots and for Build, per stp on 7 Oct 2026. This desk did not read the usage counters.
4. **Box dry run and free disk.** Not done. The box's checklist. Not measured from this desk.
5. **Share and plan lock.** The dry run held 734 tx/s at 4 processes, 97.9% of 750, so the default 25% share still fits that gate. The plan is not locked. Lock stays with stp before T0.
6. **Desk clock and desk miners.** Start offset +73 ms at 23:00 UTC, under 100 ms. Desk miners during this pre-run: 0. The end offset waits until 09:02 UTC. The miner switch is the storm control step and has not been run.
7. **Final OK on the start time.** Not given, and not a start-now. Earliest storm remains Fri 9 Oct 2026, 20:00 CEST, otherwise 13 Oct.

## Mainnet, the same night

Read because the desk was asked to watch mainnet congestion and the covenant hour on Kaspa.stream. Deliberate load stayed on TN10.

Mainnet was quiet. At about 00:41 CEST on 7 Oct the rendered [kaspa.stream](https://kaspa.stream/) homepage showed TPS 11.1 (1h average 10.4), BPS 10.6 (1h average 9.6), mempool 1, hashrate 300.5 PH/s, 29,412 transactions in the last hour, and 810 active addresses in the last hour. A direct `GET https://api.kaspa.org/info/fee-estimate` in the same hour returned 100 sompi/gram in every bucket. `GET https://api.kaspa.org/info/health` was synced, accepted-tx lag 3 seconds. That is not a congested hour.

The same homepage's last-hour covenant board:

| Label on the page | Last hour |
|---|---:|
| Covenants | 44 |
| Igra L2 | 1,498 |
| Kasplex L2 | 28 |
| Kasplay | 133 |
| KRC-721 | 79 |
| dotk | 38 |
| KaChat | 16 |
| Kasia | 2 |
| KRC-20 | 0 |

KNS was not a row on that board in this render. The page is the source. The HTML shell does not contain these figures. They are the rendered counters. No transaction ids from that page are copied here.
