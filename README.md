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
