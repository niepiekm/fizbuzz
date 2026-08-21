# Distributed Real-Time Embedded (DRE) Learning Project

## Context

This project exists to close specific, evidenced skill gaps identified while comparing the user's profile against a Software/Systems Architect role (real-time systems + event-driven architecture + distributed systems + ML/AI pipelines + containerized production deployment, in a defense-sector context). The user has 20 years of deep embedded/RTOS/C++ experience but no evidence of: event-driven architecture with message brokers (MQTT/Kafka/ZeroMQ), distributed systems design, Docker/container orchestration for production deployment, or database/high-availability data management. (ML/AI pipelines are a separate, larger gap explicitly out of scope for this project.)

The plan is to build a small but real **Distributed Real-Time Embedded (DRE) system**: an STM32 Nucleo-F411RE acts as a real-time sensor/actuator node, a Raspberry Pi 3 acts as an edge gateway, and the two communicate over USB serial. The system is built incrementally, phase by phase, each phase producing something demoable — this is a hands-on learning vehicle, not a from-scratch spec to implement in one pass.

It will live in this `fizzbuzz` repo (currently a single trivial `fizbuzz.cpp` exercise with no build system), which will be tagged for posterity and then fully repurposed.

## Key Decisions (confirmed with the user)

| Decision | Choice | Why |
|---|---|---|
| STM32 RTOS/toolchain | **Zephyr RTOS via `west`** (CMake+Ninja under the hood, Zephyr SDK toolchain, flashing via a west runner — OpenOCD/pyOCD/ST-Link compatible) | Deliberately *not* FreeRTOS — the user already has FreeRTOS production experience (Mudita), so it wouldn't close a gap. Zephyr's devicetree/Kconfig/driver-model is a genuinely new, more "architecture-flavored" skill, natively supports `nucleo_f411re`, and stays CLI-driven/scriptable. Tradeoff accepted: heavier Phase 0 setup than plain CMake+CMSIS. |
| Data source | Internal MCU temperature (ADC, periodic telemetry) + user button (discrete event) + onboard LED (inbound actuator command) | Covers all three message shapes (periodic/event/command) with zero extra hardware; clear extension point to swap in a real I2C/SPI sensor later. |
| UART framing | `START(0x7E) LEN TYPE SEQ PAYLOAD[LEN] CRC16` | Simple enough to hand-debug over a terminal; the `SEQ` field is placed now so Phase 4 dedup/idempotency reuses it rather than retrofitting. |
| Gateway language | **Python first** (`pyserial` + `paho-mqtt`), optional C++ reimplementation as a Phase 5 stretch goal | Keeps early phases focused on EDA/distributed-systems concepts, not language plumbing; the later C++ port is itself a strong "designed once, implemented twice" portfolio story matching the target job's C++/Python requirement. |
| MQTT broker | Mosquitto | Lightweight, well-supported on Raspberry Pi 3, no reason to deviate. |
| Data store | **TimescaleDB** (Postgres + extension) in Docker from Phase 3; SQLite used briefly in Phase 2 as a stepping stone only | Gives a real SQL/HA-relevant store (roles, backup, replication story) with built-in retention/hypertable support — makes "data management" concrete instead of hypothetical. |
| Raspberry Pi OS | **64-bit Raspberry Pi OS Lite** | The Pi 3's SoC (Cortex-A53) is 64-bit capable. Official arm64 Docker images for Postgres/Timescale/Grafana are healthy; arm32v7 support for these is patchy/declining. This one choice avoids most Docker-on-Pi pain. |
| Second messaging paradigm | **ZeroMQ**, for local (RPi-internal) and remote (cross-host) command/control, alongside MQTT | Directly closes the JD line item naming MQTT/Kafka/**ZeroMQ** together — MQTT alone only demonstrates the broker-based pattern. ZeroMQ's brokerless REQ/REP and PUB/SUB patterns are a deliberate contrast, and Kafka is judged too heavy/JVM-oriented to run meaningfully on a Pi 3 (noted as a future/stretch option, not pursued here). Not usable on the bare STM32 side (no native serial transport, no network stack on the Nucleo) — ZeroMQ sockets live entirely on the RPi/gateway; commands from any source still converge on one UART-command function. |
| Messaging abstraction | A `MessageTransport`-style interface in the gateway, shaped around domain operations (e.g. `send_command`, `publish`), not ZeroMQ's own socket vocabulary; ZeroMQ is the only concrete implementation for now | Requested explicitly so the ZeroMQ implementation can later be swapped for **NNG** (nanomsg-next-gen, `pynng`) without touching calling code — a deliberate ports-and-adapters exercise, not just a library choice. NNG swap itself is deferred to Phase 5. |
| Old `fizbuzz.cpp` | Tag current state as `v0-fizzbuzz`, then remove | Only 2 "Initial commit"s in history — nothing lost, and the repo's new purpose shouldn't carry dead example code. |

## Phased Plan

### Phase 0 — Repo Restructure & Environment Setup
- Tag current repo state `v0-fizzbuzz`; remove `fizbuzz.cpp`/binary/old README content.
- New directory layout (below). Rewrite `README.md` and `AGENTS.md` to describe the DRE project instead of FizzBuzz.
- Initialize the Zephyr `west` workspace (T2 topology: this repo's `firmware/` holds an app-level `west.yml` manifest pointing at the Zephyr project; `west update` fetches Zephyr/module sources into a sibling directory that is **not** committed — add it to `.gitignore`).
- Verify a stock Zephyr "blink" sample builds and flashes onto the Nucleo-F411RE via `west build` / `west flash`.
- Install 64-bit Raspberry Pi OS Lite on the Pi 3; verify Docker + `docker compose` work (`docker run hello-world`) over SSH.
- **Done when:** LED blinks under Zephyr control from a clean clone + documented bootstrap steps; `docker run hello-world` succeeds on the Pi.

### Phase 1 — Firmware Bring-up + UART Framing Protocol
- Single-application Zephyr firmware: ~1 Hz ADC temperature sampling (via devicetree `io-channels`), debounced button → event frame (GPIO interrupt callback), inbound `CMD_LED` frame toggles the onboard LED.
- Implement the frame protocol (above) on the UART tied to the ST-Link virtual COM port.
- Throwaway host-side Python script (not the real gateway yet) to sanity-check frames over serial, including injecting a bit error to confirm CRC rejection.
- **Done when:** live temp stream and button events are visible in a terminal, a command frame drives the physical LED, and a corrupted frame is visibly rejected.

### Phase 2 — Structured Concurrency + Gateway → MQTT → SQLite
- Refactor firmware into distinct Zephyr threads (sensor-sampling thread, button-callback → event queue, UART-TX thread draining a shared `k_msgq`, UART-RX/command-handler thread) — this is the hands-on lesson in producer/consumer decomposition that mirrors pub/sub thinking.
- Install Mosquitto natively on the Pi (not yet Dockerized).
- Build the Python gateway: reads framed serial data, republishes as MQTT under the topic scheme below, sets an MQTT Last Will and Testament (LWT) for presence.
- A subscriber writes telemetry/event rows into SQLite as a first durability check.
- **Done when:** `mosquitto_sub` shows live data, rows land in SQLite, and unplugging the STM32 flips a retained `status` topic to `offline` via LWT.

### Phase 2.5 — ZeroMQ Command/Control Channel Behind a Transport Abstraction
- Define a `MessageTransport` abstraction in the Python gateway, expressed in domain terms (`send_command(node_id, cmd)`, `publish(topic, payload)` or similar) rather than ZeroMQ socket-type terms — this is the seam a future NNG implementation will plug into.
- Concrete ZeroMQ implementation of that interface, using **REQ/REP** (send a command, get an ack/result — simplest pattern for command/control; note PUB/SUB or ROUTER/DEALER as later stretch options if fan-out or multiple concurrent clients are wanted):
  - **Local**: an `ipc://` (or `inproc://`) endpoint for intra-Pi use — e.g. a small local CLI/admin script that can issue commands or inspect state without going through MQTT or the network.
  - **Remote**: a `tcp://` endpoint reachable over the LAN — a small operator-console script (run from the Mac) connects and issues the same commands as the local client, over a different transport, through the same abstraction.
- Both the MQTT-originated `cmd/<actuator>` path and the new ZeroMQ path converge on the same internal "issue command to node" function, which then emits the UART `CMD_LED` frame — a concrete example of adapters converging on one core, not two parallel command paths.
- Record the abstraction seam and the deferred NNG substitution as an ADR in `docs/decisions/`.
- **Done when:** the same "toggle LED" command can be issued three ways — MQTT `cmd/led` publish, local ZeroMQ `ipc://` client, remote ZeroMQ `tcp://` client from another machine — and all three produce the same UART frame and physical LED toggle.

### Phase 3 — Dockerize the RPi Stack
- `docker-compose.yml` bringing up: `mosquitto`, `gateway` (custom Python image with `/dev/ttyACM0` passthrough), `timescaledb` with an init schema (`telemetry(node_id, ts, metric, value)`, `events(node_id, ts, type, payload)`) and a retention policy. Optional Grafana panel.
- All images arm64 (per the OS decision above).
- **Done when:** `docker compose up -d` on a clean checkout brings the full pipeline up end-to-end (STM32 → Timescale), and a `down`/`up` cycle preserves historical data via volumes.

### Phase 4 — Distributed Systems Hardening
- Reconnect/backoff for both the serial link and the MQTT connection.
- QoS 1 (at-least-once) on telemetry/event topics with dedup keyed on `(node_id, seq)` — reusing Phase 1's `SEQ` field.
- Formal presence model: clean disconnect (LWT → `offline`) vs. silent failure (heartbeat timeout → `stale`).
- A short chaos-testing runbook (kill the DB, restart the broker, unplug USB) documenting expected vs. observed recovery.
- **Done when:** every fault in the runbook self-recovers without manual intervention beyond restarting the failed component, and a forced-redelivery test produces zero duplicate rows.

### Phase 5 — Stretch Goals (optional, pick as time allows)
- Reimplement the gateway in modern C++ (`paho.mqtt.cpp` + a C++ ZeroMQ binding) as a second implementation of the same design.
- Swap the ZeroMQ `MessageTransport` implementation for **NNG** (`pynng`) — the actual point of Phase 2.5's abstraction seam — and confirm the local/remote command flow still works unchanged.
- Add a second node (real second Nucleo, or a simulated software node) to genuinely exercise the multi-node topic hierarchy.
- Swap the internal ADC temp source for a real external I2C sensor (e.g. BME280) behind the existing `TELEMETRY` message type.
- Multi-node orchestration (Swarm/k3s) — flagged as the natural next skill, out of scope here since it needs a second Pi/VM.
- COBS framing as a contrasting exercise to the length-prefixed scheme.

## Repository Structure

```
fizzbuzz/
├── docs/
│   ├── architecture.md
│   ├── decisions/          # ADR-lite notes: Zephyr choice, framing format, data store
│   └── runbooks/           # chaos-testing.md (Phase 4)
├── firmware/
│   ├── west.yml            # app-level west manifest (T2 topology)
│   ├── app/
│   │   ├── CMakeLists.txt
│   │   ├── prj.conf
│   │   ├── boards/nucleo_f411re.overlay
│   │   └── src/
│   └── .gitignore          # excludes the west-managed zephyr/ + modules/ checkout
├── proto/
│   └── framing.md          # frame spec — the contract shared by firmware and gateway
├── gateway/
│   ├── python/             # Phase 2-4 gateway (primary)
│   │   └── messaging/
│   │       ├── transport.py        # MessageTransport abstraction (Phase 2.5)
│   │       ├── zmq_transport.py    # concrete ZeroMQ implementation
│   │       └── nng_transport.py    # Phase 5 stretch: NNG implementation
│   └── cpp/                # Phase 5 stretch reimplementation
├── docker/
│   ├── docker-compose.yml
│   ├── mosquitto/mosquitto.conf
│   ├── timescaledb/init.sql
│   └── gateway.Dockerfile
└── scripts/                # flashing helpers, chaos scripts
```

## Messaging Schemes

**MQTT topics** (broker-based, telemetry/presence fan-out):
```
dre/<node_id>/status              retained, set via LWT ("online"/"offline")
dre/<node_id>/heartbeat           periodic, not retained
dre/<node_id>/telemetry/<metric>  e.g. dre/nucleo1/telemetry/temp_c
dre/<node_id>/events/<type>       e.g. dre/nucleo1/events/button
dre/<node_id>/cmd/<actuator>      e.g. dre/nucleo1/cmd/led (gateway subscribes, forwards over UART)
```
The gateway process is itself a node (`dre/gateway1/status`). A consumer that doesn't care which physical node can subscribe with `dre/+/telemetry/#`.

**ZeroMQ endpoints** (brokerless, direct command/control — Phase 2.5):
```
ipc:///tmp/dre-gateway.sock       local REQ/REP endpoint, same-host clients only
tcp://<pi-host>:5555              remote REQ/REP endpoint, LAN clients
```
Both endpoints are served by the same `MessageTransport`-backed request handler as the MQTT `cmd/<actuator>` path — three inbound paths, one internal command function.

## Raspberry Pi 3-Specific Risks

- **1GB RAM**: Mosquitto + gateway + Timescale + optional Grafana concurrently can exhaust memory — set per-container memory limits, keep Grafana optional, watch `docker stats`.
- **On-device builds are slow**: prefer pulling official images; cross-build the gateway image with `docker buildx` on the Mac and push, rather than building on the Pi.
- **SD card wear/IO**: sustained DB writes on SD are a real concern for anything framed as "HA data management" — worth noting as a footnote (better SD class or USB-SSD boot), not a Phase 0–4 blocker.

## Critical Files to Create First

- `firmware/west.yml`, `firmware/app/CMakeLists.txt`, `firmware/app/prj.conf` — anchors the Zephyr toolchain decision (Phase 0)
- `proto/framing.md` — the frame-format contract both firmware and gateway code depend on (Phase 1)
- `docker/docker-compose.yml` — anchors the Phase 3 architecture and arm64/image decisions
- `gateway/python/messaging/transport.py`, `zmq_transport.py` — anchors the Phase 2.5 abstraction seam that the Phase 5 NNG swap depends on
- `AGENTS.md` / `README.md` — full rewrite once this plan is approved, describing the new project and how to bootstrap the `west` workspace from a clean clone

## Verification (end-to-end, per phase)

- **Phase 0**: fresh clone → documented bootstrap steps → `west build && west flash` blinks the LED; `docker run hello-world` succeeds on the Pi over SSH.
- **Phase 1**: terminal shows live temp/button frames; a `CMD_LED` frame toggles the physical LED; a deliberately corrupted frame is rejected (visible CRC failure log), not silently accepted.
- **Phase 2**: `mosquitto_sub -t 'dre/#' -v` shows live telemetry/events; SQLite file has matching rows; unplugging the STM32 flips `dre/nucleo1/status` to `offline`.
- **Phase 2.5**: issuing the LED command via MQTT publish, local ZeroMQ `ipc://` client, and remote ZeroMQ `tcp://` client all produce the identical physical LED toggle and identical UART frame on the wire (verified by logging the frame before it's sent, not just observing the LED).
- **Phase 3**: `docker compose up -d` from a clean checkout brings up the full pipeline; `docker compose down && docker compose up -d` preserves historical rows in Timescale (volume-backed).
- **Phase 4**: each scripted fault in the chaos runbook recovers without manual intervention; a forced redelivery produces zero duplicate rows (checked via a `SELECT ... GROUP BY node_id, seq HAVING count(*) > 1` query).

## Deferred / Not in Scope

- ML/AI pipeline work (separate, larger gap — not addressed by this project).
- Multi-node orchestration (Swarm/k3s) — needs a second Pi/VM, flagged as the natural next project.
- External sensor hardware — start with internal ADC temp + button/LED; swapping in a real sensor is a Phase 5 stretch goal, not a blocker for Phases 0-4.
