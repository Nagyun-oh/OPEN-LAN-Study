네. 이 요구라면 **대표 장비는 AGV/AMR 계열로 잡는 게 가장 좋습니다.** 이유는 다른 항만 장비보다 **“정상 통신 행위”를 객관적으로 정의할 수 있는 공개 표준이 훨씬 구체적**하기 때문입니다.

특히 **VDA 5050 + 3GPP TS 22.104 + 3GPP RRC/NAS 규격 + NIST SP 800-82 Rev.3**를 조합하면, 지금 말씀하신

> 장비 선정 → 정상행위 정의 → 객관적 근거 → Baseline → RAN 관찰 → Anomaly 판정

구조를 거의 그대로 만들 수 있습니다.

## 1. 먼저 결론: AGV가 가장 좋은 이유

AGV를 고르면 Baseline을 최소 **3개 레이어**로 나눌 수 있습니다.

| Layer                | 무엇을 정상으로 정의?                   | 주요 근거                            |
| -------------------- | ------------------------------ | -------------------------------- |
| **Application**      | AGV가 누구와 어떤 메시지를 주고받는지         | **VDA 5050**                     |
| **5G Service/RAN**   | 요구 latency, 주기성, mobility, QoS | **3GPP TS 22.104**               |
| **Security/Anomaly** | baseline에서 벗어난 통신을 어떻게 탐지할지    | **NIST SP 800-82r3 / IEC 62443** |

이게 아주 중요합니다. **3GPP만으로는 “AGV는 MQTT broker X와 통신한다”를 정의할 수 없고**, 반대로 VDA 5050만으로는 **RRC/PDU Session/5G authentication**을 정의할 수 없습니다. 따라서 여러 표준을 레이어별로 결합해야 합니다.

---

# 2. 가장 중요한 레퍼런스: VDA 5050

### VDA 5050 — Interface for the Communication between Mobile Robots and a Fleet Control

2026년 현재 최신 VDA 페이지는 **VDA 5050 Version 3.0.0**을 “Mobile Robot Communication Interface”로 설명하며, 중앙 Fleet Control과 Mobile Robot 간 작업·상태 데이터 교환 인터페이스를 정의합니다. ([VDA][1])

이 문서가 여러분 프로젝트에서 가장 강력한 **Behavioral Baseline 근거**가 됩니다.

VDA 5050은 통신 프로토콜로 **MQTT + JSON**을 정의하며, MQTT 3.1.1 이상을 요구합니다. ([GitHub][2])

더 좋은 점은 **누가 어떤 메시지를 보내야 하는지까지 정의되어 있다는 것**입니다.

| Topic            | 송신자           | 수신자           | 의미       |
| ---------------- | ------------- | ------------- | -------- |
| `order`          | Fleet Control | AGV           | 작업 명령    |
| `instantActions` | Fleet Control | AGV           | 즉시 실행 명령 |
| `state`          | AGV           | Fleet Control | AGV 상태   |
| `visualization`  | AGV           | Visualization | 위치/경로    |
| `connection`     | AGV/Broker    | Fleet Control | 연결 상태    |
| `factsheet`      | AGV           | Fleet Control | 장비 정보    |
| `zoneSet`        | Fleet Control | AGV           | Zone 정보  |
| `responses`      | Fleet Control | AGV           | 요청 응답    |

이 방향성과 의미가 VDA 5050에 명시되어 있습니다. ([GitHub][2])

이것만으로도 상당한 Audit Rule을 만들 수 있습니다.

예를 들어:

> AGV가 `order` topic에 Publish한다 → 비정상
> Fleet Controller가 `state`를 Publish한다 → 비정상
> 등록되지 않은 AGV serialNumber가 topic에 등장한다 → 비정상 가능성

같은 룰을 **우리가 임의로 만드는 게 아니라 표준에서 파생**할 수 있습니다.

---

# 3. MQTT Topic 자체도 Baseline으로 쓸 수 있음

VDA 5050은 로컬 MQTT Broker의 topic 구조도 제안합니다.

```text
interfaceName/majorVersion/manufacturer/serialNumber/topic
```

예를 들면:

```text
vda5050/v3/KIT/0001/order
```

입니다. ([GitHub][2])

그래서 Audit 프로그램에서 아주 유용한 Feature가 생깁니다.

```text
manufacturer
serialNumber
topic
direction
message type
```

예:

```text
Normal

vda5050/v3/KIT/AGV001/state
AGV001 → FleetControl
```

반면:

```text
vda5050/v3/UNKNOWN/9999/order
UNKNOWN → Broker
```

같은 상황은 정책에 따라 이상행위 후보가 됩니다.

---

# 4. 메시지의 QoS까지 규격에 있음

VDA 5050은 MQTT QoS도 정의합니다.

`order`, `instantActions`, `state`, `factsheet`, `zoneSet`, `responses`, `visualization`은 **QoS 0**, `connection`은 **QoS 1**을 사용하도록 정의합니다. ([GitHub][2])

즉 이것도 Rule로 만들 수 있습니다.

예를 들어:

```text
Expected:
connection → MQTT QoS 1

Observed:
connection → QoS 0
```

이면

```text
Protocol Policy Violation
```

으로 잡을 수 있습니다.

다만 이건 곧바로 “공격”이라고 판정하면 안 되고 **configuration anomaly / policy violation** 정도가 더 정확합니다.

---

# 5. AGV의 정상 동작 자체도 표준화할 수 있음

VDA 5050은 AGV가 어떻게 움직이는지도 상당히 상세하게 정의합니다.

Fleet Control은 이동할 Node와 Edge를 포함한 `order`를 보내고, AGV는 해당 순서에 따라 경로를 수행합니다. 특히 `released=false`인 node/edge는 이동해서는 안 되고, Fleet Control이 허용한 경로를 따라야 합니다. ([GitHub][2])

여기서 아주 재미있는 보안 탐지 Rule을 만들 수 있습니다.

예를 들어:

```text
FleetControl Order:
Node A → Node B → Node C

AGV state:
A → B → X
```

이면

> **Communication-semantic anomaly**

로 볼 수 있습니다.

단순 네트워크 IDS보다 한 단계 발전한 것입니다.

---

# 6. VDA 5050 자체에 이미 비정상 상태가 정의되어 있음

이게 여러분 프로젝트에서 상당히 강합니다.

VDA 5050은 잘못된 order를 받았을 때 AGV가 어떤 error를 보고해야 하는지까지 정의합니다.

예를 들어:

* malformed order → `VALIDATION_FAILURE`
* unsupported parameter → `UNSUPPORTED_PARAMETER`
* 지원하지 않는 action → `INVALID_ORDER_ACTION`
* 오래된 orderUpdateId → `OUTDATED_ORDER_UPDATE`
* 동시에 다른 order → `OTHER_ORDER_ACTIVE`
* 시작 node 범위 이상 → `START_NODE_OUT_OF_RANGE`
* 목적지 경로 없음 → `NO_ROUTE_TO_TARGET`
* 알 수 없는 map → `UNKNOWN_MAP_ID`

등입니다. ([GitHub][2])

이걸 이용하면 여러분 Audit Engine에 **Signature-based rule**도 만들 수 있습니다.

```text
IF VDA_ERROR == OUTDATED_ORDER_UPDATE
THEN alert = "Unexpected order replay/update"
```

또는 반복적으로

```text
VALIDATION_FAILURE > N회 / 1분
```

이면

```text
possible malformed-message attack
```

처럼 볼 수 있습니다.

여기서 중요한 건 **“한 번 발생 = 공격”으로 단정하면 안 된다는 것**입니다.

표준상 정상적인 장애나 구현 오류로도 발생할 수 있기 때문에,

> 정상 → Warning → Suspicious → Anomaly

형태로 Risk Score를 주는 게 좋습니다.

---

# 7. 3GPP TS 22.104 — AGV의 5G 통신 특성 근거

다음으로 중요한 게 **3GPP TS 22.104 — Service requirements for cyber-physical control applications in vertical domains**입니다.

3GPP가 AGV를 명시적으로 **Mobile Robot의 하위 개념**으로 포함합니다. ([ETSI][3])

또한 AGV의 미래 구조를

> AGV ↔ reliable wireless communication ↔ centralized fleet control / edge cloud

형태로 설명합니다. ([ETSI][3])

이건 여러분 프로젝트 아키텍처와 거의 일치합니다.

```text
AGV
 │
5G UE
 │
gNB / RAN
 │
UPF
 │
MEC
 │
Fleet Control
```

---

# 8. 3GPP가 AGV Traffic 종류까지 분류함

이 부분이 특히 좋습니다.

3GPP TS 22.104는 Mobile Robot 통신을 다음과 같이 분류합니다.

* 정밀 협업 로봇 제어
* machine control
* cooperative driving
* video-operated remote control
* 일반 mobile robot operation / traffic management
* mobile robot에서 guidance control로 전송되는 실시간 영상

그리고 일부 periodic communication의 transfer interval도 **1 ms, 1–10 ms, 10–50 ms** 등의 범위로 제시합니다. ([ETSI][3])

또 AGV 요구사항을 명시적으로 설명합니다.

* 안전 관련 direct-device control은 time-critical
* 위치 및 availability 정보는 fleet/swarm management에 필요
* camera-based navigation은 높은 데이터율 요구
* sensor-based navigation은 상대적으로 낮은 데이터율 요구

([ETSI][3])

따라서 여러분 Baseline에서 traffic을 단순히 하나로 보지 말고:

```text
AGV Control Traffic
AGV Status Traffic
AGV Navigation Traffic
AGV Video Traffic
```

으로 분류하는 게 좋습니다.

---

# 9. 3GPP에서 Mobile Robot 네트워크 KPI도 얻을 수 있음

TS 22.104에는 Mobile Robot에 대한 service performance requirement도 있습니다.

예를 들어 Mobile Robot use case에서 높은 availability, 이동속도 최대 약 50 km/h, 최대 수천 UE 규모, 최대 1 km² 서비스 영역 같은 조건을 정의하고 있으며, 다른 영상 기반 mobile robot use case에는 UL 데이터율 요구도 별도로 나옵니다. ([ETSI][3])

이걸 이용해서 Audit feature를 다음처럼 만들 수 있습니다.

```text
Latency
Jitter
Packet loss
UL bitrate
DL bitrate
Traffic interval
Handover frequency
Radio link failures
```

다만 여기서 중요한 원칙:

> **QoS 초과 = 보안 공격**

으로 직접 연결하면 안 됩니다.

예:

```text
latency > baseline
```

은

```text
performance anomaly
```

이고,

여기에

```text
traffic surge
+ session flood
+ authentication failures
```

같은 다른 징후가 같이 있을 때 security anomaly score를 높이는 구조가 좋습니다.

---

# 10. RAN 레벨 Baseline은 3GPP TS 38.331

AGV의 application traffic과 별개로 **RAN에서 UE가 어떻게 행동해야 하는지**도 표준으로 정의할 수 있습니다.

3GPP TS 38.331은 NR의 RRC 상태를

```text
RRC_IDLE
RRC_INACTIVE
RRC_CONNECTED
```

로 정의합니다. 또한 연결 establish/release/resume 등의 상태 전이가 존재합니다. ([ETSI][4])

예를 들어 Audit이 다음을 수집할 수 있습니다.

```text
UE ID
RRC state
RRC setup count
RRC reconnect count
RRC resume count
handover
Radio Link Failure
measurement reports
```

그리고 비정상 후보를

```text
AGV normally:
RRC_CONNECTED → handover → RRC_CONNECTED

abnormal:
RRC setup/release 반복
RRC reconnect 급증
Radio Link Failure 급증
```

처럼 정의할 수 있습니다.

단, **“RRC reconnection이 1분에 10번이면 공격” 같은 숫자는 3GPP에 없습니다.**

이 수치는 실제 정상 테스트 데이터를 수집해서 만들어야 합니다.

---

# 11. PDU Session 역시 Baseline으로 잡을 수 있음

5G에서 AGV가 데이터를 전송하려면 PDU Session을 사용합니다.

3GPP TS 24.501은 PDU Session에 대해 IPv4, IPv6, IPv4v6, Ethernet, Unstructured type을 정의하며, establishment / authentication / modification / release 등의 절차를 규정합니다. ([ETSI][5])

TS 23.502에서는 UE가 PDU Session을 만들 때

* PDU Session ID
* DNN
* S-NSSAI
* PDU Session Type

등을 사용하도록 정의합니다. ([ETSI][6])

따라서 Private 5G를 직접 구축한다면 정상 AGV profile을 예를 들어:

```text
AGV-001

Allowed S-NSSAI = SmartPort-AGV
Allowed DNN = agv.local
Allowed PDU Session = 1
Allowed Data Network = FleetControl Network
```

로 provision해둘 수 있습니다.

그러면:

```text
AGV-001
→ unknown DNN 요청

AGV-001
→ 비허용 Slice 요청

AGV-001
→ 반복 PDU session creation
```

같은 것을 Audit 대상으로 만들 수 있습니다.

여기서 **Allowed DNN/Slice 자체는 여러분 시스템 정책**이고, PDU Session의 동작 구조는 3GPP 근거입니다.

---

# 12. 인증되지 않은 UE 탐지는 3GPP TS 33.501

5G Security의 가장 중요한 기준은 **3GPP TS 33.501**입니다.

이 규격은 5G System의 security architecture와 authentication procedure를 정의하는 표준입니다. 

따라서 다음 이벤트는 Audit 대상이 됩니다.

```text
Unknown subscriber
Authentication failure
Registration reject
Security mode failure
Repeated authentication requests
```

다만 여기서도:

> authentication failure 1회 = attack

이 아닙니다.

통신장애, USIM 문제 등도 있으므로 **반복성/빈도/시간적 패턴**을 추가해야 합니다.

---

# 13. “Baseline 기반 이상탐지” 자체의 근거는 NIST가 매우 강함

여러분 프로젝트에서 가장 중요한 문서 중 하나가 **NIST SP 800-82 Rev.3 – Guide to Operational Technology Security**입니다.

NIST는 OT Network Monitoring 시 다음을 권고합니다.

* 연결된 장비 inventory
* **typical network traffic baseline**
* **data flow baseline**
* **device-to-device communication baseline**
* anomaly/suspicious traffic 탐지

([NIST 출판물][7])

그리고 특히 중요한 내용이 있습니다.

NIST는 OT traffic은 일반 IT traffic보다 **deterministic, repeatable, predictable**한 경우가 많기 때문에 behavior anomaly detection에 적합하다고 설명합니다. 또한 정상 OT 상태를 이해하는 것이 anomaly detection의 선행조건이며, 처음에는 passive monitoring/learning 방식으로 정상/비정상 통신을 구분하는 것이 필요할 수 있다고 설명합니다. ([NIST 출판물][7])

이건 여러분 프로젝트의 연구 논리를 거의 그대로 지원합니다.

즉:

> **“왜 Baseline을 만드는가?”**

에 대한 답을 NIST에서 가져올 수 있습니다.

---

# 14. IEC 62443도 보안 감사 근거로 활용

**ISA/IEC 62443**는 산업 자동화 및 제어 시스템(IACS)에 대한 국제 보안 표준 계열입니다.

이 표준은 산업제어 환경의 전 생애주기를 대상으로 보안 요구사항, 위험평가, 보호수준 등을 제시합니다. ([isa.org][8])

특히 Network Zone / Conduit 관점으로 생각하면:

```text
AGV Zone
       │
       │ permitted communication
       ↓
Fleet Control Zone
```

같은 allowlist 모델을 구성하기 쉽습니다.

따라서:

```text
AGV → FleetControl
```

는 허용,

```text
AGV → Internet
AGV → CCTV
AGV → 다른 AGV management interface
```

는 정책에 따라 비허용으로 정의할 수 있습니다.

그리고 ISA99가 NIST CSF에 제출한 자료에서도 **실제 network flow를 baseline과 비교해 deviation을 탐지**하고, unauthorized endpoint와 비정상 protocol 사용을 모니터링하는 예가 제시됩니다. ([NIST][9])

---

# 15. 그래서 실제 Baseline은 이렇게 만들면 됩니다

제가 팀 멘토라면 **AGV Behavior Profile v1**을 다음처럼 만듭니다.

| Baseline Feature        | 정상 값의 근거                       |
| ----------------------- | ------------------------------ |
| UE identity             | Private 5G provisioning        |
| Allowed Slice           | Private 5G 정책 + 3GPP           |
| Allowed DNN             | Private 5G 정책 + 3GPP           |
| RRC states              | TS 38.331                      |
| Registration            | TS 24.501 / 33.501             |
| PDU Session             | TS 24.501 / 23.502             |
| Application protocol    | VDA 5050                       |
| MQTT Topic              | VDA 5050                       |
| Topic direction         | VDA 5050                       |
| Message format          | VDA 5050 JSON schema           |
| QoS                     | VDA 5050                       |
| AGV path/order          | VDA 5050                       |
| Allowed peer            | Deployment/Fleet configuration |
| Traffic periodicity     | 3GPP + 실측                      |
| Latency                 | 3GPP TS 22.104                 |
| Throughput              | 3GPP TS 22.104 + 실측            |
| Connection frequency    | 실측 Baseline                    |
| RRC reconnect frequency | 실측 Baseline                    |
| PDU session frequency   | 실측 Baseline                    |

여기서 핵심적으로 **세 종류를 구분해야 합니다.**

**① 표준이 직접 결정해주는 값**

```text
MQTT
topic direction
message structure
RRC states
PDU procedures
```

**② 표준이 범위만 제공하는 값**

```text
latency
data rate
transfer interval
reliability
```

**③ 우리 환경에서 학습해야 하는 값**

```text
IP address
server address
exact traffic size
connections/minute
RRC reconnections/hour
sessions/hour
평균 packet count
```

이 세 번째를 표준에서 억지로 숫자를 찾아오면 오히려 연구가 약해집니다.

---

# 16. 최종 Audit Rule 예시

그러면 프로그램의 Rule도 근거별로 구분할 수 있습니다.

```text
Rule A01
Unknown UE attempts registration
Source: 3GPP TS 33.501 + local subscriber policy
Severity: High

Rule A02
AGV publishes "order"
Source: VDA 5050
Severity: High

Rule A03
Unknown manufacturer/serialNumber appears
Source: VDA 5050 + asset inventory
Severity: High

Rule A04
AGV communicates with non-approved server
Source: NIST SP 800-82 + IEC 62443 + local allowlist
Severity: High

Rule A05
Malformed VDA5050 JSON
Source: VDA 5050 schema
Severity: Medium

Rule A06
Unexpected MQTT topic
Source: VDA 5050
Severity: Medium

Rule A07
Repeated PDU Session Establishment
Source: 3GPP normal procedure + empirical baseline
Severity: Medium/High

Rule A08
RRC reconnection spike
Source: TS 38.331 + empirical baseline
Severity: Medium

Rule A09
AGV deviates from released route
Source: VDA 5050 order/state semantics
Severity: Critical

Rule A10
Traffic rate exceeds learned range
Source: 3GPP requirement + NIST behavioral baseline
Severity: Medium
```

이렇게 하면 각 탐지 Rule에 **“왜 비정상이라고 했는가?”를 Reference로 설명할 수 있습니다.**

---

# 17. 이 프로젝트의 핵심 Reference Set

논문/프로젝트 제안서에는 최소 이 **6개를 핵심 자료**로 잡으면 됩니다.

1. **VDA 5050 v3.0.0 — Interface for Communication between Mobile Robots and Fleet Control**
   → AGV Application Behavior의 핵심 표준. ([VDA][1])

2. **3GPP TS 22.104 — Service requirements for cyber-physical control applications**
   → AGV/Mobile Robot의 latency, transfer interval, data rate, mobility 기준. ([ETSI][3])

3. **3GPP TS 38.331 — NR RRC**
   → RAN에서 관찰할 RRC state, connection, radio failure 등. ([ETSI][4])

4. **3GPP TS 24.501 / TS 23.502**
   → Registration, PDU Session, DNN, S-NSSAI와 같은 5G session behavior. ([ETSI][5])

5. **3GPP TS 33.501 — Security architecture and procedures for 5G System**
   → UE 인증과 5G 보안 절차. 

6. **NIST SP 800-82 Rev.3 — Guide to Operational Technology Security**
   → 정상 traffic/data flow/device communication을 baseline화하고 anomaly를 탐지한다는 **Audit 방법론의 직접 근거**. ([NIST 출판물][7])

보조적으로 **ISA/IEC 62443**을 붙이면 OT 보안 정책/Zone/Conduit/Continuous monitoring 근거까지 보강할 수 있습니다. ([isa.org][8])

---

## 멘토 관점에서의 추천

지금 프로젝트 방향이면 **“AGV의 모든 네트워크 패킷을 AI로 학습해서 이상탐지”부터 시작하지 않는 게 좋습니다.**

먼저 **Specification-based Behavioral Baseline**을 만드는 게 훨씬 낫습니다.

즉,

**VDA 5050로 L7 정상행위 정의**
→ **3GPP로 5G/RAN 정상행위 정의**
→ **Private 5G 정책으로 허용 UE/DNN/Slice 정의**
→ **NIST/IEC 62443으로 Baseline-based Audit 논리 확보**
→ **표준이 정하지 않는 빈도·트래픽량만 직접 실측해서 학습**

이 구조가 가장 탄탄합니다.

그리고 이 정도 자료가 확보되면 프로젝트 제목도 더 구체적으로 바꿀 수 있습니다.

> **Specification-Driven Behavioral Security Audit for AGV over Private 5G RAN in Smart Ports**

또는 한국어로는

> **스마트항만 Private 5G 환경에서 AGV의 표준 기반 정상행위 모델링 및 RAN 보안 감사 시스템 개발**

정도가 굉장히 잘 맞습니다.

다음 단계에서는 이 Reference들을 기준으로 **실제로 개발할 `AGV Behavioral Baseline Specification v1.0`을 만들어서, “수집 필드 / 정상조건 / 이상조건 / Severity / Reference / RAN에서 수집 가능한 위치”까지 약 30개 Audit Rule로 구체화**하는 게 가장 좋습니다.

[1]: https://www.vda.de/en/news/publications/publication/vda-5050 "VDA 5050"
[2]: https://github.com/VDA5050/VDA5050/blob/main/VDA5050_EN.md?plain=1 "VDA5050/VDA5050_EN.md at main · VDA5050/VDA5050 · GitHub"
[3]: https://www.etsi.org/deliver/etsi_ts/122100_122199/122104/18.04.00_60/ts_122104v180400p.pdf "TS 122 104 - V18.4.0 - 5G; Service requirements  for cyber-physical control applications in vertical domains  (3GPP TS 22.104 version 18.4.0 Release 18)"
[4]: https://www.etsi.org/deliver/etsi_ts/138300_138399/138331/16.17.00_60/ts_138331v161700p.pdf?utm_source=chatgpt.com "TS 138 331 - V16.17.0 - 5G; NR; Radio Resource Control (RRC); Protocol specification  (3GPP TS 38.331 version 16.17.0 Release 16)"
[5]: https://www.etsi.org/deliver/etsi_ts/124500_124599/124501/18.11.01_60/ts_124501v181101p.pdf?utm_source=chatgpt.com "TS 124 501 - V18.11.1 - 5G; Non-Access-Stratum (NAS) protocol for 5G System (5GS); Stage 3 (3GPP TS 24.501 version 18.11.1 Release 18)"
[6]: https://www.etsi.org/deliver/etsi_ts/123500_123599/123502/17.14.00_60/ts_123502v171400p.pdf?utm_source=chatgpt.com "TS 123 502 - V17.14.0 - 5G; Procedures for the 5G System (5GS)  (3GPP TS 23.502 version 17.14.0 Release 17)"
[7]: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-82r3.pdf?utm_source=chatgpt.com "Guide to Operational Technology (OT) Security"
[8]: https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards?utm_source=chatgpt.com "ISA/IEC 62443 Series of Standards - ISA"
[9]: https://www.nist.gov/document/11302023-isa99-comments-nist-csf-20-v03redacted?utm_source=chatgpt.com "ISA99 Committee on the Security of Industrial Automation and Control Systems (IACS)"
