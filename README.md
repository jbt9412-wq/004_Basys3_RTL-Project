# Basys3 FPGA 기반 DFT Equalizer 시스템

## 1. 프로젝트 개요

본 프로젝트는 마이크 입력 음성 신호를 FPGA에서 샘플링하고, DFT 연산을 통해 주파수 성분으로 변환한 뒤, 그 결과를 LED Matrix Equalizer 형태로 시각화하는 RTL 설계 프로젝트입니다.

전체 시스템은 **마이크 입력 → XADC 샘플링 → AXI 기반 데이터 이동 → DFT 연산 → LED Matrix 출력** 흐름으로 동작합니다.

본인은 프로젝트에서 **DFT 연산 결과를 32x8 LED Matrix로 시각화하는 출력부 RTL 설계 및 하드웨어 검증**을 담당했습니다.

---

## 2. 개발 환경

| 항목 | 내용 |
|---|---|
| FPGA Board | Basys3 FPGA Board |
| FPGA Device | Xilinx Artix-7 |
| Language | Verilog HDL |
| Tool | Vivado |
| Input | Mic Sensor, XADC |
| Data Transfer | AXI Stream, AXI DMA, AXI-Lite, AXI4 Read |
| Processing | DFT Core |
| Output Device | MAX7219 기반 8x8 LED Matrix x 4 |
| Display Size | 32 columns x 8 rows |

---

## 3. 전체 시스템 동작 흐름

전체 데이터 흐름은 다음과 같습니다.

```text
Mic Sensor
   ↓
XADC Reader
   ↓
AXI Stream / FIFO
   ↓
DMA Controller
   ↓
BRAM Input Buffer
   ↓
DFT Core
   ↓
BRAM Result Buffer
   ↓
LED Matrix Writer
   ↓
MAX7219 Controller
   ↓
MAX7219 x4 LED Matrix
```

시스템은 프레임 단위로 동작합니다.

1. 마이크 센서에서 아날로그 음성 신호가 입력됩니다.
2. XADC Reader가 아날로그 신호를 디지털 샘플 데이터로 변환합니다.
3. 샘플 데이터는 AXI Stream/FIFO/DMA 구조를 통해 BRAM Input Buffer에 저장됩니다.
4. DFT Core는 일정 개수의 샘플을 읽어 시간 영역 데이터를 주파수 영역 데이터로 변환합니다.
5. DFT 결과는 magnitude 형태로 정리되어 Result Buffer에 저장됩니다.
6. LED Matrix Writer는 Result Buffer에서 표시용 magnitude 데이터를 읽습니다.
7. 읽은 magnitude 데이터를 32x8 LED bitmap으로 변환합니다.
8. MAX7219 serial interface를 통해 4개의 LED Matrix에 출력합니다.

---

## 4. 전체 모듈 역할

| 모듈 | 역할 |
|---|---|
| Mic Sensor | 외부 음성 신호 입력 |
| XADC Reader | 아날로그 음성 신호를 디지털 샘플 데이터로 변환 |
| AXI Stream / FIFO | XADC 샘플 데이터를 DMA로 전달하기 위한 스트림 버퍼 |
| DMA Controller | 샘플 데이터를 BRAM Buffer로 이동 |
| BRAM Input Buffer | DFT 연산에 사용할 입력 샘플 저장 |
| Main Control Unit | XADC, DMA, DFT, LED Writer의 동작 순서 제어 |
| DFT Core | 시간 영역 샘플 데이터를 주파수 성분으로 변환 |
| BRAM Result Buffer | DFT magnitude 결과 저장 |
| LED Matrix Writer | DFT 결과를 LED Matrix 표시 데이터로 변환 |
| MAX7219 Controller | LED Matrix 초기화 및 row refresh 제어 |
| MAX7219 Frame Sender | DIN/CS/CLK 기반 64-bit serial frame 전송 |

---

## 5. 프레임 처리 시퀀스

Main Control Unit은 전체 시스템을 프레임 단위로 제어합니다.

```text
1. XADC 설정
2. DMA 주소 및 전송 길이 설정
3. DMA 실행
4. DMA 완료 상태 확인
5. DFT Core 설정
6. DFT Core 실행
7. DFT 완료 상태 확인
8. LED Matrix Writer 실행
9. LED 출력 완료 상태 확인
10. Buffer swap 후 다음 frame 처리
```

이 구조를 통해 입력 샘플 수집, 주파수 연산, LED 출력 갱신이 순차적으로 수행됩니다.

---

## 6. 담당 범위: LED Matrix Writer 출력부

본인이 담당한 부분은 **DFT 결과를 LED Matrix에 표시하는 출력부 RTL**입니다.

출력부는 Result Buffer에 저장된 DFT magnitude 데이터를 읽어 LED Matrix 표시용 bitmap으로 변환하고, MAX7219 driver IC에 맞는 serial frame으로 전송합니다.

담당 출력부의 내부 흐름은 다음과 같습니다.

```text
BRAM Result Buffer
   ↓
AXI4 Read Logic
   ↓
Magnitude Buffer
   ↓
LED Data Mapper
   ↓
MAX7219 Controller
   ↓
MAX7219 Frame Sender
   ↓
MAX7219 x4 LED Matrix
```

담당 구현 내용은 다음과 같습니다.

- AXI-Lite 기반 LED Writer 제어 레지스터 구성
- AXI4 Read Master를 통한 DFT magnitude 결과 read
- 8개 magnitude 값을 32x8 LED bitmap으로 변환
- MAX7219 초기화 및 row refresh FSM 설계
- 4개 cascade MAX7219에 대한 64-bit serial frame 전송
- Python 기준 데이터와 실제 LED Matrix 출력 비교 검증
- Basys3 보드 기반 하드웨어 출력 검증

---

## 7. LED Matrix Writer 소스 구조

| 파일 | 역할 |
|---|---|
| top_led_matrix_axi.v | LED Matrix 출력부 최상위 wrapper |
| led_matrix_writer_wrapper_axi_slave.v | AXI-Lite 제어, AXI4 Read, 출력 흐름 제어 |
| axil_slave_reg_if.v | AXI-Lite slave register interface |
| led_data_mapper.v | DFT magnitude → 32x8 LED bitmap 변환 |
| max7219_controller.v | MAX7219 초기화, clear, row refresh FSM |
| max7219_frame_sender.v | 64-bit serial frame을 DIN/CS/CLK로 전송 |
| top_dummy_audio_led_board.v | dummy 입력 기반 보드 출력 테스트용 top |

---

## 8. AXI-Lite 제어 구조

LED Matrix Writer는 Main Control Unit에서 AXI-Lite register를 통해 제어됩니다.

| Address | Register | 설명 |
|---|---|---|
| 0x00 | LED_CTRL | bit[0]: start/update, bit[1]: clear |
| 0x04 | LED_STATUS | bit[0]: done, bit[1]: busy, bit[2]: ready |
| 0x08 | LED_RESULT_ADDR | DFT 결과가 저장된 BRAM byte address |
| 0x0C | LED_BIN_COUNT | 읽어올 magnitude bin 개수 |
| 0x10 | LED_SCALE | LED 높이 조절용 scale shift 값 |
| 0x14 | LED_VALUE | 디버깅용 readback 값 |

제어 흐름은 다음과 같습니다.

```text
Main CU
   ↓ AXI-Lite Write
LED_RESULT_ADDR 설정
   ↓
LED_BIN_COUNT 설정
   ↓
LED_SCALE 설정
   ↓
LED_CTRL start write
   ↓
AXI4 Read로 DFT magnitude read
   ↓
LED Matrix 출력 갱신
   ↓
LED_STATUS done 확인
```

---

## 9. AXI4 Read 기반 Result Buffer 접근

LED Matrix Writer는 DFT Core가 계산한 결과를 직접 입력으로 받는 것이 아니라, BRAM Result Buffer에 저장된 값을 AXI4 Read Master로 읽습니다.

동작 방식은 다음과 같습니다.

1. LED_RESULT_ADDR register에 DFT 결과 시작 주소를 설정합니다.
2. LED_BIN_COUNT register에 읽어올 bin 개수를 설정합니다.
3. start 신호가 들어오면 AXI4 Read FSM이 동작합니다.
4. base address + index x 4 방식으로 32-bit magnitude 값을 순차 read합니다.
5. 읽은 magnitude 값은 내부 read_data_flat buffer에 저장됩니다.
6. 8개 magnitude가 모이면 LED Data Mapper로 전달됩니다.
7. Mapper가 32x8 bitmap을 생성하면 MAX7219 Controller가 출력 전송을 시작합니다.

AXI4 Read FSM은 다음 상태로 구성됩니다.

```text
S_IDLE
   ↓
S_READ_ADDR
   ↓
S_READ_DATA
   ↓
S_NEXT
   ↓
S_DONE
```

---

## 10. DFT Magnitude to LED Bitmap Mapping

### 10.1 led_data_mapper.v

LED Data Mapper는 DFT magnitude 8개를 받아 32x8 LED Matrix bitmap으로 변환하는 조합논리 모듈입니다.

입력 데이터 구조는 다음과 같습니다.

```text
magnitude_flat[31:0]      = bin0
magnitude_flat[63:32]     = bin1
magnitude_flat[95:64]     = bin2
magnitude_flat[127:96]    = bin3
magnitude_flat[159:128]   = bin4
magnitude_flat[191:160]   = bin5
magnitude_flat[223:192]   = bin6
magnitude_flat[255:224]   = bin7
```

출력 데이터 구조는 다음과 같습니다.

```text
row_bitmap_flat[31:0]      = row 0
row_bitmap_flat[63:32]     = row 1
row_bitmap_flat[95:64]     = row 2
row_bitmap_flat[127:96]    = row 3
row_bitmap_flat[159:128]   = row 4
row_bitmap_flat[191:160]   = row 5
row_bitmap_flat[223:192]   = row 6
row_bitmap_flat[255:224]   = row 7
```

표시 방식은 다음과 같습니다.

- 표시용 magnitude bin 개수: 8개
- LED Matrix 전체 크기: 32 columns x 8 rows
- bin 1개당 column 4개 할당
- 8 bin x 4 columns = 32 columns
- magnitude 값에 scale_shift 적용
- scale 적용 후 0~8 범위로 saturation
- height 값에 따라 아래 row부터 LED ON

예시:

```text
bin0 → column 0~3
bin1 → column 4~7
bin2 → column 8~11
bin3 → column 12~15
bin4 → column 16~19
bin5 → column 20~23
bin6 → column 24~27
bin7 → column 28~31
```

---

## 11. MAX7219 출력 제어

### 11.1 max7219_controller.v

MAX7219 Controller는 LED Matrix에 표시할 32x8 bitmap을 받아 MAX7219 driver에 전달하는 FSM입니다.

주요 역할은 다음과 같습니다.

- MAX7219 초기화 명령 전송
- LED Matrix clear
- row 1~8 데이터 순차 refresh
- 각 row의 32-bit bitmap을 64-bit frame으로 변환
- frame_sender에 start pulse 전달
- sender_done을 확인하며 다음 row로 이동

FSM 흐름은 다음과 같습니다.

```text
IDLE
   ↓
INIT_SEND
   ↓
INIT_WAIT
   ↓
CLEAR_SEND
   ↓
CLEAR_WAIT
   ↓
REFRESH_SEND
   ↓
REFRESH_WAIT
   ↓
row 1~8 반복 refresh
```

초기화 명령은 다음 순서로 전송됩니다.

| Register | 설정 | 의미 |
|---|---|---|
| 0x09 | 0x00 | Decode mode off |
| 0x0A | 0x03 | Intensity 설정 |
| 0x0B | 0x07 | Scan limit: 8 rows |
| 0x0C | 0x01 | Shutdown off |
| 0x0F | 0x00 | Display test off |

---

## 12. 64-bit Serial Frame 전송

### 12.1 max7219_frame_sender.v

MAX7219 1개는 16-bit frame을 입력받습니다.

```text
[address 8-bit] + [data 8-bit]
```

본 프로젝트에서는 8x8 LED Matrix 4개를 가로로 cascade 연결했기 때문에, 한 번의 row update마다 총 64-bit frame을 전송합니다.

```text
MAX7219 1개 = 16-bit
MAX7219 4개 = 16-bit x 4 = 64-bit
```

전송 방식은 다음과 같습니다.

```text
CS = 0
   ↓
64-bit frame을 MSB부터 DIN으로 출력
   ↓
CLK rising edge마다 MAX7219가 DIN sampling
   ↓
64-bit 전송 완료
   ↓
CS = 1
   ↓
각 MAX7219가 수신한 16-bit command/data latch
```

frame_sender는 100MHz FPGA clock을 divider로 분주하여 MAX7219용 serial clock을 생성합니다.

---

## 13. 하드웨어 구성

사용한 LED Matrix는 MAX7219 driver가 포함된 8x8 LED Matrix 모듈 4개입니다.

### 13.1 Basys3와 MAX7219 연결

```text
Basys3 FPGA
   ├── DIN  → MAX7219 DIN
   ├── CS   → MAX7219 CS
   ├── CLK  → MAX7219 CLK
   └── GND  → External Power GND와 공통 연결
```

### 13.2 MAX7219 cascade 연결

```text
FPGA DIN → Matrix1 DIN
Matrix1 DOUT → Matrix2 DIN
Matrix2 DOUT → Matrix3 DIN
Matrix3 DOUT → Matrix4 DIN

CS, CLK는 4개 Matrix에 공통 연결
```

### 13.3 전원 구성

LED Matrix는 외부 전원을 사용했습니다.

Basys3 FPGA와 LED Matrix의 기준 전압을 맞추기 위해 GND를 공통으로 연결했습니다. GND가 공통으로 연결되지 않으면 DIN/CS/CLK 신호의 기준 전압이 맞지 않아 LED Matrix 출력이 불안정해질 수 있습니다.

---

## 14. 검증 방법

### 14.1 Python 기준 데이터 비교

LED Matrix에 표시될 32x8 bitmap 형태를 Python으로 먼저 생성했습니다.

이후 Python 기준 출력과 실제 LED Matrix 점등 결과를 비교하여 다음 항목을 검증했습니다.

- bin별 column mapping 정상 여부
- magnitude height에 따른 row 점등 정상 여부
- row/column 방향 정상 여부
- 4개 cascade 모듈의 표시 순서 정상 여부
- 32x8 bitmap과 실제 LED Matrix 출력 일치 여부

### 14.2 Dummy 입력 기반 출력 검증

DFT Core와 완전히 통합하기 전, dummy magnitude 데이터를 사용하여 LED Matrix 출력부를 먼저 검증했습니다.

이를 통해 다음 내용을 확인했습니다.

- led_data_mapper의 bitmap 생성 동작
- MAX7219 초기화 sequence
- 64-bit frame 전송 동작
- 4개 LED Matrix cascade 표시 순서
- DIN/CS/CLK 하드웨어 연결 상태

### 14.3 실제 보드 시연

최종적으로 Basys3 보드에 bitstream을 다운로드하고, MAX7219 LED Matrix 4개를 연결하여 실제 출력 동작을 확인했습니다.

보드 검증 결과:

- MAX7219 초기화 정상 동작
- row clear 정상 동작
- 64-bit serial frame 전송 정상 동작
- 32x8 LED Matrix 출력 정상 동작
- Python 기준 데이터와 실제 LED Matrix 출력 일치 확인

---

## 15. 주요 설계 포인트

### 15.1 전체 시스템 관점

본 프로젝트는 단순 LED 출력 프로젝트가 아니라, 입력 샘플링부터 주파수 연산, 결과 시각화까지 포함한 FPGA 기반 신호 처리 시스템입니다.

```text
Analog Input
   ↓
Digital Sampling
   ↓
Memory Buffering
   ↓
Frequency Analysis
   ↓
Visual Output
```

이를 통해 FPGA 내부에서 데이터가 어떻게 수집되고, 이동하고, 연산되고, 출력되는지를 RTL 구조로 구현했습니다.

### 15.2 출력부 데이터 경로와 제어 경로 분리

LED Matrix Writer는 데이터 경로와 제어 경로를 분리하여 설계했습니다.

```text
제어 경로:
Main CU → AXI-Lite Register → start / clear / status

데이터 경로:
BRAM Result Buffer → AXI4 Read → Magnitude Buffer → LED Bitmap → MAX7219
```

이 구조를 통해 Main CU는 register 기반으로 출력부를 제어하고, 출력부는 내부 FSM을 통해 BRAM read와 LED refresh를 수행합니다.

### 15.3 계층적 모듈 설계

출력부는 기능별로 모듈을 분리했습니다.

| 모듈 | 역할 |
|---|---|
| top_led_matrix_axi | 외부 포트 및 wrapper 연결 |
| led_matrix_writer_wrapper_axi_slave | AXI 제어, BRAM read, 출력 흐름 제어 |
| axil_slave_reg_if | AXI-Lite handshake 처리 |
| led_data_mapper | magnitude → 32x8 bitmap 변환 |
| max7219_controller | MAX7219 초기화, clear, row refresh FSM |
| max7219_frame_sender | 64-bit serial frame 전송 |

모듈을 분리하여 각 기능을 독립적으로 검증할 수 있도록 구성했습니다.

---

## 16. 결과

본 프로젝트를 통해 마이크 입력 음성 신호를 FPGA에서 샘플링하고, DFT 연산을 통해 주파수 성분으로 변환한 뒤, 그 결과를 LED Matrix Equalizer 형태로 시각화하는 시스템을 구현했습니다.

담당한 LED Matrix Writer 출력부에서는 단순히 LED를 켜는 수준이 아니라, AXI 기반 result buffer read, magnitude-to-bitmap mapping, MAX7219 cascade frame 전송까지 포함한 출력 pipeline을 설계했습니다.

최종적으로 Basys3 보드와 4개의 MAX7219 LED Matrix를 연결하여 32x8 LED Matrix 출력 동작을 하드웨어에서 검증했습니다.

---

## 17. 향후 개선 방향

- LED Matrix refresh rate 최적화
- magnitude scaling 방식 개선
- bin별 peak hold 기능 추가
- AXI burst read 적용을 통한 result buffer read 효율 개선
- timing constraint 및 resource utilization 분석 보완
- RGB LED Matrix 기반 다중 색상 Equalizer 확장
