# OpenJarvis Telegram 채널 — 설정 및 사용법

**모든 기기(iPhone 포함)의 Telegram 앱**에서 로컬 `qwen-122b`와 대화하세요.
비공개 봇에 보낸 메시지는 이 Mac에서 실행 중인 OpenJarvis가 응답합니다.

- **동작 방식:** OpenJarvis는 Telegram의 **롱 폴링(long polling)** 을 사용합니다. 즉,
  Telegram 서버에서 메시지를 *가져옵니다*. 따라서 **공개 URL, 열린 포트, 터널이 전혀 필요 없습니다.**
  Mac은 인터넷 연결과 게이트웨이 데몬 실행만 있으면 됩니다.
- **모델 실행 위치:** 로컬(LiteLLM을 통한 `qwen-122b`). Telegram 서버를 거치는 것은
  메시지 텍스트뿐이며, 추론(inference)은 여러분의 기기에서 이루어집니다.
- **설정 파일:** `~/.openjarvis/config.toml` (`[channel]` / `[channel.telegram]`
  섹션은 이미 추가되어 있습니다 — 2단계 참고).

---

## 이미 완료된 작업 vs. 직접 해야 할 작업

| 완료된 작업 | 직접 해야 할 작업 (한 번만) |
|--------------|------------------------|
| `[channel] enabled = true`, `default_channel = "telegram"` 설정에 추가됨 | **@BotFather** 로 봇을 만들고 **토큰**을 발급받기 |
| `[channel.telegram]` 섹션이 플레이스홀더 토큰과 함께 구성됨 | 토큰을 설정 파일에 붙여넣기 (2단계) |
| `python-telegram-bot` 설치됨; 설정 파싱 확인 완료 | (권장) 봇을 본인 채팅 ID로 제한하기 (3단계) |

토큰이 아직 플레이스홀더 `REPLACE_WITH_BOTFATHER_TOKEN` 이기 때문에
`jarvis channel status --channel-type telegram` 은 현재 **disconnected** 를 보고합니다.

---

## 1단계 — 봇 만들기 (Telegram 앱에서)

1. Telegram을 열고 **@BotFather** (파란 체크가 있는 공식 계정)를 검색해 대화를 시작합니다.
2. `/newbot` 을 보냅니다.
3. **이름**(표시 이름, 무엇이든 가능)과 **사용자 이름**(반드시 `bot` 으로 끝나야 함,
   예: `ohama_jarvis_bot`)을 지정합니다.
4. BotFather가 다음과 같은 **토큰**으로 응답합니다:
   `123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
   비밀로 유지하세요 — 봇을 제어할 수 있는 전체 자격 증명입니다.

선택적인 BotFather 부가 기능: `/setdescription`, `/setuserpic`, 그리고 나중에 그룹 채팅에서
사용하려면 `/setprivacy` → **Disable**.

---

## 2단계 — 토큰 추가하기

`~/.openjarvis/config.toml` 을 편집해 `[channel.telegram]` 섹션에서 플레이스홀더를
교체합니다:

```toml
[channel]
enabled = true
default_channel = "telegram"

[channel.telegram]
bot_token = "123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"   # <- 여러분의 BotFather 토큰
# allowed_chat_ids = "..."   # 3단계 참고
```

**대안 (비밀 정보를 파일에서 제외):** `bot_token = ""` 로 두고 게이트웨이를 실행하는
셸에서 환경 변수를 내보냅니다:
```bash
export TELEGRAM_BOT_TOKEN="123456789:AAE..."
```
OpenJarvis는 `bot_token` 이 비어 있을 때 `TELEGRAM_BOT_TOKEN` 을 읽습니다.

---

## 3단계 — 본인 전용으로 제한하기 (권장)

허용 목록(allow-list)이 없으면 **봇의 사용자 이름을 알아낸 누구든 사용할 수 있습니다**
(그리고 여러분의 로컬 GPU를 소모하거나 활성화한 도구를 읽을 수 있습니다). 본인의 채팅 ID로
제한하세요:

1. 토큰만 설정한 상태로 게이트웨이를 임시로 시작합니다 (4단계).
2. Telegram에서 봇으로 아무 메시지나 보냅니다.
3. `jarvis channel status --channel-type telegram` 을 실행하거나 (또는 `gateway logs` 확인)
   수신된 채팅 ID를 확인합니다. 또는 Telegram에서 **@userinfobot** 에 메시지를 보내면
   숫자 ID로 응답합니다.
4. 이를 설정에 넣고 게이트웨이를 재시작합니다:
   ```toml
   [channel.telegram]
   allowed_chat_ids = "123456789"          # 여러 개는 쉼표로 구분: "111,222"
   ```

이후 다른 채팅 ID에서 오는 메시지는 무시됩니다.

---

## 4단계 — 게이트웨이 시작하기

**게이트웨이**는 활성화된 채널을 실행하는 데몬입니다.

```bash
cd /Users/ohama/projs/openjarvis-test/OpenJarvis

.venv/bin/jarvis gateway start      # 멀티 채널 게이트웨이 시작 (백그라운드 데몬)
.venv/bin/jarvis gateway status     # 실행 여부 확인
.venv/bin/jarvis gateway logs       # 로그 확인 (3단계의 채팅 ID 확인에 유용)
.venv/bin/jarvis gateway stop       # 중지
```

채널 자체가 연결되었는지 확인합니다:
```bash
.venv/bin/jarvis channel status --channel-type telegram   # 원하는 결과: Status: connected
```

> 봇이 응답하려면 Mac이 깨어 있어야 합니다. 상시 실행하려면 App Nap / 절전을
> 비활성화하거나, 전원을 연결한 채 "절전 방지"를 켜 두세요.

---

## 사용하기

- Telegram에서 봇을 열고(BotFather 링크 또는 사용자 이름 검색) **메시지를 보내기만**
  하면 됩니다 — 예: *"핵융합 에너지 관련 뉴스를 3개 항목으로 요약해 줘"*.
- 봇은 설정의 **기본 에이전트 + 모델**(`orchestrator` + `qwen-122b`)로 응답합니다.
  긴 응답은 Telegram의 4096자 제한에서 자동으로 분할됩니다.
- **Telegram에 로그인된 모든 기기**(iPhone, iPad, 데스크톱)에서 동작합니다 — 봇이
  특정 기기가 아니라 여러분의 Telegram 계정에 존재하기 때문입니다.

### CLI에서 아웃바운드 메시지 보내기 (알림)
스크립트에서 Telegram *으로* 메시지를 푸시할 수도 있습니다:
```bash
.venv/bin/jarvis channel send telegram "Build finished ✅"
```

---

## 관리 및 문제 해결

| 증상 | 유력한 원인 | 해결 방법 |
|---------|--------------|-----|
| `Status: disconnected` | 토큰이 여전히 플레이스홀더이거나 잘못됨 | 실제 BotFather 토큰을 설정에 입력 (2단계) |
| 봇이 전혀 응답하지 않음 | 게이트웨이 미실행 | `jarvis gateway start`; `gateway logs` 확인 |
| 봇이 유독 나만 무시함 | 본인 채팅 ID가 `allowed_chat_ids` 에 없음 | 본인 ID를 추가하거나, 테스트를 위해 허용 목록을 비우기 |
| 노트북을 닫으면 응답이 멈춤 | Mac이 절전 상태 | 깨어 있게 유지 / 전원 연결 |
| 다른 사람이 사용할 수 있음 | 허용 목록 없음 | `allowed_chat_ids` 설정 (3단계) |
| 토큰 유출 | — | BotFather에서 `/revoke` 로 새 토큰을 발급한 뒤 설정 업데이트 |

유용한 명령:
```bash
.venv/bin/jarvis config show                         # 채널 설정이 로드되었는지 확인
.venv/bin/jarvis channel status --channel-type telegram
.venv/bin/jarvis gateway logs
```

---

## 보안 참고 사항

- 봇을 정말로 공개하려는 것이 아니라면 **`allowed_chat_ids` 를 설정하세요** — 봇은 로컬
  컴퓨팅 자원을 소모하며 기본 에이전트가 가진 모든 도구(`shell_exec`, `file_write`, …)를 호출할 수 있습니다.
- 허용 목록을 잠그지 않을 경우 Telegram 대상 구성에 **덜 강력한 기본 에이전트/도구**를
  부여하는 것을 고려하세요.
- **토큰은 비밀입니다.** 파일에 커밋하기보다 `TELEGRAM_BOT_TOKEN` 환경 변수를 선호하세요.
  유출된 경우 BotFather에서 `/revoke` 하세요.
- 메시지 텍스트는 **Telegram 서버**를 거칩니다(봇의 경우 종단 간 암호화되지 않음).
  모델과 여러분의 데이터는 로컬에 유지되지만, 일반적인 Telegram 채팅에 넣지 않을 비밀은
  보내지 마세요.

---

## 관련 문서
- `USING-OpenJarvis.md` — 전체 CLI/사용 가이드 (Telegram은 여러 채널 중 하나)
- `LLM-STACK-REFERENCE.md` — 봇이 응답에 사용하는 로컬 모델 엔드포인트
- `SETUP-OpenJarvis.md` — 이 OpenJarvis 설치가 구축된 방법
