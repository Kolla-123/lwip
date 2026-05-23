# lwIP Repository Study Plan

This plan is for getting comfortable with this lwIP checkout as both a reader
and a future contributor. The repo is the lwIP lightweight TCP/IP stack, a C
network stack aimed at embedded systems. This tree reports version `2.2.2.dev`
in `src/Filelists.cmake`.

## Goals

- Understand how lwIP is organized and embedded into an application.
- Learn the core data structures: `pbuf`, `netif`, protocol control blocks,
  memory pools, timers, and option macros.
- Follow packets through the stack from a network interface to IP, UDP/TCP, and
  the public APIs.
- Learn how the build, examples, tests, and contrib ports fit together.
- Finish with enough confidence to debug or modify one small feature or test.

## Map Of The Repo

- `README` - project overview, features, contribution pointers, and docs links.
- `BUILDING` - how CMake and Makefile integrations are meant to be consumed.
- `src/` - the actual lwIP TCP/IP stack.
- `src/core/` - protocol implementations, raw API, buffers, memory, timers, and
  stack initialization.
- `src/core/ipv4/` and `src/core/ipv6/` - address families, fragmentation,
  ICMP, DHCP, ARP/ND, multicast support, and related helpers.
- `src/api/` - sequential, netconn, and Berkeley-like socket APIs. These sit
  above the raw callback API and require OS/thread support.
- `src/include/` - public headers and configuration defaults.
- `src/netif/` - generic network interfaces such as Ethernet, SLIP, 6LoWPAN,
  bridge, and PPP.
- `src/apps/` - bundled protocol/application modules such as HTTP, SNMP, SNTP,
  mDNS, MQTT, TFTP, and lwiperf.
- `contrib/` - ports, examples, add-ons, and helper applications.
- `test/` - unit tests, socket stress tests, and fuzz targets.
- `doc/` - focused notes and generated Doxygen inputs.

## Phase 1: Orientation

Read:

- `README`
- `FILES`
- `src/FILES`
- `BUILDING`
- `src/Filelists.cmake`
- `CMakeLists.txt`

Questions to answer:

- What does lwIP expect the host application or OS port to provide?
- Why are `src/Filelists.cmake`, `contrib/Filelists.cmake`, and the port
  filelists separate?
- Which code is core stack code, and which code is example or port code?
- What is the minimum library surface for a basic lwIP integration?

Hands-on:

- Sketch a one-page module diagram with `src/core`, `src/api`, `src/netif`,
  `src/apps`, `contrib`, and `test`.
- Find where `lwipcore_SRCS`, `lwipapi_SRCS`, `lwipnetif_SRCS`, and
  `lwipallapps_SRCS` are defined.

## Phase 2: Configuration And Porting Model

Read:

- `src/include/lwip/opt.h`
- `test/unit/lwipopts.h`
- `contrib/examples/example_app/lwipopts.h`
- `contrib/examples/example_app/lwipcfg.h.example`
- `contrib/ports/unix/README`
- `contrib/ports/win32/readme.txt`

Focus areas:

- `lwipopts.h` overrides defaults from `opt.h`.
- `NO_SYS` controls whether lwIP runs without an OS-aware threading layer.
- `SYS_LIGHTWEIGHT_PROT`, core locking, semaphores, mailboxes, and the `sys`
  layer define the concurrency contract.
- The examples provide real port configuration and are a good bridge between
  library code and an application.

Questions to answer:

- What changes when `NO_SYS == 1` versus `NO_SYS == 0`?
- Why do socket and netconn APIs depend on OS/thread support?
- Which settings would you start with for a small bare-metal board?
- Which settings would you start with for a hosted test application?

Hands-on:

- Compare `test/unit/lwipopts.h` with the example app options and list five
  options that materially change stack behavior.
- Trace where `lwipopts.h` is included from `opt.h`.

## Phase 3: Core Data Structures

Read:

- `src/include/lwip/pbuf.h` and `src/core/pbuf.c`
- `src/include/lwip/netif.h` and `src/core/netif.c`
- `src/include/lwip/mem.h` and `src/core/mem.c`
- `src/include/lwip/memp.h` and `src/core/memp.c`
- `src/include/lwip/timeouts.h` and `src/core/timeouts.c`
- `src/core/init.c`

Focus areas:

- `pbuf` chains represent packet buffers and avoid unnecessary copies.
- `netif` is the abstraction between lwIP and a concrete network interface.
- `mem` is heap-style allocation, while `memp` is fixed-size pool allocation.
- Timers drive protocol maintenance such as TCP retransmission and DHCP state.
- Initialization wires configuration, stats, memory, protocols, and timers.

Questions to answer:

- When should code allocate from `PBUF_POOL`, `PBUF_RAM`, `PBUF_ROM`, or
  `PBUF_REF`?
- What owns a `pbuf` at each stage of packet receive and transmit?
- What callbacks must a network interface provide?
- Which internal timers exist and which protocols depend on them?

Hands-on:

- Read `doc/NO_SYS_SampleCode.c` and annotate the receive path, transmit path,
  interface setup, and timer loop.
- Follow a `pbuf` from allocation to `netif.input()` to eventual free.

## Phase 4: Packet Receive And Transmit Flow

Read:

- `src/netif/ethernet.c`
- `src/core/ip.c`
- `src/core/ipv4/ip4.c`
- `src/core/ipv4/etharp.c`
- `src/core/ipv6/ip6.c`
- `src/core/ipv6/nd6.c`
- `src/core/udp.c`
- `src/core/raw.c`

Trace these flows:

- Ethernet frame receive through `ethernet_input()`.
- IPv4 receive through IP dispatch to ICMP, UDP, TCP, or raw handlers.
- IPv4 transmit through routing, ARP, and `netif->linkoutput`.
- IPv6 receive through neighbor discovery and IPv6 dispatch.
- UDP receive from packet demultiplexing to the raw callback.

Questions to answer:

- How does lwIP decide which `netif` should send a packet?
- Where are checksums verified or generated?
- How are broadcast, multicast, and unicast paths different?
- How does raw API registration affect packet handling?

Hands-on:

- Pick `udp.c` and identify the public API functions, internal input path, and
  memory ownership rules.
- Add comments in your notes for each point where a packet can be dropped.

## Phase 5: TCP Internals

Read:

- `src/include/lwip/tcp.h`
- `src/include/lwip/priv/tcp_priv.h`
- `src/core/tcp.c`
- `src/core/tcp_in.c`
- `src/core/tcp_out.c`
- `src/include/lwip/tcpbase.h`

Focus areas:

- TCP protocol control blocks and state transitions.
- Active open, passive open, accept, receive, send, close, abort.
- Segment queues, retransmission, out-of-sequence data, SACK, and windowing.
- The split between generic TCP control, input handling, and output handling.

Questions to answer:

- What is stored in a TCP PCB?
- Which functions move a connection between TCP states?
- How does lwIP queue unsent, unacked, and out-of-sequence segments?
- What timers maintain TCP behavior?

Hands-on:

- Use `test/unit/tcp/test_tcp_state.c` as a guided tour of TCP state behavior.
- Trace one simple connection lifecycle: listen, accept, receive, write, close.

## Phase 6: Public APIs

Read:

- `src/include/lwip/api.h`
- `src/api/api_lib.c`
- `src/api/api_msg.c`
- `src/api/tcpip.c`
- `src/include/lwip/sockets.h`
- `src/api/sockets.c`
- `src/api/netbuf.c`
- `src/api/netdb.c`

Focus areas:

- Raw API is callback-oriented and closest to core internals.
- Netconn API provides a sequential API over the TCP/IP thread.
- Socket API layers Berkeley-style calls over netconn behavior.
- `tcpip.c` coordinates messages into the TCP/IP thread.

Questions to answer:

- What path does `lwip_send()` take before data reaches TCP or UDP?
- Where are socket descriptors mapped to netconns?
- Which APIs are safe from application threads?
- How are errors converted between lwIP `err_t` values and socket `errno`?

Hands-on:

- Read `test/unit/api/test_sockets.c` and map tests back to `src/api/sockets.c`.
- Pick one socket operation and trace it into the netconn or core layer.

## Phase 7: Applications And Optional Modules

Read selectively based on interest:

- HTTP: `src/apps/http/httpd.c`, `src/include/lwip/apps/httpd_opts.h`
- MQTT: `src/apps/mqtt/mqtt.c`, `src/include/lwip/apps/mqtt.h`
- mDNS: `src/apps/mdns/mdns.c`, `doc/mdns.txt`
- SNTP: `src/apps/sntp/sntp.c`
- SNMP: `src/apps/snmp/`, `contrib/apps/LwipMibCompiler/`
- TLS layer: `src/core/altcp.c`, `src/core/altcp_tcp.c`,
  `src/apps/altcp_tls/`

Questions to answer:

- Which apps use raw callbacks, netconn, or sockets?
- How do app-specific option headers control module size?
- Which apps are suitable examples for embedded event-driven code?

Hands-on:

- Pick one app and identify its initialization function, option header, public
  header, and tests or examples.

## Phase 8: Tests, Fuzzing, And Debugging

Read:

- `test/unit/lwip_unittests.c`
- `test/unit/Makefile`
- `test/unit/*/test_*.c`
- `test/fuzz/README`
- `test/fuzz/Makefile`
- `.github/workflows/`

Focus areas:

- Unit tests use the Check framework and collect suites in
  `test/unit/lwip_unittests.c`.
- Tests use their own `lwipopts.h` to enable features needed by test cases.
- Fuzz targets exercise packet parsing and protocol robustness.
- CI is useful for seeing supported build/test configurations.

Questions to answer:

- Which unit suites cover core structures, protocols, APIs, and apps?
- How does the test configuration differ from a production embedded build?
- Where would you add a regression test for a UDP, TCP, DHCP, or socket bug?

Hands-on:

- Run the available build or test flow for your platform.
- Pick one small unit test and modify only your notes to explain what behavior
  it protects.

## Phase 9: Contribution-Ready Exercise

Choose one focused mini-project:

- Add a small unit test around a behavior you already traced.
- Improve a local comment or doc note where the control flow was hard to follow.
- Build one example app and document its include paths, options, and linked
  lwIP libraries.
- Trace and document one packet path in more detail: ARP, UDP, TCP close, DHCP,
  DNS, or mDNS.

Definition of done:

- You can explain the change or trace in five minutes.
- You know which files own the behavior.
- You know which tests or examples exercise it.
- You can name the configuration options that affect it.

## Suggested Four-Week Schedule

Week 1:

- Complete Phases 1 and 2.
- Produce a repo map and configuration notes.

Week 2:

- Complete Phases 3 and 4.
- Trace one receive path and one transmit path.

Week 3:

- Complete Phases 5 and 6.
- Trace one TCP lifecycle and one socket call.

Week 4:

- Complete Phases 7, 8, and 9.
- Finish one small exercise and write a short summary of what you learned.

## Daily Study Routine

1. Read one small group of files for 30 to 45 minutes.
2. Write down the main structs, public functions, callbacks, and ownership
   rules.
3. Trace one call path with `rg` before reading too broadly.
4. Check a unit test or example that exercises the same area.
5. End by writing one question and one answer in your notes.

Useful commands:

```sh
rg "struct pbuf|pbuf_alloc|pbuf_free" src test
rg "struct netif|netif_add|ethernet_input" src contrib test
rg "tcpip_callback|tcpip_api_call|netconn|lwip_send" src test
rg "Suite\\*|START_TEST|END_TEST" test/unit
```

## Glossary To Build While Reading

- `pbuf` - packet buffer, often chained, with explicit ownership rules.
- `netif` - network interface abstraction used by lwIP core code.
- `PCB` - protocol control block, used by TCP, UDP, raw, and related APIs.
- `raw API` - callback API used directly by core and many embedded apps.
- `netconn API` - sequential API that communicates with the TCP/IP thread.
- `socket API` - Berkeley-like API layered above netconn.
- `sys_arch` - OS/port layer for threading, semaphores, mailboxes, and timing.
- `lwipopts.h` - user-supplied configuration header.
- `memp` - fixed-size memory pools.
- `mem` - heap-like allocator.
