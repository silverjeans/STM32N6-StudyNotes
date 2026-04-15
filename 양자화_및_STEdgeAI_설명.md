# AI 모델 양자화 및 ST Edge AI를 이용한 STM32N6 NPU 배포

## 1. 개요

AI 모델을 STM32N6 보드의 NPU에서 실시간으로 실행하려면 두 단계의 변환이 필요하다.

1단계: 양자화 (Quantization) - 모델을 가볍게 만들기
2단계: ST Edge AI (stedgeai) - NPU가 실행할 수 있는 코드로 변환

전체 흐름:

    원본 AI 모델 (float32)
        |
        | [1단계] ONNX Runtime Quantization (양자화)
        v
    양자화된 모델 (int8)
        |
        | [2단계] ST Edge AI (stedgeai generate)
        v
    NPU 실행 코드 (network.c, network_data.hex)
        |
        | [3단계] STM32CubeIDE 빌드 + CubeProgrammer 플래시
        v
    STM32N6 보드에서 실시간 추론 (42ms)


## 2. 양자화 (Quantization)

### 2.1 양자화란

AI 모델 내부의 숫자를 float32(소수점 32비트)에서 int8(정수 8비트)로 변환하는 과정이다.

    float32: -3.4×10^38 ~ 3.4×10^38 범위, 1개 값 = 4 bytes
    int8:    -128 ~ 127 범위,             1개 값 = 1 byte

모델 크기가 약 4배 줄어들고, 정수 연산이라 속도가 빨라진다.

### 2.2 왜 필요한가

STM32N6의 Neural-ART NPU는 float32 연산을 하드웨어로 처리하지 못한다. float32 모델을 넣으면 모든 레이어가 CPU 소프트웨어 폴백(SW fallback)으로 실행되어 매우 느리거나 크래시가 발생한다.

    float32 모델 → NPU에서 실행 시도 → 전부 SW fallback → 크래시
    int8 모델    → NPU에서 실행 시도 → Conv 등은 HW 가속  → 정상 동작

따라서 int8 양자화는 STM32N6 NPU 배포의 필수 조건이다.

### 2.3 양자화 원리

float32 값을 int8 범위로 선형 매핑한다.

    int8_value = round(float_value / scale) + zero_point
    float_value = (int8_value - zero_point) x scale

예시:
    원본 float32 값: [0.0, 0.5, 1.0, 0.3, 0.8]
    scale = 1.0 / 255 = 0.00392
    zero_point = 0
    양자화 int8 값:   [0, 128, 255, 77, 204]

scale과 zero_point를 정확히 구하는 것이 양자화 품질의 핵심이다.

### 2.4 Calibration (보정)

scale과 zero_point를 구하려면 모델 내부 각 레이어의 activation 값 분포(최솟값, 최댓값)를 알아야 한다. 이를 위해 대표 이미지(calibration 데이터)를 모델에 넣어서 분포를 측정하는 과정을 calibration이라 한다.

    calibration 데이터 500장을 모델에 입력
        → 각 레이어에서 activation 값의 min/max 측정
        → 예: 특정 레이어에서 min=-2.5, max=3.1
        → scale = (3.1 - (-2.5)) / 255 = 0.022
        → 이 scale로 해당 레이어의 값을 int8로 변환

calibration 데이터의 품질이 양자화 정확도를 결정한다.

    랜덤 이미지로 calibration → min/max가 실제와 다름 → scale 부정확 → 출력 폭발
    실제 도메인 이미지로 calibration → min/max가 정확 → scale 정확 → 출력 정상

### 2.5 실제 calibration 결과 비교

본 프로젝트에서 calibration 데이터에 따른 양자화 오차를 측정한 결과:

    | calibration 데이터           | 이미지 수 | float32 대비 오차  |
    |------------------------------|-----------|-------------------|
    | 랜덤 데이터                  | 300장     | 좌표 폭발 (사용 불가) |
    | pupil 데이터 전체 리사이즈   | 300장     | 동작하지만 부정확    |
    | pupil 데이터 bbox 크롭       | 500장     | 2.10px (3.3%)      |
    | GI4E 데이터 iris center 크롭 | 2,472장   | 0.46px (0.7%)      |

실제 사용 환경과 유사한 이미지(눈 클로즈업)로 calibration할수록 오차가 줄어들었다.

### 2.6 QDQ (Quantize-Dequantize) 방식

본 프로젝트에서 사용한 QuantFormat.QDQ 방식은 각 레이어 앞뒤에 양자화/역양자화 노드를 삽입하는 방식이다.

    입력 float32
        → QuantizeLinear (float32 → int8)
        → Conv2D (int8 연산, NPU HW에서 실행)
        → DequantizeLinear (int8 → float32)
        → 다음 레이어로 전달

레이어별로 독립적인 scale을 사용하므로 정밀도가 높다.

### 2.7 사용 도구 및 환경

    도구: ONNX Runtime Quantization (quantize_static 함수)
    실행 환경: WSL2 Ubuntu, Python 가상환경 (~/tf_gpu/)
    입력: float32 ONNX 모델 + calibration 이미지
    출력: int8 양자화 ONNX 모델

양자화 코드 예시:
    from onnxruntime.quantization import quantize_static, QuantFormat, QuantType
    
    quantize_static(
        model_input="iris_landmark_preproc.onnx",
        model_output="iris_landmark_int8.onnx",
        calibration_data_reader=eye_crop_reader,  # 눈 크롭 이미지
        quant_format=QuantFormat.QDQ,
        per_channel=True,
        weight_type=QuantType.QInt8,
        activation_type=QuantType.QInt8,
    )


## 3. ST Edge AI (stedgeai)

### 3.1 ST Edge AI란

STMicroelectronics에서 제공하는 공식 AI 배포 도구로, ONNX/TFLite 등의 AI 모델을 STM32 MCU/MPU에서 실행할 수 있는 C 코드로 변환한다.

설치 경로: C:\ST\STEdgeAI\4.0\

### 3.2 역할

양자화된 int8 ONNX 모델을 입력받아, STM32N6의 Neural-ART NPU(ATON 가속기)에서 실행할 수 있는 최적화된 C 코드와 바이너리를 생성한다.

    입력: iris_landmark_64x64_int8.onnx (양자화된 모델)
    
    [stedgeai generate 실행]
    
    출력 파일들:
        network.c           - 모델 구조 (레이어 연결, 연산 순서)
        stai_network.c/h     - NPU API 래퍼 코드 (init, run, get_output 등)
        network_ecblobs.h    - NPU 하드웨어 설정값 (Epoch Controller 설정)
        network_atonbuf.raw  - 모델 가중치(weight) 바이너리
                                  ↓ arm-none-eabi-objcopy로 변환
                              network_data.hex  - 외부 플래시에 굽는 파일

### 3.3 핵심 기능: 레이어별 HW/SW 분배

stedgeai는 모델의 각 레이어를 분석하여 NPU 하드웨어(EC)에서 실행할 수 있는 것과 CPU 소프트웨어(SW)로 폴백해야 하는 것을 자동 분배한다.

    레이어 타입     | 실행 위치 | 설명
    ----------------|-----------|---------------------------
    Conv2D (81개)   | EC (HW)   | NPU 하드웨어 가속, 빠름
    Add (26개)      | EC (HW)   | NPU 하드웨어 가속
    MaxPool (6개)   | EC (HW)   | NPU 하드웨어 가속
    PReLU (53개)    | SW (CPU)  | NPU가 지원하지 않음, CPU에서 실행
    Reshape (2개)   | SW (CPU)  | 데이터 재배치
    
    EC = Epoch Controller (NPU 하드웨어 실행 엔진)
    SW = Software Fallback (CPU에서 실행)

무거운 Conv 연산(81개)이 전부 NPU HW에서 돌아가므로 42ms 내에 추론이 완료된다.

### 3.4 메모리 풀 설정 (user_neuralart.json)

stedgeai에 전달하는 메모리 풀 설정 파일로, 모델의 가중치와 중간 연산 결과(activation)를 STM32N6의 어느 메모리 영역에 배치할지 정의한다.

    메모리 영역      | 주소         | 크기    | 용도
    -----------------|-------------|---------|---------------------
    cpuRAM2          | 0x34100000  | 1 MB    | activation 버퍼
    npuRAM3~6        | 0x34200000~ | 각 448KB | NPU 연산 중간 버퍼
    hyperRAM (xSPI1) | 0x90000000  | 16 MB   | 큰 가중치 저장
    octoFlash (xSPI2)| 0x70380000  | 61 MB   | 모델 데이터 (읽기 전용)

network_data.hex 파일이 외부 플래시(octoFlash)의 0x70380000 주소에 기록된다.

### 3.5 생성 명령어

    stedgeai generate
        --model iris_landmark_64x64_int8.onnx         (입력 모델)
        --target stm32n6                                (타겟 칩)
        --st-neural-art default@user_neuralart.json    (NPU 메모리/최적화 설정)
        --input-data-type float32                       (NPU 입력 데이터 타입)
        --output-data-type float32                      (NPU 출력 데이터 타입)
        -o st_ai_output                                 (출력 폴더)

input-data-type 옵션:
    uint8   - 카메라 출력(uint8)을 NPU에 직접 전달 (모델이 uint8 입력일 때)
    float32 - float32 버퍼를 NPU에 전달 (본 프로젝트에서 사용, main.c에서 /255 변환)

### 3.6 생성된 코드의 사용법 (main.c에서)

    // 초기화
    stai_runtime_init();
    stai_network_init(network_context);
    
    // 입력 버퍼 주소 얻기
    stai_network_get_inputs(network_context, &nn_in, &n_inputs);
    
    // 입력 데이터를 nn_in 버퍼에 복사 (카메라 uint8 → /255.0 → float32)
    Convert_uint8_to_float32_normalized(camera_buffer, (float*)nn_in, 64*64*3);
    
    // NPU 추론 실행
    stai_network_run(network_context, STAI_MODE_ASYNC);
    
    // 출력 결과 읽기
    stai_network_get_outputs(network_context, nn_out, &n_outputs);
    float *iris_landmarks = (float*)nn_out[1];  // 홍채 5점 좌표


## 4. 전체 배포 파이프라인 요약

    [PC - WSL2 Ubuntu]
        원본 모델: iris_landmark.tflite (MediaPipe, float32)
            ↓
        stedgeai가 변환한 ONNX: iris_landmark_OE_3_3_1.onnx (float32)
            ↓
        ONNX Runtime으로 int8 양자화 (GI4E 눈 크롭 2,472장 calibration)
            ↓
        양자화된 모델: iris_landmark_64x64_int8.onnx
    
    [PC - Windows]
        stedgeai generate 실행
            ↓
        NPU 코드 생성: network.c, stai_network.h, network_data.hex
            ↓
        STM32CubeIDE에서 빌드 → firmware.elf
    
    [STM32N6 DK 보드]
        CubeProgrammer로 network_data.hex를 외부 플래시(0x70380000)에 기록
        CubeIDE에서 firmware를 내부 플래시에 기록 + 실행
            ↓
        카메라 192x192 캡처 → 중앙 크롭 → 64x64 → /255 정규화
            → NPU 추론 (42ms) → 홍채 중심 좌표 출력


## 5. 디지털 리프랙터에 활용 가능한 AI 기능

양자화 + ST Edge AI를 활용하면, PC 없이 STM32N6 보드 하나에서 다양한 AI 기능을 실시간으로 실행할 수 있다. 아래는 디지털 리프랙터에 적용 가능한 AI 기능과 각각에 필요한 데이터를 정리한 것이다.

### 5.1 동공 중심 추적 (현재 구현 완료)

    기능: 카메라 영상에서 동공(홍채) 중심 좌표를 실시간 추출
    용도: 측정 렌즈를 환자 시축에 정렬하여 굴절도 측정 정확도 향상
    모델: MediaPipe Iris Landmark (int8 양자화)
    추론 시간: 42ms (약 24fps)
    
    필요 데이터:
        학습용: 눈 클로즈업 이미지 + 홍채 중심 (x, y) 좌표 라벨
                - GI4E 데이터셋: 1,237장, iris center 좌표 포함
                - LPW 데이터셋: 66개 영상, pupil center 좌표 포함
        양자화 calibration용: 눈 클로즈업 이미지 500~2,000장 (라벨 불필요)
                - 실제 사용 환경(IR 카메라, 눈 클로즈업)과 유사할수록 좋음

### 5.2 각막 반사점 검출

    기능: IR LED가 각막에 반사된 Purkinje image의 위치를 AI로 정밀 검출
    용도: 각막 곡률 측정(케라토미터). 기존 영상처리 방식은 반사 노이즈에 약함
    모델: 경량 landmark regression 모델 (MobileNet 기반)
    예상 추론 시간: 10~20ms
    
    필요 데이터:
        학습용: IR 카메라로 촬영한 각막 반사 이미지 + 반사점 좌표 (x, y) 라벨
                - 최소 1,000장 이상
                - 다양한 각막 곡률, 눈 위치, 조명 조건 포함
                - 리프랙터의 IR LED 패턴으로 직접 촬영한 데이터가 가장 이상적
        양자화 calibration용: 같은 조건의 각막 반사 이미지 300~500장

### 5.3 눈 상태 분류 (깜빡임 감지)

    기능: 눈 뜸 / 감음 / 반쯤 뜸 / 깜빡이는 중 자동 분류
    용도: 측정 중 깜빡임 구간 자동 제외, "눈을 뜨세요" 안내 자동 트리거
    모델: 경량 분류 모델 (MobileNetV2 또는 커스텀 CNN)
    예상 추론 시간: 5~10ms
    
    필요 데이터:
        학습용: 눈 이미지 + 상태 분류 라벨 (open/closed/half/blink)
                - 클래스당 500장 이상 (총 2,000장+)
                - 다양한 인종, 나이, 눈 크기 포함
                - 공개 데이터셋: CEW (Closed Eyes in the Wild), MRL Eye Dataset
        양자화 calibration용: 각 상태별 이미지 100장씩

### 5.4 안구 질환 스크리닝

    기능: 전안부 이미지에서 백내장, 각막혼탁, 익상편 등 이상 소견 감지
    용도: 굴절 검사와 동시에 질환 스크리닝하여 조기 발견
    모델: 이미지 분류 또는 세그멘테이션 모델
    예상 추론 시간: 50~100ms
    
    필요 데이터:
        학습용: 전안부 촬영 이미지 + 질환 분류 라벨 또는 병변 영역 마스크
                - 정상/백내장/각막혼탁/익상편 등 클래스별 1,000장+
                - 의료 전문가의 라벨링 필수 (의사 또는 안과 기사)
                - 공개 데이터셋: ODIR (Ocular Disease Intelligent Recognition)
                - 주의: 의료 데이터는 IRB(연구윤리) 승인 필요할 수 있음
        양자화 calibration용: 전안부 이미지 500장

### 5.5 노이즈 제거 / 영상 개선

    기능: IR 카메라 영상의 노이즈 제거, 콘트라스트 향상
    용도: 저조도/반사 환경에서도 선명한 동공 영상 확보 → 후단 AI 정확도 향상
    모델: 경량 denoising 모델 (UNet-lite 등)
    예상 추론 시간: 20~40ms
    
    필요 데이터:
        학습용: 노이즈 있는 이미지 + 깨끗한 이미지 쌍 (paired data)
                - 같은 장면을 노이즈 있는/없는 조건으로 촬영
                - 또는 깨끗한 이미지에 인위적 노이즈 추가하여 생성
                - 500~1,000쌍
        양자화 calibration용: 노이즈 있는 이미지 300장

### 5.6 안경 렌즈 도수 자동 추천

    기능: 측정 데이터 + 환자 정보 → 최적 처방 도수 추천
    용도: 검사자 경험에 의존하는 처방 과정을 AI가 보조
    모델: 테이블형 데이터 모델 (MLP 또는 경량 추론 모델)
    예상 추론 시간: 1ms 이내
    
    필요 데이터:
        학습용: 과거 처방 데이터 (구조화된 수치 데이터)
                - 입력: 자동굴절검사 결과(S, C, Axis), 나이, 주소증, 동공크기
                - 출력: 최종 처방 도수(S, C, Axis), ADD(가입도)
                - 최소 5,000건 이상의 처방 기록
                - 안과 또는 안경원의 과거 처방 DB에서 추출
        양자화 calibration용: 입력 데이터 100건 (수치 데이터라 양자화 영향 작음)

### 5.7 다중 모델 파이프라인

    기능: NPU에서 여러 AI 모델을 순차 실행하여 복합 기능 구현
    
    예시 파이프라인:
        카메라 프레임 입력
            → [모델1] 눈 상태 분류 (5ms)    → 눈 감으면 측정 중지
            → [모델2] 동공 정밀 추적 (42ms)  → 동공 좌표 추출
            → [모델3] 주시 안정성 판단 (5ms)  → 안정 시 측정 트리거
            → 총 52ms (약 19fps)
    
    STM32N6 NPU는 모델별 별도 context를 사용하여 순차 실행을 지원한다.
    stedgeai generate 시 --network-name 옵션으로 각 모델에 고유 이름을 부여한다.


## 6. 필요 데이터 총 정리

    | AI 기능              | 학습 데이터                          | 수량      | 라벨 형태           | 공개 데이터셋           |
    |----------------------|--------------------------------------|-----------|---------------------|------------------------|
    | 동공 추적            | 눈 클로즈업 이미지                   | 1,000장+  | 중심 좌표 (x,y)     | GI4E, LPW, ExCuSe     |
    | 각막 반사점 검출     | IR 각막 반사 이미지                  | 1,000장+  | 반사점 좌표 (x,y)   | 자체 촬영 필요          |
    | 눈 상태 분류         | 눈 이미지                            | 2,000장+  | 클래스 (open/closed) | CEW, MRL Eye Dataset   |
    | 안구 질환 스크리닝   | 전안부 이미지                        | 5,000장+  | 질환 분류/마스크     | ODIR (IRB 필요 가능)   |
    | 노이즈 제거          | 노이즈/깨끗한 이미지 쌍              | 1,000쌍   | 깨끗한 이미지        | 인위적 생성 가능        |
    | 도수 자동 추천       | 과거 처방 기록 (수치)                | 5,000건+  | 처방 도수            | 사내 처방 DB           |
    | 양자화 calibration   | 각 기능별 실사용 환경 이미지         | 300~500장 | 불필요               | 위 데이터에서 추출      |

    참고: 양자화 calibration에는 라벨이 필요 없다. 
    모델 학습에만 라벨이 필요하고, calibration은 입력 이미지만 있으면 된다.
    실제 사용 환경(IR 카메라, 리프랙터 조명)과 유사한 이미지일수록 양자화 품질이 높아진다.
