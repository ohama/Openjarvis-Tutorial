# OpenJarvis Tutorial

> Apple Silicon 맥에서 **로컬 Qwen(122B / 35B)** 을 **LiteLLM + MLX** 로 서빙하고,
> 그 위에 **OpenJarvis** 로컬 우선(local-first) AI 에이전트를 구축·운용하는 실전 튜토리얼.
> 설치부터 사용법, 아이폰에서 쓰기(Telegram), 대안 프레임워크 비교까지 — 전부 실제로 검증한 절차만.

한국어 / English 문서를 함께 제공합니다 · Available in Korean and English.

## 📖 Documentation

**https://ohama.github.io/Openjarvis-Tutorial/**

## 이 튜토리얼로 얻는 것 (What you'll build)

- Ollama 없이 기존 **LiteLLM 프록시(`:4000`) + MLX** 스택에 OpenJarvis를 연결
- `jarvis ask` / `chat` / 서버 모드로 로컬 `qwen-122b` 활용
- **아이폰에서** Telegram 봇으로 로컬 모델과 대화
- OpenJarvis vs. OpenHands vs. GSD vs. LangGraph — 언제 무엇을 쓸지 판단

## 목차 (Contents)

| # | 장 (Chapter) | 한국어 | English |
|---|--------------|--------|---------|
| 1 | 설치 및 설정 / Setup | [KO](SETUP-OpenJarvis.ko.md) | [EN](SETUP-OpenJarvis.md) |
| 2 | 로컬 LLM 스택 레퍼런스 / LLM Stack Reference | [KO](LLM-STACK-REFERENCE.ko.md) | [EN](LLM-STACK-REFERENCE.md) |
| 3 | 사용법 / Usage | [KO](USING-OpenJarvis.ko.md) | [EN](USING-OpenJarvis.md) |
| 4 | Telegram 채널 설정 / Telegram Channel | [KO](TELEGRAM-Channel.ko.md) | [EN](TELEGRAM-Channel.md) |
| 5 | 프레임워크 비교 / Framework Comparison | [KO](COMPARISON-Agent-Frameworks.ko.md) | [EN](COMPARISON-Agent-Frameworks.md) |

## 대상 환경 (Target environment)

| 항목 | 값 |
|------|-----|
| 하드웨어 | Apple M4 Max · 128 GB RAM · macOS · arm64 |
| 서빙 | `mlx_lm.server`(MLX) + **LiteLLM** 프록시 `http://localhost:4000/v1` |
| 모델 | `qwen-122b` (고품질) · `qwen-35b` (빠름) |

## 로컬 빌드 (Build locally)

```bash
# mdBook 설치 (한 번만): brew install mdbook  또는  cargo install mdbook
mdbook serve .     # http://localhost:3000 에서 미리보기
mdbook build .     # docs/ 에 정적 사이트 생성
```

## 배포 (Deployment)

- `master`/`main`에 push하면 GitHub Actions(`.github/workflows/mdbook.yml`)가 `docs/`를
  자동으로 다시 빌드해 커밋합니다.
- 최초 1회: **Settings → Pages → Source: Deploy from a branch → Branch: `master`, Folder: `/docs`**.

## 라이선스 및 출처 (Notes)

본 튜토리얼은 [OpenJarvis](https://github.com/open-jarvis/OpenJarvis) 사용법을 정리한 비공식
학습 자료입니다. 각 장의 명령·설정 값은 위 대상 환경에서 실제 실행해 검증했습니다.

---

*Built with [mdBook](https://rust-lang.github.io/mdBook/) · deployed via GitHub Pages*
