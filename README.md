Basys3 FPGA 기반 DFT Equalizer 출력부 설계
1. 프로젝트 개요

본 프로젝트는 마이크로 입력된 아날로그 음성 신호를 FPGA 내부에서 샘플링하고, DFT 연산을 통해 주파수 성분으로 변환한 뒤, 그 결과를 32x8 LED Matrix Equalizer 형태로 시각화하는 RTL 설계 프로젝트입니다.

전체 시스템은 Mic Sensor → XADC → AXI 기반 데이터 이동 → Buffer → DFT Core → LED Matrix Writer → MAX7219 LED Matrix 흐름으로 구성됩니다.

본인은 프로젝트 중 DFT 결과 데이터를 32x8 LED Matrix로 출력하는 LED Matrix Writer 출력부 RTL 설계 및 하드웨어 검증을 담당했습니다.

2. 개발 환경
항목	내용
FPGA Board	Basys3 FPGA Board
FPGA Device	Xilinx Artix-7
Language	Verilog HDL
Tool	Vivado
Input	Mic Sensor, XADC
Processing	AXI DMA / Buffer / DFT Core
Output	MAX7219 기반 8x8 LED Matrix x 4
Display Size	32 columns x 8 rows
Interface	AXI-Lite, AXI4 Read, MAX7219 Serial Interface
3. 주요 기능
마이크 입력 음성 신호를 XADC를 통해 디지털 샘플 데이터로 변환
AXI 기반 데이터 이동 구조를 통해 샘플 데이터를 Buffer 및 DFT Core로 전달
DFT Core에서 시간 영역 데이터를 주파수 성분으로 변환
DFT magnitude 결과 중 표시용 8개 bin 데이터를 LED 높이 값으로 변환
8개 magnitude 값을 32x8 LED bitmap으로 매핑
4개의 MAX7219 LED Matrix 모듈을 cascade 구조로 연결하여 32x8 Equalizer 출력 구현
Python 기준 데이터와 실제 LED Matrix 출력을 비교하여 bitmap mapping 검증
4. 전체 시스템 구조

Mic Sensor
↓
XADC Reader
↓
AXI Stream / DMA / Buffer
↓
DFT Core
↓
DFT Magnitude Result Buffer
↓
LED Matrix Writer
↓
MAX7219 Controller
↓
MAX7219 x4 LED Matrix

구분	역할
입력부	Mic Sensor의 아날로그 음성 신호를 XADC로 샘플링
데이터 이동부	AXI Stream, AXI DMA, Buffer를 이용해 샘플 데이터 이동
연산부	DFT Core에서 주파수 성분 및 magnitude 계산
출력부	DFT 결과를 LED Matrix bitmap으로 변환 후 MAX7219로 출력
5. 담당 범위

본인이 담당한 범위는 LED Matrix 출력부입니다.

DFT Core에서 계산된 magnitude 결과를 받아 LED Matrix에 표시하기 위해 다음 구조를 설계했습니다.

DFT Magnitude Data
↓
AXI Read Logic
↓
LED Data Mapper
↓
MAX7219 Controller
↓
MAX7219 Frame Sender
↓
32x8 LED Matrix

담당 구현 내용은 다음과 같습니다.

AXI-Lite 기반 제어 레지스터 구성
AXI4 Read Master를 통한 DFT 결과값 read
8개 magnitude 데이터를 32x8 LED bitmap으로 변환
MAX7219 초기화 및 row refresh FSM 설계
4개 cascade MAX7219에 대한 64-bit serial frame 전송
실제 Basys3 보드와 LED Matrix 하드웨어 연결 및 출력 검증
6. LED Matrix Writer 구조
6.1 top_led_matrix_axi.v

FPGA 외부 인터페이스와 LED Matrix Writer wrapper를 연결하는 최상위 출력부 모듈입니다.

주요 역할:

AXI-Lite Slave 제어 채널 연결
AXI4 Read Master 채널 연결
MAX7219 출력 핀 연결
Basys3 clock/reset 연결

출력 핀:

max7219_din
max7219_cs
max7219_clk
6.2 led_matrix_writer_wrapper_axi_slave.v

LED Matrix 출력부의 중심 모듈입니다.

주요 역할:

AXI-Lite register write/read 처리
DFT 결과가 저장된 BRAM 주소 설정
AXI4 Read Master로 magnitude 데이터 읽기
읽은 magnitude 데이터를 내부 buffer에 저장
LED Data Mapper로 bitmap 변환
MAX7219 Controller 시작 신호 생성
7. AXI-Lite Register Map
Address	Register	설명
0x00	LED_CTRL	bit[0]: start/update, bit[1]: clear
0x04	LED_STATUS	bit[0]: done, bit[1]: busy, bit[2]: ready
0x08	LED_RESULT_ADDR	DFT 결과가 저장된 BRAM byte address
0x0C	LED_BIN_COUNT	읽어올 magnitude bin 개수
0x10	LED_SCALE	LED 높이 조절용 scale shift 값
0x14	LED_VALUE	디버깅용 readback 값

제어 흐름:

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
LED Matrix 출력
↓
LED_STATUS done 확인

8. DFT Magnitude to LED Bitmap Mapping
led_data_mapper.v

DFT 결과 magnitude 8개를 32x8 LED Matrix bitmap으로 변환하는 조합논리 모듈입니다.

입력 데이터 구조:

magnitude_flat[31:0] = bin0
magnitude_flat[63:32] = bin1
magnitude_flat[95:64] = bin2
magnitude_flat[127:96] = bin3
magnitude_flat[159:128] = bin4
magnitude_flat[191:160] = bin5
magnitude_flat[223:192] = bin6
magnitude_flat[255:224] = bin7

출력 데이터 구조:

row_bitmap_flat[31:0] = row 0
row_bitmap_flat[63:32] = row 1
row_bitmap_flat[95:64] = row 2
row_bitmap_flat[127:96] = row 3
row_bitmap_flat[159:128] = row 4
row_bitmap_flat[191:160] = row 5
row_bitmap_flat[223:192] = row 6
row_bitmap_flat[255:224] = row 7

표시 방식:

총 8개 bin 사용
bin 1개당 LED column 4개 할당
8 bin x 4 columns = 32 columns
magnitude 값은 scale_shift를 적용해 0~8 높이로 제한
height 값에 따라 아래 row부터 LED ON
9. MAX7219 Controller
max7219_controller.v

32x8 bitmap 데이터를 MAX7219 LED Matrix에 표시하기 위한 제어 FSM입니다.

주요 역할:

MAX7219 초기화 명령 전송
전체 row clear
row 1~8 데이터 순차 refresh
각 row 데이터를 64-bit frame으로 구성
frame_sender를 통해 DIN / CS / CLK 출력

FSM 흐름:

IDLE
↓
INIT_SEND / INIT_WAIT
↓
CLEAR_SEND / CLEAR_WAIT
↓
REFRESH_SEND / REFRESH_WAIT
↓
row 1~8 반복 갱신

초기화 명령:

Register	설정
0x09	Decode mode off
0x0A	Intensity 설정
0x0B	Scan limit 8 rows
0x0C	Shutdown off
0x0F	Display test off
10. MAX7219 Frame Sender
max7219_frame_sender.v

MAX7219 4개에 전달할 64-bit serial frame을 실제 DIN / CS / CLK 신호로 변환하는 모듈입니다.

MAX7219 1개는 다음과 같은 16-bit frame을 받습니다.

[address 8-bit] + [data 8-bit]

본 프로젝트에서는 MAX7219 LED Matrix 4개를 cascade로 연결했기 때문에, 한 번의 row update마다 총 64-bit frame을 전송합니다.

MAX7219 1개 = 16-bit
MAX7219 4개 = 16-bit x 4 = 64-bit

전송 방식:

CS = 0
↓
64-bit data를 MSB부터 DIN으로 출력
↓
CLK rising edge마다 MAX7219가 DIN sampling
↓
64-bit 전송 완료
↓
CS = 1
↓
각 MAX7219가 수신한 16-bit command/data latch

11. 하드웨어 구성

사용한 LED Matrix는 MAX7219 driver가 포함된 8x8 LED Matrix 모듈 4개입니다.

연결 구조:

Basys3 DIN → MAX7219 DIN
Basys3 CS → MAX7219 CS
Basys3 CLK → MAX7219 CLK
Basys3 GND → External Power GND와 공통 연결

MAX7219 Matrix cascade 연결:

FPGA DIN → Matrix1 DIN
Matrix1 DOUT → Matrix2 DIN
Matrix2 DOUT → Matrix3 DIN
Matrix3 DOUT → Matrix4 DIN

CS, CLK는 4개 Matrix에 공통 연결됩니다.

전원 구성:

LED Matrix는 외부 전원 사용
Basys3와 LED Matrix 전원부 GND 공통 연결
GND 공통이 되지 않으면 DIN / CS / CLK 기준 전압이 맞지 않아 출력이 불안정해질 수 있음
12. 검증 방법
12.1 Python 기준 데이터 비교

LED Matrix에 표시될 32x8 bitmap 형태를 Python으로 먼저 생성했습니다.

이후 Python 기준 출력과 실제 LED Matrix 점등 결과를 비교하여 다음을 확인했습니다.

bin별 column mapping 정상 여부
height 값에 따른 row 점등 정상 여부
좌우 방향 및 row/column mapping 정상 여부
4개 cascade 모듈의 표시 순서 정상 여부
12.2 실제 보드 검증

Basys3 보드에 bitstream을 다운로드한 뒤, MAX7219 LED Matrix 4개를 연결하여 실제 출력 동작을 확인했습니다.

검증 내용:

MAX7219 초기화 정상 동작
row clear 정상 동작
64-bit frame 전송 정상 동작
32x8 bitmap 출력 정상 동작
Python 기준 데이터와 실제 LED Matrix 출력 일치 확인
13. 주요 설계 포인트
13.1 AXI 기반 출력부 제어

LED Matrix Writer는 단순 GPIO 출력이 아니라 AXI-Lite register와 AXI4 Read Master 구조를 포함합니다.

AXI-Lite: start, clear, status, result address, scale 설정
AXI4 Read: BRAM에 저장된 DFT magnitude 결과 읽기
내부 FSM: read request, read data capture, done pulse, display start 제어
13.2 데이터 경로와 제어 경로 분리

출력부는 데이터 경로와 제어 경로를 분리해 설계했습니다.

제어 경로:

Main CU → AXI-Lite Register → start / clear / status

데이터 경로:

BRAM Result Buffer → AXI4 Read → magnitude buffer → LED bitmap → MAX7219

13.3 계층적 모듈 설계
모듈	역할
top_led_matrix_axi	외부 포트 및 wrapper 연결
led_matrix_writer_wrapper_axi_slave	AXI 제어, BRAM read, 전체 출력 흐름 제어
led_data_mapper	magnitude → 32x8 bitmap 변환
max7219_controller	MAX7219 초기화, clear, row refresh FSM
max7219_frame_sender	64-bit serial frame 전송
14. 결과

본 프로젝트를 통해 마이크 입력 기반 주파수 분석 결과를 FPGA 내부 RTL 흐름으로 처리하고, LED Matrix Equalizer 형태로 시각화하는 시스템을 구현했습니다.

특히 담당한 LED Matrix 출력부에서는 DFT magnitude 데이터를 단순 표시값으로 사용하는 것이 아니라, AXI 기반 result buffer read, bitmap mapping, MAX7219 cascade frame 전송까지 포함한 출력 pipeline을 설계했습니다.

최종적으로 Basys3 보드와 4개의 MAX7219 LED Matrix를 연결하여 32x8 LED Matrix 출력 동작을 하드웨어에서 검증했습니다.

15. 향후 개선 방향
LED Matrix refresh rate 최적화
magnitude scaling 방식 개선
bin별 peak hold 기능 추가
주파수 대역별 색상 표현이 가능한 RGB Matrix 확장
AXI burst read 적용을 통한 result buffer read 효율 개선
timing constraint 및 resource utilization 분석 보완
