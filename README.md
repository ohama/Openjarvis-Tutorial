# OpenJarvis Tutorial

Apple Silicon(M4 Max) 맥에서 **로컬 Qwen(122B / 35B)** 을 **LiteLLM + MLX** 로 서빙하고
그 위에 **OpenJarvis** 로컬 우선 AI 에이전트를 구축·운용하는 실전 튜토리얼입니다.
한국어 / 영어 문서를 함께 제공합니다.

## Documentation

📖 **[https://ohama.github.io/Openjarvis-Tutorial/](https://ohama.github.io/Openjarvis-Tutorial/)**

## 목차 (Contents)

| # | 한국어 | English |
|---|--------|---------|
| 1 | [설치 및 설정](SETUP-OpenJarvis.ko.md) | [Setup](SETUP-OpenJarvis.md) |
| 2 | [로컬 LLM 스택 레퍼런스](LLM-STACK-REFERENCE.ko.md) | [LLM Stack Reference](LLM-STACK-REFERENCE.md) |
| 3 | [사용법](USING-OpenJarvis.ko.md) | [Usage](USING-OpenJarvis.md) |
| 4 | [Telegram 채널 설정](TELEGRAM-Channel.ko.md) | [Telegram Channel](TELEGRAM-Channel.md) |
| 5 | [프레임워크 비교](COMPARISON-Agent-Frameworks.ko.md) | [Framework Comparison](COMPARISON-Agent-Frameworks.md) |

## 로컬 빌드 (Build locally)

```bash
# mdBook 설치 (한 번만): brew install mdbook  또는  cargo install mdbook
mdbook serve .     # http://localhost:3000 에서 미리보기
mdbook build .     # docs/ 에 정적 사이트 생성
```

## 배포 (Deployment)

- `master`/`main`에 push하면 GitHub Actions(`.github/workflows/mdbook.yml`)가 `docs/`를
  자동으로 다시 빌드해 커밋합니다.
- GitHub 저장소 **Settings → Pages → Source: Deploy from a branch → Branch: `master`,
  Folder: `/docs`** 로 설정하면 위 URL에서 책이 공개됩니다.

---

*대상 환경: Apple M4 Max · 128 GB · macOS · `qwen-122b`/`qwen-35b` via LiteLLM `:4000`*
