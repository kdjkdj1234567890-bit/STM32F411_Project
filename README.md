# STM32F411 GPIO 실습 — LD2 LED 제어 및 레지스터 관찰

> 실습일: **2026-11-11**  
> 주제: GPIO 구조, LD2 LED 회로, HAL 기반 LED 토글, GPIOA 레지스터 디버깅, 오실로스코프 에지 측정  
> 장비: STM32F411 계열 NUCLEO 보드(제공 사진은 NUCLEO-F411RE 계열로 보임), STM32CubeIDE, Tektronix TDS 2012B 오실로스코프

## 1. 실습 목표

1. MCU의 GPIO가 어떻게 외부 LED를 제어하는지 이해한다.
2. 메모리 매핑 GPIO 레지스터(MODER, IDR, ODR, BSRR 등)의 역할과 주소를 확인한다.
3. `HAL_GPIO_TogglePin()`과 `HAL_Delay(1000)`을 사용해 LED 출력을 전환한다.
4. 디버거의 Expressions에서 GPIOA 레지스터를 확인하고, 오실로스코프로 상승·하강 에지를 관찰한다.
5. 실제 LED 부품의 회로도, PCB 패턴, 데이터시트를 연결해 읽는다.

> **수업 과제**: 2026-11-11 Google Classroom 공지 **「GPIO F/W 깃허브 정리」**에 맞춰 작성한 학습 기록이다.  
> **자료 구분**: [교수님 GPIO 학습 자료](https://github.com/gotree94/STM32/tree/main/NUCLEO_F411RE/GPIO_DOC)와 제공된 PDF/회로 자료를 참고하고, 오늘 촬영한 코드·디버거·파형 사진을 실습 증거로 구분했다. 교수님 원문 압축파일이나 PDF 전문을 재게시하지 않았다.

## 2. GPIO와 시스템 아키텍처 요약

- **GPIO(General Purpose Input/Output)**: MCU 핀을 입력/출력/아날로그/대체 기능으로 설정하여 외부 회로와 신호를 교환한다.
- **Cortex-M4 CPU**: 프로그램을 실행하고 GPIO 레지스터를 읽고 쓴다.
- **AHB1 버스**: STM32F411에서 GPIO 주변장치로 접근하는 버스. **이번 실습은 STM32F411의 GPIOA와 AHB1 연결만 다룬다**.
- **RCC 클록 활성화**: GPIOA를 사용하기 전 GPIOA 주변장치 클록을 켠다(STM32CubeMX와 `MX_GPIO_Init()`에서 설정 가능).
- **SFR(Special Function Register)**: GPIO 설정/상태를 제어하는 메모리 맵 레지스터. 일반 변수와 달리 쓰기 동작이 하드웨어에 영향을 준다.

## 3. GPIOA 레지스터와 주소

GPIOA 기준 주소(Base address)는 **`0x40020000`**이다.

| 레지스터 | 오프셋 | 주소 | 기능 |
|---|---:|---|---|
| `MODER` | `0x00` | `0x40020000` | 핀 모드: 입력(00), 출력(01), 대체기능(10), 아날로그(11) |
| `OTYPER` | `0x04` | `0x40020004` | Push-pull(0) / Open-drain(1) |
| `OSPEEDR` | `0x08` | `0x40020008` | GPIO 출력 속도 |
| `PUPDR` | `0x0C` | `0x4002000C` | 풀업 / 풀다운 설정 |
| `IDR` | `0x10` | `0x40020010` | 입력 핀의 실제 논리 상태 읽기 |
| `ODR` | `0x14` | `0x40020014` | 출력 데이터의 현재 상태 |
| `BSRR` | `0x18` | `0x40020018` | 비트 설정(Set)·리셋(Reset), 원자적 쓰기 지원 |
| `AFRL` | `0x20` | `0x40020020` | PA0~PA7 대체 기능 선택 |
| `AFRH` | `0x24` | `0x40020024` | PA8~PA15 대체 기능 선택 |

`MODER`에서 핀 하나는 2비트를 사용한다. 예를 들어 PA5 모드는 `MODER[11:10]`이다.

![GPIO 설정 관련 수업 필기](images/05_gpio_register_whiteboard.png)

### 디버거 실제 화면

![GPIOA 레지스터 디버깅](images/02_gpioa_register_debug.jpg)

사진에서 읽히는 값:

- `GPIOA` 베이스 주소: `0x40020000`
- `MODER`: `0xA80010A0`
- `ODR`: `0x00000000`(사진이 찍힌 시점의 값)
- 나머지 레지스터의 값은 디버깅 시점마다 달라질 수 있다.

**확인 필요 — 중요한 불일치:** 촬영한 `MODER = 0xA80010A0`을 비트로 해석하면 **PA5의 모드 비트 [11:10]는 `00`(입력)**, **PA6의 [13:12]는 `01`(출력)**이다. 반면 일반적인 NUCLEO-F411RE의 사용자 LED(LD2)는 **PA5**에 연결된다. 따라서 해당 화면이 실제 LED 토글 중인 설정인지, 프로젝트의 `LD2_Pin`이 무엇인지, 사용하는 보드/보드 리비전이 맞는지 **`main.h`와 `MX_GPIO_Init()`을 대조해야 한다**. 이 불일치를 해결하기 전에는 'PA5 출력 설정이 레지스터로 검증되었다'고 주장하지 않는다.

## 4. LD2 LED 회로와 PCB

### 보드 사진

![보드의 LD2와 R31](images/06_nucleo_ld2_photo.png)

### 제공된 LD2 회로도

![LD2 회로도](images/07_ld2_schematic.png)

회로도 발췌를 보면 LD2의 **캐소드(-)** 쪽은 GND에 연결되고, **애노드(+)** 쪽은 직렬 저항 `R31`과 솔더 브리지 `SB21` 방향으로 연결된다. 자료의 표기값은 **R31 = 510Ω**이다. 해당 연결이 PA5로 이어지는지는 **전체 보드 회로도**에서 최종 확인한다.

### PCB 배치 화면

![LD2와 R31 PCB 배치](images/08_ld2_pcb_layout.png)

PCB 화면에서 `LD2`, `R31`, `U4`, `C21`의 물리적 위치를 구분할 수 있다. PCB 패턴 그림만으로 전체 연결 경로를 확정하지 않고 회로도 및 넷 이름과 함께 본다.

> **부품 표기 재확인**: 보드 확대 사진의 R31 부품 표기가 '151'처럼 보이는 반면, 제공된 회로도에는 `510`이라고 쓰여 있다. 사진 해상도·보드 리비전·부품값 중 어느 차이인지 확인이 필요하다. 이 자료만으로 실제 저항값을 단정하지 않는다.

## 5. LED 부품 데이터시트 읽기

함께 제공된 Everlight LED 데이터시트 **`19-213SYGC/S530-XX/TR8`**의 주요 조건은 다음과 같다.

| 항목 | 데이터시트 값 | 비고 |
|---|---|---|
| 발광색 | Super Yellow Green | 해당 부품 모델 기준 |
| 순방향 전압 `VF` | Typ. 2.0 V, Max. 2.4 V | `IF = 20 mA`, `Ta = 25°C` |
| 허용 순방향 전류 `IF` | 최대 25 mA | 절대최대정격 |
| 피크 파장 | Typ. 575 nm | `IF = 20 mA` |
| 지향각 | Typ. 120° | 데이터시트 조건 |

LED는 전류 제한 저항이 필요하다. 다만 **이 데이터시트 모델이 실제 납땜된 LD2와 정확히 동일한지는 부품 BOM/보드 리비전 확인이 필요**하다. 데이터시트의 20mA 조건값을 실제 회로 전류의 측정값처럼 쓰지 않는다.

## 6. 오늘 작성한 LED 토글 코드

STM32CubeIDE에서 확인한 핵심 코드(주석과 무관한 부분을 간추림):

```c
/* MX_GPIO_Init(); 등 주변장치 초기화 후 */
int counter = 0;

while (1)
{
    counter++;
    HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
    HAL_Delay(1000);  // 1000 ms = 약 1초 대기
}
```

![STM32CubeIDE 코드 화면](images/01_led_toggle_code.jpg)

- `HAL_GPIO_TogglePin(...)`: 현재 핀 출력 상태를 반전한다(High ↔ Low).
- `HAL_Delay(1000)`: 다음 반전까지 약 1초 대기한다. 실제 지연은 클록 설정과 실행 상태의 영향을 받을 수 있다.
- **예상 동작**: High 1초 + Low 1초 → **전체 점멸 주기 약 2초**, **반전 간격 약 1초**.
- `counter++`: 반복 횟수를 세는 변수. 단순 선언·증가만으로는 LED 동작에 필요하지 않다.
- `HAL_GPIO_WritePin()`으로 1초 켜고 1초 끄는 예제(교수님 자료)와는 달리, **오늘 촬영된 코드는 `TogglePin()` 방식**이다. 결과는 유사하지만 코드 방식은 구분한다.

## 7. 오실로스코프 측정

사용 장비: Tektronix **TDS 2012B** / 채널 `CH2`.

### 상승 에지(Low → High)

![상승 에지](images/03_oscilloscope_rising.jpg)

### 하강 에지(High → Low)

![하강 에지](images/04_oscilloscope_falling.jpg)

**관찰된 사실**

- 두 사진에 각각 Low → High, High → Low 신호 전환이 보인다.
- 화면 하단에 `CH2 1.00V`(수직 스케일 약 1 V/div)와 `M 10.0ms`(수평 스케일 10 ms/div)가 표시된다.
- 두 에지가 있다는 것은 출력이 전환되었음을 보여 주지만, **사진 한 장의 시간 범위(약 100 ms)만으로 1초 간격·2초 주기를 직접 입증할 수는 없다**.
- 정확한 진폭·주기는 프로브 감쇠 설정과 오실로스코프 커서/자동 측정값을 추가로 확인해야 한다.

**재측정 제안**: 수평 스케일을 **200~500 ms/div**로 조정하여 연속 파형 여러 에지를 한 화면에 나타내고 `Period`, `Frequency`, `Vmax`, `Vmin`을 확인한다.

## 8. 오실로스코프 프로브 사양과 측정 시 주의점

수업에서 제공된 **프로브 사양표 사진**으로 X1/X10 설정을 비교했다. 이 표는 20·40·60·100MHz 등급의 프로브 사양을 함께 제시한 자료이며, 오늘 실제 연결한 프로브가 어느 등급인지 사진만으로 확정할 수 없다.

![수업에서 제공된 오실로스코프 프로브 사양표](images/09_oscilloscope_probe_specifications.jpg)

| 항목 | X1 | X10 |
|---|---|---|
| 감쇠비 | 1:1 | 10:1 |
| 입력 저항 | 1MΩ ±2% | 10MΩ ±2% |
| 입력 정전용량 | 85~115pF | 18.5~22.5pF(20·40MHz 등급), 14.5~17.5pF(60·100MHz 등급) |
| 사양표상 시스템 대역폭 | DC~6MHz | 등급별 DC~20/40/60/100MHz |

**측정 절차에서 배운 점**

1. **프로브 스위치가 X1인지 X10인지 먼저 확인한다.** 프로브의 물리적 감쇠비와 오실로스코프의 CH2 `Probe` 설정을 일치시켜야 전압을 올바르게 읽을 수 있다.
2. **X10이 신호에 주는 부하가 일반적으로 더 적다.** 특히 빠른 상승·하강 에지를 측정할 때 유리하지만, 신호 크기와 오실로스코프 노이즈 수준도 함께 고려한다.
3. **측정 전 프로브 보상(compensation)을 점검한다.** 기준 사각파에 연결해 에지가 과보상·미보상되지 않는지 확인한다. 이번 실습에서 보상을 실제 수행했는지는 기록이 없다.
4. 사양표는 오실로스코프 **입력 임피던스 1MΩ** 및 장비의 **예열 20분 이상** 조건을 명시한다.
5. 앞서 촬영한 CH2 파형의 `1.00V/div` 표시만으로 실제 프로브 X1/X10 설정, 진폭 및 주기를 확정할 수 없다.

> **아직 확인되지 않은 사항**: 프로브의 실제 대역 등급, 당시 X1/X10 스위치 위치, 오실로스코프 CH2 프로브 배율, 프로브 보상 여부. 따라서 사진만으로 측정 전압이 정확했다고 단정하지 않는다.

## 9. 실습 결과 및 배운 점

**이번 실습에서 정리한 내용**

1. MCU는 GPIO 관련 SFR을 통해 출력 핀의 상태를 제어한다.
2. LED는 극성·직렬 저항·PCB 연결을 함께 확인해야 한다.
3. `TogglePin()`을 1초 지연과 함께 반복하면 코드상 2초 주기의 LED 점멸이 예상된다.
4. 디버거에서 GPIOA의 베이스 주소와 레지스터 값을 읽었으며 오실로스코프에서 양방향 에지를 관찰했다.

**후속 검증**

- [ ] `main.h`에서 `LD2_Pin`과 `LD2_GPIO_Port`의 실제 정의 확인
- [ ] `MX_GPIO_Init()`에서 PA5/PA6 중 어느 핀이 출력으로 설정됐는지 확인
- [ ] GPIOA `MODER` 스냅샷과 실제 동작 핀의 대응 확인
- [ ] R31 부품 마킹과 회로도 표기값 및 보드 리비전 확인
- [ ] 오실로스코프 CH2 설정과 실제 프로브 감쇠비(X1/X10) 일치 확인
- [ ] 오실로스코프로 실제 주기·진폭 수치 측정

## 10. 참고 자료

- [교수님 GPIO 학습 자료 — gotree94/STM32](https://github.com/gotree94/STM32/tree/main/NUCLEO_F411RE/GPIO_DOC)
- [STMicroelectronics STM32F411xC/xE 데이터시트](https://www.st.com/resource/en/datasheet/stm32f411ce.pdf)
- [STMicroelectronics RM0383 레퍼런스 매뉴얼](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- Everlight `19-213SYGC/S530-XX/TR8` 데이터시트 — 실습 시 제공된 PDF(본문에 핵심 사양 정리, 원본 파일은 별도)
- 오실로스코프 프로브 X1/X10 사양표 — 수업에서 제공된 사진(위 8절)

---

**기록 원칙:** 실제 실습 사진에서 확인한 내용, 교수 자료/데이터시트의 설명, 아직 검증하지 못한 추정을 분리하여 작성했다.


---

---

## 교수님 자료 — STM32F411 전용 접기식 학습 자료

> [출처: gotree94 / STM32 / NUCLEO_F411RE / GPIO_DOC](https://github.com/gotree94/STM32/tree/main/NUCLEO_F411RE/GPIO_DOC). 다음 5개 항목은 교수님이 제공한 자료의 **F411 설명과 그림만** 학습 목적에 맞게 재구성하였다. 아래 그림은 교수님 자료에서 가져왔으며, 오늘 내가 직접 수행한 실험·측정 사진과 혼동하지 않는다.

**목차:** [Architecture](#11-architecture-시스템-아키텍처) · [CLOCK](#12-clock-클록) · [Memory Map](#13-memory-map-메모리-맵) · [GPIO](#14-gpio-입출력-및-레지스터) · [Test](#15-test-gpio-fw-디버깅)

## 11. Architecture (시스템 아키텍처)

<details>
<summary><strong>F411 — 눌러서 펼치기</strong></summary>

STM32F411은 **Arm Cortex-M4** 코어를 사용한다. CPU에서 실행한 명령이 GPIO를 움직이려면 버스를 거쳐 **주변장치 레지스터(SFR)**에 접근해야 한다.

- **CPU**: 명령어 실행.
- **Flash**: 프로그램 코드 저장.
- **SRAM**: 실행 중 변수와 스택 저장.
- **AHB 버스 매트릭스**: CPU, DMA, 메모리, GPIO 등 장치를 연결하고 접근을 조정.
- **AHB1**: STM32F411의 GPIO를 연결하는 버스 영역.

**LED 제어의 흐름:** `C 코드 → Cortex-M4 → AHB1 → GPIO 레지스터 → 핀 전압 → LD2 LED`

교수님 그림은 시스템 전체 구조를 보여 주며, 실제 LED 회로는 위 4절에서 따로 확인할 수 있다.

**그림 1. STM32F411 버스 매트릭스의 주요 연결**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/arm-arch-2.png" alt="그림 1. STM32F411 버스 매트릭스의 주요 연결 (교수님 STM32F411 수업 자료)" width="760">

**그림 2. STM32F411 내부 버스와 주변장치 구조**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/arm-arch-1.png" alt="그림 2. STM32F411 내부 버스와 주변장치 구조 (교수님 STM32F411 수업 자료)" width="760">

</details>

## 12. CLOCK (클록)

<details>
<summary><strong>F411 — 눌러서 펼치기</strong></summary>

클록은 MCU 내부 회로가 순서대로 동작할 수 있도록 하는 **시간 기준 신호**이다. STM32F411은 RCC(Reset and Clock Control)를 통해 클록 소스를 선택하고 주변장치 클록을 켜거나 끈다.

1. `HAL_Init()` — HAL 사용에 필요한 기본 초기화.
2. `SystemClock_Config()` — 시스템 클록 구성.
3. `MX_GPIO_Init()` — GPIOA 주변장치 클록을 활성화하고 핀을 초기화.
4. `HAL_GPIO_TogglePin()` — 출력 상태 반전.
5. `HAL_Delay(1000)` — HAL 시간 기준 약 1초 지연.

`RCC_AHB1ENR`의 GPIOA 클록 활성화 비트를 설정해야 GPIOA 레지스터를 정상적으로 사용할 수 있다. **MCU 최대 100 MHz**는 데이터시트상 상한이며 이번 실습의 실측 클록 속도라는 뜻은 아니다.

**그림 3. STM32F411 클록 트리 1**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/411-clk1.png" alt="그림 3. STM32F411 클록 트리 1 (교수님 STM32F411 수업 자료)" width="760">

**그림 4. STM32F411 클록 트리 2**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/411-clk2.png" alt="그림 4. STM32F411 클록 트리 2 (교수님 STM32F411 수업 자료)" width="760">

</details>

## 13. Memory Map (메모리 맵)

<details>
<summary><strong>F411 — 눌러서 펼치기</strong></summary>

STM32F411은 **메모리 매핑 입출력(Memory-Mapped I/O)**을 사용한다. CPU는 일반 메모리를 읽고 쓰듯이 정해진 주변장치 주소에 접근해 GPIO를 제어할 수 있다.

| 구역 | 대표 주소 | 역할 |
|---|---|---|
| Flash | `0x08000000` | 사용자 프로그램 코드 |
| SRAM | `0x20000000` | 실행 중 데이터 |
| GPIOA | `0x40020000` | GPIOA 설정·입출력 레지스터 |

**GPIOA 주소 예시**: `MODER=0x40020000`, `ODR=0x40020014`, `BSRR=0x40020018`. 따라서 디버거 **Memory 창**에 `0x40020000`을 넣으면 GPIOA 영역을 볼 수 있다.

**주의:** 프로그램 코드가 저장되는 Flash 주소와 GPIO 레지스터 주소는 서로 다르다.

**그림 5. STM32F411 메모리 맵 개요**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/007-1.png" alt="그림 5. STM32F411 메모리 맵 개요 (교수님 STM32F411 수업 자료)" width="760">

**그림 6. STM32F411 주변장치 주소 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/009-1.png" alt="그림 6. STM32F411 주변장치 주소 표 (교수님 STM32F411 수업 자료)" width="760">

**그림 7. AHB 관련 레지스터 경계 주소**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/010-1.png" alt="그림 7. AHB 관련 레지스터 경계 주소 (교수님 STM32F411 수업 자료)" width="760">

**그림 8. GPIO 및 주변장치 주소 참고 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/011-1.png" alt="그림 8. GPIO 및 주변장치 주소 참고 표 (교수님 STM32F411 수업 자료)" width="760">

</details>

## 14. GPIO (입출력 및 레지스터)

<details>
<summary><strong>F411 — 눌러서 펼치기</strong></summary>

**GPIO**는 외부 신호를 입력받거나 출력하는 핀이다. 각 포트는 MODER(모드), OTYPER(출력 방식), OSPEEDR(출력 속도), PUPDR(풀업/풀다운), IDR(입력), ODR(출력), BSRR(설정/리셋), AFR(대체 기능) 등의 레지스터로 제어한다.

**MODER는 핀당 2비트를 사용**한다.

| 설정 | 의미 | 예시 |
|---|---|---|
| `00` | 입력 | 스위치 입력 |
| `01` | 출력 | LED 제어 |
| `10` | 대체 기능 | UART, SPI 연결 |
| `11` | 아날로그 | ADC 입력 |

**IDR과 ODR의 차이**: `IDR`은 핀의 현재 논리 상태를 읽고, `ODR`은 출력 데이터 레지스터의 값을 읽는다.

**푸시풀/오픈드레인**: 푸시풀은 High와 Low를 능동적으로 구동한다. 오픈드레인은 Low 또는 Hi-Z 상태로 동작하므로 외부 풀업 등이 필요할 수 있다.

**BSRR**은 특정 비트만 설정/초기화할 때 사용한다. 하위 16비트에 `1`을 쓰면 Set, 상위 16비트에 `1`을 쓰면 Reset이다.

```c
/* 원리 설명용 예시: PA5가 출력으로 설정되어 있을 때 */
GPIOA->BSRR = (1U << 5);         // PA5 High
GPIOA->BSRR = (1U << (5 + 16));  // PA5 Low
```

이 코드는 이번 실습에서 직접 실행한 코드가 아니라, **레지스터 원리를 이해하기 위한 예시**이다. 실제 실습 코드는 위 6절의 `HAL_GPIO_TogglePin()`이다.

**그림 9. GPIO 한 핀의 기본 회로 구조**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F16.png" alt="그림 9. GPIO 한 핀의 기본 회로 구조 (교수님 STM32F411 수업 자료)" width="760">

**그림 10. 입력 핀 구성**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F18.png" alt="그림 10. 입력 핀 구성 (교수님 STM32F411 수업 자료)" width="760">

**그림 11. 출력 핀 구성**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F19.png" alt="그림 11. 출력 핀 구성 (교수님 STM32F411 수업 자료)" width="760">

**그림 12. 대체 기능 구성**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F20.png" alt="그림 12. 대체 기능 구성 (교수님 STM32F411 수업 자료)" width="760">

**그림 13. 아날로그 모드 구성**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F21.png" alt="그림 13. 아날로그 모드 구성 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 841.png: MODER 레지스터 비트 배치**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/841.png" alt="수업 자료 841.png: MODER 레지스터 비트 배치 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 842.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/842.png" alt="수업 자료 842.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 843.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/843.png" alt="수업 자료 843.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 844.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/844.png" alt="수업 자료 844.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 845.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/845.png" alt="수업 자료 845.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 846.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/846.png" alt="수업 자료 846.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 847.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/847.png" alt="수업 자료 847.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 848.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/848.png" alt="수업 자료 848.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 849.png: GPIO 레지스터 관련 표**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/849.png" alt="수업 자료 849.png: GPIO 레지스터 관련 표 (교수님 STM32F411 수업 자료)" width="760">

**수업 자료 8410.png: AFR 레지스터 비트 배치**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/8410.png" alt="수업 자료 8410.png: AFR 레지스터 비트 배치 (교수님 STM32F411 수업 자료)" width="760">

**내 화면과 대조할 때 주의:** 촬영된 `MODER=0xA80010A0`은 PA5=입력(00), PA6=출력(01)로 해석된다. 일반적인 NUCLEO-F411RE의 LD2가 PA5에 연결되는 점과 차이가 있으므로, `LD2_Pin`과 `MX_GPIO_Init()`을 직접 확인하기 전에는 정상 설정으로 확정하지 않는다.

</details>

## 15. Test (GPIO F/W 디버깅)

<details>
<summary><strong>F411 — 눌러서 펼치기</strong></summary>

교수님 F411 자료의 디버깅 순서를 정리했다. 아래 사진은 **교수님 자료의 예시 화면**이며, 오늘 본인이 직접 촬영한 실습 사진은 위 3·6·7절에 별도로 있다.

**진행 순서:** `Debug 시작 → Expressions → SFR 확인 → Memory 확인 → 초기화 함수 단계별 실행 → LED 상태 변경 → 레지스터 비교`

| 순서 | 확인 항목 | 배울 점 |
|---:|---|---|
| 1 | `LD2_GPIO_Port`를 Expressions에 등록 | 현재 사용 중인 포트 주소를 확인 |
| 2 | SFR 창에서 GPIO 관련 레지스터 확인 | MODER/ODR/IDR 기능과 값 비교 |
| 3 | Core Registers 및 Disassembly | CPU의 PC, LR, SP와 기계어 실행 흐름 확인 |
| 4 | Memory 창에 `0x40020000` 입력 | GPIOA 메모리 매핑 직접 관찰 |
| 5 | `HAL_Init()` → `SystemClock_Config()` → `MX_GPIO_Init()` | 초기화 과정에서 설정 변경 확인 |
| 6 | `HAL_GPIO_WritePin()`과 `HAL_Delay(1000)` | 교수님 예제의 LED 켜기/끄기 실행 |

**구분할 점:** 교수님 캡처는 `HAL_GPIO_WritePin(..., 1/0)`로 출력을 각각 설정한다. **오늘 실습 사진의 코드는 `HAL_GPIO_TogglePin()`**으로 현재 상태를 반전한다. 함수가 다르므로 같은 코드를 사용했다고 기록하지 않는다.

**코어 레지스터 vs GPIO 레지스터**: PC/LR/SP는 CPU 실행 흐름용이고, MODER/ODR/BSRR은 GPIO 제어용이다. `HAL_GPIO_WritePin()`의 내부 구현에서는 **BSRR**을 사용해 비트를 바꾸는 경로를 볼 수 있다.

**F411 실습 화면 01. Expressions에 LD2_GPIO_Port 추가**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_001.png" alt="F411 실습 화면 01. Expressions에 LD2_GPIO_Port 추가 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 02. SFR 레지스터 창 관찰**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_002.png" alt="F411 실습 화면 02. SFR 레지스터 창 관찰 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 03. CPU 코어 레지스터 관찰**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_003.png" alt="F411 실습 화면 03. CPU 코어 레지스터 관찰 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 04. Disassembly 메뉴 열기**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_013.png" alt="F411 실습 화면 04. Disassembly 메뉴 열기 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 05. Disassembly 화면**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_014.png" alt="F411 실습 화면 05. Disassembly 화면 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 06. CPU 코어의 주요 구성요소**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_MCU_Core.png" alt="F411 실습 화면 06. CPU 코어의 주요 구성요소 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 07. Memory 창에서 GPIOA 시작 주소 확인**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_004.png" alt="F411 실습 화면 07. Memory 창에서 GPIOA 시작 주소 확인 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 08. HAL_Init 실행**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_005.png" alt="F411 실습 화면 08. HAL_Init 실행 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 09. SystemClock_Config 실행**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_006.png" alt="F411 실습 화면 09. SystemClock_Config 실행 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 10. MX_GPIO_Init 실행**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_007.png" alt="F411 실습 화면 10. MX_GPIO_Init 실행 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 11. USART2 초기화 예제(해당 기능을 쓰는 프로젝트에서만)**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_008.png" alt="F411 실습 화면 11. USART2 초기화 예제(해당 기능을 쓰는 프로젝트에서만) (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 12. GPIO 핀 High 설정 예제**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_009.png" alt="F411 실습 화면 12. GPIO 핀 High 설정 예제 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 13. HAL_Delay(1000) 실행**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_010.png" alt="F411 실습 화면 13. HAL_Delay(1000) 실행 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 14. GPIO 핀 Low 설정 예제**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_011.png" alt="F411 실습 화면 14. GPIO 핀 Low 설정 예제 (교수님 STM32F411 수업 자료)" width="760">

**F411 실습 화면 15. HAL_Delay(1000) 후 상태 확인**

<img src="https://raw.githubusercontent.com/gotree94/STM32/main/NUCLEO_F411RE/GPIO_DOC/img/F411_R_012.png" alt="F411 실습 화면 15. HAL_Delay(1000) 후 상태 확인 (교수님 STM32F411 수업 자료)" width="760">

### 내 실습에 연결한 결과

- [x] `HAL_GPIO_TogglePin()`과 `HAL_Delay(1000)` 코드 사진 확보
- [x] GPIOA 디버거 화면 확보
- [x] 상승·하강 에지 사진 확보
- [ ] `LD2_Pin` 정의와 MODER 값 일치 여부 확인
- [ ] 실제 주기와 출력 전압의 계측값 기록

이미지에서 확인할 수 없는 결과는 임의로 만들어 넣지 않았다.

</details>

---

**실습 범위:** STM32F411 전용. 교수님 예제 화면과 본인의 실습 결과는 명확히 구분하였다.
