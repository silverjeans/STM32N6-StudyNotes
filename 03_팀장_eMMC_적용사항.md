# 팀장님 eMMC 적용사항 (Marsboard A20)

> 팀장님(최부장님)이 2020년에 적용한 U-Boot/eMMC 수정 내용 정리
> 2026-04-07 공유받음

---

## 1. 문제 상황

### 증상
SD 카드 없이 eMMC만 있는 상태에서 부팅 실패:

```
U-Boot SPL 2014.04-10734-gfec9bf7-dirty (Apr 09 2020 - 13:48:46)
Board: Marsboard_A20
DRAM: 1024 MiB
CPU: 960000000Hz, AXI/AHB/APB: 3/2/2
auto detect MMC0 or MMC2 boot
MMC0 card boot
Card did not respond to voltage select!
eschoi-spl: mmc init failed: err - -17
### ERROR ### Please RESET the board ###
```

### 원인
- SPL의 `board_mmc_init()`가 CD핀으로 SD/eMMC 자동 감지
- CD핀이 LOW → MMC0(SD) 선택 → SD 없으면 에러 -17 → `hang()` 호출
- **eMMC로 폴백하는 로직이 없었음**

---

## 2. SPL 수정 (이전 → 변경)

### 이전 버전 (hang으로 멈춤)

```
부팅 시작
  ↓
BROM → SPL 로드 (8KB)
  ↓
board_mmc_init() 실행
  ├─ CD pin 체크
  ├─ LOW → MMC0(SD) 초기화
  └─ HIGH → MMC2(eMMC) 초기화
  ↓
mmc_init() 실행
  ├─ 성공 → u-boot.img 로드 (40KB) → U-Boot 메인 → 커널 부팅
  └─ 실패 (에러 -17) → hang()  ← 여기서 멈춤!
```

### 변경 버전 (eMMC 폴백 추가)

```
부팅 시작
  ↓
BROM → SPL 로드 (8KB)
  ↓
board_mmc_init() 실행
  ├─ CD pin 체크
  ├─ LOW → MMC0(SD) 초기화
  └─ HIGH → MMC2(eMMC) 초기화
  ↓
mmc_init() 실행
  ├─ 성공 → u-boot.img 로드 (40KB)
  └─ 실패 (에러 -17) → spl_mmc_load_image_eschoi()
       ↓
       board_mmc_init_eschoi() 호출
       ↓
       MMC2(eMMC) 강제 초기화
       ↓
       u-boot.img 로드 재시도
  ↓
U-Boot 메인 실행
  ↓
initr_mmc() → uboot_mmc_check_image()
  ├─ MMC 체크
  └─ 실패 시 → mmc_initialize_eschoi()
  ↓
커널 부팅
```

### 부트 방식 비교 (팀장님 판단)

| 방식 | 장점 | 단점 |
|------|------|------|
| SD 완전 제거 | 코드 단순 | 복구 어려움 (FEL만 가능) |
| **eMMC 우선 + SD 폴백** | **복구 쉬움** | 코드 약간 복잡 |
| 현재 방식 (CD핀 감지) | 자동 선택 | 불안정할 수 있음 |

> **팀장님 권장**: SD 폴백을 남겨두는 것을 강력 권장. 양산 후 현장에서 문제 발생 시 SD로 복구 가능

### 참고
- 변경 내용: 2020년에 최부장님이 수정
- 당시 SD 고장 상황은 고려하지 않음

---

## 3. U-Boot 이미지 적용법

### 전체 적용 (SPL + U-Boot 둘 다)

빌드 후 SD 카드의 eMMC 폴더 내부에 `emmc_loader_uboot.raw` 이름으로 복사:

```bash
# Git Bash (관리자 권한)에서
cp u-boot-sunxi-with-spl.bin emmc_loader_uboot.raw
truncate -s 1040384 emmc_loader_uboot.raw
```

### SPL만 교체

```bash
# SPL 파일 준비 (GitHub Actions에서 다운로드한 spl/sunxi-spl.bin)
# SPL은 24KB 크기여야 함
dd if=spl/sunxi-spl.bin of=spl_only.bin bs=1024 count=24 conv=sync

# 기존 emmc_loader_uboot.raw의 SPL 부분만 교체
# 1. 먼저 기존 파일 복사
cp /g/emmc/emmc_loader_uboot.raw emmc_loader_uboot_new.raw

# 2. SPL 부분만 덮어쓰기 (처음 24KB)
dd if=spl/sunxi-spl.bin of=emmc_loader_uboot_new.raw bs=1024 count=24 conv=notrunc
```

---

## 4. eMMC 호환성 확인 (Phison PSE2AOSL 8GB)

**eMMC 5.1 HS200 → A20 플랫폼 호환성: 확인됨**

### 커널 드라이버

| 항목 | 상태 | 비고 |
|------|------|------|
| eMMC 5.1 지원 | 지원 | EXT_CSD_REV 8까지 지원 |
| HS200 지원 | 지원 | MMC_CAP2_HS200_1_8V_SDR 설정됨 |
| 최대 클럭 120MHz | 지원 | HS200용 |
| 튜닝 기능 | 구현됨 | execute_tuning |

> **실제 동작은 Legacy 25MHz** (HS200 미사용, 안정성 우선)

### FEX 설정

| 항목 | 상태 | 설정값 |
|------|------|--------|
| mmc2 (eMMC) 활성화 | 설정됨 | sdc_used = 1 |
| 4-bit 버스 | 설정됨 | sdc_buswidth = 4 |
| Non-removable | 설정됨 | sdc_detmode = 3 |
| 핀맵 | 설정됨 | PC06-PC11 (표준) |

---

## 5. BKOPS 구현 현황

### 접근 방식

- **Phase 1 (현재)**: Auto BKOPS 사용
  - BKOPS_EN = 1 설정
  - eMMC가 알아서 관리
  - 안정적이고 간단

- **Phase 2 (나중에 필요하면)**: Manual BKOPS 추가
  - 호스트가 적극 관리
  - 더 공격적인 BKOPS
  - 복잡하고 위험

### 구현된 기능 (커널 코드)

| 기능 | 구현 위치 | 설명 |
|------|-----------|------|
| 자동 감지 | mmc.c:548-563 | eMMC 초기화 시 BKOPS 지원 감지 |
| 상태 체크 | core.c:2596-2624 | mmc_check_bkops() - Level 0-3 반환 |
| 수동 시작 | core.c:2626-2669 | mmc_start_bkops() - BKOPS 실행 |
| 강제 중지 | core.c:2671-2701 | mmc_stop_bkops() - HPI로 중단 |
| 자동 실행 | core.c:2703-2735 | 30초마다 자동 체크 및 실행 |
| I/O 충돌 방지 | block.c:1351-1353 | I/O 시작 전 BKOPS 자동 중지 |

### 전원 차단 안전 대책

| 시나리오 | 보호 동작 | 코드 위치 | 디버깅 메시지 |
|----------|----------|----------|--------------|
| 일반 I/O 요청 | BKOPS 즉시 중지 (HPI) | block.c:1351-1353 | |
| 시스템 Suspend | BKOPS 중지 + 캐시 플러시 | mmc.c:1395-1407 | "Stopping BKOPS before suspend" |
| 시스템 Shutdown | BKOPS 중지 | mmc.c:1395-1407 | |
| 카드 제거 | BKOPS 중지 + 캐시 플러시 | mmc.c:1294-1307 | |
| Resume 후 | BKOPS 워커 재시작 | mmc.c:1448-1453 | |
| 갑작스런 차단 | 할 수 없음 (전원 없음) | 하드웨어 | 캐패시터로 긴급 저장 |

### 고급 기능 현황

| 기능 | 구현 여부 | 대체 방안 | 영향 |
|------|----------|----------|------|
| CMDQ | 없음 | - | 멀티태스크 성능 (임베디드에선 미미) |
| Cache Barrier | 없음 | sync로 대체 가능 | 데이터 안전성 |
| Cache Flush Report | 없음 | - | 전력 최적화 |
| **BKOP Control** | **구현됨** | - | **수명 관리** |

---

## 6. BKOPS 동작 확인 방법

### 부팅 시 로그 확인

```bash
dmesg | grep -i bkop

# 예상:
# [    2.345] mmc2: BKOPS supported and enabled
# [    2.346] mmc2: BKOPS auto-check scheduled
```

### sysfs 상태 확인 스크립트

```bash
echo "=== eMMC BKOPS Status ==="
echo "Card: $(cat /sys/class/mmc_host/mmc2/mmc2:0001/name 2>/dev/null)"
dmesg | grep -i "BKOPS supported"
echo "PRE_EOL: $(cat /sys/class/mmc_host/mmc2/mmc2:0001/pre_eol_info 2>/dev/null)"
echo "Life A: $(cat /sys/class/mmc_host/mmc2/mmc2:0001/life_time_typ_a 2>/dev/null)0%"
echo "Life B: $(cat /sys/class/mmc_host/mmc2/mmc2:0001/life_time_typ_b 2>/dev/null)0%"
dmesg | grep -i bkops | tail -10
```

### BKOPS 트리거 테스트

```bash
# 대량 쓰기로 BKOPS 필요 상황 생성
dd if=/dev/zero of=/mnt/emmc/testfile bs=1M count=1000
sync && rm /mnt/emmc/testfile && sync

# 30~60초 후 로그 확인
dmesg | tail -20
# 예상: "Starting auto BKOPS (level=2)"
```

### I/O 시 BKOPS 중지 확인

```bash
# Terminal 1: 모니터링
watch -n 1 'dmesg | grep -i bkops | tail -5'

# Terminal 2: I/O 발생
dd if=/dev/mmcblk0 of=/dev/null bs=1M count=100
# "BKOPS stopped" 메시지 확인
```

### 디버깅 메시지 목록

**Cache Barrier:**
- `Cache barrier not supported`
- `Cache is disabled, barrier not needed`
- `Failed to set cache barrier`
- `Cache barrier set`

**BKOP Control:**
- `Failed to start BKOPS`
- `BKOPS started (level=`
- `Failed to stop BKOPS (HPI):`
- `BKOPS stopped`
- `Failed to check BKOPS:`
- `Starting auto BKOPS (level=`
- `KOPS level 1, will check again later`
- `BKOPS auto-check scheduled`
- `BKOPS work cancelled`

---

## 7. eMMC 수명 모니터링

```bash
cat /sys/kernel/debug/mmc2/mmc2:0001/ext_csd
```

### PRE_EOL_INFO (예비 블록 소진도)

| 값 | 의미 | 상태 | 조치 |
|----|------|------|------|
| 0x01 | Normal | 정상 | 없음 |
| 0x02 | Warning | 경고 | 80% 소진, 교체 준비 |
| 0x03 | Urgent | 긴급 | 90% 소진, 즉시 교체! |

### DEVICE_LIFE_TIME_EST (수명 사용량)

| 값 | 사용량 | 상태 |
|----|--------|------|
| 0x01 | 0-10% | 새 제품 |
| 0x02~0x03 | 10-30% | 양호 |
| 0x04~0x05 | 30-50% | 정상 |
| 0x06~0x07 | 50-70% | 주의 |
| 0x08 | 70-80% | 경고 |
| 0x09 | 80-90% | 교체 준비 |
| 0x0A | 90-100% | 긴급 교체 |
| 0x0B | 100% 초과 | 수명 종료 |
