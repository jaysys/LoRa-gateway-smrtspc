LoRa는 단말(Node)에서 데이터를 **무선으로 전송**하므로, 이를 수신하고 처리하는 별도의 장치 또는 서버(게이트웨이 및 서버) 가 필요합니다.

LoRa 네트워크의 기본 구성과 수신 서버 구성 방법을 정리합니다. 그리고 소스 리포지터리는 LoRa Gateway 역할을 담당하는 기능에 대한 소스를 담고 있습니다.

---

# 1. LoRa 네트워크

```
[LoRa Node] → → → [LoRa Gateway] → → → [네트워크 서버] → [애플리케이션 서버 / DB / 클라우드]
```

| 구성 요소                     | 설명                                           |
| ----------------------------- | ---------------------------------------------- |
| **LoRa Node**                 | 센서 또는 장치 (ESP32 + LoRa 모듈 등)          |
| **LoRa Gateway**              | LoRa 무선신호 → IP 패킷으로 변환 (중계기 역할) |
| **LoRa Network Server (LNS)** | 장치 인증, 라우팅, 중복 제거, 메시지 필터링 등 |
| **Application Server**        | 사용자 데이터 저장, 시각화, 알림 등            |

---

LoRa(로라)는 전 세계적으로 다양한 실제 IoT 시스템에 사용되고 있으며, 특히 **전력, 농업, 물류, 스마트시티, 환경 감시** 등 **광범위한 분야**에 적용되어 왔습니다. 여기에는 대기업/정부 프로젝트도 포함됩니다. 아래에 실제 사례를 정리해드릴게요.

---

# 2. LoRa Node

| 구성 요소                  | 설명                                                            |
| -------------------------- | --------------------------------------------------------------- |
| **MCU (마이크로컨트롤러)** | 데이터를 수집하고 LoRa로 송신 (예: **ESP32**, STM32, ATmega 등) |
| **LoRa 모듈**              | 무선 송신/수신 장치 (예: **SX1276/78**, E32-900M 등)            |
| **센서**                   | 온도, 습도, 가스, 움직임, 토양 수분 등                          |
| **전원**                   | 배터리 (리튬이온, AA, 태양광 등) 또는 USB                       |
| **안테나**                 | LoRa 통신 거리에 중요 (433MHz/868/915MHz에 맞는 안테나 사용)    |

---

## 2.1 Recommended Hardware Examples

### ESP32 + LoRa 일체형 보드

| 모델                    | 특징                                       |
| ----------------------- | ------------------------------------------ |
| **LilyGO LoRa32**       | ESP32 + SX1276 + 0.96인치 OLED             |
| **Heltec WiFi LoRa 32** | ESP32 + OLED + LoRa (868/915MHz)           |
| **TTGO T-Beam**         | ESP32 + LoRa + GPS + 배터리 충전 회로 내장 |

➡️ 위 보드는 **센서만 연결하면 바로 LoRa 노드로 개발 가능**합니다.

---

## 2.2 LoRa Node Operation Structure

```
[센서 읽기] → [데이터 처리 (ESP32)] → [LoRa 송신] → [게이트웨이]
```

- 일정 주기마다 센서 데이터 읽기
- LoRa 모듈에 데이터 전송
- 간단한 프로토콜 (예: `"ID:23,TEMP:25.4,HUM:48.2"`)

---

## 2.3 LoRa Node Development (ESP32)

### 2.3.1 Arduino IDE 설정

1. **보드 매니저에 ESP32 추가**
   URL: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
2. **라이브러리 설치**:

   - `LoRa by Sandeep Mistry`
   - `Adafruit Sensor`, `DHT`, `OneWire`, `DallasTemperature` 등 센서 라이브러리

---

### 2.3.2 기본 예제 코드 (LoRa 송신)

```cpp
#include <SPI.h>
#include <LoRa.h>

void setup() {
  Serial.begin(9600);
  while (!Serial);

  // LoRa 초기화 (915MHz 또는 868/433MHz)
  if (!LoRa.begin(915E6)) {
    Serial.println("LoRa init failed. Check your connections.");
    while (1);
  }
  Serial.println("LoRa Sender Ready");
}

void loop() {
  // 예시 데이터
  float temp = 24.6;
  float hum = 45.2;

  String msg = "NODE01:TEMP=" + String(temp) + ",HUM=" + String(hum);

  LoRa.beginPacket();
  LoRa.print(msg);
  LoRa.endPacket();

  Serial.println("Sent: " + msg);

  delay(5000); // 5초 주기
}
```

---

## 2.4 Sensor Connection Examples

| 센서          | 연결 방식     | 아두이노 라이브러리 |
| ------------- | ------------- | ------------------- |
| DHT11/DHT22   | Digital (1핀) | `DHT.h`             |
| Soil Moisture | Analog        | 직접 `analogRead`   |
| MQ-2 가스센서 | Analog        | `analogRead`        |
| DS18B20       | OneWire       | `DallasTemperature` |

---

## 2.5 Considerations for Managing Multiple LoRa Nodes

| 항목            | 설명                                                      |
| --------------- | --------------------------------------------------------- |
| Node ID         | 각 노드마다 고유 ID 부여 (`NODE01`, `NODE02` 등)          |
| Duty Cycle 제한 | 충돌 방지 위해 전송 주기 분산 (예: 10\~60초 간격)         |
| 전원 최적화     | `deepSleep` 사용 시 수년간 배터리로 운영 가능             |
| 프로토콜 정의   | CSV, JSON 또는 단순 문자열 (게이트웨이에서 파싱 용이하게) |

---

## 2.6 Receiver Test Example (Without Gateway)

LoRa 수신기 역할을 하는 **또 다른 ESP32 + LoRa 보드**를 준비하여 확인:

```cpp
// LoRa 수신기 코드
#include <SPI.h>
#include <LoRa.h>

void setup() {
  Serial.begin(9600);
  if (!LoRa.begin(915E6)) {
    Serial.println("LoRa init failed!");
    while (1);
  }
  Serial.println("LoRa Receiver");
}

void loop() {
  int packetSize = LoRa.parsePacket();
  if (packetSize) {
    String msg = "";
    while (LoRa.available()) {
      msg += (char)LoRa.read();
    }
    Serial.println("Received: " + msg);
  }
}
```

---

## 2.7 Summary

| 항목      | 내용                                             |
| --------- | ------------------------------------------------ |
| MCU       | ESP32 추천 (무선 + 연산 + 개발 편의성)           |
| LoRa 모듈 | SX1276 계열, E32-900M, 또는 보드 일체형 사용     |
| 센서      | 디지털/아날로그 모두 가능                        |
| 개발환경  | Arduino IDE + LoRa 라이브러리                    |
| 전송 포맷 | NodeID + 센서값, 예: `NODE01:TEMP=24.5`          |
| 고급 기능 | deepSleep, OTA, 메시 라우팅(중계), AES 암호화 등 |

---

# 3. LoRa Gateway (Receiver) Configuration

## 3.1 상용 LoRa 게이트웨이 사용

- 예시:

  - **RAK Wireless** (RAK7249, RAK7258 등)
  - **Dragino LG01 / LG308**
  - **TTN Gateway (The Things Network Gateway)**

- 특징:

  - 여러 노드의 LoRa 데이터를 동시에 수신 가능
  - 이더넷/Wi-Fi/4G 통해 인터넷 전송

## 3.2 직접 구성: **라즈베리파이 + LoRa Concentrator**

- **필요 부품**:

  - Raspberry Pi 3/4/5
  - SX1302 or SX1301 LoRa Concentrator (RAK2245/RAK2287 등)

- **소프트웨어**:

  - [LoRa Packet Forwarder](https://github.com/Lora-net/packet_forwarder)

- **기능**:

  - LoRa 패킷 수신 → MQTT or UDP로 서버로 전달

---

# 4. LoRa Network Server (LNS) 구성

## 4.1 공개/클라우드 네트워크 사용

- **The Things Network (TTN)**: 무료로 글로벌 LoRaWAN 서버 제공

  - [https://www.thethingsnetwork.org/](https://www.thethingsnetwork.org/)
  - 게이트웨이를 등록하고, Application → Device 설정하면 바로 사용 가능
  - 데이터는 MQTT/Webhook/HTTP API 등으로 연동 가능

## 4.2 자체 서버 구축 (사설망)

- **ChirpStack** (오픈소스 LoRa 서버 플랫폼)

  - Docker 또는 리눅스 서버에 설치 가능
  - UI 포함 / MQTT로 애플리케이션 연동
  - [https://www.chirpstack.io/](https://www.chirpstack.io/)

- 서버 구조:

  - Gateway Bridge
  - Network Server
  - Application Server
  - PostgreSQL + Redis

---

## 4.3 수신 데이터 처리 예시

| 방식        | 기술 스택                                       | 설명               |
| ----------- | ----------------------------------------------- | ------------------ |
| MQTT        | ChirpStack / TTN → MQTT Broker → Python/Node.js | 실시간 처리        |
| Webhook     | TTN/ChirpStack → HTTP 서버 (Flask/Express 등)   | REST API 기반      |
| 데이터 저장 | MySQL, InfluxDB, Firebase 등                    | 추후 시각화 / 분석 |
| 시각화      | Grafana, Node-RED, Vue.js 등                    | 대시보드 구성 가능 |

---

### 4.3.1 예시 구성: 직접 LoRa 수신 서버 운영

| 구성 요소   | 선택 예시                                  |
| ----------- | ------------------------------------------ |
| 게이트웨이  | Dragino LG01-P 또는 Raspberry Pi + RAK2245 |
| 서버        | Ubuntu + ChirpStack or TTN Cloud           |
| 데이터 처리 | Python + MQTT or Webhook                   |
| 저장소      | SQLite / InfluxDB / Firebase               |
| 시각화      | Grafana or Node-RED or 웹앱                |

---

- **LoRa 통신은 직접 서버를 구축하거나, TTN 같은 무료 클라우드 플랫폼을 이용하여 수신할 수 있습니다.**
- **게이트웨이는 LoRa 전파를 인터넷으로 전환하는 다리 역할**을 하며,
- \*\*LNS(Network Server)\*\*는 인증, 라우팅, 중복 제거 등을 담당합니다.
- 최종 데이터는 웹 서버, MQTT 브로커, 데이터베이스, 대시보드 등에 연결되어 활용됩니다.

--

현재 폴더에 구성된 소스 리포지터리는 LoRa Gateway를 구성에 필요한 소스를 담고 있습니다.

## Lora network packet forwarder project

```
/ _____)             _              | |
( (____  _____ ____ _| |_ _____  ____| |__
 \____ \| ___ |    (_   _) ___ |/ ___)  _ \
 _____) ) ____| | | || |_| ____( (___| | | |
(______/|_____)_|_|_| \__)_____)\____)_| |_|
(C)2013 Semtech-Cycleo
```

1. Core program: lora_pkt_fwd

---

The packet forwarder is a program running on the host of a Lora gateway that
forwards RF packets receive by the concentrator to a server through a IP/UDP
link, and emits RF packets that are sent by the server. It can also emit a
network-wide GPS-synchronous beacon signal used for coordinating all nodes of
the network.

    ((( Y )))
        |
        |
    +- -|- - - - - - - - - - - - -+        xxxxxxxxxxxx          +--------+
    |+--+-----------+     +------+|       xx x  x     xxx        |        |
    ||              |     |      ||      xx  Internet  xx        |        |
    || Concentrator |<----+ Host |<------xx     or    xx-------->|        |
    ||              | SPI |      ||      xx  Intranet  xx        | Server |
    |+--------------+     +------+|       xxxx   x   xxxx        |        |
    |   ^                    ^    |           xxxxxxxx           |        |
    |   | PPS  +-----+  NMEA |    |                              |        |
    |   +------| GPS |-------+    |                              +--------+
    |          +-----+            |
    |                             |
    |            Gateway          |
    +- - - - - - - - - - - - - - -+

Uplink: radio packets received by the gateway, with metadata added by the
gateway, forwarded to the server. Might also include gateway status.

Downlink: packets generated by the server, with additional metadata, to be
transmitted by the gateway on the radio channel. Might also include
configuration data for the gateway.

2. Helper programs

---

Those programs are included in the project to provide examples on how to
communicate with the packet forwarder, and to help the system builder use it
without having to implement a full Lora network server.

### 3.1. util_sink

The packet sink is a simple helper program listening on a single port for UDP
datagrams, and displaying a message each time one is received. The content of
the datagram itself is ignored.

### 3.2. util_ack

The packet acknowledger is a simple helper program listening on a single UDP
port and responding to PUSH_DATA datagrams with PUSH_ACK, and to PULL_DATA
datagrams with PULL_ACK.

### 3.3. util_tx_test

The network packet sender is a simple helper program used to send packets
through the gateway-to-server downlink route.

4. Helper scripts

---

### 4.1. lora_gateway/reset_lgw.sh

This script, provided with the HAL (lora_gateway), must be launched on IoT Start
Kit platform to reset concentrator chip through GPIO, before starting any
application using the concentrator, like the packet forwarder.

### 4.2. packet_forwarder/lora_pkt_fwd/update_gwid.sh

This script allows automatic update of Gateway_ID with unique MAC address, in
packet forwarder JSON configuration file.
Please refer to the script header for more details.

5. Changelog

---

### v4.0.1 - 2017-03-16

- Class-B: Added xtal error correction to beacon frequency
- Class-B: Added support for all regions to beacon frame format (various
  datarates imply different frame sizes), as defined by LoRaWAN v1.1.

### v4.0.0 - 2017-01-10

- Added Class-B support, as defined in LoRaWAN v1.1
- Downlink only support "tmst" or "tmms" timestamp. "time" is not supported
  anymore ("time" field is kept in Uplink as an informative field).
- Reworked thread_gps to handle GPS UBX messages for native GPS time.
- Updated Gateway <-> NetworkServer protocol to describe the new "tmms" field.
- Updated global_conf.PCB286\*.json to remove indexes of the TX gain LUT above
  20dBm. Use PCB336 (aka GW v1.5) to comply with ETSI TX mask between 20dBm and
  27dBm.

### v3.1.0 - 2016-09-07

- Updated "Listen-Before-Talk" JSON configuration to match with LBT rework.
- Added TX Notch Filter JSON configuration.
- Updated Parson library to latest version
- Fixed Class-B beacon CRC-16 calculation
- Removed JiT time_on_air local function, and use lgw_time_on_air() function

### v3.0.0 - 2016-05-19

- Merged all different flavours of packet forwarder into one unique lora_pkt_fwd
  Note: Various flavours can still be achieved using the corresponding
  global_conf.json.XXX file provided in lora_pkt_fwd/cfg.
- Added downlink "just-in-time" scheduling to optimize downlink capacity.
- Updated Gateway <-> NetworkServer protocol to describe the new format of
  "tx_ack" message.
- Added "Listen-Before-Talk" JSON configuration.
- Splitted reset_pkt_fwd.sh script in 2 different scripts:
  - reset_lgw.sh, provided with the HAL (lora_gateway)
  - update_gwid.sh, provided with lora_pkt_fwd

WARNING: Gateway <-> Network Server protocol version has changed. Please refer
to PROTOCOL.txt file.

### v2.2.1 - 2016-04-12

- util_tx_test: added FSK support and specific payload for easier PER testing.
- base64: fixed padding check.
- Updated all makefiles to handle the creation of obj directory when necessary.
- [gps/beacon]\_pkt_fwd: fixed crash on exit when GPS not enabled.
- [*]\_pkt_fwd: added a cfg/ directory containing different flavours or the
  global_conf.json file for different boards: Ref Design PCB_E336 (GW1.5-27dBm),
  Ref Design PCB_E286 (GW1.0), Ref Design with US902 frequency plan.

### v2.2.0 - 2015-10-08

- Removed FTDI support in makefiles to align with HAL v3.2.0.
- Force IPv4 mode usage on UDP socket, instead of auto. The auto mode was
  causing an issue to properly resolve LoRa server hostname given in JSON
  configuration file (MariaDB issue: https://mariadb.atlassian.net/browse/MDEV-4356,
  https://mariadb.atlassian.net/browse/MDEV-4379).

### v2.1.0 - 2015-06-29

- Added helper script for concentrator reset through GPIO, needed on IoT
  Starter Kit (reset_pkt_fwd.sh).
- The same reset_pkt_fwd.sh script also allows to automatically update the
  Gateway_ID field in JSON configuration file, with board MAC address.
- Updated JSON configuration file with proper default value for IoT Starter
  Kit: server address set to local server, GPS device path set to proper value
  (/dev/ttyAMA0).

### v2.0.0 - 2015-04-30

- Changed: Several configuration parameters are now set dynamically from the
  JSON configuration file: RSSI offset, concentrator clock source, radio type,
  TX gain table, network type. The HAL does not need to be recompiled any more to
  update those parameters. An example for IoT Starter Kit platform is provided in
  global_conf.json for basic, gps and beacon packet_forwarder.
- Removed: Band frequency JSON configuration file has been removed. An example
  for EU 868MHz is provided in global_conf.json for basic, gps and beacon packet
  forwarder.
- Changed: Updated makefiles to allow cross compilation from environment
  variable (ARCH, CROSS_COMPILE).

** WARNING: **
** Update your JSON configuration file with new dynamic parameters. **

### v1.4.1 - 2015-01-23

- Bugfix: fixed LP-116, fdev parameter parsed incorrectly, making FSK TX fail.
- Bugfix: fixed a platform-dependant minor rounding issue.
- Beta: updated beacon format, partially aligned with latest class B proposal.

### v1.4.0 - 2014-10-16

- Feature: Adding TX FSK support.
- Feature: optional auto-quit if a certain number of PULL_ACK is missed.
- Feature: upstream and downstream ping time is displayed on console.
- Bugfix: some beacons were missed at high beaconing frequency.
- Bugfix: critical snprintf error caused a crash for long payloads.
- FSK bitrate now appears in the upstream JSON.

### v1.3.0 - 2014-03-28

- Feature: adding preliminary beacon support for class B development.
- Solved warnings with 64b integer printf when compiling on x86_64.
- Updated build system for easier deployment on various hardware.
- Changed threads organization in the forwarder programs.
- Removed duplicate protocol documentation.

### v1.2.0 - 2014-02-03

- Feature: added a GPS-enabled packet forwarder, used to timestamp received
  packet with a globally-synchronous microsecond-accurate timestamp.
- Feature: GPS packet forwarder sends status report on the uplink, protocol
  specification updated accordingly (report include gateway geolocation).
- Feature: packets can be sent without CRC at radio layer.
- Bugfix: no more crash with base64 padded input.
- Bugfix: no more rounding errors on the 'freq' value sent to server.
- A minimum preamble of 6 Lora symbol is enforced for optimum sensitivity.
- Padded Base64 is sent on uplink, downlink accepts padded and unpadded Base64.
- Updated the Parson JSON library to a version that supports comments.
- Added .md (Markdown) extension to readme files for better Github viewing.

### v1.1.0 - 2013-12-09

- Feature: added packet filtering parameters to the JSON configuration files.
- Bugfix: will not send a datagram if all the packets returned by the receive()
  function have been filtered out.
- Bugfix: removed leading zeros for the timestamp in the upstream JSON because
  it is not compliant with JSON standard (might be interpreted as octal number).
- Removed TXT extension for README files for better Github integration.
- Cleaned-up documentation, moving change log to top README.
- Modified Makefiles to ease cross-compilation.

### v1.0.0 - 2013-11-22

- Initial release of the packet forwarder, protocol specifications and helper
  programs.

6. Legal notice

---

The information presented in this project documentation does not form part of
any quotation or contract, is believed to be accurate and reliable and may be
changed without notice. No liability will be accepted by the publisher for any
consequence of its use. Publication thereof does not convey nor imply any
license under patent or other industrial or intellectual property rights.
Semtech assumes no responsibility or liability whatsoever for any failure or
unexpected operation resulting from misuse, neglect improper installation,
repair or improper handling or unusual physical or electrical stress
including, but not limited to, exposure to parameters beyond the specified
maximum ratings or operation outside the specified range.

SEMTECH PRODUCTS ARE NOT DESIGNED, INTENDED, AUTHORIZED OR WARRANTED TO BE
SUITABLE FOR USE IN LIFE-SUPPORT APPLICATIONS, DEVICES OR SYSTEMS OR OTHER
CRITICAL APPLICATIONS. INCLUSION OF SEMTECH PRODUCTS IN SUCH APPLICATIONS IS
UNDERSTOOD TO BE UNDERTAKEN SOLELY AT THE CUSTOMERS OWN RISK. Should a
customer purchase or use Semtech products for any such unauthorized
application, the customer shall indemnify and hold Semtech and its officers,
employees, subsidiaries, affiliates, and distributors harmless against all
claims, costs damages and attorney fees which could arise.

_EOF_
