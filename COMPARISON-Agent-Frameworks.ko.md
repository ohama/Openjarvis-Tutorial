# OpenJarvis vs. OpenHands vs. GSD vs. LangGraph — 상세 비교

네 가지 인기 에이전트 도구의 기능 비교이며, **이 머신의 구성**(Apple M4 Max에서 LiteLLM을 통해 구동되는 로컬 `qwen-122b`/`qwen-35b`)을 염두에 두고 작성했습니다.

> **먼저 읽어주세요 — 이들은 같은 범주가 아닙니다.** 이들의 "기능"을 정면으로 맞대어
> 비교하는 것은 다소 사과와 오렌지를 비교하는 격입니다. 이들은 서로 다른 계층에 위치합니다:
>
> | 도구 | 근본적으로 *무엇인가* |
> |------|----------------------------|
> | **OpenJarvis** | **개인 비서 런타임/플랫폼** — *당신*을 위해 로컬 우선(local-first) AI 에이전트를 구동(채팅, 다이제스트, 모니터, 리서치)하며, 다양한 채널에서 접근 가능 |
> | **OpenHands** | **자율 소프트웨어 엔지니어링 에이전트** — 코드를 작성하고, 터미널을 실행하며, PR을 엽니다 |
> | **GSD** (Get Shit Done) | 코딩 에이전트(Claude Code 등) *위에* 얹는 **명세 기반(spec-driven) 워크플로/방법론** — 사람+AI의 *프로세스*를 오케스트레이션하며, 런타임이 아닙니다 |
> | **LangGraph** | *개발자*가 Python/JS로 상태를 가진 에이전트 그래프를 구축하기 위한 **저수준 라이브러리/SDK** |
>
> 따라서 진짜 질문은 대개 "어느 것이 가장 좋은가"가 아니라 "나에게 어느 계층이 필요한가"입니다.

---

## 1. 한눈에 보는 매트릭스

| 항목 | **OpenJarvis** | **OpenHands** | **GSD** | **LangGraph** |
|-----------|---------------|---------------|---------|---------------|
| **범주** | 개인 AI 에이전트 플랫폼 | 자율 SWE 에이전트 | 개발 워크플로 / 메타 프롬프팅 시스템 | 에이전트 오케스트레이션 라이브러리 |
| **주요 사용자** | 최종 사용자 / 취미 개발자 | 개발자 / 팀 | Claude Code 등을 사용하는 개발자 | 에이전트를 구축하는 개발자 |
| **상호작용 방식** | CLI, 채팅, 메시징 채널, GUI, HTTP API | 브라우저 "Agent Canvas", SDK, CLI | Claude Code 내부의 슬래시 커맨드 | 직접 작성한 코드(Python/JS) |
| **핵심 역할** | 로컬 개인 비서가 되는 것 | 코딩 작업을 처음부터 끝까지 완수 | 장기 프로젝트의 일관성 유지(context rot 극복) | 프로그래밍할 그래프/상태 기계 제공 |
| **로컬 우선 / 프라이버시** | ✅ 핵심 설계 목표(당신의 HW에서 구동) | ⚠️ 셀프 호스팅 가능하나 샌드박스 + 클라우드 지향 | ➖ 호스트 에이전트를 상속(예: Claude Code → 클라우드) | ➖ 프로바이더 비종속적; 직접 선택 |
| **로컬 모델 구동(Ollama/MLX/vLLM)** | ✅ 내장 엔진 + OpenAI 호환 | ✅ LiteLLM을 통해 Qwen/Llama 지원 | ➖ 호스트 CLI가 사용하는 것 무엇이든 | ✅ LangChain 프로바이더 바인딩을 통해 |
| **멀티 에이전트 오케스트레이션** | ✅ 에이전트 + 게이트웨이 + 오퍼레이터 | ✅ 멀티 에이전트, 위임 | ✅ 병렬 "wave" 서브에이전트 | ✅✅ 핵심 강점(그래프) |
| **장기 기억** | ✅ 지식 저장소(BM25+밀집 검색), 커넥터 | ➖ 세션/워크스페이스 상태 | ✅ 디스크 상의 파일(PROJECT/PLAN/phase 문서) | ✅ 체크포인터 + 메모리 저장소 |
| **Human-in-the-loop(사람 개입)** | 채팅 네이티브 | UI에서 검토/승인 | ✅ 명시적 discuss/verify 게이트 | ✅ 일급 지원(상태 일시정지/검사/편집) |
| **도구 사용 / 코드 실행** | 도구 + MCP + 샌드박스 | ✅ 완전 지원: 셸, 브라우저, PR | 호스트 에이전트의 도구를 통해 | 도구를 노드에 직접 연결 |
| **배포** | 노트북/Mac mini/서버, 터널, 채널 | 노트북 → 클라우드 서버(Docker) | 해당 없음(에디터/CLI 내부에서 구동) | 라이브러리 → LangGraph Platform/Cloud |
| **"고집스러움"의 정도** | 중간(배터리 포함형) | 중상(SWE 루프) | 높음(프로세스를 규정) | 낮음(모든 것을 직접 결정) |
| **사용에 필요한 코딩 수준** | 적음(CLI/설정) | 적음(UI) | 없음(프롬프트) | 많음(라이브러리이므로) |
| **라이선스 / 모델** | 오픈 소스 | 오픈 소스(+ 호스팅 클라우드) | 오픈 소스 | 오픈 소스(+ 유료 플랫폼) |
| **대략적 규모(2026)** | 신흥 단계 | ~70k★, SWE-bench ~53% | ~58k★, 2025년 12월 출시 | v1.0, ~30k★, Uber/LinkedIn/Klarna 프로덕션 |

범례: ✅ 강력 / 내장 · ⚠️ 부분적 또는 단서 조건 있음 · ➖ 주력 대상 아님(간접적으로 가능)

---

## 2. 도구별 상세

### OpenJarvis — *당신*의 로컬 개인 비서
- **용도:** 당신의 하드웨어 위에서 살아가며 *개인적인* 작업을 수행하는 프라이빗 AI —
  채팅, 아침 다이제스트, 자신의 문서를 대상으로 한 심층 리서치, 예약된 모니터,
  코드 어시스턴트를 제공하며, Telegram/iMessage/Slack/WhatsApp, 데스크톱 GUI,
  또는 OpenAI 호환 HTTP API에서 접근할 수 있습니다.
- **두드러진 기능:** 13개 이상의 추론 엔진(Ollama, MLX, vLLM,
  llama.cpp, LiteLLM, 클라우드…)을 지원하는 기본 로컬 구동, 멀티 채널 게이트웨이, 개인 지식 저장소, MCP
  도구 지원, 자동화를 위한 오퍼레이터/스케줄러, 그리고 트레이스 기반 학습.
- **약점:** 전문 코딩 에이전트가 아니며(그 영역은 OpenHands가 더 강력함), 더 새롭고
  생태계가 작으며, 정해진 방법을 따르기보다 자신만의 워크플로를 직접 조립해야 합니다.
- **이 머신에 완벽히 부합:** 당신이 이미 갖춘 로컬 모델 스택(LiteLLM을 통한 `qwen-122b`)을
  정확히 구동하도록 만들어졌습니다. `USING-OpenJarvis.md`를 참고하세요.

### OpenHands (구 OpenDevin) — 자율 소프트웨어 엔지니어
- **용도:** GitHub 이슈나 작업을 건네면 계획을 세우고, 코드를 작성하며, 터미널을
  실행하고, 문서를 탐색하며, 테스트를 고치고, PR을 엽니다 — 샌드박스화된 Docker 환경에서의
  완전한 에이전트형 개발 루프입니다.
- **두드러진 기능:** 강력한 모델과 함께 SWE-bench Verified에서 ~53%+; V1 모듈형
  **Software Agent SDK**(이벤트 소싱 상태, 선택적 샌드박싱, REST/WebSocket 서버);
  브라우저 "Agent Canvas" UI; VS Code/VNC/브라우저 워크스페이스; Slack/GitHub에서 트리거 가능.
- **모델 지원:** LiteLLM을 통한 Claude, GPT, Gemini 및 오픈 모델(Llama, **Qwen**) —
  따라서 로컬 `qwen-122b`에 대해 *구동할 수 있으나*, 최고 점수는 프런티어 모델에서 나옵니다.
- **약점:** 구동이 더 무거우며(Docker 샌드박스, 이상적으로는 서버 필요), 일반적인 개인 비서 작업이 아니라
  소프트웨어 엔지니어링에 집중하고, 클라우드 서버 배포에서 진가를 발휘합니다.

### GSD (Get Shit Done) — 명세 기반 *프로세스*
- **용도:** **크고 여러 세션에 걸친 프로젝트의 일관성 유지**. 런타임이 아니라
  Claude Code(및 OpenCode/Gemini CLI/Codex)를 위한 슬래시 커맨드 워크플로 모음으로,
  **discuss → plan → execute → verify → ship** 사이클을 강제합니다. *(이 머신에 설치되어 있습니다 —
  `/gsd:*` 커맨드와 `.planning/`을 참고하세요.)*
- **해결하는 문제:** *context rot* — 하나의 긴 세션이 컨텍스트 윈도우를 채워가며 품질이
  저하되는 현상. GSD는 작업을 **phase(단계)**로 분할하고, 깨끗한 200k 토큰 컨텍스트를 가진
  **새로운 서브에이전트**를 생성하며, 독립적인 작업을 병렬 "**wave(웨이브)**"로 실행하고, `PLAN.md`
  파일을 서브에이전트가 직접 읽는 *바로 그* 실행 가능한 지시서로 만듭니다.
- **두드러진 기능:** 인터뷰 기반 명세 생성, 병렬 리서치 에이전트,
  목표 역방향(goal-backward) 검증, 영속적 `.planning/` 문서(PROJECT/REQUIREMENTS/MILESTONES,
  phase별 PLAN/RESEARCH/VERIFICATION). 이 머신의 설정: `mode=yolo`, `depth=standard`,
  `parallelization=true`, `model_profile=balanced`.
- **약점:** 방법론이므로 **호스트 에이전트의 모델/비용/프라이버시를 상속합니다**
  (CLI를 로컬 모델로 가리키지 않는 한 Claude Code → 클라우드); *일하는 방식*을 다스릴 뿐,
  스스로 추론을 실행하거나 제품 런타임을 제공하지는 않습니다.

### LangGraph — 직접 만드는 오케스트레이션 라이브러리
- **용도:** 커스텀하고 상태를 가지며 오래 실행되는 에이전트를 구축하는 **개발자**를 위한 것. 에이전트
  로직을 **그래프/상태 기계**로 모델링합니다: 노드 = 액션(LLM 호출, 도구, 서브에이전트),
  엣지 = 조건부 제어 흐름.
- **두드러진 기능:** 체크포인팅을 통한 내구성 있는 실행(크래시 후 재개),
  일급 human-in-the-loop(실행 중 상태 검사/수정), 네이티브 토큰 스트리밍,
  단일/멀티/계층형 에이전트 토폴로지, 내장 메모리. 안정 버전 **v1.0**(2025년 말),
  Klarna, LinkedIn, Uber, Replit에서 프로덕션 사용; DeltaChannel은 체크포인트 저장량을 ~73,000× 줄입니다.
- **모델 지원:** LangChain을 통해 프로바이더 비종속적 — 로컬 모델을 포함하므로
  당신의 `qwen` 엔드포인트를 대상으로 삼을 수 있습니다.
- **약점:** 인프라이므로 — **직접 코드를 작성해야 합니다**; 즉시 사용 가능한 어시스턴트,
  코딩 에이전트, UI가 없습니다. 최대의 제어, 최대의 노력.

---

## 3. 서로 어떻게 관련되는가 (스택으로 쌓을 수 있음)

이들은 순수한 경쟁 관계만은 아닙니다 — 일부는 조합됩니다:
- **GSD + OpenHands / Claude Code:** GSD는 *프로세스*를 오케스트레이션하고, 코딩 에이전트가
  *작업*을 수행합니다. (이 머신에는 실제로 `gsd-openhands` 프로젝트가 있습니다.)
- **제품 내부의 LangGraph:** LangGraph를 하부 엔진으로 삼아 OpenJarvis 유사 또는 OpenHands 유사
  동작을 구축할 수 있습니다.
- **로컬 두뇌로서의 OpenJarvis:** 그 OpenAI 호환 서버는 다른 도구를 뒷받침할 수 있으며,
  OpenHands/LangGraph는 OpenJarvis가 사용하는 *동일한* 로컬 `qwen` 엔드포인트를 가리킬 수 있습니다.

진정한 겹침 지점: **OpenJarvis ↔ OpenHands**는 둘 다 "작업을 수행하는 에이전트를 구동"하며(개인 vs.
소프트웨어 엔지니어링 초점); **GSD ↔ LangGraph**는 둘 다 오케스트레이션/상태를 다룹니다(규정된
*방법* vs. 프로그래밍 가능한 *라이브러리*).

---

## 4. 무엇을 사용해야 하나? (로컬 Qwen Mac 기준)

| 당신의 목표 | 최선의 선택 | 이유 |
|-----------|-----------|-----|
| 자신의 하드웨어에서 구동되는 프라이빗 비서(채팅, 다이제스트, 리서치, 휴대폰에서 접근) | **OpenJarvis** | 로컬 우선으로 특별히 설계됨; 이미 당신의 `qwen-122b`에 연결됨 |
| "이 버그를 고쳐라 / 이 기능을 만들어라"를 자율적으로, PR까지 전부 | **OpenHands** | 전용 SWE 에이전트; LiteLLM을 통해 로컬 Qwen 사용 가능 |
| 크고 여러 주에 걸친 빌드를 context rot 없이 일관되게 유지 | **GSD** | phase/verify 프로세스를 규정; 이미 여기 설치됨 |
| 상태와 흐름을 정밀하게 제어하는 *자신만의* 맞춤 에이전트 구축 | **LangGraph** | 저수준 그래프 엔진; 필요한 것을 정확히 프로그래밍 |

**경험 법칙**
- 사용할 *제품*을 원한다면 → OpenJarvis(개인) 또는 OpenHands(코딩).
- 따를 *프로세스*를 원한다면 → GSD.
- 그 위에 구축할 *라이브러리*를 원한다면 → LangGraph.
- **프라이버시 / 오프라인이 가장 중요하다면** → OpenJarvis만이 로컬 우선으로 설계되었습니다;
  나머지는 로컬 모델을 *사용할 수* 있으나 그 목표를 중심으로 만들어지지는 않았습니다.

---

## Sources
- OpenHands: [openhands.dev](https://www.openhands.dev/) · [GitHub](https://github.com/OpenHands/OpenHands) · [Software Agent SDK (arXiv)](https://arxiv.org/html/2511.03690v1) · [SDK 문서](https://docs.openhands.dev/sdk)
- LangGraph: [langchain.com/langgraph](https://www.langchain.com/langgraph) · [GitHub](https://github.com/langchain-ai/langgraph) · [문서](https://docs.langchain.com/oss/python/langgraph/overview) · [Review 2026](https://toolbrain.net/blog/langgraph-review-2026/)
- GSD: [GitHub (gsd-build/get-shit-done)](https://github.com/gsd-build/get-shit-done) · [문서](https://gsd-build-get-shit-done.mintlify.app/introduction) · [Augment Code 소개글](https://www.augmentcode.com/learn/gsd-58k-stars-claude-code) · [MindStudio 가이드](https://www.mindstudio.ai/blog/gsd-framework-claude-code-clean-context-phases)
- OpenJarvis: [GitHub (open-jarvis/OpenJarvis)](https://github.com/open-jarvis/OpenJarvis) + 로컬 설치 문서(`SETUP-OpenJarvis.md`, `USING-OpenJarvis.md`, `LLM-STACK-REFERENCE.md`)

*2026-07-01 작성. 스타 수 / 벤치마크 수치는 특정 시점 기준이며 변동됩니다.*
