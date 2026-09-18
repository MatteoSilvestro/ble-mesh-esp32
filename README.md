# BLE Mesh on ESP32 — instrumenting and measuring Packet Delivery Ratio

Project for the **Computer Communication Networks** course — Tongji University.

A three-node **Bluetooth Mesh** network built on ESP-IDF and instrumented to measure,
at runtime, the **Packet Delivery Ratio (PDR)**, the number of lost packets and the
inter-packet arrival delta over a multi-hop link.

The firmware derives from the official Espressif example
[`esp_ble_mesh/sensor_models`](https://github.com/espressif/esp-idf/tree/master/examples/bluetooth/esp_ble_mesh/sensor_models)
(Apache-2.0). All changes are concentrated in the two `main.c` files; everything else
is unmodified.

---

## Experimental topology

| Node | Role | Unicast address | Firmware to flash |
|---|---|---|---|
| 1 | Sensor Client / Provisioner | `0x0001` | `sensor_client` |
| 2 | Sensor Server — traffic generator | `0x0005` | `sensor_server`, with `main.c` |
| 3 | Relay — forwarding only | `0x0006` | `sensor_server`, with `main_relay.c` |

```
[Server 0x0005] --(TTL=3)--> [Relay 0x0006] --(TTL=2)--> [Client 0x0001]
```

Hardware: 3 ESP32 boards (`CONFIG_IDF_TARGET=esp32`). Server and Relay must be powered
from separate supplies: a shared USB hub causes voltage drops and Bluetooth controller
crashes when both radios come up at the same time.

---

## Changes with respect to the Espressif example

### `sensor_server/main/main.c` — sequence number injection

* New FreeRTOS task `mesh_stress_test_task`, created at the end of `app_main()`, which
  publishes a `SENSOR_STATUS` message at a fixed interval regardless of the model's
  Publish Address state.
* The `sensor_data_0` / `sensor_data_1` buffers grow from 1 to 5 bytes (`.length` from
  `0` to `4`, the field being zero-based) to hold **1 temperature byte + a 4-byte
  little-endian sequence number**.
* The application sequence number `stress_seq_num` is incremented on every
  transmission: it is the quantity the Client uses to derive packet loss.
* Transmission period: **100 ms** (10 Hz) in stress-test mode.

### `sensor_client/main/main.c` — runtime PDR computation

Inside the `ESP_BLE_MESH_MODEL_OP_SENSOR_GET` callback, for property `0x0056`
(Present Indoor Ambient Temperature) carrying a 4-byte payload:

* the received sequence number is reconstructed from the 4 little-endian bytes;
* it is compared against `expected_seq`: a forward jump counts as loss and emits a
  `COLLISION DETECTED: Lost N packets` warning;
* a Server reboot is recognised (`rx_seq == 0`) and the counters are reset;
* `PDR: xx.xx% | Received: N | Lost: M` is printed continuously.

### Server firmware variants

The three files in `sensor_server/main/` are alternatives to be copied over `main.c`
before building (`main/CMakeLists.txt` compiles only `main.c` and `board.c`):

| File | Transmission period | Transmit task | Used for |
|---|---|---|---|
| `main.c` | 100 ms | active | Node 2, stress test |
| `main_server.c` | 500 ms | active | Node 2, relaxed traffic |
| `main_relay.c` | 500 ms | **commented out** | Node 3, forwarding-only node |

The Relay is therefore a "mute" Server: same image, with the `xTaskCreate` call
disabled so that the node only regenerates and re-transmits other nodes' PDUs.

---

## Build and flash

Requires ESP-IDF (tested on the `esp32` target).

```bash
. $IDF_PATH/export.sh
cd sensor_server          # or sensor_client
idf.py set-target esp32
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

For the Relay node, before building:

```bash
cd sensor_server/main
cp main.c main_stress.bak
cp main_relay.c main.c
```

The `sdkconfig` files are tracked on purpose: they document the configuration actually
used during the experiments, in particular `CONFIG_BLE_MESH_RELAY=y` and
`CONFIG_BLE_MESH_NODE=y` on the server.

---

## Repository layout

```
sensor_client/       Provisioner firmware + PDR computation
sensor_server/       Server firmware, with the three variants under main/
docs/report.md       full report (theory, architecture, results)
docs/img/            screenshots of the UART logs referenced by the report
```

---

## Known limitation: multi-node provisioning

The Espressif `sensor_client` example does not handle sequential provisioning of
multi-node networks. When provisioning completes, the Provisioner runs

```c
server_address = primary_addr;
```

that is, it keeps a **single** server address in a static variable, overwritten every
time a new node joins. The observed consequences:

* with simultaneous provisioning, the Client latches onto the first node that answers
  and hands it the AppKey; the Relay stays outside the cryptographic network, receives
  the PDUs at physical level but discards them at the Network Layer because the NetMIC
  cannot be validated;
* with sequential provisioning, the Relay does join, but it overwrites
  `server_address`, and the Client starts polling a node that was programmed not to
  transmit.

The fix requires replacing the static variable with a dynamic table of primary
addresses on the Provisioner, iterating NetKey and AppKey distribution over every node
before enabling sensor data exchange. It is not implemented in this repository.

---

## License and attribution

Source code distributed under the **Apache License 2.0**, like the Espressif example it
derives from; the original SPDX headers are preserved in every file. See
[`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).
