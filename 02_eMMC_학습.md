# eMMC 학습 노트

> 순수 eMMC 기술 학습용. 팀장님 적용사항은 `03_팀장_eMMC_적용사항.md` 참고

---

## 1. NAND Flash 기초 (eMMC를 이해하기 위한 선행 지식)

### 1-1. 플래시 메모리 종류

```
비휘발성 메모리
├── NOR Flash  ← 코드 실행 가능 (XIP), 읽기 빠름, 쓰기 느림, 비쌈
│   └── 용도: 부트로더, 펌웨어 (STM32N6의 xSPI NOR Flash)
│
└── NAND Flash ← 대용량, 쓰기 빠름, 읽기는 페이지 단위, 저렴
    ├── SLC (Single Level Cell) ← 1bit/cell, 빠르고 수명 길고 비쌈
    ├── MLC (Multi Level Cell)  ← 2bit/cell, 균형
    ├── TLC (Triple Level Cell) ← 3bit/cell, 대용량, 수명 짧음
    └── QLC (Quad Level Cell)   ← 4bit/cell, 최대 용량, 수명 최단
```

### 1-2. NAND Flash 물리 구조

```
NAND Flash 칩
├── Die (다이)
│   ├── Plane (플레인) 0
│   │   ├── Block 0          ← 삭제(Erase) 최소 단위
│   │   │   ├── Page 0       ← 읽기/쓰기 최소 단위
│   │   │   │   ├── Data area (2KB~16KB)
│   │   │   │   └── Spare area (64~512B) ← ECC, 메타데이터
│   │   │   ├── Page 1
│   │   │   ├── ...
│   │   │   └── Page 63 (또는 127, 255)
│   │   ├── Block 1
│   │   └── ...
│   └── Plane 1
└── (추가 Die)
```

핵심 포인트:
- **읽기/쓰기**: Page 단위 (4KB~16KB)
- **삭제**: Block 단위 (수백 KB ~ 수 MB)
- **쓰기 전에 반드시 삭제 필요** (1→0만 가능, 0→1은 삭제로만 가능)

### 1-3. NAND Flash의 3대 문제

| 문제 | 설명 | 해결책 |
|------|------|--------|
| **Bad Block** | 공장 출하 시 또는 사용 중 불량 블록 발생 | Bad Block Table (BBT) 관리 |
| **Wear Out** | 각 블록의 P/E cycle 제한 (SLC: 10만회, TLC: 1천회) | Wear Leveling (균등 분배) |
| **Bit Flip** | 읽기/쓰기 시 비트 반전 발생 가능 | ECC (Error Correction Code) |

**이 3가지를 직접 관리하면 = Raw NAND**
**컨트롤러가 자동 관리하면 = eMMC (또는 SSD)**

---

## 2. eMMC란?

**eMMC (embedded Multi-Media Card)** = NAND Flash + 컨트롤러를 하나의 BGA 패키지에 통합

```
┌─────────────────────────────────┐
│          eMMC BGA 패키지         │
│                                 │
│  ┌──────────┐  ┌──────────────┐ │
│  │  NAND    │  │  컨트롤러     │ │
│  │  Flash   │  │              │ │
│  │          │  │ ● ECC 처리   │ │
│  │  실제    │  │ ● Wear Level │ │
│  │  저장소   │  │ ● Bad Block  │ │
│  │          │  │ ● FTL        │ │
│  │  4~128GB │  │ ● Cache      │ │
│  └──────────┘  └──────────────┘ │
│                                 │
│    CLK  CMD  DAT0~DAT7  VCC     │
└─────────────┬───────────────────┘
              │
        호스트 (SoC/MCU)
```

### 왜 eMMC를 쓰는가?

| 비교 | Raw NAND | eMMC |
|------|----------|------|
| FTL (Flash Translation Layer) | 호스트 SW에서 구현 | 내장 |
| Bad Block 관리 | 호스트가 직접 | 내장 |
| Wear Leveling | 호스트가 직접 | 내장 |
| ECC | 호스트가 직접 | 내장 |
| 개발 난이도 | 매우 높음 | 낮음 (블록 디바이스처럼 사용) |
| 비용 | NAND 자체는 저렴 | 컨트롤러 포함으로 약간 비쌈 |
| 호스트 인터페이스 | NAND 전용 (8-bit parallel) | MMC 프로토콜 (범용) |

> **임베디드에서 eMMC를 쓰는 이유**: 호스트가 단순히 "주소 X에 써줘"만 하면 됨. NAND의 복잡한 관리는 eMMC 컨트롤러가 전부 처리.

### SD 카드 vs eMMC vs SSD 비교

| 항목 | SD 카드 | eMMC | SSD |
|------|---------|------|-----|
| 형태 | 탈착식 카드 | BGA 납땜 | M.2/SATA 모듈 |
| 인터페이스 | SD (4-bit) | MMC (4/8-bit) | NVMe/SATA |
| 순차 읽기 | ~100 MB/s | ~250 MB/s | ~3500 MB/s |
| 순차 쓰기 | ~60 MB/s | ~125 MB/s | ~3000 MB/s |
| 랜덤 4K IOPS | ~2K | ~10K | ~500K |
| 내구성 | 낮음 | 중간 | 높음 |
| 용도 | 카메라, 라즈베리파이 | 임베디드, 스마트폰 | PC, 서버 |
| 크기 | 15x11mm (microSD) | 11.5x13mm (BGA) | 22x80mm (M.2) |

---

## 3. eMMC 버전 역사

| eMMC 버전 | 연도 | EXT_CSD Rev | 최대 속도 | 핵심 추가 기능 |
|-----------|------|-------------|----------|---------------|
| 4.3 | 2007 | 3 | 26 MB/s | DDR 모드 |
| 4.4 | 2009 | 5 | 52 MB/s (DDR) | 8-bit DDR |
| 4.41 | 2010 | 5 | 52 MB/s | Boot partition, RPMB |
| 4.5 | 2012 | 6 | 200 MB/s | **HS200**, TRIM, Sanitize |
| 4.51 | 2013 | 7 | 200 MB/s | BKOPS, HPI, Cache |
| 5.0 | 2014 | 7 | 200 MB/s | Field firmware update |
| **5.1** | **2015** | **8** | **400 MB/s** | **HS400, CMDQ, Cache Barrier** |
| 5.1A | 2016 | 9 | 400 MB/s | 보안 강화, Secure Write Protect |

---

## 4. eMMC 하드웨어 인터페이스

### 4-1. 핀 구성

| 핀 | 방향 | 설명 |
|----|------|------|
| **CLK** | 호스트 → eMMC | 클럭 신호 (동기화 기준) |
| **CMD** | 양방향 | 명령 전송(호스트→) / 응답 수신(→호스트) |
| **DAT0~DAT7** | 양방향 | 데이터 전송 (1/4/8-bit 선택) |
| **DS** | eMMC → 호스트 | Data Strobe (HS400 전용) |
| **RST_n** | 호스트 → eMMC | 하드웨어 리셋 (선택사항) |
| **VCC** | 전원 | 3.3V (NAND Flash 전원) |
| **VCCQ** | 전원 | 1.8V 또는 3.3V (I/O 인터페이스 전원) |

### 4-2. 버스 모드

```
1-bit:   CLK ─── CMD ─── DAT0                       (초기화)
4-bit:   CLK ─── CMD ─── DAT0 DAT1 DAT2 DAT3        (SD 호환)
8-bit:   CLK ─── CMD ─── DAT0 DAT1 ... DAT6 DAT7    (eMMC 전용)
```

### 4-3. 속도 모드 상세

| 모드 | 클럭 | 전압 | 데이터율 | 버스 폭 | 최대 속도 | 특징 |
|------|------|------|---------|---------|----------|------|
| Legacy | 0~26 MHz | 3.3V | SDR | 8-bit | 26 MB/s | 초기화 모드 |
| High Speed SDR | 0~52 MHz | 3.3V | SDR | 8-bit | 52 MB/s | 기본 |
| High Speed DDR | 0~52 MHz | 3.3/1.8V | DDR | 8-bit | 104 MB/s | 양쪽 엣지 샘플링 |
| **HS200** | 0~200 MHz | **1.8V** | SDR | 8-bit | 200 MB/s | 튜닝 필수 |
| **HS400** | 0~200 MHz | **1.8V** | DDR | 8-bit | 400 MB/s | DS핀 사용, 최고 속도 |

> **SDR vs DDR**: SDR은 클럭 상승 엣지에서만 데이터 전송, DDR은 상승+하강 엣지 모두 → 같은 클럭에서 2배 속도

### 4-4. HS200/HS400 전환 과정

```
초기화 (Legacy 400KHz)
  ↓
CMD6 (SWITCH) → HS Timing 설정
  ↓
클럭 올리기 → 52MHz
  ↓
CMD6 → HS200 Timing 설정
  ↓
클럭 올리기 → 200MHz
  ↓
CMD21 (SEND_TUNING_BLOCK) → 튜닝 실행 ★
  ↓  (최적 샘플링 포인트 찾기)
HS200 동작 중
  ↓ (HS400으로 가려면)
CMD6 → High Speed 모드로 복귀
  ↓
CMD6 → DDR 모드 전환 + HS400 Timing
  ↓
CMD6 → DS핀 드라이버 강도 설정
  ↓
클럭 올리기 → 200MHz (DDR)
  ↓
HS400 동작 (400 MB/s)
```

> **튜닝(Tuning)**: HS200에서 200MHz로 동작할 때 신호 지연이 생김. 호스트가 여러 딜레이 값을 시도하며 최적의 데이터 캡처 타이밍을 찾는 과정. 반드시 필요.

---

## 5. eMMC 내부 구조

### 5-1. 파티션 구조

```
┌─────────────────────────────────────────────────┐
│                    eMMC                          │
│                                                 │
│  ┌───────────────┐  ┌───────────────┐           │
│  │  Boot Area 1  │  │  Boot Area 2  │  (각 4MB) │
│  │  (부트로더)    │  │  (백업 부트)   │           │
│  └───────────────┘  └───────────────┘           │
│                                                 │
│  ┌──────────────────────────────────┐           │
│  │         RPMB (Replay            │  (보안용)  │
│  │    Protected Memory Block)      │           │
│  └──────────────────────────────────┘           │
│                                                 │
│  ┌──────────────────────────────────┐           │
│  │      User Data Area             │  (메인)    │
│  │  ┌────────┬────────┬────────┐   │           │
│  │  │ GPP 1  │ GPP 2  │ GPP 3  │   │ (선택)    │
│  │  └────────┴────────┴────────┘   │           │
│  └──────────────────────────────────┘           │
└─────────────────────────────────────────────────┘
```

| 파티션 | 용도 | 크기 | 접근 방식 |
|--------|------|------|----------|
| **Boot Area 1/2** | 부트로더 (SPL, U-Boot) | 각 최대 4MB | PARTITION_ACCESS 레지스터로 전환 |
| **RPMB** | 보안 키, DRM, 인증 데이터 | 최대 16MB | 인증 키 필요 (HMAC-SHA256) |
| **User Data Area** | OS, 파일시스템, 사용자 데이터 | 나머지 전체 | 기본 접근 영역 |
| **GPP 1~4** | 사용자 정의 파티션 | 설정 가능 | 선택적, 한번 설정하면 변경 불가 |

### 5-2. FTL (Flash Translation Layer)

eMMC 컨트롤러 내부의 핵심 소프트웨어:

```
호스트가 보는 것:             eMMC 내부 실제:
┌──────────────┐            ┌──────────────┐
│ LBA 0        │  ──FTL──→ │ Block 5, Page 3  │
│ LBA 1        │  ──FTL──→ │ Block 12, Page 0 │
│ LBA 2        │  ──FTL──→ │ Block 5, Page 7  │
│ ...          │            │ ...              │
│ LBA N        │  ──FTL──→ │ Block 23, Page 1 │
└──────────────┘            └──────────────┘

LBA = Logical Block Address (호스트가 사용하는 논리 주소)
PBA = Physical Block Address (실제 NAND 위치)
```

FTL이 하는 일:
1. **주소 변환**: LBA → PBA 매핑 테이블 관리
2. **Wear Leveling**: 쓰기를 여러 블록에 분산
3. **Garbage Collection**: 유효 페이지를 새 블록에 모으고 옛 블록 삭제
4. **Bad Block Management**: 불량 블록 회피 및 교체 블록 할당

### 5-3. Garbage Collection (GC) 상세

NAND는 "덮어쓰기" 불가. 삭제 후 쓰기만 가능. 그래서 GC가 필요:

```
[쓰기 전]                      [쓰기 후 (업데이트)]
Block A:                       Block A:
┌─────────────┐                ┌─────────────┐
│ Page 0: 유효 │                │ Page 0: 유효  │
│ Page 1: 유효 │ ← LBA 5 업데이트 │ Page 1: 무효★ │ (옛 데이터)
│ Page 2: 유효 │                │ Page 2: 유효  │
│ Page 3: 빈칸 │                │ Page 3: 유효★ │ (LBA 5 새 데이터)
└─────────────┘                └─────────────┘

[GC 실행]
1. 새 Block B 할당
2. Block A의 유효 페이지만 Block B로 복사
3. Block A 전체 삭제 (Erase)
4. Block A를 빈 블록 풀에 반환

Block B (GC 후):               Block A (삭제됨):
┌─────────────┐                ┌─────────────┐
│ Page 0: 유효 │                │ (빈 블록)     │
│ Page 1: 유효 │                │              │
│ Page 2: 유효 │                │              │
│ Page 3: 빈칸 │                │              │
└─────────────┘                └─────────────┘
```

> GC는 **Write Amplification(쓰기 증폭)**을 발생시킴: 호스트가 1페이지만 썼는데 내부적으로는 블록 전체를 읽고+쓰고+삭제. 이것이 eMMC가 느려지는 주된 원인이며, BKOPS가 이 문제를 해결.

---

## 6. eMMC 레지스터

### 6-1. 핵심 레지스터 3개

| 레지스터 | 크기 | 읽기 명령 | 주요 정보 |
|----------|------|----------|----------|
| **CID** | 128-bit | CMD2, CMD10 | 제조사 ID, 제품명, 시리얼 번호, 제조일 |
| **CSD** | 128-bit | CMD9 | 용량, 최대 클럭, 읽기/쓰기 블록 크기 |
| **EXT_CSD** | 512-byte | CMD8 | eMMC 확장 설정 전부 (버전, 기능, 수명 등) |

### 6-2. CID 레지스터 필드

| 필드 | 비트 | 설명 | 예시 |
|------|------|------|------|
| MID | [127:120] | 제조사 ID | 0x15 = Samsung, 0x45 = SanDisk |
| CBX | [113:112] | 카드/BGA 타입 | 0=카드, 1=BGA |
| OID | [111:104] | OEM ID | |
| PNM | [103:56] | 제품명 (6자) | "PSE2AO" |
| PRV | [55:48] | 제품 리비전 | |
| PSN | [47:16] | 시리얼 번호 | |
| MDT | [15:8] | 제조일 (년/월) | |

### 6-3. EXT_CSD 중요 필드

```
리눅스에서 읽기: cat /sys/kernel/debug/mmc2/mmc2:0001/ext_csd
U-Boot에서 읽기: mmc extcsd read 2
```

| 바이트 | 필드명 | R/W | 설명 |
|--------|--------|-----|------|
| 192 | EXT_CSD_REV | R | eMMC 버전 (5→4.4, 6→4.5, 7→5.0, 8→5.1) |
| 196 | DEVICE_TYPE | R | 지원 속도 모드 비트맵 |
| 179 | PARTITION_CONFIG | R/W | 부팅 파티션 설정 |
| 177 | BOOT_BUS_CONDITIONS | R/W | 부트 버스 폭/속도 |
| 175 | ERASE_GROUP_DEF | R/W | 삭제 그룹 정의 |
| 163 | BKOPS_EN | R/W | BKOPS 활성화 (0=Off, 1=On) |
| 246 | BKOPS_STATUS | R | BKOPS 긴급도 (0~3) |
| 253 | DATA_SECTOR_SIZE | R | 섹터 크기 (0=512B, 1=4KB) |
| 267 | PRE_EOL_INFO | R | 수명 소진도 (1=정상, 2=경고, 3=긴급) |
| 268 | DEVICE_LIFE_TIME_EST_A | R | Type A (SLC) 수명 |
| 269 | DEVICE_LIFE_TIME_EST_B | R | Type B (MLC/TLC) 수명 |
| 254 | CACHE_SIZE[3:0] | R | 캐시 크기 (KB) |
| 308 | CMDQ_SUPPORT | R | CMDQ 지원 여부 |

---

## 7. eMMC 명령 체계 (MMC Protocol)

### 7-1. 명령 구조

```
CMD 라인 명령 포맷 (48-bit):
┌──┬────────┬──────────────────────────────┬───────┬──┐
│01│ index  │          argument            │  CRC  │1 │
│  │(6-bit) │         (32-bit)             │(7-bit)│  │
└──┴────────┴──────────────────────────────┴───────┴──┘
 시작  CMD번호       파라미터                CRC7   끝
```

### 7-2. 주요 명령어

| 명령 | 이름 | 용도 | 응답 |
|------|------|------|------|
| **CMD0** | GO_IDLE_STATE | 리셋, Idle 상태로 | 없음 |
| **CMD1** | SEND_OP_COND | 전압 협상 (반복) | R3 (OCR) |
| **CMD2** | ALL_SEND_CID | CID 읽기 | R2 (CID) |
| **CMD3** | SET_RELATIVE_ADDR | RCA 주소 할당 | R1 |
| CMD6 | SWITCH | EXT_CSD 레지스터 변경 | R1b |
| **CMD7** | SELECT_CARD | 카드 선택/해제 | R1 |
| **CMD8** | SEND_EXT_CSD | EXT_CSD 읽기 | R1 |
| CMD9 | SEND_CSD | CSD 읽기 | R2 |
| CMD12 | STOP_TRANSMISSION | 멀티블록 전송 중단 | R1b |
| CMD13 | SEND_STATUS | 카드 상태 확인 | R1 |
| **CMD17** | READ_SINGLE_BLOCK | 1블록 읽기 | R1 |
| **CMD18** | READ_MULTIPLE_BLOCK | 멀티블록 읽기 | R1 |
| CMD23 | SET_BLOCK_COUNT | 블록 수 설정 | R1 |
| **CMD24** | WRITE_BLOCK | 1블록 쓰기 | R1 |
| **CMD25** | WRITE_MULTIPLE_BLOCK | 멀티블록 쓰기 | R1 |
| CMD35 | ERASE_GROUP_START | 삭제 시작 주소 | R1 |
| CMD36 | ERASE_GROUP_END | 삭제 끝 주소 | R1 |
| CMD38 | ERASE | 삭제 실행 | R1b |

### 7-3. 카드 상태 머신

```
Power Off
  │ 전원 인가
  ▼
Idle ──CMD1──→ Ready ──CMD2──→ Ident ──CMD3──→ Stand-by
                                                  │
                                              CMD7│
                                                  ▼
                 Data ←──CMD17/18/24/25──── Transfer
                  │                            │
                  └──────CMD12──────────────→──┘
```

---

## 8. eMMC 초기화 과정

### 상세 시퀀스

```
1. 전원 인가
   │  VCC=3.3V, VCCQ=3.3V (또는 1.8V)
   │  최소 1ms 대기 (전원 안정화)
   │
2. 클럭 시작 (400 KHz 이하)
   │  74 클럭 이상 대기 (eMMC 내부 초기화)
   │
3. CMD0 (GO_IDLE) → Idle 상태로 리셋
   │
4. CMD1 (SEND_OP_COND) → 전압 협상
   │  인자: 지원 전압 범위 + 섹터 모드 비트
   │  응답: OCR 레지스터
   │  ├─ busy=0 → 아직 준비 안됨 → CMD1 반복 (최대 1초)
   │  └─ busy=1 → 준비 완료 → 다음 단계
   │
5. CMD2 (ALL_SEND_CID) → CID 읽기
   │  제조사, 제품명, 시리얼 번호 확인
   │
6. CMD3 (SET_RELATIVE_ADDR) → RCA 할당
   │  eMMC: 호스트가 RCA 지정 (보통 0x0001)
   │  SD: 카드가 RCA 자동 생성 (차이점!)
   │
7. CMD9 (SEND_CSD) → CSD 읽기
   │  용량, 최대 클럭 정보 확인
   │
8. CMD7 (SELECT_CARD) → Transfer 상태로 전환
   │
9. CMD8 (SEND_EXT_CSD) → EXT_CSD 512바이트 읽기
   │  eMMC 버전, 지원 기능, 수명 정보 확인
   │
10. CMD6 (SWITCH) → 버스 폭 변경
    │  EXT_CSD[183] BUS_WIDTH = 2 (8-bit)
    │
11. CMD6 (SWITCH) → 속도 모드 변경
    │  EXT_CSD[185] HS_TIMING = 2 (HS200)
    │
12. 클럭 변경 → 200 MHz
    │
13. CMD21 (SEND_TUNING_BLOCK) → 튜닝
    │  최적 샘플링 포인트 결정
    │
14. 초기화 완료 → 정상 R/W 가능
```

### 초기화 에러와 원인

| 에러 코드 | 이름 | 원인 | 해결 |
|-----------|------|------|------|
| -17 | EEXIST | 이미 초기화됨 또는 CD핀 오감지 | 폴백 로직 추가 |
| -110 | ETIMEDOUT | CMD1 응답 없음 (전원/배선 불량) | HW 확인 |
| -22 | EINVAL | 전압 불일치 | OCR 설정 확인 |
| -5 | EIO | 통신 에러 | 클럭/신호 무결성 확인 |
| -84 | EILSEQ | CRC 에러 (신호 품질 불량) | 튜닝, 배선 길이 확인 |

---

## 9. eMMC 읽기/쓰기 동작

### 9-1. 단일 블록 읽기

```
호스트                              eMMC
  │                                  │
  │──CMD17 (READ_SINGLE_BLOCK)────→ │  주소 전달
  │←────── R1 응답 ──────────────── │
  │                                  │  내부 NAND 읽기
  │←── DAT0~7: 데이터 (512B) ────── │  데이터 전송
  │←── DAT0~7: CRC16 ───────────── │  에러 체크
  │                                  │
```

### 9-2. 멀티블록 쓰기 (가장 흔한 패턴)

```
호스트                              eMMC
  │                                  │
  │──CMD23 (SET_BLOCK_COUNT)──────→ │  블록 수 설정
  │←────── R1 ──────────────────── │
  │──CMD25 (WRITE_MULTIPLE)───────→ │  쓰기 시작
  │←────── R1 ──────────────────── │
  │──DAT: Block 0 데이터 ──────────→│
  │←── DAT0: CRC status ─────────  │  CRC 확인
  │──DAT: Block 1 데이터 ──────────→│
  │←── DAT0: CRC status ─────────  │
  │  ...반복...                      │
  │──DAT: Block N 데이터 ──────────→│
  │←── DAT0: CRC status ─────────  │
  │                                  │  내부 프로그래밍
  │  (DAT0 LOW = busy)              │  (NAND에 실제 쓰기)
  │←── DAT0 HIGH ─────────────────  │  완료
  │                                  │
```

> **busy 신호**: eMMC가 내부 작업(NAND 프로그래밍) 중일 때 DAT0을 LOW로 유지. 호스트는 이 동안 대기해야 함.

### 9-3. TRIM과 Discard

파일 삭제 시 OS가 eMMC에게 "이 영역은 더 이상 안 쓰임"을 알려줌:

| 명령 | 동작 | 속도 | 데이터 복구 |
|------|------|------|-----------|
| ERASE | 물리적 삭제 | 느림 | 불가 |
| **TRIM** | GC 대상 마킹 (삭제는 나중에) | 빠름 | 불가 |
| DISCARD | GC 대상 마킹 (삭제 보장 안됨) | 매우 빠름 | 가능할 수도 |
| SANITIZE | 전체 무효 데이터 완전 삭제 | 매우 느림 | 불가 |

> **TRIM을 지원하면 eMMC 수명과 성능이 크게 향상됨.** 파일시스템에서 `discard` 마운트 옵션 사용 권장.

---

## 10. eMMC 고급 기능

### 10-1. BKOPS (Background Operations) - eMMC 4.51+

eMMC 컨트롤러가 **유휴 시간에 수행**하는 유지보수 작업:
- Garbage Collection (삭제된 블록 정리)
- Wear Leveling (쓰기 횟수 균등 분배)
- Bad Block 교체

```
BKOPS 상태 레벨 (EXT_CSD[246]):
  Level 0: 불필요 (깨끗한 상태)
  Level 1: 비중요 (여유 있을 때 해도 됨)
  Level 2: 성능 영향 시작 (곧 해야 함)
  Level 3: 긴급! (성능 심각 저하, 즉시 필요)
```

#### Auto BKOPS vs Manual BKOPS

| 구분 | Auto BKOPS | Manual BKOPS |
|------|------------|--------------|
| 동작 | eMMC가 유휴 시 자동 실행 | 호스트가 CMD6으로 시작/중지 |
| 설정 | EXT_CSD[163] BKOPS_EN = 1 | 호스트 코드 필요 |
| 장점 | 간단, 안정적 | 세밀한 제어, I/O 타이밍 관리 |
| 단점 | I/O 지연 발생 가능 | 구현 복잡, 버그 위험 |
| 적합 | 대부분의 임베디드 | 실시간 시스템 |

#### BKOPS + I/O 충돌 처리

```
eMMC가 BKOPS 실행 중일 때 호스트가 I/O 요청을 보내면?

방법 1: HPI (High Priority Interrupt) - eMMC 4.41+
  호스트 → CMD12/CMD13 (HPI 비트 설정) → eMMC
  eMMC: BKOPS 중단 → I/O 처리 → BKOPS 재개

방법 2: 그냥 기다림
  BKOPS 완료될 때까지 I/O 대기 (지연 발생)
```

### 10-2. Cache - eMMC 4.51+

eMMC 내부의 **고속 쓰기 버퍼**:

```
호스트 → 데이터 쓰기 → [Cache (RAM)] → [NAND Flash]
                        ↑ 빠름!         ↑ 느림
                        (즉시 완료 응답)  (나중에 flush)
```

| 항목 | 설명 |
|------|------|
| 크기 | 보통 512KB ~ 수 MB |
| 장점 | 쓰기 속도 대폭 향상 (특히 랜덤 쓰기) |
| 위험 | 캐시에만 있고 NAND에 안 쓴 상태에서 전원 차단 → **데이터 손실** |
| 해결 | `CMD6 (FLUSH_CACHE)` 또는 `sync` 명령으로 강제 flush |

#### Cache Barrier - eMMC 5.1

```
일반 캐시:
  메타데이터 쓰기 → 데이터 쓰기 → Flush
  (순서 보장 안됨! 전원 차단 시 파일시스템 깨질 수 있음)

Cache Barrier:
  메타데이터 쓰기 → [Barrier] → 데이터 쓰기 → Flush
  (메타데이터가 반드시 먼저 NAND에 기록됨)
```

### 10-3. CMDQ (Command Queuing) - eMMC 5.1

```
일반:     CMD → 완료대기 → CMD → 완료대기 → CMD → 완료대기
CMDQ:     CMD1 ─┐
          CMD2 ─┼─→ eMMC가 내부적으로 최적 순서로 처리
          CMD3 ─┘
```

- 최대 32개 명령 동시 큐잉
- eMMC가 NAND의 물리 위치를 고려해 최적 순서로 실행
- 스마트폰처럼 멀티앱 환경에서 효과적
- **임베디드 단일 태스크에서는 효과 미미**

### 10-4. Secure Erase / Sanitize

| 기능 | 설명 | 용도 |
|------|------|------|
| Secure Erase | 지정 영역의 데이터를 복구 불가능하게 삭제 | 보안 데이터 폐기 |
| Secure TRIM | TRIM + 복구 불가 보장 | 보안 파일 삭제 |
| Sanitize | 모든 무효 데이터 완전 삭제 | 기기 폐기/재활용 전 |

### 10-5. RPMB (Replay Protected Memory Block)

```
일반 영역:  누구나 읽기/쓰기 가능
RPMB:      인증 키(HMAC-SHA256) + 카운터로 보호

쓰기: 호스트 → 데이터 + HMAC 서명 → eMMC (서명 검증 후 쓰기)
읽기: 호스트 → 요청 → eMMC → 데이터 + HMAC 서명 (호스트가 검증)
```

- 재생 공격(Replay Attack) 방지: 쓰기 카운터가 항상 증가
- 용도: DRM 키, Secure Boot 인증서, 민감한 설정값

---

## 11. eMMC 수명 관리

### 11-1. 수명에 영향을 주는 요인

| 요인 | 영향 | 대책 |
|------|------|------|
| P/E Cycle 소진 | NAND 셀 열화 | Wear Leveling, 쓰기 최소화 |
| Write Amplification | 실제 쓰기량 > 호스트 쓰기량 | TRIM 사용, 순차 쓰기 |
| 고온 환경 | 데이터 보존력 감소 | 방열 설계 |
| 갑작스런 전원 차단 | 메타데이터 손상 | Cache Flush, 캐패시터 |

### 11-2. 수명 모니터링

```bash
# 리눅스에서 EXT_CSD 전체 읽기
cat /sys/kernel/debug/mmc2/mmc2:0001/ext_csd

# sysfs에서 개별 값 읽기
cat /sys/class/mmc_host/mmc2/mmc2:0001/pre_eol_info
cat /sys/class/mmc_host/mmc2/mmc2:0001/life_time_typ_a
cat /sys/class/mmc_host/mmc2/mmc2:0001/life_time_typ_b
```

### 11-3. PRE_EOL_INFO (예비 블록 소진도)

| 값 | 의미 | 상태 | 조치 |
|----|------|------|------|
| 0x01 | Normal | 정상 | 없음 |
| 0x02 | Warning | 경고 | 예비 블록 80% 소진, 교체 준비 |
| 0x03 | Urgent | 긴급 | 예비 블록 90% 소진, 즉시 교체! |

### 11-4. DEVICE_LIFE_TIME_EST (수명 사용량)

| 값 | 사용량 | 상태 | 권장 조치 |
|----|--------|------|----------|
| 0x00 | 정의 안 됨 | - | |
| 0x01 | 0-10% | 새 제품 | |
| 0x02 | 10-20% | 양호 | |
| 0x03 | 20-30% | 양호 | |
| 0x04 | 30-40% | 정상 | |
| 0x05 | 40-50% | 정상 | |
| 0x06 | 50-60% | 주의 | 모니터링 주기 단축 |
| 0x07 | 60-70% | 주의 | 교체 일정 수립 |
| 0x08 | 70-80% | 경고 | 교체 부품 확보 |
| 0x09 | 80-90% | 교체 준비 | 교체 일정 확정 |
| 0x0A | 90-100% | 긴급 교체 | 즉시 교체 |
| 0x0B | 100% 초과 | 수명 종료 | 데이터 손실 위험 |

> Type A = SLC 영역 수명, Type B = MLC/TLC 영역 수명

### 11-5. 양산 제품 수명 관리 전략

```
[개발 단계]
  1. TRIM/Discard 지원 확인 및 활성화
  2. BKOPS 활성화 (Auto BKOPS 권장)
  3. 불필요한 로그/임시파일 쓰기 최소화
  4. 로그는 tmpfs(RAM)에 쓰고 주기적으로 flush

[양산 단계]
  1. 수명 모니터링 데몬 구현
  2. PRE_EOL 경고 시 원격 알림
  3. 교체 기준 수립 (예: LIFE_TIME 0x08 이상이면 교체)

[운영 단계]
  1. 주기적 수명 체크 (1일 1회)
  2. 쓰기량 통계 수집
  3. 이상 징후 시 사전 교체
```

---

## 12. eMMC와 부팅

### 12-1. 일반적인 eMMC 부팅 과정

```
SoC 전원 ON
  ↓
Boot ROM (SoC 내장)
  ↓ Boot Area에서 첫 번째 부트로더 읽기
SPL (Secondary Program Loader)
  ↓ DRAM 초기화 + 메인 부트로더 로드
U-Boot (또는 다른 부트로더)
  ↓ 커널 + DTB + rootfs 로드
Linux Kernel
  ↓
User Space
```

### 12-2. Boot Partition 설정

EXT_CSD[179] PARTITION_CONFIG:

```
비트 [5:3] BOOT_PARTITION_ENABLE:
  0 = 부트 비활성화
  1 = Boot Area 1에서 부팅
  2 = Boot Area 2에서 부팅
  7 = User Data Area에서 부팅

비트 [2:0] PARTITION_ACCESS:
  0 = User Data Area 접근
  1 = Boot Area 1 접근
  2 = Boot Area 2 접근
  3 = RPMB 접근
```

### 12-3. Boot Area에 부트로더 쓰기 (리눅스 예시)

```bash
# Boot Area 1에 SPL 쓰기
echo 1 > /sys/block/mmcblk0boot0/force_ro  # 쓰기 보호 해제 (0)
echo 0 > /sys/block/mmcblk0boot0/force_ro
dd if=spl.bin of=/dev/mmcblk0boot0 bs=512

# Boot Partition 1에서 부팅하도록 설정
mmc bootpart enable 1 0 /dev/mmcblk0
```

---

## 13. eMMC 전원 안전성

### 13-1. 전원 차단 시 위험 시나리오

```
정상 종료:
  BKOPS 중지 → Cache Flush → 전원 차단 ✓

갑작스런 차단:
  쓰기 중 전원 차단 → Cache 데이터 손실 ✗
  BKOPS 중 전원 차단 → 메타데이터 불일치 가능 ✗
  GC 중 전원 차단 → 블록 매핑 손상 가능 ✗
```

### 13-2. 보호 방법

| 레벨 | 방법 | 설명 |
|------|------|------|
| **하드웨어** | 캐패시터 | 전원 차단 감지 → 캐패시터 전력으로 Cache Flush |
| **소프트웨어** | Reliable Write | 쓰기 중 전원 차단 시에도 원자성 보장 |
| **소프트웨어** | Power-off Notification | CMD6으로 "곧 꺼짐" 알림 → eMMC가 정리 |
| **파일시스템** | Journaling (ext4) | 메타데이터 저널링으로 복구 가능 |

### 13-3. Power-off Notification (eMMC 4.41+)

```
정상 종료 시:
  CMD6 → POWER_OFF_NOTIFICATION = 0x01 (POWERED_ON)
  CMD6 → POWER_OFF_NOTIFICATION = 0x02 (POWER_OFF_SHORT)
  (eMMC가 Cache Flush + 내부 정리)
  전원 차단

긴급 종료 시:
  CMD6 → POWER_OFF_NOTIFICATION = 0x03 (POWER_OFF_LONG)
  (eMMC가 최대한 데이터 보존 시도)
  전원 차단
```

---

## 14. 임베디드에서의 eMMC 실전 팁

### 14-1. 쓰기 수명 늘리기

| 방법 | 효과 | 구현 |
|------|------|------|
| tmpfs 활용 | 임시 파일 RAM에 쓰기 | `/tmp`을 tmpfs로 마운트 |
| 로그 회전 | 로그 파일 크기 제한 | logrotate 설정 |
| noatime 마운트 | 읽기 시 접근시간 안 씀 | `mount -o noatime` |
| TRIM/Discard | GC 효율 향상 | `mount -o discard` |
| 쓰기 버퍼링 | 작은 쓰기 모아서 한번에 | application 레벨 버퍼 |

### 14-2. 파일시스템 선택

| 파일시스템 | 장점 | 단점 | 적합 |
|-----------|------|------|------|
| **ext4** | 안정적, 저널링 | TRIM 지원 필요 | 범용 |
| **F2FS** | Flash 최적화, 낮은 WAF | 상대적으로 새로움 | eMMC 최적 |
| SquashFS | 읽기 전용, 압축 | 쓰기 불가 | 펌웨어 이미지 |
| UBIFS | 원래 Raw NAND용 | eMMC에는 과잉 | 사용 안 함 |

> **eMMC에는 F2FS를 가장 권장.** Flash 특성에 맞게 설계되어 WAF가 낮고 수명이 길어짐.

### 14-3. 디바이스 노드 (리눅스)

```bash
/dev/mmcblk0         # User Data Area 전체
/dev/mmcblk0p1       # User Area의 파티션 1
/dev/mmcblk0p2       # User Area의 파티션 2
/dev/mmcblk0boot0    # Boot Area 1
/dev/mmcblk0boot1    # Boot Area 2
/dev/mmcblk0rpmb     # RPMB
```

---

## 15. [실무 심화] U-Boot SPL에서의 MMC 초기화

> 팀장님 적용사항 분석 결과, **SPL 단계의 MMC 초기화/폴백**이 가장 핵심 업무.
> 이 섹션은 U-Boot SPL이 eMMC를 어떻게 초기화하는지 코드 레벨로 학습.

### 15-1. 부팅 전체 흐름에서 SPL의 위치

```
┌──────────────────────────────────────────────────────────────┐
│ SoC 전원 ON                                                  │
│  ↓                                                           │
│ BROM (Boot ROM, SoC 내장, 수정 불가)                          │
│  │ ● 내장 SRAM에 SPL 로드 (eMMC Boot Area 또는 SD 카드에서)   │
│  │ ● BROM은 최소한의 MMC 초기화만 수행 (1-bit, 저속)           │
│  ↓                                                           │
│ ★ SPL (Secondary Program Loader, 우리가 수정하는 코드) ★       │
│  │ ● DRAM 초기화 (SPL 전에는 SRAM만 사용 가능)                 │
│  │ ● MMC 재초기화 (더 빠른 모드로)                             │
│  │ ● U-Boot 본체를 DRAM으로 로드                               │
│  ↓                                                           │
│ U-Boot (메인 부트로더, DRAM에서 실행)                          │
│  │ ● 전체 하드웨어 초기화                                      │
│  │ ● 환경변수 로드                                             │
│  │ ● 커널 + DTB + rootfs 로드                                  │
│  ↓                                                           │
│ Linux Kernel                                                  │
└──────────────────────────────────────────────────────────────┘
```

### 15-2. SPL의 크기 제약과 설계

SPL은 SoC 내장 SRAM에서 실행되므로 **크기 제한이 매우 엄격함**:

| SoC | 내장 SRAM | SPL 최대 크기 | 비고 |
|-----|-----------|-------------|------|
| Allwinner A20 | 32KB | ~24KB | 나머지는 스택/BSS |
| STM32MP1 | 256KB | ~200KB | 여유 있음 |
| i.MX6 | 96KB | ~68KB | |

SPL에서 할 수 있는 것:
- 클럭 설정 (PLL)
- DRAM 컨트롤러 초기화
- **MMC 초기화 (핵심!)**
- UART 출력 (디버그용)
- U-Boot 이미지 로드

SPL에서 할 수 **없는** 것:
- 파일시스템 접근 (드라이버가 너무 큼)
- 네트워크 (드라이버 크기 초과)
- 복잡한 에러 처리 (코드 공간 부족)

### 15-3. SPL의 MMC 초기화 코드 흐름 (U-Boot 기준)

```c
// arch/arm/lib/spl.c (또는 common/spl/spl_mmc.c)
void board_init_f(ulong dummy)
{
    // 1. 기본 하드웨어 초기화
    board_early_init_f();    // 클럭, 핀먹스
    
    // 2. DRAM 초기화
    dram_init();             // DDR 컨트롤러 설정
    
    // 3. SPL 메인 로직
    board_init_r();
}

void board_init_r(void)
{
    // 4. MMC 초기화
    spl_mmc_load_image();    // ★ 여기서 eMMC/SD 초기화
}
```

#### spl_mmc_load_image() 상세

```c
int spl_mmc_load_image(void)
{
    struct mmc *mmc;
    int err;
    
    // Step 1: 어떤 MMC를 쓸지 결정
    err = board_mmc_init();     // ★ 보드별 커스텀 (팀장님이 수정한 부분)
    
    // Step 2: MMC 구조체 가져오기
    mmc = find_mmc_device(boot_device);
    
    // Step 3: MMC 초기화 (CMD0 → CMD1 → ... 전체 시퀀스)
    err = mmc_init(mmc);
    if (err) {
        // ★★★ 기존: hang() → 변경: 폴백 로직 ★★★
        printf("spl: mmc init failed: err - %d\n", err);
        
        // [기존 코드] hang();  // 여기서 멈춤!
        
        // [팀장님 수정] 폴백 시도
        err = spl_mmc_load_image_eschoi();  // eMMC 강제 초기화
        if (err)
            hang();  // 폴백도 실패하면 그때 hang
    }
    
    // Step 4: U-Boot 이미지 로드
    err = mmc_load_image_raw(mmc, sector_offset);
    // sector_offset: U-Boot 이미지가 저장된 섹터 위치
    
    return err;
}
```

### 15-4. board_mmc_init()의 역할 (보드별 커스텀)

```c
// board/sunxi/board.c (Allwinner A20 예시)
int board_mmc_init(bd_t *bis)
{
    // CD (Card Detect) 핀 읽기
    int cd_pin = sunxi_mmc_get_cd_pin();
    int cd_state = gpio_get_value(cd_pin);
    
    if (cd_state == 0) {
        // CD핀 LOW = SD 카드 삽입됨 (또는 감지 오류!)
        printf("MMC0 card boot\n");
        sunxi_mmc_init(0);    // MMC0 (SD 카드 슬롯)
    } else {
        // CD핀 HIGH = SD 카드 없음
        printf("MMC2 eMMC boot\n");
        sunxi_mmc_init(2);    // MMC2 (eMMC)
    }
    return 0;
}
```

**문제점**: CD핀이 풀업/풀다운 저항 문제로 잘못된 값을 줄 수 있음
- SD 카드가 없는데 LOW로 읽힘 → MMC0 선택 → 초기화 실패 → hang()

### 15-5. mmc_init() 내부 (MMC 프로토콜 초기화)

```c
int mmc_init(struct mmc *mmc)
{
    int err;
    
    // 1. 호스트 컨트롤러 초기화
    err = mmc->cfg->ops->init(mmc);     // SDMMC 페리페럴 설정
    
    // 2. 클럭 설정 (400 KHz, 초기화 속도)
    mmc_set_clock(mmc, 400000);
    
    // 3. 버스 폭 1-bit으로 시작
    mmc_set_bus_width(mmc, 1);
    
    // 4. CMD0 - GO_IDLE
    err = mmc_go_idle(mmc);              // 리셋
    
    // 5. CMD1 - SEND_OP_COND (반복)
    err = mmc_send_op_cond(mmc);         // 전압 협상
    //   ↑ 여기서 "Card did not respond to voltage select!" 에러 발생
    //     SD가 없으면 CMD1 응답이 안 옴 → 타임아웃 → 에러 -17
    
    // 6. CMD2 - ALL_SEND_CID
    err = mmc_all_send_cid(mmc);         // CID 읽기
    
    // 7. CMD3 - SET_RELATIVE_ADDR
    err = mmc_set_relative_addr(mmc);    // RCA 할당
    
    // 8. CMD9 - SEND_CSD
    err = mmc_send_csd(mmc);             // CSD 읽기
    
    // 9. CMD7 - SELECT_CARD
    err = mmc_select_card(mmc);          // Transfer 상태로
    
    // 10. CMD8 - SEND_EXT_CSD
    err = mmc_send_ext_csd(mmc);         // EXT_CSD 읽기 (eMMC만)
    
    // 11. 버스 폭/속도 업그레이드
    err = mmc_change_freq(mmc);          // HS/HS200 전환
    err = mmc_set_bus_width(mmc, 8);     // 8-bit 모드
    
    return 0;
}
```

### 15-6. 폴백 설계 패턴 (팀장님 방식 분석)

```
[설계 원칙]
1차 시도 실패 → 2차 대안 시도 → 최종 실패 시에만 hang

[팀장님 구현]
┌─ 1차: CD핀 기반 선택 (기존 로직 유지)
│   ├─ 성공 → 정상 부팅
│   └─ 실패 → 폴백
│
├─ 2차: eMMC 강제 초기화 (추가된 로직)
│   ├─ board_mmc_init_eschoi()
│   │   └─ MMC2 직접 초기화 (CD핀 무시)
│   ├─ mmc_init() 재시도
│   ├─ 성공 → 정상 부팅
│   └─ 실패 → hang
│
└─ hang() (최종 실패)
```

**이 패턴이 좋은 이유:**
1. 기존 코드 최소 수정 (CD핀 로직은 그대로)
2. 실패 시에만 추가 코드 실행 (정상 경로 성능 영향 없음)
3. SD 폴백 유지 (양산 복구용)

### 15-7. SPL 디버깅 방법

SPL은 DRAM 초기화 전이라 디버깅이 어려움:

| 방법 | 설명 | 난이도 |
|------|------|--------|
| **UART printf** | SPL에서 시리얼 출력 (가장 기본) | 쉬움 |
| **GPIO 토글** | LED/GPIO로 진행 상태 표시 | 쉬움 |
| **JTAG/SWD** | 하드웨어 디버거 연결 | 중간 |
| **FEL 모드** | USB로 SPL 직접 로드/테스트 (Allwinner) | 중간 |

```c
// SPL 디버깅 예시
void spl_mmc_load_image(void)
{
    printf("SPL: starting MMC init\n");       // UART 출력
    
    err = board_mmc_init();
    printf("SPL: board_mmc_init ret=%d\n", err);
    
    err = mmc_init(mmc);
    printf("SPL: mmc_init ret=%d\n", err);    // 여기서 에러 확인
    
    if (err) {
        printf("SPL: trying fallback...\n");
        // 폴백 로직
    }
}
```

---

## 16. [실무 심화] BKOPS 커널 드라이버 구현 분석

> 팀장님이 구현한 BKOPS 제어 코드의 원리를 이해하기 위한 학습

### 16-1. 리눅스 MMC 서브시스템 구조

```
┌──────────────────────────────────────────┐
│              User Space                   │
│  (read/write /dev/mmcblk0)               │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│           Block Layer                     │
│  (I/O 스케줄링, 요청 큐)                   │
│  drivers/mmc/core/block.c                 │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│           MMC Core                        │
│  (MMC 프로토콜, 카드 관리)                  │
│  drivers/mmc/core/core.c    ← BKOPS 구현  │
│  drivers/mmc/core/mmc.c     ← eMMC 초기화  │
│  drivers/mmc/core/mmc_ops.c ← CMD 발행     │
└────────────────┬─────────────────────────┘
                 │
┌────────────────▼─────────────────────────┐
│        Host Controller Driver             │
│  (SDMMC 하드웨어 제어)                     │
│  drivers/mmc/host/sunxi-mmc.c (A20용)     │
│  drivers/mmc/host/stm32_sdmmc2.c (STM32)  │
└────────────────┬─────────────────────────┘
                 │
           ┌─────▼─────┐
           │   eMMC     │
           │  하드웨어   │
           └───────────┘
```

### 16-2. BKOPS 초기화 흐름 (mmc.c)

```c
// drivers/mmc/core/mmc.c
static int mmc_init_card(struct mmc_host *host, u32 ocr, ...)
{
    // ... (CMD0~CMD8 초기화) ...
    
    // EXT_CSD 읽기
    err = mmc_read_ext_csd(card);
    
    // ★ BKOPS 지원 여부 확인
    if (card->ext_csd.bkops_support) {
        // BKOPS_EN 비트가 설정되어 있는지 확인
        if (card->ext_csd.bkops_en) {
            // Auto BKOPS 활성화됨
            pr_info("mmc%d: BKOPS supported and enabled\n", host->index);
            
            // BKOPS 자동 체크 워커 스케줄링
            schedule_delayed_work(&card->bkops_work, 
                                  msecs_to_jiffies(30000));  // 30초 후 첫 체크
        }
    }
}
```

### 16-3. BKOPS 상태 체크 (core.c)

```c
// drivers/mmc/core/core.c

// ★ BKOPS 상태 확인 함수
int mmc_check_bkops(struct mmc_card *card)
{
    int err;
    u8 bkops_status;
    
    // EXT_CSD[246] BKOPS_STATUS 읽기
    err = mmc_send_ext_csd(card, ext_csd);
    bkops_status = ext_csd[EXT_CSD_BKOPS_STATUS];  // Byte 246
    
    // 레벨 판단
    switch (bkops_status) {
        case 0:  // Level 0: 불필요
            pr_debug("BKOPS level 0 (OK)\n");
            return 0;
            
        case 1:  // Level 1: 비중요
            pr_info("BKOPS level 1, will check again later\n");
            return 0;  // 지금은 안 함
            
        case 2:  // Level 2: 성능 영향 시작
        case 3:  // Level 3: 긴급
            pr_info("Starting auto BKOPS (level=%d)\n", bkops_status);
            return mmc_start_bkops(card);  // BKOPS 시작!
    }
}
```

### 16-4. BKOPS 시작/중지 (core.c)

```c
// ★ BKOPS 시작
int mmc_start_bkops(struct mmc_card *card)
{
    int err;
    
    // 이미 BKOPS 중이면 리턴
    if (card->bkops_running)
        return 0;
    
    // CMD6 (SWITCH) - BKOPS_START (EXT_CSD[164])
    err = mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
                     EXT_CSD_BKOPS_START, 1, 0);
    
    if (err) {
        pr_err("Failed to start BKOPS\n");
        return err;
    }
    
    card->bkops_running = true;
    pr_info("BKOPS started (level=%d)\n", card->bkops_status);
    return 0;
}

// ★ BKOPS 중지 (HPI 사용)
int mmc_stop_bkops(struct mmc_card *card)
{
    int err;
    
    if (!card->bkops_running)
        return 0;
    
    // HPI (High Priority Interrupt) 발행
    // CMD12 또는 CMD13 with HPI bit
    err = mmc_send_hpi_cmd(card);
    
    if (err) {
        pr_err("Failed to stop BKOPS (HPI): %d\n", err);
        return err;
    }
    
    card->bkops_running = false;
    pr_info("BKOPS stopped\n");
    return 0;
}
```

### 16-5. I/O 요청 시 BKOPS 자동 중지 (block.c)

```c
// drivers/mmc/core/block.c

// 블록 I/O 요청 처리 함수
static int mmc_blk_issue_rq(struct mmc_queue *mq, struct request *req)
{
    struct mmc_card *card = mq->card;
    
    // ★★★ I/O 시작 전 BKOPS 중지 ★★★
    if (card->bkops_running) {
        mmc_stop_bkops(card);  // HPI로 즉시 중단
    }
    
    // 실제 I/O 처리
    if (req_op(req) == REQ_OP_READ)
        return mmc_blk_issue_read(mq, req);
    else if (req_op(req) == REQ_OP_WRITE)
        return mmc_blk_issue_write(mq, req);
    // ...
}
```

### 16-6. 30초 주기 자동 체크 워커

```c
// drivers/mmc/core/core.c

// 주기적 BKOPS 체크 워커
static void mmc_bkops_work(struct work_struct *work)
{
    struct mmc_card *card = container_of(work, ...);
    
    // 1. 카드가 유휴 상태인지 확인
    if (card->bkops_running || mmc_card_doing_io(card))
        goto reschedule;
    
    // 2. BKOPS 상태 체크
    mmc_check_bkops(card);
    
reschedule:
    // 3. 30초 후 다시 체크 스케줄
    schedule_delayed_work(&card->bkops_work,
                          msecs_to_jiffies(30000));
}
```

### 16-7. Suspend/Resume 시 BKOPS 처리

```c
// drivers/mmc/core/mmc.c

// Suspend 시
static int mmc_suspend(struct mmc_host *host)
{
    struct mmc_card *card = host->card;
    
    // 1. BKOPS 워커 취소
    cancel_delayed_work_sync(&card->bkops_work);
    pr_info("BKOPS work cancelled\n");
    
    // 2. 실행 중인 BKOPS 중지
    if (card->bkops_running) {
        mmc_stop_bkops(card);
        pr_info("Stopping BKOPS before suspend\n");
    }
    
    // 3. Cache Flush
    mmc_flush_cache(card);
    pr_info("Cache flush completed\n");
    
    return 0;
}

// Resume 시
static int mmc_resume(struct mmc_host *host)
{
    struct mmc_card *card = host->card;
    
    // BKOPS 워커 재시작
    schedule_delayed_work(&card->bkops_work,
                          msecs_to_jiffies(30000));
    pr_info("BKOPS work restarted after resume\n");
    
    return 0;
}
```

### 16-8. BKOPS 전체 상태 머신

```
              ┌──────────┐
              │  Idle    │ ← BKOPS 안 함
              │ (정상)    │
              └────┬─────┘
                   │ 30초 타이머 만료
                   ▼
              ┌──────────┐
              │  Check   │ ← EXT_CSD[246] 읽기
              │ (상태확인)│
              └────┬─────┘
                   │
          ┌────────┼────────┐
          │        │        │
     Level 0-1  Level 2   Level 3
     (불필요)   (필요)    (긴급)
          │        │        │
          ▼        ▼        ▼
     ┌────────┐ ┌──────────┐
     │  Idle  │ │ Running  │ ← BKOPS 실행 중
     │(재스케줄)│ │ (GC 등)  │
     └────────┘ └────┬─────┘
                     │
              ┌──────┼──────┐
              │             │
         I/O 요청       30초 후
         (HPI 중지)     자동 완료
              │             │
              ▼             ▼
         ┌────────┐   ┌────────┐
         │ I/O    │   │  Idle  │
         │ 처리   │   │(재스케줄)│
         └───┬────┘   └────────┘
             │
             ▼
        ┌────────┐
        │  Idle  │
        │(재스케줄)│
        └────────┘
```

---

## 17. [실무 심화] eMMC 이미지 플래싱 완전 가이드

> 팀장님이 dd 명령으로 이미지를 적용하는 과정의 원리

### 17-1. eMMC 이미지 레이아웃 (Allwinner A20)

```
emmc_loader_uboot.raw (1,040,384 bytes = 1016 KB)
┌────────────────────────────────────────────────────┐
│ Offset 0                                           │
│ ┌────────────────────────┐                         │
│ │      SPL (24KB)        │ ← sunxi-spl.bin         │
│ │  (Boot ROM이 읽는 부분) │                         │
│ └────────────────────────┘                         │
│ Offset 24KB                                        │
│ ┌────────────────────────┐                         │
│ │                        │                         │
│ │    U-Boot (나머지)      │ ← u-boot.img            │
│ │                        │                         │
│ └────────────────────────┘                         │
│ Offset 1016KB (= 1,040,384 bytes)                  │
└────────────────────────────────────────────────────┘
```

### 17-2. dd 명령어 완전 이해

```bash
dd if=INPUT of=OUTPUT bs=BLOCK_SIZE count=COUNT [옵션]
```

| 파라미터 | 의미 | 예시 |
|----------|------|------|
| `if` | Input File (읽을 파일) | if=spl.bin |
| `of` | Output File (쓸 파일) | of=emmc.raw |
| `bs` | Block Size (한번에 읽고 쓰는 크기) | bs=1024 (1KB) |
| `count` | 복사할 블록 수 | count=24 (24블록) |
| `skip` | 입력 파일에서 건너뛸 블록 수 | skip=10 |
| `seek` | 출력 파일에서 건너뛸 블록 수 | seek=10 |
| `conv=sync` | 입력이 bs보다 작으면 0으로 패딩 | 정확한 크기 보장 |
| `conv=notrunc` | 출력 파일을 자르지 않음 | 부분 덮어쓰기용 |

### 17-3. 명령별 동작 원리

**전체 이미지 생성:**
```bash
cp u-boot-sunxi-with-spl.bin emmc_loader_uboot.raw
truncate -s 1040384 emmc_loader_uboot.raw
```
```
동작:
1. cp: 빌드 결과물(SPL+U-Boot 합본)을 복사
2. truncate: 정확히 1,040,384 바이트로 크기 조정
   - 파일이 작으면: 0x00으로 패딩 (뒤에 빈 공간 추가)
   - 파일이 크면: 뒤를 잘라냄
   
왜 1040384?
  = 1016 * 1024 = eMMC의 부트 영역 중 U-Boot 파티션 크기
  SoC의 BROM이 이 크기만큼 읽도록 설정되어 있음
```

**SPL만 교체:**
```bash
# Step 1: SPL 바이너리를 정확히 24KB로 만들기
dd if=spl/sunxi-spl.bin of=spl_only.bin bs=1024 count=24 conv=sync
```
```
동작:
  sunxi-spl.bin (실제 크기: ~8KB)
  ┌────────────────┬─────────────────────────┐
  │  실제 코드 8KB  │  0x00 패딩 16KB (sync)   │
  └────────────────┴─────────────────────────┘
  = spl_only.bin (정확히 24KB)
  
  conv=sync가 하는 일:
  - 마지막 블록이 bs(1024)보다 작으면 0x00으로 채움
  - count=24이므로 24 * 1024 = 24,576 바이트 보장
```

```bash
# Step 2: 기존 이미지의 SPL 부분만 덮어쓰기
dd if=spl/sunxi-spl.bin of=emmc_loader_uboot_new.raw bs=1024 count=24 conv=notrunc
```
```
동작:
  [덮어쓰기 전]
  emmc_loader_uboot_new.raw (1016KB):
  ┌──────────────┬──────────────────────────────┐
  │ 옛 SPL 24KB  │    기존 U-Boot (992KB)        │
  └──────────────┴──────────────────────────────┘
  
  [덮어쓰기 후]
  ┌──────────────┬──────────────────────────────┐
  │ 새 SPL 24KB  │    기존 U-Boot (992KB) 유지!  │
  └──────────────┴──────────────────────────────┘
  
  conv=notrunc가 핵심!
  - 없으면: 출력 파일이 24KB로 잘림 (U-Boot 부분 소실)
  - 있으면: 처음 24KB만 덮어쓰고 나머지는 유지
```

### 17-4. eMMC에 직접 쓰기 (리눅스 타겟에서)

```bash
# 방법 1: User Data Area에 직접 쓰기
dd if=emmc_loader_uboot.raw of=/dev/mmcblk0 bs=1024 seek=8
#                                                    ↑ 8KB 오프셋
#   Allwinner: 섹터 16 (8KB)부터 부트로더 시작

# 방법 2: Boot Partition에 쓰기
echo 0 > /sys/block/mmcblk0boot0/force_ro   # 쓰기 허용
dd if=emmc_loader_uboot.raw of=/dev/mmcblk0boot0 bs=512
echo 1 > /sys/block/mmcblk0boot0/force_ro   # 다시 읽기 전용
```

### 17-5. SD → eMMC 복사 (양산 플래싱)

```bash
#!/bin/bash
# SD 카드의 이미지를 eMMC에 플래싱하는 스크립트

EMMC=/dev/mmcblk0        # eMMC
SD_IMAGE=/mnt/sd/emmc_loader_uboot.raw

echo "=== eMMC Flashing ==="

# 1. Boot Area 쓰기 보호 해제
echo 0 > /sys/block/mmcblk0boot0/force_ro

# 2. 부트로더 쓰기
dd if=$SD_IMAGE of=/dev/mmcblk0boot0 bs=512
sync

# 3. 쓰기 보호 복원
echo 1 > /sys/block/mmcblk0boot0/force_ro

# 4. Boot Partition 설정 (Boot Area 1에서 부팅)
mmc bootpart enable 1 0 $EMMC

# 5. 검증
dd if=/dev/mmcblk0boot0 of=/tmp/verify.bin bs=512 count=2032
md5sum $SD_IMAGE /tmp/verify.bin

echo "=== Flashing Complete ==="
```

### 17-6. 플래싱 시 주의사항

| 주의 | 설명 |
|------|------|
| **전원 차단 금지** | 플래싱 중 전원 끊기면 벽돌(brick) 됨 |
| **sync 필수** | dd 후 반드시 `sync`로 캐시 flush |
| **크기 확인** | truncate로 정확한 크기 보장 |
| **boot partition 설정** | mmc bootpart enable 안 하면 부팅 안 됨 |
| **force_ro** | Boot Area는 기본 읽기 전용, 해제 필요 |
| **검증** | md5sum으로 쓴 내용 비교 확인 |

---

## 18. [실무 심화] eMMC 수명 모니터링 시스템 설계

> 양산 제품에서 eMMC 수명을 추적하고 사전 교체하기 위한 시스템

### 18-1. 왜 모니터링이 필요한가?

```
임베디드 제품의 eMMC 수명 시나리오:

[정상 운영] ─────────────────────────────→ [수명 종료]
0%         30%        60%        90%      100%
│          │          │          │         │
│          │          │     ★ 여기서 경고  │ 데이터 손실!
│          │          │     ★ 여기서 교체  │
│          │          │                    │
└── 모니터링 없이는 여기까지 모름 ──────────┘

eMMC는 SSD와 달리 SMART 정보가 제한적
→ EXT_CSD의 3개 필드만 사용 가능
→ 주기적으로 읽어서 기록해야 함
```

### 18-2. 모니터링 대상 3가지

```
1. PRE_EOL_INFO (EXT_CSD[267])
   ● 예비 블록 소진도
   ● 1=정상, 2=80% 소진(경고), 3=90% 소진(긴급)
   ● 이게 2가 되면 교체 계획 수립

2. DEVICE_LIFE_TIME_EST_A (EXT_CSD[268])
   ● SLC 영역 수명 (0x01~0x0B)
   ● Boot Area, 메타데이터 등 빈번한 쓰기 영역

3. DEVICE_LIFE_TIME_EST_B (EXT_CSD[269])
   ● MLC/TLC 영역 수명 (0x01~0x0B)
   ● User Data Area (대용량 데이터 저장)
   ● 보통 이게 먼저 소진됨
```

### 18-3. 모니터링 데몬 구현 예시

```bash
#!/bin/bash
# /usr/local/bin/emmc_health_monitor.sh
# crontab: 0 3 * * * /usr/local/bin/emmc_health_monitor.sh

SYSFS="/sys/class/mmc_host/mmc2/mmc2:0001"
LOG="/var/log/emmc_health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# 값 읽기
PRE_EOL=$(cat $SYSFS/pre_eol_info 2>/dev/null)
LIFE_A=$(cat $SYSFS/life_time_typ_a 2>/dev/null)
LIFE_B=$(cat $SYSFS/life_time_typ_b 2>/dev/null)

# 로그 기록
echo "$DATE PRE_EOL=$PRE_EOL LIFE_A=$LIFE_A LIFE_B=$LIFE_B" >> $LOG

# 경고 판단
if [ "$PRE_EOL" = "0x02" ] || [ "$LIFE_B" -ge 8 ] 2>/dev/null; then
    # WARNING: 교체 준비 필요
    echo "$DATE WARNING: eMMC approaching end of life!" >> $LOG
    # 원격 알림 (예: MQTT, HTTP)
    # curl -X POST http://monitor-server/alert -d "device=$(hostname)&status=warning"
fi

if [ "$PRE_EOL" = "0x03" ] || [ "$LIFE_B" -ge 10 ] 2>/dev/null; then
    # CRITICAL: 즉시 교체
    echo "$DATE CRITICAL: eMMC replacement urgent!" >> $LOG
    # 긴급 알림
fi
```

### 18-4. 쓰기량 추적 (Write Amplification 확인)

```bash
# /sys/block/mmcblk0/stat 에서 I/O 통계 읽기
#  3번째 필드: 읽기 섹터 수
#  7번째 필드: 쓰기 섹터 수

cat /sys/block/mmcblk0/stat
#  읽기:완료 읽기:병합 읽기:섹터 읽기:ms 쓰기:완료 쓰기:병합 쓰기:섹터 쓰기:ms ...

# 일일 쓰기량 계산 예시
SECTORS=$(awk '{print $7}' /sys/block/mmcblk0/stat)
WRITTEN_MB=$((SECTORS * 512 / 1024 / 1024))
echo "Total written: ${WRITTEN_MB} MB"
```

### 18-5. 수명 예측 공식

```
[단순 선형 예측]

현재 LIFE_TIME_EST = 3 (20-30%)
운영 기간 = 2년
예상 사용률 = 25% (중간값)

남은 수명 = (100% - 25%) / (25% / 2년) = 6년

→ 하지만 실제로는:
  - 쓰기 패턴에 따라 비선형
  - 온도에 따라 가속 열화
  - GC/Wear Leveling 효율 변화
  
→ 실무에서는 LIFE_TIME_EST 값의 변화 추세를 기록하고
   2단계 이상 증가 속도가 빨라지면 조기 교체 판단
```

### 18-6. 수명 연장 실전 설정 (리눅스)

```bash
# /etc/fstab 예시 (eMMC 수명 최적화)

# User Data (noatime + discard로 수명 보호)
/dev/mmcblk0p2  /  ext4  noatime,nodiratime,discard,commit=60  0  1

# 임시 파일은 RAM에 (eMMC 쓰기 방지)
tmpfs  /tmp      tmpfs  defaults,noatime,size=64M  0  0
tmpfs  /var/tmp  tmpfs  defaults,noatime,size=32M  0  0
tmpfs  /var/log  tmpfs  defaults,noatime,size=16M  0  0
#                       ↑ 주의: 재부팅 시 로그 사라짐
#                         중요 로그는 별도 처리 필요
```

| 옵션 | 효과 | 쓰기 절감 |
|------|------|----------|
| `noatime` | 파일 읽기 시 접근시간 안 씀 | 큼 (모든 읽기마다 쓰기 방지) |
| `nodiratime` | 디렉토리 접근시간 안 씀 | 중간 |
| `discard` | TRIM 자동 발행 | GC 효율↑, 간접적 수명↑ |
| `commit=60` | 저널 커밋 주기 60초 (기본 5초) | 중간 (데이터 안전성↓) |
| `tmpfs` | RAM 디스크 사용 | 매우 큼 (해당 경로 eMMC 쓰기 0) |

---

## 19. [실무 제안] 추가로 쓰면 좋은 기능들

> 팀장님 현재 구현: SPL 폴백 + Auto BKOPS + 수명 모니터링
> 아래는 양산 품질/안정성을 높이기 위해 추가 적용을 검토할 기능들

### 현재 vs 추가 제안 전체 맵

```
                    현재 구현됨          추가 제안
                    ──────────          ──────────
[부팅 안정성]
  SPL 폴백            ★                
  Boot Partition 이중화                   ● A/B 부팅
  Watchdog 부팅 감시                      ● 부트 카운터

[데이터 안전성]
  Auto BKOPS           ★
  Cache Flush                            ● Power-off Notification
  Reliable Write                         ● 중요 데이터용
  파일시스템 저널링                        ● F2FS 전환

[수명 관리]
  PRE_EOL 모니터링      ★
  TRIM/Discard                           ● 쓰기 최적화
  쓰기량 제한                             ● Write Budget

[보안]
  (없음)                                 ● Secure Boot (RPMB)

[성능]
  Legacy 25MHz          ★
  HS200 튜닝                             ● 속도 8배↑

[복구]
  SD 폴백              ★
  OTA 업데이트                            ● 원격 펌웨어 업데이트
```

---

### 19-1. A/B Boot Partition (부팅 이중화) -- 우선순위: 높음

**현재 문제**: 부트로더 업데이트 중 전원 차단 → 벽돌(brick)

```
[현재]
Boot Area 1: U-Boot  ← 이거 망가지면 끝
Boot Area 2: (비어있음)

[제안: A/B 부팅]
Boot Area 1: U-Boot (Slot A) ← 정상 시 여기서 부팅
Boot Area 2: U-Boot (Slot B) ← 업데이트 시 여기에 먼저 쓰기

업데이트 절차:
1. Slot B에 새 U-Boot 쓰기
2. PARTITION_CONFIG를 Slot B로 변경
3. 재부팅 → Slot B에서 부팅 시도
4. 성공 → Slot B가 새 기본
5. 실패 → Slot A로 자동 폴백 (기존 버전)
```

```bash
# 구현 예시
# Slot B에 새 부트로더 쓰기
echo 0 > /sys/block/mmcblk0boot1/force_ro
dd if=new_uboot.bin of=/dev/mmcblk0boot1 bs=512
sync

# 부팅 파티션을 Slot B로 변경
mmc bootpart enable 2 0 /dev/mmcblk0
# ↑ 2 = Boot Area 2

# 재부팅 후 정상이면 OK
# 비정상이면 수동으로 Slot A 복원:
mmc bootpart enable 1 0 /dev/mmcblk0
```

**효과**: 부트로더 업데이트 실패해도 이전 버전으로 복구 가능

---

### 19-2. Boot Counter + Watchdog (부팅 감시) -- 우선순위: 높음

**현재 문제**: 커널이 부팅 중 멈추면 수동 복구만 가능

```
[제안: 부트 카운터]
U-Boot 환경변수에 부팅 시도 횟수 저장:

U-Boot 시작
  ↓
boot_count 읽기
  ├─ boot_count < 3 → boot_count++ → 정상 부팅 시도
  │                     ↓
  │                   커널 부팅
  │                     ↓
  │                   정상 시작 확인 → boot_count = 0 (리셋)
  │
  └─ boot_count >= 3 → 복구 모드!
       ├─ SD 카드에서 복구 이미지 부팅
       └─ 또는 공장 초기화 파티션으로 전환
```

```c
// U-Boot 환경변수 예시
// bootcmd에 카운터 로직 추가

setenv bootcmd '
    setexpr boot_count ${boot_count} + 1;
    saveenv;
    if test ${boot_count} -gt 3; then
        echo "Boot failed 3 times! Entering recovery...";
        run recovery_cmd;
    else
        run normal_boot;
    fi
'

// 커널 부팅 성공 후 (init 스크립트에서)
// fw_setenv boot_count 0
```

```bash
# 리눅스 시작 시 (systemd service 또는 init.d)
#!/bin/bash
# /etc/init.d/boot_success.sh
fw_setenv boot_count 0
echo "Boot successful, counter reset"
```

**효과**: 부팅 실패 시 자동 복구, 현장 출동 감소

---

### 19-3. Power-off Notification -- 우선순위: 높음

**현재 문제**: 갑작스런 전원 차단 시 eMMC Cache 데이터 손실 가능

```
[현재]
전원 차단 감지 → (아무것도 안 함) → 전원 꺼짐
                                    ↑ Cache 데이터 손실!

[제안]
전원 차단 감지(GPIO 인터럽트)
  ↓
캐패시터 전력으로 동작 (50~200ms)
  ↓
커널에게 긴급 종료 시그널
  ↓
mmc_poweroff_notify(card, EXT_CSD_POWER_OFF_SHORT)
  ↓
eMMC가 Cache Flush + 내부 정리
  ↓
안전한 전원 차단 ✓
```

```c
// 커널 드라이버에서 Power-off Notification 구현
// drivers/mmc/core/mmc.c 에 이미 있음, 활성화만 하면 됨

static int mmc_poweroff_notify(struct mmc_card *card, unsigned int notify_type)
{
    // EXT_CSD[34] POWER_OFF_NOTIFICATION 에 쓰기
    // notify_type:
    //   0x01 = POWERED_ON (정상 상태)
    //   0x02 = POWER_OFF_SHORT (곧 꺼짐, 빨리 정리해)
    //   0x03 = POWER_OFF_LONG (시간 여유 있음, 천천히 정리)
    
    return mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
                      EXT_CSD_POWER_OFF_NOTIFICATION,
                      notify_type, card->ext_csd.generic_cmd6_time);
}
```

**하드웨어 요구사항**:
| 항목 | 사양 |
|------|------|
| 캐패시터 | 100~470uF, 전원 차단 후 50~200ms 유지 |
| 전압 감지 | GPIO + 비교기 (전원전압 모니터링) |
| 인터럽트 | 전압 하강 시 즉시 커널에 통보 |

**효과**: 전원 차단 시 데이터 무결성 보장

---

### 19-4. TRIM/Discard 활성화 -- 우선순위: 높음

**현재 문제**: TRIM 없으면 삭제된 블록을 eMMC가 모름 → GC 비효율 → 성능 저하 + 수명 단축

```
[TRIM 없을 때]
호스트: 파일 삭제 → 파일시스템에서 제거
eMMC:   해당 블록이 "유효"인지 "무효"인지 모름
        → GC가 무효 데이터도 복사 → Write Amplification ↑↑

[TRIM 있을 때]  
호스트: 파일 삭제 → TRIM 명령 발행 → "이 블록 더 안 씀"
eMMC:   해당 블록을 "무효"로 마킹
        → GC가 무효 블록 건너뜀 → Write Amplification ↓↓
```

```bash
# 적용 방법 1: 마운트 옵션 (실시간 TRIM)
mount -o remount,discard /

# 적용 방법 2: fstab 수정 (영구 적용)
# /etc/fstab
/dev/mmcblk0p2  /  ext4  noatime,discard  0  1

# 적용 방법 3: 주기적 TRIM (성능 영향 최소화, 권장)
# 실시간 discard 대신 하루 1회 fstrim 실행
# /etc/cron.daily/fstrim
#!/bin/bash
fstrim -v /
```

**discard vs fstrim 비교**:
| 방식 | 동작 | 성능 영향 | 적합 |
|------|------|----------|------|
| `discard` (마운트) | 파일 삭제 즉시 TRIM | 삭제 시 약간 지연 | 쓰기 적은 시스템 |
| `fstrim` (크론) | 주기적으로 한꺼번에 | 실행 시에만 부하 | **양산 제품 권장** |

**효과**: eMMC 수명 20~40% 연장, 쓰기 성능 유지

---

### 19-5. Reliable Write (신뢰성 쓰기) -- 우선순위: 중간

**현재 문제**: 일반 쓰기 중 전원 차단 → 일부만 쓰임 (부분 쓰기)

```
[일반 쓰기]
호스트 → 데이터 4KB 쓰기 → eMMC
  전원 차단 발생!
  → 2KB만 쓰임, 나머지 2KB는 이전 데이터도 아니고 새 데이터도 아님 (깨짐)

[Reliable Write]
호스트 → CMD23 (SET_BLOCK_COUNT, Reliable Write 비트 ON) → eMMC
호스트 → 데이터 4KB 쓰기 → eMMC
  전원 차단 발생!
  → 원자성 보장: 4KB 전체가 쓰이거나 OR 전체가 안 쓰임 (깨끗)
```

```c
// EXT_CSD[166] WR_REL_SET 으로 파티션별 Reliable Write 설정
// 설정 후 변경 불가! (OTP, One Time Programmable)

// U-Boot에서 설정
mmc rrel_write dev_num partition_num
```

| 적용 대상 | 필요성 |
|-----------|--------|
| 부트로더 영역 | 매우 높음 (깨지면 벽돌) |
| 파일시스템 저널 | 높음 (깨지면 fsck 필요) |
| 환경변수 영역 | 높음 (설정 손실) |
| 일반 데이터 | 낮음 (파일시스템 저널이 보호) |

**효과**: 전원 차단 시 중요 데이터 원자성 보장

---

### 19-6. HS200 속도 모드 활성화 -- 우선순위: 중간

**현재 상태**: eMMC는 HS200 지원하지만 Legacy 25MHz로 동작 (안정성 우선)

```
현재: 25 MB/s (Legacy 25MHz, 8-bit)
HS200: 200 MB/s (200MHz, 8-bit) → 8배 속도↑
```

**활성화 전 확인 사항**:
| 항목 | 확인 방법 | 필요 조건 |
|------|----------|----------|
| VCCQ 전압 | 회로도 확인 | 1.8V 필요 (3.3V에서는 불가) |
| 배선 길이 | PCB 확인 | 짧을수록 좋음 (< 50mm) |
| 임피던스 매칭 | SI 시뮬레이션 | 50옴 ±10% |
| 튜닝 지원 | 커널 드라이버 | execute_tuning 구현 확인 |

```bash
# 커널에서 HS200 활성화 확인
dmesg | grep -E "HS200|new.*MMC"

# 속도 측정
dd if=/dev/mmcblk0 of=/dev/null bs=1M count=100 iflag=direct
# Legacy: ~20 MB/s
# HS200:  ~150 MB/s (실측, 이론 200)
```

**주의**: 신호 품질이 안 좋으면 CRC 에러로 오히려 불안정해짐. 오실로스코프로 Eye Diagram 확인 후 적용.

**효과**: 부팅 시간 단축, 대용량 데이터 처리 성능 향상

---

### 19-7. F2FS 파일시스템 전환 -- 우선순위: 중간

**현재 (추정)**: ext4 사용

```
[ext4 on eMMC]
● 저널링으로 안전하지만 eMMC에 최적화되지 않음
● 제자리 업데이트(in-place update) → Write Amplification 높음
● GC와 ext4의 블록 할당이 충돌 가능

[F2FS on eMMC]  
● Flash 특성에 맞게 설계 (Log-structured)
● 항상 새 위치에 쓰기 (append-only) → Write Amplification 낮음
● TRIM을 내부적으로 최적 처리
● 멀티헤드 로깅으로 핫/콜드 데이터 분리
```

```bash
# F2FS 적용 방법
# 1. 커널 설정에서 F2FS 활성화
CONFIG_F2FS_FS=y

# 2. 파티션 포맷
mkfs.f2fs -l rootfs /dev/mmcblk0p2

# 3. 마운트
mount -t f2fs -o noatime /dev/mmcblk0p2 /mnt

# 4. fstab
/dev/mmcblk0p2  /  f2fs  noatime,background_gc=on,discard  0  0
```

| 비교 | ext4 | F2FS |
|------|------|------|
| 순차 쓰기 | 비슷 | 비슷 |
| 랜덤 쓰기 | 기준 | **2~3배 빠름** |
| Write Amplification | 높음 | **낮음** |
| eMMC 수명 | 기준 | **20~50% 연장** |
| 안정성 | 매우 검증됨 | 안정 (삼성/구글 사용) |
| fsck 속도 | 느림 | 빠름 |

**효과**: eMMC 수명 연장 + 랜덤 쓰기 성능 향상

---

### 19-8. Secure Boot with RPMB -- 우선순위: 낮음 (제품 요구사항에 따라)

```
[현재]
부트로더 → 커널 → (무결성 검증 없이 실행)
→ 누군가 eMMC 내용을 변조해도 모름

[Secure Boot]
RPMB에 인증 키 저장
  ↓
부트로더 → 커널 이미지 해시 검증 → 일치 시만 부팅
  ↓
변조 감지 시 → 부팅 거부 또는 복구 모드
```

의료기기(검안기) 특성상 보안이 중요할 수 있음:
- 펌웨어 변조 방지
- 환자 데이터 보호
- 인증 기관 요구사항 충족

---

### 19-9. OTA (Over-The-Air) 업데이트 -- 우선순위: 낮음

```
[양산 후 펌웨어 업데이트가 필요한 경우]

현재: SD 카드에 이미지 넣고 현장 방문
제안: 네트워크로 원격 업데이트

A/B 파티션 + OTA 조합:
  ┌────────────────┬────────────────┐
  │  Slot A (현재)  │  Slot B (대기)  │
  │  커널 + rootfs  │  (빈 공간)      │
  └────────────────┴────────────────┘
  
  1. 서버에서 새 이미지 다운로드
  2. Slot B에 쓰기
  3. 부팅 파티션을 Slot B로 변경
  4. 재부팅 → Slot B에서 부팅
  5. 성공 → 완료
  6. 실패 → Slot A로 자동 롤백
```

도구: SWUpdate, RAUC, Mender 등

---

### 19-10. 추가 기능 우선순위 정리

| 순위 | 기능 | 난이도 | 효과 | 비고 |
|------|------|--------|------|------|
| **1** | **TRIM/fstrim** | 쉬움 | 수명↑ 성능↑ | fstab 한 줄 수정 |
| **2** | **Power-off Notification** | 중간 | 데이터 안전↑↑ | HW 캐패시터 필요 |
| **3** | **A/B Boot Partition** | 중간 | 업데이트 안전↑ | Boot Area 2 활용 |
| **4** | **Boot Counter** | 쉬움 | 자동 복구↑ | U-Boot 환경변수만 |
| **5** | **F2FS 전환** | 중간 | 수명↑↑ 성능↑ | 포맷 필요 (신규 양산부터) |
| **6** | **Reliable Write** | 쉬움 | 중요 데이터 안전↑ | OTP라 신중히 |
| **7** | **HS200 활성화** | 어려움 | 속도 8배↑ | SI 검증 필요 |
| **8** | **Secure Boot** | 어려움 | 보안↑↑ | 의료기기 인증 시 |
| **9** | **OTA 업데이트** | 어려움 | 유지보수↑↑ | 네트워크 필요 |

> **당장 적용 추천**: 1~4번은 소프트웨어만으로 가능하고 효과가 큼.
> 특히 **TRIM + Power-off Notification**은 eMMC 제품이면 기본으로 넣어야 함.

---

## 20. 핵심 요약

```
eMMC = NAND Flash + 컨트롤러 (FTL, ECC, Wear Leveling 내장)

꼭 알아야 할 것:
1. 기본: MMC 프로토콜 (CMD + DAT), 블록 단위 R/W
2. 구조: Boot Area + User Area + RPMB
3. 레지스터: CID (신원), CSD (능력), EXT_CSD (모든 것)
4. 초기화: CMD0→CMD1(반복)→CMD2→CMD3→CMD7→CMD8→SWITCH
5. 속도: Legacy → HS → DDR → HS200 → HS400
6. 수명: BKOPS + TRIM + PRE_EOL/LIFE_TIME 모니터링
7. 안전: Cache Flush → Power-off Notification → 전원 차단
8. 파일시스템: F2FS 권장, ext4도 OK (noatime + discard)
```

---

## 참고 문서

- JEDEC eMMC 5.1 표준 (JESD84-B51)
- JEDEC eMMC 5.1A 표준 (JESD84-B51A)
- 리눅스 커널 `drivers/mmc/core/` 소스코드
- U-Boot `drivers/mmc/` 소스코드
- Samsung eMMC Product Family Application Notes
