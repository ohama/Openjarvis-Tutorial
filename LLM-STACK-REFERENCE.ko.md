# 로컬 LLM 스택 레퍼런스 — LiteLLM → Qwen 122B / 35B (MLX)

**목적:** 이 문서는 다른 애플리케이션이 이 머신의 로컬 Qwen 모델과 통신하는 데 필요한 모든 것을 담고 있습니다. 어떤 소프트웨어든 **OpenAI 호환(OpenAI-compatible)** 엔드포인트를 지정할 수 있다면, 스택을 직접 들여다볼 필요 없이 이 문서만으로 설정이 가능합니다.

- **호스트:** Apple M4 Max, 128 GB, macOS, arm64
- **서빙:** **LiteLLM** 프록시 뒤에서 동작하는 Apple **MLX** (`mlx_lm.server`)
- **한 가지만 기억하세요:** 앱을 **`http://localhost:4000/v1`** 로 지정하고, 아래 표에서 모델 이름을 하나 고르면 됩니다.

---

## TL;DR — 대부분의 앱에 필요한 값은 단 두 가지

```
OpenAI-compatible Base URL : http://localhost:4000/v1
API key                    : (none required — send any non-empty string if the field is mandatory)
Model name                 : qwen-122b   (heavy)   |   qwen-35b   (fast)
```

> **OpenAI 호환(OpenAI-compatible):** 위 base URL은 표준 OpenAI REST API
> (`/v1/models`, `/v1/chat/completions`, SSE를 통한 스트리밍)를 구현합니다. 커스텀 OpenAI base URL
> 또는 "OpenAI 호환 프로바이더(OpenAI-compatible provider)"를 지원하는 소프트웨어(OpenJarvis, Open WebUI,
> LibreChat, Continue, Cursor, LangChain, `openai` SDK 등)라면 해당 URL과 모델 이름을 설정하는 것만으로
> 동작합니다. MLX 고유의 요소는 클라이언트에 전혀 노출되지 않습니다.

---

## 아키텍처

```
                 ┌────────────────────────────────────────────────────────────┐
   Your app ───► │  LiteLLM proxy      :4000   (OpenAI-compatible front door)   │
 (OpenAI API)    └───────┬───────────────────────────┬────────────────────────┘
                         │                            │
          plain chat     │                            │  role-shim path (system-msg safe)
                         ▼                            ▼
             ┌──────────────────────┐     ┌──────────────────────────────┐
             │ mlx_lm.server :8001  │     │ role_shim :8011 ──► :8001     │  (122B)
             │  qwen122b  (122B)    │     │ role_shim :8012 ──► :8000     │  (35B)
             ├──────────────────────┤     └──────────────────────────────┘
             │ mlx_lm.server :8000  │
             │  qwen36-35b (35B)    │
             └──────────────────────┘

  (claude-code-proxy :8082 sits in front for Anthropic-API clients — see note below)
```

모든 구성 요소는 `127.0.0.1`(루프백 전용)에서 동작합니다. 의도된 진입점은 LiteLLM 프록시이며,
`:8000`/`:8001` MLX 서버와 shim은 내부용입니다.

---

## 모델 카탈로그 (`:4000` 상의 LiteLLM 별칭)

| 모델 이름 | 크기 | 라우팅 대상 | 경유 경로 | 용도 |
|------------|------|-----------|--------------|------------|
| `qwen-122b` | 122B | MLX `:8001` | 직접 | 최고 품질 / 에이전트 / 추론 |
| `qwen-35b` | 35B | MLX `:8000` | 직접 | 빠름 / 경량 / 고처리량 |
| `qwen-local` | 35B | MLX `:8000` | 직접 | `qwen-35b`의 레거시 별칭 |
| `qwen-122b-claude` | 122B | MLX `:8001` | role-shim `:8011` 경유 | 클라이언트가 system/developer 메시지를 순서에 맞지 않게 보낼 때의 122B |
| `qwen-35b-claude` | 35B | MLX `:8000` | role-shim `:8012` 경유 | 35B, 동일한 role-safe 경로 |
| `qwen-122b-codex` | 122B | MLX `:8001` | Responses→chat 브릿지 + role-shim | OpenAI **Responses API** / Codex 스타일 클라이언트 |

**어떤 것을 고를까:**
- 일반적인 OpenAI 호환 앱 → **`qwen-122b`** 또는 **`qwen-35b`**.
- MLX 오류 *"System message must be at the beginning"* 이 발생하면(일부 에이전트
  프레임워크가 대화 중간에 system 메시지를 삽입함) → 메시지 role을 정규화하는 **`*-claude`**
  변형으로 전환하세요.
- 클라이언트가 **Responses API**(chat/completions가 아님)를 사용한다면 → **`qwen-122b-codex`**.

---

## 서빙 파라미터 (실행 시 고정)

| 백엔드 | 모델 디렉터리 | 포트 | 최대 토큰 | Thinking |
|---------|-----------|------|-----------|----------|
| 122B | `/Users/ohama/llm-system/models/qwen122b` | `:8001` | 16384 | 비활성화 (`enable_thinking:false`) |
| 35B | `/Users/ohama/llm-system/models/qwen36-35b` | `:8000` | 8192 | 비활성화 (`enable_thinking:false`) |

- `max-tokens`는 **출력** 길이에 대한 서버 측 상한입니다. 더 많이 요청하면 이 값으로 제한됩니다.
- Thinking/추론 트레이스는 서버 측에서 비활성화되어 있으므로, 응답은 최종 답변만 포함합니다.

---

## 스택이 정상 동작하는지 확인 (앱을 설정하기 전에 실행)

```bash
# 1. Is the proxy answering, and what models does it expose?
curl -s http://localhost:4000/v1/models | python3 -m json.tool

# 2. Does the 122B model actually generate?
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-122b","messages":[{"role":"user","content":"say OK"}],"max_tokens":5}'

# 3. Same check for the 35B
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-35b","messages":[{"role":"user","content":"say OK"}],"max_tokens":5}'
```

(1)이 별칭들을 나열하고 (2)/(3)이 completion을 반환한다면, 어떤 OpenAI 호환 앱이든
동작합니다. 그렇지 않다면 문제는 앱이 아니라 스택에 있습니다.

---

## 일반적인 OpenAI 호환 소프트웨어 설정 방법

base URL과 모델을 설정하고, 키는 비워 두거나 아무 플레이스홀더 값이나 사용하세요.

**일반 (env vars used by the `openai` SDK, LangChain, many tools):**
```bash
export OPENAI_BASE_URL="http://localhost:4000/v1"
export OPENAI_API_KEY="local"        # any non-empty value; the proxy ignores it
# then use model = "qwen-122b"
```

**Python `openai` SDK:**
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:4000/v1", api_key="local")
resp = client.chat.completions.create(
    model="qwen-122b",
    messages=[{"role": "user", "content": "Hello"}],
)
```

**TOML/YAML 설정 앱 (예: OpenJarvis의 OpenAI 호환 엔진):**
```toml
# OpenAI-compatible engine → point host at the LiteLLM proxy, pick a model alias
[engine]
default = "mlx"                     # any "OpenAI-compatible" engine label works
[engine.mlx]
host = "http://localhost:4000"      # NOT the raw MLX port; the proxy is the front door
# model = "qwen-122b"
# API key only needed if you later enable a LiteLLM master key (see below)
```

**경험칙:** 앱이 "OpenAI API base URL", "커스텀 엔드포인트(custom endpoint)",
또는 "OpenAI 호환 프로바이더(OpenAI-compatible provider)"를 요구하는 곳이면 어디든
`http://localhost:4000/v1` 을 입력하고 카탈로그에서 모델 이름을 선택하세요.

---

## 참고 사항 및 주의점

- **현재 인증 없음.** 프록시는 현재 인증되지 않은 요청을 허용합니다.
  `agent-stack/litellm/config.yaml` 내부의 API 키는 LiteLLM이 MLX 백엔드에 접근할 때 사용되며,
  클라이언트는 이 키가 필요 없습니다. 나중에 LiteLLM **master key**를 활성화하면, 모든 앱이
  `Authorization: Bearer <key>` 를 전송해야 합니다.
- **`/v1` 접미사.** 앱이 전체 base URL을 요구할 때는 `/v1` 을 포함하세요. 일부 앱은
  이를 스스로 덧붙이므로, 오류에 `/v1/v1` 이 보이면 접미사를 제거하세요.
- **포트 `:8000` 은 사용 중** 입니다. 35B MLX 서버가 점유하고 있습니다. 자체 서버를 가진 다른 앱을
  실행한다면 다른 포트에 바인딩하세요.
- **루프백 전용.** LAN에 노출되는 것은 없습니다. 다른 기기에 서비스를 제공하려면 프록시 바인드
  주소를 변경하고 키를 추가해야 하는데, 이는 이 문서의 범위를 벗어납니다.
- **Anthropic-API 클라이언트**(예: Claude Code)는 `:8082` 의 `claude-code-proxy` 를 사용할 수 있으며,
  이는 Anthropic → chat으로 변환하여 동일한 스택으로 전달합니다. 표준 앱은 위의
  OpenAI 호환 `:4000` 경로를 사용해야 합니다.

---

## 컴포넌트 맵 (유지보수/디버깅 전용 — 앱에는 필요 없음)

| 컴포넌트 | 명령 | 포트 | 설정 / 로그 |
|-----------|---------|------|---------------|
| LiteLLM proxy | `agent-stack/venv/bin/litellm --config agent-stack/litellm/config.yaml --port 4000` | `:4000` | `agent-stack/litellm/config.yaml` |
| MLX 122B | `mlx_lm.server --model .../qwen122b --port 8001 --max-tokens 16384` | `:8001` | `llm-system/services/logs/122b.*` |
| MLX 35B | `mlx_lm.server --model .../qwen36-35b --port 8000 --max-tokens 8192` | `:8000` | `llm-system/services/logs/36-35b.*` |
| Role-shim (122B) | `role_shim.py --listen 8011 --upstream http://localhost:8001` | `:8011` | `llm-system/services/logs/role-shim.*` |
| Role-shim (35B) | `role_shim.py --listen 8012 --upstream http://localhost:8000` | `:8012` | `llm-system/services/logs/role-shim-35b.*` |
| Claude-code proxy | `claude-code-proxy -s` | `:8082` | `agent-stack/claude-code-proxy/` |

**Role-shim의 목적:** `mlx_lm.server` 는 모든 system 메시지가 첫 번째 메시지여야 함을 요구하며,
OpenAI Responses의 `developer` role을 거부합니다. shim은 메시지를 재작성하여
(첫 번째 `system`/`developer` → `system` 으로 맨 앞에 유지, 이후의 것들 → `user`), 스트리밍을 포함한
나머지 모든 것을 바이트 단위 그대로 전달합니다. 이것이 `*-claude` / `-codex`
별칭이 존재하는 이유입니다.
