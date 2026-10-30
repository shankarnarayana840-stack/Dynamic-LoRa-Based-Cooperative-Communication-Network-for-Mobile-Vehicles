# Dynamic LoRa Mesh Network for Military Vehicle Communication

A four-node LoRa communication prototype built with **ESP32-C6** microcontrollers and **Reyax RYLR998** LoRa modules. The project explores master-coordinated node discovery and cooperative searching when some nodes cannot communicate directly with the master.

> **Project status:** Experimental four-node prototype. The results below describe the tests performed with the current hardware setup; they should not be interpreted as guaranteed field performance or as validation of a production-ready military communication system.

## 1. Project Overview

Reliable communication between mobile vehicles can be difficult in remote areas where cellular networks, Wi-Fi, or wired infrastructure are unavailable. LoRa provides a low-power radio link for exchanging small messages over relatively long distances.

This project implements an application-level communication protocol on ESP32-C6 boards connected to RYLR998 LoRa modules. One node acts as the **master** and three nodes act as **slaves**. The master broadcasts discovery messages and records which slaves respond. When one or more slaves do not respond directly, the master can ask already responding nodes to assist with a cooperative search.

The purpose of the prototype is to investigate whether reachable nodes can help the master discover nodes beyond its direct communication range.

## 2. Main Features

- Four-node prototype: one master and three slave nodes.
- UART control of RYLR998 LoRa modules using AT commands.
- Master-initiated broadcast discovery and acknowledgement collection.
- Tracking of directly responding and unresponsive node IDs.
- Cooperative search requests sent to known/responding nodes.
- Randomized backoff at slave nodes to reduce simultaneous response transmissions.
- Acknowledgement timeout and bounded retry handling for search requests.
- Experimental comparison using LoRa Spreading Factors 8 and 9.

## 3. Hardware Requirements

| Component | Quantity | Purpose |
|---|---:|---|
| ESP32-C6 development board | 4 | Runs the master/slave firmware and protocol logic. |
| Reyax RYLR998 LoRa module | 4 | Provides wireless transmission and reception. |
| Compatible LoRa antennas | 4 | Connect to the radio modules for RF communication. |
| USB cables | As required | Programming, power, and serial debugging. |
| Jumper wires | As required | UART and power connections between each ESP32-C6 and RYLR998. |
| Computer | 1 | Firmware development, flashing, and monitoring. |

### ESP32-C6 to RYLR998 wiring

| ESP32-C6 | RYLR998 | Connection |
|---|---|---|
| 3V3 | VCC | Power |
| GND | GND | Common ground |
| GPIO 20 (TX) | RXD | ESP32 transmit to module receive |
| GPIO 21 (RX) | TXD | ESP32 receive from module transmit |

The UART data lines are crossed: TX connects to RXD, and RX connects to TXD. Use the 3.3 V supply connection as specified for the board/module and verify the exact pinout and power requirements of the hardware revision before powering it.



## 4. Software and Development Tools

- **Visual Studio Code** – development environment.
- **ESP-IDF** – ESP32-C6 firmware build, flash, and debugging framework.
- **C** – implementation language for the master/slave application protocol.
- **ESP-IDF UART driver** – serial communication between the ESP32-C6 and RYLR998.
- **RYLR998 AT commands** – module configuration and radio message transmission.
- **Serial monitor** – inspection of module responses and protocol logs.

## 5. Node Addressing and Radio Configuration

| Node | Role | Address |
|---|---|---:|
| Master | Discovery coordinator and search-request initiator | 4 |
| Slave 1 | Responding/search-assistance node | 1 |
| Slave 2 | Responding/search-assistance node | 2 |
| Slave 3 | Responding/search-assistance node | 3 |
| Broadcast | Broadcast destination | 0 |

The project configuration previously used Network ID `18`, UART baud rate `115200`, and RYLR998 parameter setting `8,7,1,12` for the SF8 experiment. SF9 was used in the comparative experiment. The radio frequency and all settings must be configured consistently across communicating nodes. Verify that the selected frequency and transmit-power settings comply with the radio regulations applicable to the deployment location.

### UART pin configuration in the master firmware

| Setting | Value |
|---|---|
| UART peripheral | `UART_NUM_1` |
| ESP32-C6 TX | GPIO 20 |
| ESP32-C6 RX | GPIO 21 |
| UART baud rate | `115200` |

## 6. Application Packet Types

The application protocol uses the following message names:

| Packet | Purpose |
|---|---|
| `DISCOVERY` | Master broadcast used to discover reachable slave nodes. |
| `ACK_DISCOVERY` | Slave acknowledgement for a discovery message. |
| `SEARCH` | Request for a responding node to search for specified undiscovered nodes. |
| `ACK_SEARCH` | Acknowledgement of a search request. |
| `PROBE` | Probe message used as part of the delegated search procedure. |
| `ACK_PROBE` | Response to a probe message. |
| `RESULT:F:N` | Search-result message format used by the application protocol; document the exact meanings of `F` and `N` alongside the firmware definition. |
| `FWD` | Forwarding-related message. |
| `ACK_FWD` | Acknowledgement associated with a forwarding message. |
| `FWD_FAIL` | Indicates a forwarding failure. |

The RYLR998 wraps application payloads in its UART AT-command interface. A transmit command has the general form:

```text
AT+SEND=<destination>,<payload_length>,<payload>
```

A received message is reported in the general form:

```text
+RCV=<source>,<payload_length>,<payload>,<RSSI>,<SNR>
```

For example, the serial monitor may show:

```text
AT+SEND=2,19,FWDS=3;SEARCH_REQ=1
RX: +OK
RX: +RCV=2,10,ACK_SEARCH,-16,14
```

`+OK` indicates that the module accepted the send command; it is not, by itself, confirmation that the destination received the packet. `ACK_SEARCH` is the application-level acknowledgement.

> 

## 7. Master Discovery and Cooperative Search

The master performs the following high-level sequence:

1. Initialize the ESP32-C6 UART interface and communicate with the RYLR998 module.
2. Broadcast `DISCOVERY` to address `0`.
3. Collect `ACK_DISCOVERY` responses and update the list of directly discovered slaves.
4. Identify slave IDs that did not respond during the discovery window.
5. Build a search request containing the unresolved node IDs.
6. Send the search request to a known/responding node.
7. Wait for `ACK_SEARCH`. If no acknowledgement arrives within the configured timeout, retry the request up to the configured limit.
8. After acknowledgement, wait for the search result. If the selected node does not acknowledge after the allowed attempts, proceed to the next known node.
9. Update the master’s record based on the returned search result.

The master firmware includes functions for UART initialization, AT-command transmission, packet parsing, discovery processing, unresolved-node tracking, search-request construction, and cooperative search handling.

### Timing settings in the master source file shared

The uploaded master source currently defines the following values:

| Parameter | Value in shared source | Meaning |
|---|---:|---|
| `MAX_TRIES` | 2 | Total initial discovery attempts in the current `app_main()` loop. |
| `DISCOVERY_WAIT_MS` | 1500 ms | Discovery response window in `discovery_process()`. |
| `SEARCH_RETRIES` | 2 | Additional search retries; up to 3 sends in total to one known node. |
| `SEARCH_ACK_WAIT_MS` | 1000 ms | Time to wait for a search acknowledgement. |
| `SEARCH_RESULT_WAIT_MS` | 15000 ms | Time to wait for a search result after the search acknowledgement. |
| `NEXT_KNOWN_NODE_WAIT_MS` | 1000 ms | Delay before trying the next known node. |

Confirm these values against the final firmware you intend to publish. They differ from some earlier draft/report values discussed during development.

## 8. Randomized Backoff

Slave nodes apply a randomized delay before transmitting their responses. The project has used a backoff interval between 100 ms and 1000 ms. Randomizing response times reduces the chance that multiple slaves transmit at exactly the same time after receiving a common discovery message. It reduces collision likelihood but does not guarantee collision-free communication.

## 9. Experimental Setup and Results

A scooter was used as a mobile platform to vary the separation between nodes. The direct communication test and two cooperative-search arrangements were examined using SF8 and SF9.

### Case 1 – Direct communication

The maximum observed distance at which the master received acknowledgements from all three slave nodes was:

| Spreading factor | Maximum observed direct distance |
|---|---:|
| SF8 | Approximately 450 m |
| SF9 | Approximately 470 m |

These are observations from the specific test setup, not guaranteed LoRa ranges.

### Case 2 – One responding node assists the search

One responding slave was positioned approximately 450 m from the master. The other two nodes were approximately 250 m and 280 m from the responding node, under non-line-of-sight conditions. The master initiated the cooperative search through the responding node.

### Case 3 – Two responding nodes assist the search

Two slave nodes, approximately 450–460 m from the master, responded directly. The remaining node was approximately 250–300 m from one of the responding nodes. The cooperative search mechanism was used, and the master discovered all three slave nodes in this test scenario.

### Bitrate and transmission-time analysis

The experiments compared SF8 and SF9. Bitrate and packet transmission-time values were calculated using the radio parameters and packet lengths considered in the project. Since application payloads have different lengths, minimum, average, and maximum transmission-time values were considered rather than presenting only one value for every packet type. State clearly in the results whether a value is calculated from a formula or measured from the device.

### LOS and NLOS observations

A clear line-of-sight path generally supports better reception than an obstructed path. Obstacles can weaken the received signal and lead to intermittent reception or missed acknowledgements. A distance approaching 1 km was discussed as a possible LOS target, but it is not listed here as a measured result because the reported direct tests achieved approximately 450 m (SF8) and 470 m (SF9).

## 10. Limitations

- The prototype uses four nodes; behaviour with a larger network has not been evaluated.
- The measured distances apply to the particular antennas, locations, radio settings, and test conditions used.
- The tests used a scooter as a mobile platform; extensive multi-vehicle mobility testing has not been performed.
- Packet Delivery Ratio (PDR) and end-to-end delay have not been measured.
- Security features such as encryption and authentication have not been implemented or evaluated.
- The prototype should not be considered a field-ready military communication system.

## 11. Future Work

- Expand the network and evaluate the discovery mechanism with more nodes.
- Improve node selection and route management using suitable link-quality or position information.
- Measure PDR, end-to-end delay, and network recovery time.
- Test with multiple moving nodes under different LOS and NLOS conditions.
- Add authentication and encryption for protected communication.

## 12. Repository Structure



```text
Dynamic-LoRa-Mesh-Network/
├── README.md
├── Firmware/
│   ├── Master_Node/
│   │   └── main.c
│   └── Slave_Node/
│       └── main.c
├── Hardware/
│   ├── esp32c6_rylr998_wiring.png
├── Protocol/
│   ├── packet_structure.md
│   └── discovery_flowchart.png
├── Experimental_Results/
│   ├── observations.xlsx
│   ├── sf8_results.png
│   ├── sf9_results.png
│   └── test_photos/
└── Documentation/
    └── project_report.pdf
```



## 13. Build and Run

1. Install Visual Studio Code and the ESP-IDF extension/toolchain for ESP32-C6.
2. Open the master or slave ESP-IDF project directory.
3. Select the correct ESP32-C6 target and serial port.
4. Check the UART pins, baud rate, node address, Network ID, and LoRa radio settings in the firmware.
5. Connect the RYLR998 module to the ESP32-C6 using the wiring table above.
6. Build and flash the firmware to each board.
7. Open the serial monitor and verify module initialization and packet exchange.
8. Start the master and slave nodes according to the test scenario.

The exact build commands and project paths should be added after confirming the repository's final ESP-IDF project layout.

## 14. Acknowledgement

This project was developed as an academic embedded-systems and wireless-communication prototype. The implementation and measurements are intended for learning and experimental evaluation.
