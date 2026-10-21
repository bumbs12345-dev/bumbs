# Claude Code 도구 정리 (2026-10-20)

오늘 대화에서 살펴본 세 가지를 정리한 문서입니다.

1. [Claude Code 사무실 화면 만들기](#1-claude-code-사무실-화면-만들기) (스레드 cowboy76 게시물)
2. [Headroom: 토큰 압축 도구](#2-headroom-토큰-압축-도구) (틱톡 "신영선의 AI탐구")
3. [task-observer: 스킬을 개선해 주는 스킬](#3-task-observer-스킬을-개선해-주는-스킬) (틱톡 "신영선의 AI탐구")

> ⚠️ 셋 다 **내 PC에서 실행하는 Claude Code**에서 쓰는 도구예요.
> Claude 앱의 클라우드 세션에는 바로 적용되지 않아요.

---

## 1. Claude Code 사무실 화면 만들기

Claude Code가 지금 무엇을 하는지 아이소메트릭 사무실 화면으로 보여 주는 프로젝트예요.
파일을 읽을 때는 자료실, 명령을 실행할 때는 서버실로 캐릭터가 이동하고, 서브에이전트는 엘리베이터를 타고 출근해요.

### 작동 원리

```
Claude Code (터미널)
   │  hooks: 도구를 쓸 때마다 이벤트 발생
   ▼
로컬 서버 (Node.js 등)  ← 이벤트를 받아서
   │  WebSocket으로 실시간 전달
   ▼
브라우저 화면 (아이소메트릭 사무실)
   → 캐릭터가 도구 종류에 맞는 방으로 이동
```

### 1단계: hooks 등록 (`~/.claude/settings.json`)

```json
{
  "hooks": {
    "PreToolUse":   [{ "matcher": "*", "hooks": [{ "type": "command", "command": "curl -s -X POST localhost:4000/event -d @-" }] }],
    "PostToolUse":  [{ "matcher": "*", "hooks": [{ "type": "command", "command": "curl -s -X POST localhost:4000/event -d @-" }] }],
    "Notification": [{ "hooks": [{ "type": "command", "command": "curl -s -X POST localhost:4000/event -d @-" }] }],
    "Stop":         [{ "hooks": [{ "type": "command", "command": "curl -s -X POST localhost:4000/event -d @-" }] }],
    "SubagentStop": [{ "hooks": [{ "type": "command", "command": "curl -s -X POST localhost:4000/event -d @-" }] }]
  }
}
```

hook은 표준입력으로 JSON을 받아요. 이 JSON에는 `tool_name`, `tool_input`, `session_id`, `cwd`가 들어 있어요.

### 2단계: 도구를 사무실 공간에 대응시키기

| Claude 동작 | 사무실 위치 |
|---|---|
| `Read`, `Grep`, `Glob` | 📚 자료실 (책 펴기) |
| `Bash` | 🖥️ 서버실 랙 앞 |
| `Edit`, `Write` | 💻 내 자리 책상 |
| `WebSearch`, `WebFetch` | 🌐 정보검색실 |
| `Notification` (사용자 응답 대기) | 🤔 자리에서 턱 괴고 고민 |
| `Task`/`Agent` (서브에이전트) | 🛗 새 캐릭터가 엘리베이터 타고 출근 |
| `Stop` | ☕ 휴게실 |

### 3단계: 화면 만들기

- **로컬 서버**: Express와 `ws`(WebSocket)로 POST 이벤트를 받아서 브라우저에 전달해요.
- **프론트엔드**: Phaser나 PixiJS를 쓰거나, CSS transform만으로 아이소메트릭 맵을 만들어요.
- **층 안내**: hook JSON의 `cwd`(프로젝트 폴더)와 `session_id`로 층을 나눠요.
- **활동 로그**: 이벤트를 시간순으로 쌓아서 보여 줘요.
- **과거 기록**: `~/.claude/projects/*/*.jsonl`에 있는 대화 기록 파일을 읽어요.

### 해 볼 만한가?

- 👍 블로그나 스레드 **글감**으로 좋아요. 원 게시물 반응이 좋아요 2천, 리포스트 546개였어요.
- 👍 내 PC에서 Claude Code로 여러 작업을 동시에 돌리면 상태를 한눈에 볼 수 있어요.
- 👎 실용성만 보면 꼭 필요하지는 않아요. 예쁘게 만들려면 그림 작업에 시간이 많이 들어요.
- **추천 순서**: 방 4개와 캐릭터 1명으로 간단히 만들기 → 녹화해서 글로 올리기 → 반응이 좋으면 기능 추가하기

---

## 2. Headroom: 토큰 압축 도구

- 저장소: https://github.com/headroomlabs-ai/headroom
- 라이선스: Apache-2.0 (무료, 오픈소스)

**하는 일:** Claude가 읽는 파일, 로그, 명령 실행 결과를 **Claude에게 보내기 전에 압축해서** 토큰을 줄여 줘요.
압축은 내 컴퓨터 안에서만 하고, 원본은 따로 보관해 두다가 필요하면 Claude가 `headroom_retrieve`로 다시 꺼내 볼 수 있어요.

### 설치와 사용

```bash
uv tool install --python 3.13 "headroom-ai[all]"   # 또는 pip install "headroom-ai[all]"
headroom wrap claude       # Headroom을 거쳐서 Claude Code 실행
headroom dashboard         # 얼마나 아꼈는지 확인 (proxy 실행 중일 때)
headroom doctor            # 상태 점검
headroom unwrap claude     # 원래대로 되돌리기
```

### 알아 둘 점

- 절약 효과는 공식 자료 기준 **21~57%**예요. 긴 작업이나 로그, 코드를 많이 읽을 때 효과가 크고, 짧은 대화나 일반 글에는 거의 효과가 없어요.
- **익명 사용 통계 전송이 기본으로 켜져 있어요.** 끄려면 `HEADROOM_OFFLINE=1`(또는 `DO_NOT_TRACK=1`)을 설정하세요.
- `wrap` 하면 코드 탐색 도구 Serena가 `~/.claude.json`에 같이 등록돼요. 빼려면 `--code-memory none` 옵션을 쓰세요.
- 애플실리콘 맥과 리눅스는 바로 설치돼요. 인텔 맥은 Docker로 설치하거나 직접 따로 설정해야 해요.
- 일부 기능(모델 라우팅, 보안 스캔)은 유료 Enterprise 기능이에요.

**추천도: ⭐**. 내 PC에서 Claude Code로 긴 작업을 많이 돌리고 사용량 한도에 자주 걸릴 때 써 보세요.

---

## 3. task-observer: 스킬을 개선해 주는 스킬

- 저장소: https://github.com/rebelytics/one-skill-to-rule-them-all
- 라이선스: CC BY 4.0 (무료, 출처 표기 필요)

**하는 일:** 작업하는 동안 사용자가 고친 부분, 반복되는 패턴, 빠진 부분을 관찰했다가
"이 스킬은 이렇게 고치면 좋겠다", "이런 새 스킬을 만들면 좋겠다"고 **제안**해 줘요.
마음대로 바꾸지 않고, 사용자가 승인해야 반영돼요.

### 설치

```bash
# Node.js 필요, -g는 모든 프로젝트에 적용
npx skills add rebelytics/one-skill-to-rule-them-all --skill task-observer -g
```

수동으로 설치하려면 폴더를 `.claude/skills/task-observer/`에 복사하세요. `references/`와 `scripts/` 하위 폴더도 함께 복사해야 해요.

### 활성화 (꼭 필요)

설치만으로는 동작하지 않아요. 저장소의 `references/environments.md`에 있는 **활성화 문구를 CLAUDE.md에 추가**하거나
session-start hook을 설치한 다음, 새 세션에서 동작하는지 확인하세요.

**추천도: ⭐⭐⭐**. 쿠팡파트너스 글, 애드센스 글, 티스토리 홈처럼 반복해서 쓰는 스킬을 점점 다듬는 데 잘 맞아요.
