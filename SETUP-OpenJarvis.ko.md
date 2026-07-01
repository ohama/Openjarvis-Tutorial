# OpenJarvis 설치 — Apple Silicon(M4 Max)에서 로컬 qwen-122b 구동

OpenJarvis를 설치하고, 이미 **LiteLLM 프록시 + MLX** 스택을 통해 서비스되고 있는
**로컬 Qwen** 모델에 연결하기 위한 재현 가능한 가이드입니다. 모든 단계에는 *이유*가
포함되어 있어, 환경이 다를 경우 이를 참고해 적절히 조정할 수 있습니다.

- **머신:** Apple M4 Max, 128 GB, macOS 26.x, arm64
- **목표:** OpenJarvis 에이전트를 로컬 `qwen-122b`에서 구동 (Ollama 없이, 클라우드 없이)
- **결과:** 정상 동작 확인됨 (`jarvis ask`가 로컬 모델을 통해 올바른 답변을 반환함)

---

## 0. 사전 요구 사항 — 이미 실행 중이어야 하는 LLM 스택

이 가이드는 이미 자체 추론 스택을 운영하고 있다고 가정합니다(여기서 설치하지 않았으며,
OpenJarvis를 여기에 연결하기만 했습니다). 참고로 제 환경은 다음과 같습니다:

| 포트 | 프로세스 | 역할 |
|------|---------|------|
| `:4000` | `litellm --config agent-stack/litellm/config.yaml` | OpenAI 호환 **프록시 / 라우터** (진입점) |
| `:8000` | `mlx_lm.server --model .../qwen36-35b` | 35B 백엔드 |
| `:8001` | `mlx_lm.server --model .../qwen122b` | 122B 백엔드 |
| `:8011` / `:8012` | `role_shim.py` | MLX를 위한 시스템 메시지 순서 정규화 |
| `:8082` | `claude-code-proxy` | Anthropic→chat 브리지 (OpenJarvis와 무관) |

LiteLLM 프록시는 다음 모델 **별칭(alias)**을 노출합니다: `qwen-122b`, `qwen-35b`,
`qwen-local`, `qwen-122b-codex`, `qwen-122b-claude`, `qwen-35b-claude`.

**OpenJarvis를 건드리기 전에 스택이 정상 동작 중인지 확인하세요:**

```bash
# 프록시가 노출하는 모델 목록 확인
curl -s http://127.0.0.1:4000/v1/models | python3 -m json.tool

# 모델이 실제로 응답하는지 확인
curl -s http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-122b","messages":[{"role":"user","content":"say OK"}],"max_tokens":5}'
```

*먼저 확인하는 이유:* OpenJarvis는 단지 클라이언트일 뿐입니다. `:4000`이 여기서 응답하지
않으면 그 하위의 어떤 것도 동작하지 않으며, 엉뚱한 계층을 디버깅하느라 시간을 낭비하게 됩니다.

---

## 1. 두 가지 설계 결정 이해하기 (설치 전에 읽어 보세요)

OpenJarvis에는 플러그인 방식의 "엔진" 계층이 있습니다. 연결 방식은 서로 독립적인 두 가지
선택으로 정의됩니다:

### 결정 A — 원시 MLX(`:8001`)가 아니라 LiteLLM 프록시(`:4000`)에 연결

OpenJarvis를 MLX에 직접 연결하지 않고 **프록시**를 가리키게 합니다.

*이유:* 프록시는 이미 진입점 역할을 합니다. 프록시를 거치면 OpenJarvis가 깔끔한 별칭
(`qwen-122b`, `qwen-35b`)과 `role_shim` 라우팅을 재사용할 수 있습니다. 한 단어짜리 폴백
모델을 사용할 수 있고, 문자열 하나만 바꾸면 role-shim으로 보호되는 별칭으로 전환할 수
있습니다. `:8001`에 직접 연결하면 모델의 **파일 시스템 경로**를 이름으로 하드코딩해야 하고,
직접 구축한 계층을 우회하게 됩니다.

### 결정 B — "OpenAI 호환" 엔진 선택

OpenJarvis에서 `vllm`, `sglang`, `llamacpp`, `mlx`, `lmstudio` 등의 엔진은 **동일한
코드**입니다. 각 엔진은 그저 `{host}/v1/chat/completions`에 POST 요청을 보낼 뿐입니다.
이름은 기본 포트와 라벨만 바꿀 뿐, **vLLM이나 MLX를 실행하지는 않습니다.**

*`mlx`를 쓰는 이유:* 백엔드가 실제로 MLX이므로, 정직한 라벨로서 `[engine.mlx]`를
사용합니다. (`vllm`도 동일하게 동작합니다 — 이름은 표면적인 것일 뿐입니다 — 그러나 `mlx`가
더 명확합니다.) 진정으로 범용적인 `openai-compat` 엔진도 존재하지만 `config.toml`에서는
**선택할 수 없으므로**(오직 `jarvis eval --base-url`을 통해서만 가능), 등록된 이름 하나를
빌려 씁니다.

> 전체 흐름: `OpenJarvis → 범용 OpenAI HTTP 클라이언트 → LiteLLM :4000 → MLX :8001`

---

## 2. 저장소 클론

```bash
cd /Users/ohama/projs/openjarvis-test
git clone --depth 1 https://github.com/open-jarvis/OpenJarvis.git
cd OpenJarvis
```

*한 줄짜리 설치 스크립트*(`curl … install.sh | bash`)*를 쓰지 않는 이유:* 이 스크립트는
**Ollama**를 설치하고 스타터 모델을 내려받습니다. 이미 더 나은 로컬 스택(MLX)이 있으므로
그 단계는 불필요하며, 경쟁하는 모델 서버를 추가하게 됩니다. pip 패키지를 직접 설치하면
그 모든 과정을 건너뛸 수 있습니다.

---

## 3. uv로 Python 3.13 가상환경(venv) 생성

```bash
uv venv --python 3.13 .venv
```

*특별히 3.13을 쓰는 이유:* OpenJarvis는 `python >=3.10,<3.14`를 요구합니다. 여기 시스템
Python은 **3.14**로 *너무 최신*이라 실패합니다. `uv`는 호환되는 3.13 인터프리터를
내려받아 시스템 Python은 건드리지 않고 venv에 고정합니다.

---

## 4. 패키지 설치 (base + server extra)

```bash
VIRTUAL_ENV=.venv uv pip install -e ".[server]"
```

*`[server]`만 쓰고 더 무거운 것은 쓰지 않는 이유:*
- **Base** 의존성은 순수 Python이며 이미 `openai` 클라이언트를 포함합니다 — OpenAI 호환
  엔진에 필요한 것은 그게 전부입니다. 따라서 base 설치만으로도 `jarvis ask`가 동작합니다.
- **`server`**는 FastAPI/uvicorn을 추가하여, 이후 `jarvis serve` / 데스크톱 앱을 실행할
  수 있게 합니다.
- **`desktop`** extra는 일부러 피했습니다. 이것만이 **Rust** 확장(`openjarvis-rust`)을
  가져와 컴파일을 유발합니다. CLI 사용에는 필요하지 않습니다.
- 웹 검색 도구를 사용하는 경우에만 나중에 `tools-search`를 추가하세요
  (`".[server,tools-search]"`).

---

## 5. 설정 작성 및 설치

OpenJarvis는 `~/.openjarvis/config.toml`을 읽습니다. 다음 파일을 생성하세요:

```toml
# OpenJarvis 설정 — LiteLLM 프록시(:4000)를 통해 로컬 qwen-122b에 연결됨

[intelligence]
default_model = "qwen-122b"       # LiteLLM 별칭 -> mlx qwen122b (:8001)
fallback_model = "qwen-35b"       # 더 가볍고 빠른 폴백 (:8000)
preferred_engine = "mlx"          # OpenAI 호환 클라이언트 (host는 아래에서 :4000을 가리킴)
provider = "local"
quantization = "none"
temperature = 0.2
max_tokens = 4096
top_p = 0.9

[agent]
default_agent = "orchestrator"    # 도구 선택이 가능한 멀티턴 방식
max_turns = 10
context_from_memory = true

[tools]
enabled = ["code_interpreter", "file_read", "file_write", "shell_exec", "web_search", "think", "calculator"]

[tools.storage]
default_backend = "sqlite"
db_path = "~/.openjarvis/memory.db"

[tools.mcp]
enabled = true

[engine]
default = "mlx"                   # OpenAI 호환 엔진 이름이면 무엇이든 동작함; mlx가 정직한 라벨

[engine.mlx]
host = "http://localhost:4000"    # <-- 원시 mlx :8001이 아니라 LiteLLM 프록시
# 프록시는 현재 클라이언트 키 없이 요청을 받습니다. 이후 LiteLLM 마스터 키를 활성화한다면,
# 하드코딩하지 말고 export하세요:  export MLX_API_KEY="sk-..."
# (환경 변수 = <ENGINE_ID>_API_KEY, 대문자로)

[server]
host = "127.0.0.1"
port = 8090                       # :8000에서 옮김 — 이유는 아래 참고

[traces]
enabled = true
db_path = "~/.openjarvis/traces.db"
```

설치:

```bash
mkdir -p ~/.openjarvis
cp config.toml ~/.openjarvis/config.toml
```

*이 값들을 쓰는 이유:*
- **`host = :4000`** — 결정 A. 모든 것을 프록시를 통해 보냅니다.
- **`default_model = "qwen-122b"`** — 경로가 아니라 프록시 별칭. 깔끔하고 교체하기 쉽습니다.
- **`fallback_model = "qwen-35b"`** — 동작하는 별칭이므로, 성능 저하가 부드럽게 이루어집니다.
- **`[server] port = 8090`** — OpenJarvis 기본값은 `8000`인데, 이는 **`:8000`의
  `mlx_lm.server`와 충돌합니다.** 포트를 옮기면 시작 시 포트 충돌을 방지할 수 있습니다.
- **API 키 없음** — 0단계에서 프록시가 인증되지 않은 호출을 받아들였습니다.
  `agent-stack/litellm/config.yaml` 안의 키는 프록시→MLX 전용이며, 클라이언트에는
  필요하지 않습니다.

---

## 6. 설정이 의도대로 로드되는지 확인

```bash
.venv/bin/jarvis config show
```

예상 결과: `Default Engine: mlx`, `Default Model: qwen-122b`,
`Fallback Model: qwen-35b`, `Default Agent: orchestrator`.

*이유:* 답변이 올바른지에 의존하기 **전에**, OpenJarvis가 파일을 파싱하고 엔진/모델을
해석했는지 확인합니다. (`jarvis doctor`는 더 심층적인 진단을 수행합니다.)

---

## 7. 스모크 테스트

```bash
# 7a. 가장 단순한 단일 턴 에이전트로 모델 경로를 분리해 확인
.venv/bin/jarvis ask --agent simple "Reply with exactly this and nothing else: JARVIS-WIRED-OK"

# 7b. 추론 작업으로 설정된 기본값(orchestrator)을 실행
.venv/bin/jarvis ask "What is 17 * 23? Reply with just the number."
```

예상 결과: `JARVIS-WIRED-OK`, 그다음 `391`.

*두 가지 테스트를 하는 이유:* 7a는 원시 **엔진→프록시→모델** 경로를 단독으로 검증합니다
(도구 없이, 멀티턴 없이) — 실패한다면 문제는 연결이지 에이전트가 아닙니다. 7b는 실제로
설정한 **기본 에이전트**가 처음부터 끝까지(end-to-end) 동작함을 검증합니다.

---

## 8. 일상적인 사용

```bash
cd /Users/ohama/projs/openjarvis-test/OpenJarvis
.venv/bin/jarvis ask "..."      # 일회성
.venv/bin/jarvis chat           # 대화형 세션
.venv/bin/jarvis serve          # :8090에서 HTTP API ([server] extra 필요)
```

선택적 편의 설정 — venv를 활성화하거나 PATH에 추가하면 그냥 `jarvis`만 입력할 수 있습니다:

```bash
source .venv/bin/activate       # 셸별로
# 또는 심볼릭 링크: ln -s "$PWD/.venv/bin/jarvis" ~/bin/jarvis
```

---

## 문제 해결

| 증상 | 원인 | 해결 |
|---------|-------|-----|
| Python 버전에서 `jarvis` 설치 실패 | 시스템 Python이 ≥3.14 | venv 재생성: `uv venv --python 3.13 .venv` (3단계) |
| 연결 거부 / 엔진에 도달 불가 | LiteLLM `:4000` 또는 MLX 백엔드가 중단됨 | 0단계 `curl` 확인을 다시 실행; 스택 재시작 |
| `System message must be at the beginning` (MLX 오류) | 어떤 에이전트가 대화 중간에 시스템 메시지를 삽입했으며, 일반 `qwen-122b`는 role-shim을 건너뜀 | `default_model = "qwen-122b-claude"`로 설정 (`:8011`의 role-shim을 경유) |
| 서버가 시작되지 않음 / 포트 사용 중 | `:8000`이 `mlx_lm.server`와 충돌 | `[server] port = 8090`인지 확인 (5단계) |
| `web_search` 도구 오류 | 검색 extra가 설치되지 않음 | `VIRTUAL_ENV=.venv uv pip install -e ".[server,tools-search]"` |

---

## 설치/변경된 항목 (정리 또는 감사용)

- 저장소 클론: `/Users/ohama/projs/openjarvis-test/OpenJarvis` (~144 MB)
- 가상환경: 그 안의 `.venv` (Python 3.13.13, uv로 프로비저닝됨)
- 설정: `~/.openjarvis/config.toml`
- 런타임 상태 (사용 시 생성됨): `~/.openjarvis/memory.db`, `~/.openjarvis/traces.db`
- **변경하지 않음:** LiteLLM/MLX 스택, 시스템 Python, Ollama (설치한 적 없음)

OpenJarvis를 완전히 제거하려면: 클론 디렉터리와 `~/.openjarvis/`를 삭제하세요.

---

## 버전 참고

`main` 클론은 자신을 개발 빌드(`0.0.1.dev1+…`)로 보고하며, 릴리스 `v1.0.3`이 사용
가능하다고 표시합니다. 개발 빌드는 정상적으로 테스트되었습니다. 대신 태그된 릴리스를
사용하려면:

```bash
cd OpenJarvis && git fetch --tags && git checkout v1.0.3 && VIRTUAL_ENV=.venv uv pip install -e ".[server]"
```
