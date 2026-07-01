# OpenJarvis Tutorial

Apple Silicon(M4 Max, 128GB) 맥에서 **로컬 Qwen(122B / 35B)** 모델을 **LiteLLM + MLX**
스택으로 서빙하고, 그 위에 **OpenJarvis** 로컬 우선(local-first) AI 에이전트를 구축·운용하는
실전 튜토리얼입니다. 설치부터 사용법, Telegram으로 아이폰에서 쓰기, 대안 프레임워크 비교까지
직접 검증한 절차만 담았습니다.

> 이 문서는 한국어와 영어 두 가지 버전을 함께 제공합니다. 왼쪽 목차에서 언어 섹션을 선택하세요.
> This book is available in both Korean and English — pick a language section in the sidebar.

## 무엇을 다루나 (What's inside)

| # | 장 | 내용 |
|---|----|------|
| 1 | **설치 및 설정** | Ollama 없이 pip으로 OpenJarvis 설치, 로컬 Qwen에 연결 (왜 그렇게 하는지 포함) |
| 2 | **로컬 LLM 스택 레퍼런스** | LiteLLM → Qwen 122B/35B(MLX) 엔드포인트 — OpenAI 호환 앱이면 이 문서만으로 연결 가능 |
| 3 | **사용법** | `jarvis ask`/`chat`, 에이전트·도구·메모리, 서버 모드, **"아이폰에서 쓸 수 있나요?"** |
| 4 | **Telegram 채널 설정** | 텔레그램 봇으로 아이폰에서 로컬 Qwen과 대화하기 (설정 + 사용법) |
| 5 | **에이전트 프레임워크 비교** | OpenJarvis vs. OpenHands vs. GSD vs. LangGraph 상세 비교 |

## 이 튜토리얼의 대상 환경 (Target environment)

- **하드웨어:** Apple M4 Max, 128 GB RAM, macOS, arm64
- **서빙:** `mlx_lm.server`(MLX) 백엔드 + **LiteLLM** 프록시(`http://localhost:4000/v1`)
- **모델:** `qwen-122b`(고품질) / `qwen-35b`(빠름)

## 시작하기 (Get started)

[1. 설치 및 설정](SETUP-OpenJarvis.ko.md)부터 시작하세요.
Prefer English? Start with [1. Setup](SETUP-OpenJarvis.md).
