# OpenJarvis 사용하기 — 상세 가이드

이 머신에 설치된 OpenJarvis를 실제로 *사용하는* 방법 (로컬
`qwen-122b` / `qwen-35b`에 연결됨 — 설치 방법은 `SETUP-OpenJarvis.md`를,
모델 엔드포인트 세부 정보는 `LLM-STACK-REFERENCE.md`를 참고하세요).

- **CLI 바이너리:** `/Users/ohama/projs/openjarvis-test/OpenJarvis/.venv/bin/jarvis`
- **설정:** `~/.openjarvis/config.toml` (모델 `qwen-122b`, 엔진 → LiteLLM `:4000`, API 서버 포트 `8090`)
- 이 문서 전반에서 `jarvis`는 해당 venv 바이너리를 의미합니다. 그냥 `jarvis`만 입력해서
  실행하려면 `source .venv/bin/activate`를 실행하거나 PATH에 심볼릭 링크를 걸어두세요.

> **"아이폰에서 사용할 수 있나요?"에 대한 답변을 포함합니다 — 7번 섹션을 참고하세요.**

---

## 1. 가장 빠르게 사용하는 방법

```bash
cd /Users/ohama/projs/openjarvis-test/OpenJarvis

# 단발성 질문 (기본 에이전트 + qwen-122b 사용)
.venv/bin/jarvis ask "Summarize the theory of relativity in 3 bullets"

# 대화형 멀티턴 대화
.venv/bin/jarvis chat

# 신규 사용자를 위한 가이드 설정 / 기능 둘러보기
.venv/bin/jarvis quickstart

# 진단 — 설정, 엔진 접근 가능 여부, 모델 등을 점검
.venv/bin/jarvis doctor
```

`ask`는 기본적으로 답변을 스트리밍합니다. 하나의 블록으로 받으려면 `--no-stream`을,
원시 결과 객체를 받으려면 (스크립팅에 유용) `--json`을 추가하세요.

---

## 2. `jarvis ask` — 핵심 명령

자주 쓰는 옵션 (`jarvis ask --help` 기준):

| 플래그 | 용도 |
|------|------|
| `-m, --model` | 모델을 재정의, 예: 더 빠른 답변을 위한 `-m qwen-35b` |
| `-a, --agent` | 에이전트 선택: `simple` (싱글턴, 도구 없음), `orchestrator` (멀티턴 + 도구), `react`, `openhands`. `--agent ''` = 모델과 직접 대화 |
| `--tools` | 이번 호출에 사용할 도구를 제한, 예: `--tools calculator,think` |
| `-t, --temperature` / `--max-tokens` | 샘플링 재정의 |
| `--research` | 인덱싱된 개인 지식(BM25 + 임베딩)으로부터 답변 |
| `-i, --image FILE` / `-S, --screen` | 비전 모델에 이미지 / 스크린샷 전송 |
| `--profile` | 해당 호출의 지연 시간 / 토큰 / 에너지 텔레메트리 출력 |
| `--no-context` | 메모리 컨텍스트 주입 건너뛰기 |

예시:
```bash
.venv/bin/jarvis ask -m qwen-35b "quick: capital of Japan?"
.venv/bin/jarvis ask --agent simple --tools calculator "what is 19% of 4,320?"
.venv/bin/jarvis ask -S "what's on my screen right now?"      # 비전 모델 필요
```

---

## 3. 에이전트, 도구, 스킬

- **에이전트(Agents)**는 요청이 처리되는 *방식*을 결정합니다. `simple`은 그저 답변만 하고,
  `orchestrator`는 여러 턴에 걸쳐 도구를 호출할 수 있으며, `openhands`/`react`는 코드/에이전트
  스타일입니다. 기본값은 `config.toml`(`[agent] default_agent`)에서 설정하거나 호출마다 재정의하세요.
- **영속 에이전트(Persistent agents):** `jarvis agents` — 고유한 페르소나/설정을 가진 이름 있는
  에이전트를 생성하고, 살펴보고, 대화할 수 있습니다.
- **도구(Tools)**는 에이전트가 호출할 수 있는 기능입니다 (`code_interpreter`, `file_read`,
  `file_write`, `shell_exec`, `web_search`, `calculator`, `think`, 메모리/지식
  검색 등). `jarvis tool list`로 목록을 확인하세요. 호출마다 `--tools`로, 또는
  전역적으로 `[tools] enabled = [...]`에서 활성화합니다.
- **스킬(Skills):** `jarvis skill` — 재사용 가능한 저장된 기능. **MCP 서버:** `jarvis add`로
  외부 도구 서버를 추가합니다 (그리고 `[tools.mcp] enabled = true`).

> 참고: 일부 도구는 추가 설치가 필요합니다. `web_search`는 search extra를 필요로 합니다:
> `VIRTUAL_ENV=.venv uv pip install -e ".[server,tools-search]"`.

---

## 4. 메모리, 데이터 연결 및 딥 리서치

OpenJarvis는 *당신의* 데이터를 대상으로 답변할 수 있습니다:

```bash
# 데이터 소스 연결 (Gmail, Obsidian, 로컬 폴더 등)
.venv/bin/jarvis connect            # 사용 가능한 커넥터 확인

# 문서를 지식 저장소에 인덱싱
.venv/bin/jarvis memory index ~/Documents/papers/

# 인덱싱된 데이터를 검색하는 질문
.venv/bin/jarvis ask --research "what did the Q2 planning notes conclude?"

# 또는 로컬 소스를 감지하고 리서치를 실행하는 단발성 자동 설정
.venv/bin/jarvis deep-research-setup
```

메모리/지식은 `~/.openjarvis/*.db`에 저장됩니다 (설정에 따라 기본적으로 sqlite).

---

## 5. 자동화 — 스케줄러, 오퍼레이터, 다이제스트, 워크플로

- `jarvis digest` — "모닝 다이제스트" (다이제스트 설정/에이전트 필요).
- `jarvis scheduler` — 반복 작업 예약 (cron 스타일).
- `jarvis operators` — 영속적이고 예약된 자율 에이전트 (예: 모니터).
- `jarvis workflow` — 다단계 워크플로 목록/실행/검사.

이러한 기능은 Jarvis를 온디맨드 어시스턴트에서 백그라운드 어시스턴트로 바꿔줍니다.

---

## 6. 서버 / 데몬으로 실행하기 (원격 및 앱 접근의 기반)

OpenJarvis는 **OpenAI 호환 HTTP API**를 노출하고 백그라운드 데몬으로 실행할 수 있습니다.

```bash
# 포그라운드 서버 (Ctrl-C로 중지)
.venv/bin/jarvis serve                       # 설정값에 바인딩: 127.0.0.1:8090

# 백그라운드 데몬 + 라이프사이클
.venv/bin/jarvis start                        # 데몬 시작
.venv/bin/jarvis status                       # 실행 중인가?
.venv/bin/jarvis restart / stop
```

서빙을 시작하면 표준 OpenAI API를 사용하므로, LiteLLM을 가리키던 것과 마찬가지로
**다른 소프트웨어가 이를 가리키도록** 할 수 있습니다:

```bash
curl http://localhost:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-122b","messages":[{"role":"user","content":"hi"}]}'
```

> `server` extra가 필요합니다 (여기에는 이미 설치됨). 기본적으로 루프백에만
> 바인딩됩니다 — 다른 기기에서 접근하려면 다음 섹션을 참고하세요.

---

## 7. 아이폰에서 사용할 수 있나요?

**짧은 답변: 예 — 다만 전용 App Store 앱을 통해서는 아닙니다.** OpenJarvis는
데스크톱 GUI(`.dmg`/`.exe`/`.deb`/…)와 CLI를 제공하며, **네이티브 iOS 앱은 없습니다**.
아이폰에서 사용하려면 이 Mac에서 실행 중인 인스턴스에 세 가지 방법 중 하나로 연결합니다.
어떤 방법이든 Mac이 **켜져 있고 해당 데몬이 실행 중**이어야 합니다.

### 옵션 A — 메시징 채널 (가장 쉬움, 네이티브에 가까운 느낌) ✅ 권장

OpenJarvis에는 **멀티채널 게이트웨이(multi-channel gateway)**가 있습니다: 아이폰에 이미 설치된
메시징 앱 안에서 Jarvis와 대화하면, 로컬 `qwen-122b`를 사용해 응답합니다.

지원 채널에는 **Telegram, WhatsApp, Slack, Discord, iMessage/SMS**가 포함됩니다
(iMessage/SMS는 서드파티 **SendBlue** 서비스를 통해).

```bash
.venv/bin/jarvis channel list          # 채널 유형 확인
.venv/bin/jarvis gateway start         # 멀티채널 게이트웨이 데몬 실행
.venv/bin/jarvis gateway status
.venv/bin/jarvis channels imessage-start <CHAT_IDENTIFIER>   # iMessage 경로
```

- **Telegram**이 대개 가장 간단합니다: 봇 토큰을 만들고, 채널 설정에 넣은 뒤,
  `gateway start`를 실행하고, 아이폰의 Telegram 앱에서 봇에게 메시지를 보내면 됩니다.
- **iMessage**는 기본 메시지 앱에서 Jarvis에게 문자를 보낼 수 있지만, SendBlue
  계정(유료 서드파티)이 필요합니다 — 설정이 더 번거롭습니다.
- **아이폰에 이 방법이 가장 좋은 이유:** 네트워크에 아무것도 노출할 필요가 없고,
  인터넷만 있으면 어디서든 동작하며, UI가 이미 신뢰하는 채팅 앱입니다.

### 옵션 B — 네트워크를 통해 HTTP API 접근 🌐

서버를 LAN에 바인딩해서 실행하고 아이폰에서 접속합니다. 이 방법에는 키가 필요합니다
(시작 시 키 없이 루프백이 아닌 바인딩을 **거부**합니다):

```bash
.venv/bin/jarvis auth generate-key                  # API 키 생성
.venv/bin/jarvis serve --host 0.0.0.0 --port 8090   # LAN에 바인딩
```

그런 다음 아이폰에서 (같은 Wi-Fi):
- 아무 **OpenAI 호환 iOS 채팅 앱**(또는 "Get Contents of URL" POST를 사용하는 Apple
  **Shortcuts**)을 `http://<mac-LAN-IP>:8090/v1`, 모델 `qwen-122b`로 가리키고,
  API 키를 `Authorization: Bearer <key>`로 전송합니다.
- 또는 GUI/웹 엔드포인트가 서빙 중이라면 Safari에서 그냥 `http://<mac-LAN-IP>:8090/`를 엽니다.

Mac의 LAN IP는 `ipconfig getifaddr en0`로 확인합니다. 이 방법은 아이폰이 Mac과 **같은
네트워크**에 있을 때만 동작합니다. **외부에서** 접근하려면 **Tailscale** 같은 프라이빗
메시 VPN을 사용하세요 (두 기기에 모두 설치하고, LAN IP 대신 Mac의 Tailscale IP를 사용) —
공용 인터넷에 포트를 노출하지 않습니다.

### 옵션 C — 공용 터널 (어디서든 접근) 🚇

OpenJarvis에는 내장 Cloudflare 터널이 있습니다:

```bash
.venv/bin/jarvis serve --host 127.0.0.1 --port 8090   # 루프백 유지
.venv/bin/jarvis tunnel --port 8090                   # Cloudflare를 통해 노출
.venv/bin/jarvis tunnel status
```

이렇게 하면 아이폰에서 어디서든 열 수 있는 공용 HTTPS URL이 생깁니다. 이 방법을 쓸 때는
**항상 API 키를 활성화**하세요 — 터널은 URL을 아는 누구나 접근할 수 있습니다.

### 어떤 것을 선택해야 할까?

| 원하는 것 | 사용 |
|------|------|
| 가장 간단하고, 채팅 앱 느낌이며, 어디서든 동작 | **A — Telegram 채널** |
| 메시지 앱에서 Jarvis에게 문자 보내기 | A — iMessage (SendBlue 필요) |
| OpenAI 호환 iOS 앱 / Shortcuts를 집에서 사용 | **B — LAN 서버 + 키** |
| 위와 같으나 외부에서도 프라이빗하게 | B + **Tailscale** |
| 어디서든 접근 가능한 공용 URL | **C — Cloudflare 터널** (+ 키) |

### 아이폰 접근 시 보안 유의사항
- API 키 없이 `0.0.0.0`에 바인딩하거나 터널을 열지 **마세요**.
- 라우터의 포트 포워딩보다 Tailscale을 선호하세요.
- 프롬프트가 Mac을 벗어나는 것은 선택한 채널/터널 제공자로 향할 때뿐이며, 모델
  자체는 여전히 로컬의 `qwen-122b`에서 실행됩니다.

---

## 8. 유지 관리

```bash
.venv/bin/jarvis config show      # 어떤 설정이 로드되었는지
.venv/bin/jarvis doctor           # 상태 점검
.venv/bin/jarvis scan             # 환경에 대한 개인정보/보안 감사
.venv/bin/jarvis telemetry ...    # 과거 호출의 지연 시간/토큰/에너지 통계
.venv/bin/jarvis self-update      # 업그레이드 (또는 SETUP 문서의 git/uv 방법 사용)
```

**이 폴더의 관련 문서:**
- `SETUP-OpenJarvis.md` — 이 설치가 어떻게 구성되었는지 (그리고 다시 하는 방법)
- `LLM-STACK-REFERENCE.md` — OpenAI 호환 앱을 위한 LiteLLM → Qwen 엔드포인트 레퍼런스
