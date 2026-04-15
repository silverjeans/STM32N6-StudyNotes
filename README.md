# STM32N6 스터디 노트

STM32N6570-DK 보드를 활용한 동공 추적 리프랙터 개발 과정 스터디 노트입니다.

## 목차

| 파일 | 내용 |
|------|------|
| 01_환경구축_및_보드개요.md | NUCLEO-N657X0-Q 보드 특징, 카메라/NPU 파이프라인, 스터디 로드맵 |
| 02_동공검출_모델학습_환경구축.md | YOLOX 동공 검출 모델 학습 환경 (Colab, WSL2, RTX 4060) |
| 02_eMMC_학습.md | eMMC 플래시 메모리 학습 |
| 03_모델배포_시도_및_트러블슈팅.md | YOLOX/Iris 모델 DK 보드 배포 삽질 기록 (앵커 불일치, HardFault 등) |
| 양자화_및_STEdgeAI_설명.md | int8 양자화 원리, ST Edge AI 사용법, 리프랙터 AI 활용 방안 |

## 관련 프로젝트

- [STM32N6-IrisLandmark](https://github.com/silverjeans/STM32N6-IrisLandmark) - MediaPipe Iris 동공 추적 구현
