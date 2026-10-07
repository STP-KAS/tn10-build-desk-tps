> **Experimental. We are just trying this.**
>
> Good intentions, shaky hands. STP does not know what he is doing. We test, we write down what we think we saw, and that is the whole product. A number here is not the truth. A chart is not the truth. Any other sentence that sounds sure of itself is not the truth either. Do not count any of it as a claim.
>
> [Disclaimer](DISCLAIMER.md)

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

## Long holds, 6-7 Oct 2026

The 20-second and 45-second runs above are the burst record. A hold has to keep submitting after the pipe fills. Every run below has the first and last per-second log line in UTC. Rates are submits, or seen accepts, divided by that wall time. Same signed one-input one-output. Desk miners: 0. The fee cap stayed 600. Stopped on request at 2026-10-07T05:54:55Z, before the 09:02 UTC deadline.

| Run | First log | Last log | Wall | What it did |
|---|---|---|---|---|
| Depth 8, six nodes | 2026-10-06T21:22:15.436Z | 2026-10-06T22:53:03.590Z | 1 h 30 m 48 s | 539 tx/s submit, 521 tx/s seen accepted. Most seconds were zero. |
| Shell start A | 2026-10-06T23:02:31.904Z | 2026-10-06T23:02:31.946Z | under 1 s | 17,454 submits, 0 accepts. The processes died with the shell. |
| Shell start B | 2026-10-06T23:05:56.265Z | 2026-10-06T23:05:57.269Z | 1.0 s | 14,771 submits, 987 accepts, 0 rejects. Same shell death. |
| Depth 2, feed stuck | 2026-10-06T23:08:02.667Z | 2026-10-06T23:11:51.851Z | 3 m 49 s | Five nodes, one burst second, then accept 0. |
| Muon-10, older process | 2026-10-06T23:08:03.706Z | 2026-10-06T23:51:44.209Z | 43 m 41 s | 465 tx/s submit, 463 tx/s seen accepted. |
| Depth 2, five nodes | 2026-10-06T23:12:22.136Z | 2026-10-06T23:52:27.579Z | 40 m 5 s | 1,655 tx/s submit, 1,641 tx/s seen accepted. |
| Watcher muon blip | 2026-10-06T23:51:58.316Z | 2026-10-06T23:52:27.501Z | 29.2 s | 7,936 submits, 5,811 seen accepted. Stopped with the others. |
| Two signers, fee 200 and 300 | 2026-10-06T23:53:27.326Z | 2026-10-07T05:54:53.185Z | 6 h 1 m 26 s | **2,210 tx/s submit, 2,207 tx/s seen accepted.** |

### Depth 8, 2026-10-06T21:22:15.436Z to 2026-10-06T22:53:03.590Z

Six processes, one per public node, chains 8 deep, in-flight cap 96, target 8,000 tx/s, fees 100 and 150 sompi/gram. Armed for 7,200 seconds. The logs stop at 22:53:03.590Z. No summary line.

| Node | Lines | Submit-OK | Seen accepted | Rejects | Seconds at 0 | Submit tx/s |
|---|---:|---:|---:|---:|---:|---:|
| vector-10 | 5,398 | 328,646 | 316,542 | 0 | 4,254 | 60 |
| proton-10 | 5,398 | 329,615 | 316,199 | 0 | 4,111 | 61 |
| electron-10 | 5,398 | 349,645 | 335,445 | 0 | 4,139 | 64 |
| muon-10 | 5,407 | 1,242,943 | 1,210,943 | 6,996 | 2,726 | 228 |
| quark-10 | 5,398 | 352,104 | 337,784 | 5,393 | 4,130 | 65 |
| neutrino-10 | 5,398 | 333,140 | 319,596 | 0 | 4,172 | 61 |
| All six | | 2,936,093 | 2,836,509 | 12,389 | | 539 |

The smaller nodes filled about 1,500–1,800 lanes in the first seconds and then sat at zero for about four fifths of the run. Muon-10 had 4,000 lanes, the cap in that build, and was the only node that kept moving. Seen accepted over the wall was 521 tx/s. A long run with a low duty cycle.

### Two starts that died with the shell

2026-10-06T23:02:31.904Z to 2026-10-06T23:02:31.946Z. Five lane processes and the ordered stream. One second. 17,454 submits, 0 accepts. The ordered stream's first line is 23:02:30.869Z and its last is 23:02:31.832Z.

2026-10-06T23:05:56.265Z to 2026-10-06T23:05:57.269Z. Same five nodes. 14,771 submits, 987 accepts, 0 rejects. The ordered stream is 23:05:55.533Z to 23:05:56.501Z.

A process started inside the short-lived shell died when that shell's job closed, including a detached Node child. Later parents were created with `Win32_Process.Create` and left running. Those stayed up.

### Depth 2, feed stuck, 2026-10-06T23:08:02.667Z to 2026-10-06T23:11:51.851Z

Five nodes, depth 2, in-flight 48. Each logged one burst second (1,237 to 3,502 submits) and then 227 seconds at 0. Seen accepted: 0. Vector-10 also logged 44,520 rejects. The ordered stream, 23:08:01.643Z to 23:11:51.911Z, submitted 4 and accepted 0, with 45,158 rejects.

Muon-10 did not stick. Its older process, in-flight 160, ran 2026-10-06T23:08:03.706Z to 2026-10-06T23:51:44.209Z. Wall 2,620.5 s. Submit 1,218,343 (465 tx/s). Seen accepted 1,213,183 (463 tx/s). Rejects 18,120. Eight seconds at 0.

### Depth 2, five nodes, 2026-10-06T23:12:22.136Z to 2026-10-06T23:52:27.579Z

Depth 2, in-flight 48, one process per node, fees frozen at the arm. Vector-10 was 136 and 204 sompi/gram. The other four were 100 and 150. The virtual-chain feed stayed up. The last half minute of these files is a watcher restart after the 23:51:44Z stop; it was stopped again at 23:52:27Z. The totals below are the whole file.

| Node | Submit | Seen accepted | Rejects | Submit tx/s | Accepted tx/s |
|---|---:|---:|---:|---:|---:|
| vector-10 | 1,380,182 | 1,373,941 | 7,813 | 574 | 571 |
| proton-10 | 633,355 | 626,709 | 4,088 | 263 | 261 |
| electron-10 | 677,292 | 669,978 | 4,904 | 282 | 279 |
| quark-10 | 658,497 | 651,545 | 4,768 | 274 | 271 |
| neutrino-10 | 631,680 | 625,015 | 4,376 | 263 | 260 |
| Five nodes | 3,981,006 | 3,947,188 | 25,949 | 1,655 | 1,641 |

Plus the muon process above, the overlap held about 2,100 tx/s. The closing minutes before the stop were about 2,050 tx/s with 0 rejects, and every lane was already two deep. Desk CPU was about 5%.

The ordered stream on proton-10, 2026-10-06T23:12:21.107Z to 2026-10-06T23:52:26.521Z, submitted 9,501 (3.95 tx/s) against a target of 4.

At 23:13 UTC the public mempools were about 16,000 to 18,000. `https://api-tn10.kaspa.org/info/health` was HTTP 200, synced, accepted-tx lag 3 seconds. The indexer did not freeze in that minute. By 23:47 the normal fee quote on vector-10 was about 186, while these signers were still frozen at 100 and 150.

Desk clock against time.windows.com at 2026-10-06T23:00:16Z: +73 ms.

### Two signers, 2026-10-06T23:53:27.326Z to 2026-10-07T05:54:53.185Z

Twelve lane signers, two per public node, each with its own half of that node's coins. Depth 2. In-flight 64. Fee frozen at 200 and 300 sompi/gram. Cap 600. Stopped on request at 2026-10-07T05:54:55Z. Wall 21,685.9 s, which is 6 h 1 m 26 s.

| Node | Submit | Seen accepted | Rejects | Submit tx/s | Accepted tx/s |
|---|---:|---:|---:|---:|---:|
| vector-10 | 11,896,821 | 11,893,161 | 321 | 549 | 548 |
| proton-10 | 6,935,035 | 6,933,161 | 7,638 | 320 | 320 |
| electron-10 | 7,088,908 | 7,087,002 | 7,883 | 327 | 327 |
| quark-10 | 7,113,789 | 7,111,871 | 7,685 | 328 | 328 |
| neutrino-10 | 6,742,978 | 6,698,439 | 7,763 | 311 | 309 |
| muon-10 | 8,145,325 | 8,140,351 | 4,349 | 376 | 375 |
| Twelve signers | 47,922,856 | 47,863,985 | 35,639 | 2,210 | 2,207 |

Seen accepted stayed with submit for the whole six hours. Rejects are 0.07% of submits. Neutrino-10's second signer had 1,436 seconds at 0. The other signers had 30 to 108 such seconds. The first minute was higher, about 2,530 submit and 2,500 seen accepted, and the rate then settled at the table. A signed one-input one-output is about 1,624 grams, so about 3,080 of them fill a 500,000-gram block at 10 blocks per second. This hold is under that ceiling.

The twelve signers used about one core. The machine stayed near 5% CPU, with the pipes full. More signers had no empty lane.

The ordered stream, 2026-10-06T23:53:26.324Z to 2026-10-07T05:54:52.833Z, submitted 85,916 (3.96 tx/s). Its own seen-accept counter was 8,938, and it logged 28,620 rejects. The lane result above does not depend on that counter.

At 2026-10-07T05:55:18Z the desk clock was 407 ms ahead of the Date header from `https://www.microsoft.com`. That header is whole seconds, so the offset is inside one second.

### After the stop

The senders stopped at 2026-10-07T05:54:55Z. Covenant activity on TN10 is gaining traction again. The covenant apps did not change. The plain transfers stopped taking the block.

Three slices from `https://api-tn10.kaspa.org`, 40 selected-chain blocks each, coinbase left out. A covenant transaction is one with a covenant on an output. The two times are the ends of that walk. A few seconds is not an hour rate. This desk's render of the TN10 homepage left the last-hour Covenants card empty, so the figures are these blocks.

| When | Earlier end | Later end | User txs | Covenant txs | Covenant outputs | User txs per block |
|---|---|---|---:|---:|---:|---:|
| During the hold | 2026-10-07T02:59:49Z | 2026-10-07T02:59:58Z | 12,218 | 3 | 4 | about 305 |
| 15 min after the stop | 2026-10-07T06:09:38Z | 2026-10-07T06:09:43Z | 61 | 3 | 8 | about 1.5 |
| 22 min after the stop | 2026-10-07T06:17:11Z | 2026-10-07T06:17:19Z | 279 | 7 | 26 | about 7 |

During the hold the slice held 12,218 user transactions, 3 of them covenant, about 305 per block. Fifteen minutes after the stop the same kind of slice held 61 user transactions, 3 of them covenant. Twenty-two minutes after, it held 279 user transactions, 7 of them covenant. The plain flood had left the blocks. Covenant transactions were being included in the room that opened.

Why. The six-hour hold included 2,207 plain one-input one-output transfers per second, about 1,624 grams each, at 200 and 300 sompi per gram. About 3,080 of those fill a 500,000-gram block at 10 blocks per second. The hold was most of that mass. Miners order by fee per gram. A covenant step often weighs more, so it needs a higher total fee to stand level with a 1,624-gram transfer at 200 or 300. A step that stays near the quiet floor waits. The next covenant step is spent from the previous output, so one waiting step stops the flow, and the covenant lane looks idle. After the stop, a block has room, the waiting step is included, and the app posts the next step. That second step is the traction. It shows up because the flood stopped, not because the covenant program sped up.

The mempool count stayed high through this. At 2026-10-07T06:11:31Z the highest public mempool was 52,530. At 2026-10-07T06:27:46Z it was 49,141, still falling by only a few per second. A vector-10 fee read at 2026-10-07T06:28Z was about 162 and 131 sompi per gram in the normal buckets, and about 243 in the priority bucket. During the hold the normal quote was about 186–194 and the priority quote was about 876. The 06:09Z slice was about 1.5 user transactions per block while that count was still near 50,900. The count is the wrong instrument for "is the next block full."

This is TN10. The mainnet hour the same night is in the section below. The builder view is point 9 of the [early recommendations](https://github.com/STP-KAS/tn10-storm-throughput-questions#early-recommendations-for-builders).

## Saved for the 9 Oct test

The shape that held is in [plan/DESK-SHAPE-9-OCT.md](https://github.com/STP-KAS/tn10-storm-throughput-questions/blob/main/plan/DESK-SHAPE-9-OCT.md). The test-day steps are in [plan/TESTDAY.md](https://github.com/STP-KAS/tn10-storm-throughput-questions/blob/main/plan/TESTDAY.md). Starting that prompt on test day is the run. Paced steps stay at 4 processes, depth 2, in-flight 48. The long hold and the uncapped max step use two signers on each public node, depth 2, in-flight 64, and a fee frozen at 200 and 300. The fee cap stays 600. That note does not start the storm.

## Tasks for stp, from this desk

These are the tasks in [tn10-storm-throughput-questions](https://github.com/STP-KAS/tn10-storm-throughput-questions#tasks-for-stp-before-the-storm). Scored at the stop, 2026-10-07T05:55:18Z. The storm itself is still the later GO. This pre-run does not start it.

1. **Dry run.** Done on the desk for the planned share. Four fixed processes held 734 tx/s, 97.9% of 750, over 3 minutes, with 0 rejects and 0 seconds at 0. The lower steps of 25, 100, 225 and 475 tx/s each hit the target on every second. The per-transaction logs for that run are on the desk. Matching those txids to n0 is the box's check, and that match is still open.
2. **Enough tKAS.** OK for both wallets. [Grok Build](https://github.com/STP-KAS/groks-wallet#grok-build) ([TN10 page](https://tn10.kaspa.stream/addresses/kaspatest:qp4jge54eztxewf8r53rtjdvxakmatsu6tjd0nn9sjhgvzxknsfvjvmwurqhd)) and [Grok Bot](https://github.com/STP-KAS/groks-wallet#grok-bot) ([TN10 page](https://tn10.kaspa.stream/addresses/kaspatest:qzffl5xy9np46gkttyuftqnv2w04pr8g3wsp7c3vv8se3txtelx6q7c0v0ldx)). At 23:00 UTC the desk read the Build wallet at 3,612,867 tKAS and 15,944 coins of at least 2 tKAS. The Bot wallet is OK on stp's word. The box balance was not read from this desk.
3. **Usage resets.** OK for the bots and for Build, per stp on 7 Oct 2026. This desk did not read the usage counters.
4. **Box dry run and free disk.** Not done. The box's checklist. Not measured from this desk.
5. **Share and plan lock.** The dry run held 734 tx/s at 4 processes, 97.9% of 750, so the default 25% share still fits that gate. The plan is not locked. Lock stays with stp before T0.
6. **Desk clock and desk miners.** Start offset +73 ms at 2026-10-06T23:00:16Z. At the stop, 2026-10-07T05:55:18Z, the desk was 407 ms ahead of a whole-second Date header, so the offset is inside one second. Desk miners during this pre-run: 0. The miner switch is the storm control step and has not been run.
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
