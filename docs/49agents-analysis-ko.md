# 49 Agents IDE 분석 정리 (한국어)

> 작성일: 2026-09-18
> 대상 저장소: https://github.com/bmshin94/49-IDE (내 포크)
> 원본 저장소: https://github.com/49Agents/49Agents
> 호스팅 서비스: https://app.49agents.com

---

## 1. 이게 뭐하는 프로젝트야?

**"터미널 탭 14개 알트탭 지옥"을 무한 캔버스 하나로 합쳐주는 2D 에이전트 IDE.**

Claude Code 같은 AI 에이전트를 여러 개 동시에 돌릴 때 창을 왔다갔다 하는 문제를,
줌 가능한 무한 캔버스 위에 터미널·에디터·깃 그래프를 판때기(pane)로 배치해서 해결한다.

| Before | 49 Agents |
|---|---|
| 터미널 탭 14개 | 줌 되는 캔버스 하나 |
| 머신마다 SSH | SSH 없이 전 머신 |
| 알트탭으로 Claude 확인 | 모든 pane에 Claude 상태 표시 |
| 폰에서 작업 불가 | 어떤 기기에서든 접속 |

---

## 2. 폴더 구조

| 경로 | 역할 |
|---|---|
| `agent/` | 내 PC에서 도는 일꾼. tmux 세션 생성, 깃/이슈/파일 정보 수집 후 WebSocket 전송 |
| `agent/services/tmux.js` | 실제 터미널(tmux) 제어 |
| `agent/services/conversations.js` | `~/.claude/projects/*.jsonl` 파싱 → Claude 작업 상태 판별 |
| `agent/services/gitGraph.js`, `beads.js` | 깃 그래프 / Beads 이슈 연동 |
| `cloud/` | 중계(릴레이) 서버. Express + SQLite + ws |
| `cloud/src/ws/relay.js` | 에이전트 ↔ 브라우저 중계 핵심 |
| `cloud/src/billing/` | 요금제 정의(`tiers.js`) + 서버측 제한 적용(`enforcement.js`) |
| `cloud/src-client/` | 캔버스 UI 소스 (바닐라 JS 모듈 30여 개) |
| `desktop/` | macOS 트레이 앱 (Electron) |
| `49ctl` | 만능 실행 스크립트 |

### 아키텍처

```
내 PC (49-agent)  ⇄  릴레이 서버  ⇄  브라우저 (폰/태블릿/노트북)
                       WSS              WSS
```

터미널 I/O는 **중계만 하고 서버에 저장하지 않는다.**

---

## 3. 설치 및 사용법

### 사전 요구사항
- Node.js 18+ (`.nvmrc`)
- **tmux** — 실제 터미널 세션
- **ttyd** — 터미널을 웹으로 노출

```bash
brew install tmux ttyd          # macOS
git clone https://github.com/49Agents/49Agents.git
cd 49Agents
./49ctl setup    # 1) 단일 머신  2) 여러 머신
./49ctl start
```

→ 브라우저에서 `http://localhost:1071` (회원가입·로그인 없음)

### 주요 명령어

| 명령어 | 설명 |
|---|---|
| `./49ctl status` | 실행 중인 프로세스 확인 |
| `./49ctl logs [cloud\|agent\|all]` | 로그 확인 |
| `./49ctl stop` / `restart` | 중지 / 재시작 |
| `./49ctl build` | 클라이언트 자산 + 에이전트 tarball 재빌드 |

macOS는 GitHub Releases의 `.dmg`로도 설치 가능
(`xattr -cr /Applications/49Agents.app` 한 번 실행 필요).

---

## 4. 플러그인? 스킬? MCP? → 셋 다 아님

**독립적인 웹 애플리케이션**이다. Claude Code에 붙는 확장이 아니라,
Claude Code를 tmux 안에서 실행시키고 ttyd로 웹에 띄워 캔버스에 배치하는
**"관제탑" 성격의 별도 앱**이다.

---

## 5. API 토큰이 필요한가? → 기본은 불필요

터미널에서 `claude`를 그대로 실행하므로 기존 구독을 그대로 사용한다.
다만 코드상 토큰 관련 3가지가 존재:

| 종류 | 위치 / 설명 |
|---|---|
| 에이전트 토큰 | `~/.49agents/agent.json` (자동 생성, PC↔서버 연결용) |
| Claude 사용량 HUD | `~/.claude/.credentials.json` 또는 macOS 키체인의 OAuth 토큰을 **읽어서** `api.anthropic.com/api/oauth/usage` 호출 |
| 로그인 | 호스팅판만 `AUTH_MODE=oauth` (GitHub/Google). 자체 호스팅은 `open` |

---

## 6. 왜 GitHub에서 주목받았나 (분석)

1. **실제로 아픈 문제** — 멀티 에이전트 병렬 작업의 창 관리 지옥
2. **강력한 Before/After 데모** — README 상단 영상·스크린샷
3. **진입장벽 제로** — 두 줄 실행, 로그인 없음
4. **프라이버시** — 터미널 데이터 서버 미저장
5. **영리한 재활용** — 터미널을 직접 구현하지 않고 검증된 tmux + ttyd 조합
6. **새 카테고리 선점** — VSCode·Cursor와 경쟁하지 않는 "2D IDE"

---

## 7. 로컬 에이전트 구축에 참고할 부분

| 배울 것 | 위치 |
|---|---|
| 릴레이 패턴 (포트 개방 없이 외부 접근) | `cloud/src/ws/relay.js`, `agentHandler.js`, `browserHandler.js` |
| 에이전트 통신 규약 (양쪽 공유) | `agent/src/protocol.js` + `cloud/src/protocol.js` |
| CLI 감싸기 | `agent/services/tmux.js` |
| **Claude 상태 판별 (핵심)** | `agent/services/conversations.js` — JSONL 파싱 |
| 요금제 서버측 강제 | `cloud/src/billing/enforcement.js` |

---

## 8. 라이선스 (중요)

**BSL 1.1**, Change Date **2030-02-26** → 이후 MIT 전환.

### Additional Use Grant
상용 목적 포함 프로덕션 사용 **가능**. 단 조직이 아래를 **모두** 만족할 것:
- (a) 최근 12개월 총매출 **100만 USD 미만**
- (b) 총 투자유치/부채 **100만 USD 미만**

둘 중 하나라도 초과하면 상용 라이선스 구매 필요.
개인·비상업적 사용은 항상 허용.

### 안전 체크리스트
- [x] 매출/투자 100만 달러 미만 유지
- [x] `LICENSE` 파일 유지 (BSL 고지 의무)
- [ ] **"49Agents" 이름·로고 사용 금지** (상표권은 라이선스와 별개 — 다른 이름 사용)
- [x] 출처 명시 ("49Agents 기반")

---

## 9. 수익화 아이디어

### 참고: 원본의 수익 모델 (`cloud/src/billing/tiers.js`)

| 등급 | 에이전트 | 터미널 | 노트 이미지 |
|---|---|---|---|
| free | 2 | 7 | 10 |
| pro | 6 | 40 | 100 |
| poweruser | 24 | 160 | 400 |

`enforcement.js` 주석: *"The agent has zero awareness of tiers"*
→ **자체 호스팅은 무제한, 호스팅 서비스만 과금**하는 오픈소스 수익화 정석 구조.

### 아이디어 7개

| # | 아이디어 | 과금 | 난이도 | 비고 |
|---|---|---|---|---|
| 1 | **한국형 호스팅 릴레이** | 월 9,900 / 29,000원 | 중 | 한국어 UI, 카카오톡 알림, 카카오·네이버 로그인, 토스페이먼츠, 국내 리전 |
| 2 | **기업 온프레미스 구축·유지보수** | 구축 500만~2,000만 / 월 50만~150만 | 하 | 금융·공공·게임사 타깃. 현금화 최속 |
| 3 | **AI 사용량·생산성 분석 SaaS** | 좌석당 월 5,000원 | 중~상 | 독자 구현 → 라이선스 무관. **본진 추천** |
| 4 | 카카오톡 알림 애드온 | 월 3,000원 | 하 | `cloud/src/notifications/`에 현재 `discord.js`만 존재 |
| 5 | 교육·콘텐츠 (유튜브/강의/뉴스레터) | 강의 15~20만원 | 하 | 위험 0, 1~3번의 마케팅 채널 |
| 6 | 워크플로 템플릿 마켓 | 건당 5,000~15,000원 | 하 | `routes/layouts.js` 활용 |
| 7 | 업스트림 기여 → 커리어 자산 | 간접 | 하 | 프리랜서 단가·신뢰도 상승 |

### 추천 로드맵

```
0~1개월   5번(콘텐츠) 시작 + 4번(카톡 알림) 개발    → 저비용 반응 확인
1~3개월   1번(한국형 호스팅) 오픈                  → 첫 유료 고객
3~6개월   3번(분석 SaaS) 개발 + 2번 기업 문의 대응  → 본진
```

---

## 10. React / PHP로 재구현 가능한가?

### React → 가능, 오히려 유리
현재는 바닐라 JS(`cloud/src-client/modules/` 30여 개). React로 간다면:
- 캔버스: **React Flow** 또는 **tldraw**
- 터미널: `xterm.js` + `@xterm/addon-fit`
- 상태: Zustand

### PHP → 비추천
핵심이 **WebSocket 상시 연결**이라 요청-응답 모델인 PHP와 체질이 안 맞음.
(Swoole/Ratchet 필요 → 배포 복잡도 급상승, tmux·ttyd 프로세스 관리도 불편)

### 권장 조합

```
프론트  →  React
백엔드  →  Node.js (현행 유지) 또는 Go
PHP     →  관리자 페이지 · 결제 · 통계 API 정도만
```

---

## 참고 링크

- 원본 저장소: https://github.com/49Agents/49Agents
- 내 포크: https://github.com/bmshin94/49-IDE
- 호스팅 서비스: https://app.49agents.com
- 릴리스(.dmg): https://github.com/49Agents/49Agents/releases/latest
- Beads 이슈 트래커: https://github.com/steveyegge/beads
- ttyd: https://github.com/tsl0922/ttyd
