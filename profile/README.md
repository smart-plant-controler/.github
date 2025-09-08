# Smart Plant Controller (STM32 NUCLEO‑F103RB)

스마트 화분(스마트팜 미니)용 **토양 습도/온도/Ph 모니터링 & 자동 급수 컨트롤러** 프로젝트입니다. STM32 NUCLEO‑F103RB 보드와 릴레이 모듈, 토양 센서, 플로트 스위치, ESP32 UART 브리지를 사용합니다.

> 예시 저장소: `smart-plant-controler/STM32NUCLEO-F103RB` (저장소 이름/링크는 프로젝트에 맞게 수정하세요)

---

## ✨ 주요 기능

* 토양 **습도(Analog)** 측정 및 임계값 기반 펌프 제어
* **DS18B20 방수형 센서**로 토양 온도 측정
* **플로트 스위치**로 저수조 수위 보호 (수위 낮음 시 펌프 차단)
* **릴레이 모듈**로 펌프 ON/OFF 제어 (Active‑HIGH/LOW 설정 가능)
* 주기적 측정 스케줄러(타이머 인터럽트 기반)
* **UART 시리얼 프롬프트** 상태 조회/설정
* **ESP32**와 UART 연동하여 사용자와 통신

---

## 🧰 하드웨어 구성

| 모듈         | 예시 모델                   | 연결 요약                                 |
| ---------- | ----------------------- | ------------------------------------- |
| MCU        | **STM32 NUCLEO‑F103RB** | 기본 보드                                 |
| 토양 습도 센서   | Analog 타입 (Vout)        | Vout → **A0(Arduino 헤더)**, VCC, GND   |
| 릴레이 모듈     | 5V 또는 3.3V 릴레이, SSR 가능  | **Signal → D4(Arduino 헤더)**, VCC, GND |
| 펌프         | 5–12V DC 펌프             | 별도 전원/다이오드, 릴레이 COM/NO 이용             |
| 플로트 스위치    | NO/NC 타입                | 스위치 한 쪽 GND, 다른 쪽 **디지털 입력**(풀업)      |
| 온도 센서  | DS18B20 방수형             | 1‑Wire, 4.7kΩ 풀업 → 지정 핀               |
| ESP32 | UART 브리지                | STM32 **TX/RX ↔ RX/TX**, 공통 GND       |

> ⚠️ 펌프/릴레이 전원은 **보드와 분리**하고, **공통 접지**를 꼭 맞춰 주세요. 인덕티브 부하라면 플라이백 다이오드를 권장합니다.

---

## 🔌 배선 예시 (간단)

* 습도 센서 `Vout → A0`
* 릴레이 `IN → D4` (보드 3.3V 논리 호환 릴레이 사용 권장)
* 플로트 스위치 `→ D5` (내부 풀업 입력으로 사용, GND로 당겨지면 LOW)
* (옵션) DS18B20 `DATA → D6` (4.7kΩ to VCC 풀업)
* (옵션) ESP32 UART: `STM32 TX(USARTx_TX) → ESP32 RX`, `STM32 RX(USARTx_RX) → ESP32 TX`, **GND 공통**

핀은 `CubeMX`에서 실제 MCU 핀(예: `PA0` 등)으로 매핑하세요. 보드 실크의 `A0/D4`는 Arduino 호환 헤더 기준입니다.

---

## 🗂️ 폴더 구조 (예시)

```
firmware/
  moisture_scheduler/
    Core/
    Drivers/
    Middlewares/
    Inc/
    Src/
    README_firmware.md
docs/
  wiring_diagram.png
  system_overview.drawio
scripts/
  flash_openocd.sh
  monitor.py
```

---

## ⚙️ 펌웨어 빌드

### STM32CubeIDE

1. 저장소 클론 후 `firmware/moisture_scheduler` 프로젝트 Import
2. `Build` → `Debug` 또는 `Release`
3. `Run/Debug`로 보드에 다운로드 및 시리얼 모니터 연결



---

## 🔧 주요 설정 (컴파일 타임)

`Inc/app_config.h` (예시)

```c
#define RELAY_ACTIVE_HIGH   1   // 1: High=ON, 0: Low=ON
#define MOIST_THRESHOLD     650 // ADC 임계값 (0~4095)
#define SAMPLE_PERIOD_MS    1000
#define UART_BAUDRATE       115200
```

---

## 🖥️ 시리얼 프롬프트 (Tera Term 등)

부팅 메시지 예:

```
BOOT: ready
H:512  T:23.5C  W:OK  PUMP:OFF
```

명령어 예시:

```
help            # 명령어 목록
stat            # 현재 측정값/펌프 상태
set th <val>    # 습도 임계값 설정 (예: set th 700)
set pump on/off # 펌프 수동 제어
save            # 설정 저장(옵션)
```

출력 항목:

* `H`: 습도 ADC(0\~4095)
* `T`: DS18B20 온도(°C, 옵션)
* `W`: 수위(OK/LOW)

---

## 🧪 동작 로직 (요약)

1. 주기마다 ADC로 습도 측정 → `H`
2. 플로트 스위치 확인 → `W=LOW`면 펌프 강제 OFF
3. `H < 임계값`이며 `W=OK`이면 펌프 ON, 아니면 OFF
4. (옵션) DS18B20 온도 읽어 함께 출력
5. UART 명령으로 임계값/펌프 수동제어 가능

---

## 🛠️ 트러블슈팅

* **BOOT만 보이고 갱신이 없을 때**: 타이머/USART 초기화 확인, 우선 간단 루프에서 `HAL_Delay(1000); printf("ping\r\n");`로 UART 단독 점검
* **릴레이가 반대로 동작**: `RELAY_ACTIVE_HIGH` 토글
* **ESP32와 통신 불가**: RX/TX 교차, **GND 공통**, 보레이트 동일 여부 확인
* **펌프 미동작**: 외부 전원, 릴레이 결선(COM/NO), 플라이백 다이오드, 최대전류 확인

---

## ✅ 체크리스트 (초기 bring‑up)

* [ ] 공통 GND 연결 확인
* [ ] ADC 채널/핀 매핑 확인 (A0→PA0 등)
* [ ] 타이머/USART 클럭 enable
* [ ] 릴레이 구동 전 보드/릴레이 **전압 호환** 확인
* [ ] 시리얼 115200 8N1 설정

---

## 🤝 컨트리뷰션 가이드 (팀 협업)

* 브랜치 전략: `main`(안정), `dev`(통합), `feat/*`, `fix/*`
* 커밋 컨벤션: Conventional Commits (예: `feat: add UART prompt`)
* PR 템플릿 사용, 코드 리뷰 1인 이상 승인
* 이슈 템플릿으로 버그/기능 요청 관리

---

## 🪪 라이선스

MIT (원하는 라이선스로 교체 가능)

---

## 📸 자료/이미지

* `docs/wiring_diagram.png` : 배선도
* `docs/system_overview.drawio` : 시스템 구성도
* 동작 사진/영상/GIF 추가 권장

---

## 🗺️ 로드맵 (예시)

* [ ] EEPROM/Flash에 설정 저장
* [ ] 펌프 가동 최소/최대 시간 세이프가드
* [ ] 이동 평균/스로틀로 노이즈 완화
* [ ] ESP32 MQTT/HTTP 연동 (상태 업로드)
* [ ] 웹 대시보드 (그래프/알림)

---

## 🛡️ 배지(선택)

README 상단에 다음과 같은 배지를 추가할 수 있습니다.

```
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Board](https://img.shields.io/badge/Board-NUCLEO--F103RB-informational)
![STM32](https://img.shields.io/badge/STM32-CubeIDE-blue)
```

배지는 [https://shields.io](https://shields.io) 에서 커스터마이즈하세요.

---

## 📎 빠른 시작 (요약)

1. 하드웨어 배선 (공통 GND 필수)
2. 펌프 전원/릴레이 결선 확인
3. 펌웨어 빌드 & 다운로드
4. 시리얼 연결(115200) → `stat`로 상태 확인
5. `set th 650` 등으로 임계값 조정 후 운전 시작

