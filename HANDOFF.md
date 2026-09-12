# HANDOFF — 에이전트 통제 수단(Agent Control Surface) 조사

- 작업 유형: 조사(코드 변경 없음)
- 브랜치: `claude/buzz-agent-control-investigation-dyml01`
- 기준 커밋: `8d50870` (main 머지 시점)
- 질문: "buzz 리포지토리에 Buzz 클라이언트처럼 에이전트를 통제할 수 있는 수단이 있는가"
- 결론: **있다.** 다만 하나가 아니라 5개 계층이고, 그중 데스크톱 클라이언트 전용인 것과
  어떤 클라이언트/CLI에서도 쓸 수 있는 것이 나뉜다. 도구 호출 단위 사전 승인(HITL)은 **없다.**

---

## 1. 채팅 내 오너 제어 명령 — 클라이언트 무관, CLI로 실행 가능

| 명령 | 효과 |
|------|------|
| `!shutdown` | 하네스 프로세스 graceful 종료 |
| `!cancel` | 해당 세션 스코프의 in-flight 턴 취소(유휴면 no-op) |
| `!rotate` | 해당 스코프의 ACP 세션 폐기 → 다음 이벤트부터 새 세션 |

- 판정: `crates/buzz-acp/src/lib.rs:4157` `is_owner_control_command()`
  → **kind:9 이고, `content.trim()`이 명령과 정확히 일치하고, 별도 `p` 태그로 해당 에이전트를 멘션**해야 함.
- 처리: `crates/buzz-acp/src/lib.rs:3347`(shutdown) / `:3372`(cancel) / `:3420`(rotate).
  발신자 == 검증된 owner일 때만 소비되고, 아니면 평범한 메시지로 흘러간다.
- **author gate보다 먼저** 검사된다(`:3483` 주석) → `respond_to` 모드가 무엇이든 오너는 항상 제어 가능.
- 범위: `session_policy`(channel | thread)에 따라 `scope::SessionScope::derive`로 결정.
  기본 `channel` 정책이면 채널 전체, `thread` 정책이면 해당 스레드에 답글로 보내야 그 스레드만 취소.
- CLI 실행 경로(확인됨):
  ```sh
  buzz messages send --channel <uuid> --mention <agent-pubkey> --content '!cancel'
  buzz messages send --channel <uuid> --reply-to <thread-root> --mention <agent-pubkey> --content '!rotate'
  ```
  `--mention`은 content를 건드리지 않고 `p` 태그만 붙이며(`crates/buzz-cli/src/commands/messages.rs:231,249,636`),
  `--kind` 미지정 시 기본이 kind 9(`messages.rs:700`)이라 정확 일치 조건을 만족한다.
  단 멘션 대상은 해당 채널 멤버여야 한다(`messages.rs:638` missing_members 검사).
- ⚠️ 문서 stale: `docs/welcome-kickoff-silent-failures.md:408`은 "모든 제품 표면에서 도달 불가
  (CLI에 mention 플래그 없음)"라고 적고 있으나, 현재 CLI에는 `--mention`이 있어 CLI 경로는 해소됨.
  다만 같은 절의 나머지 지적(1턴·1에이전트·1채널만 취소되어 루프 차단기는 아님)은 여전히 유효.
  → 후속 작업자가 그 문서를 갱신하면 좋다(이번 조사에서는 코드/문서 미변경).

## 2. 옵저버 제어 프레임(kind 24200, NIP-AO) — 암호화 owner→agent 제어 채널

- 스펙: `docs/nips/NIP-AO.md`. 태그 `["frame","control"]`, `p`=에이전트, `agent`=에이전트, content는 NIP-44.
- 구현된 제어 타입(`crates/buzz-acp/src/lib.rs:1611-1620`):
  `cancel_turn`, `switch_model`, `publish_project_owner_announcements`.
  ⚠️ NIP-AO 본문(`NIP-AO.md:125`)은 `cancel_turn`만 정의 — 구현이 스펙보다 앞서 있다(스펙 미갱신).
- 릴레이 강제(`crates/buzz-relay/src/handlers/event.rs`):
  방향 판정 `agent_observer_route()`(:1154), ±5분 freshness(:1035),
  `is_agent_owner` DB 확인(:1085, NIP-OA 세션이면 fast path), rate limit은 **telemetry에만** 100/s(:1116),
  영속화 없이 in-memory pub/sub 팬아웃.
- 표면:
  - 데스크톱: `desktop/src/shared/api/agentControl.ts`(cancel_turn/switch_model),
    `desktop/src/features/projects/projectOwnerControl.ts`, 전송은 `shared/api/observerRelay.ts`.
  - 모바일: **읽기 전용**(`mobile/lib/features/channels/agent_activity/observer_subscription.dart`) — 제어 전송 없음.
  - CLI: **제어 전송 서브커맨드 없음.** `buzz agents draft-create/draft-update`는 반대 방향
    (agent→owner, telemetry 프레임에 실린 승인 요청)이다: `crates/buzz-cli/src/agent_management.rs`.
  - 단 `buzz_sdk::build_agent_observer_frame`(`crates/buzz-sdk/src/builders.rs:270`)이 public이고
    두 방향 모두 허용하므로, 스크립트로 control 프레임을 직접 만들어 `POST /events`로 보낼 수는 있다.
- 한계: 타깃이 **채널 단위**. 채널에 세션 스코프가 여러 개면 하네스가 `ambiguous_target`으로 거부
  (`desktop/src/features/agents/AGENTS.md` "Channel-only runtime controls").
  전달은 best-effort → `control_result` 프레임으로만 성공 확인 가능(릴레이 수락은 증거가 아님).

## 3. 런타임 라이프사이클 — 데스크톱(Tauri) 전용

- `desktop/src-tauri/src/managed_agents/runtime_commands.rs`:
  `start_managed_agent_runtime`(:236), `stop_managed_agent_runtime`(:319),
  `restart_managed_agent_runtime`(:386), `reconcile_managed_agent_runtimes`(:466),
  `put_managed_agent_runtime_lifecycle`(:103).
- 원격(provider/K8s) 백엔드는 **Stop이 provider 연산이 아니다**: 데스크톱이 `!shutdown`을 발행한다
  (`docs/remote-agents.md:886`, `desktop/src/features/agents/lib/managedAgentControlActions.ts`).
  `VISION_REMOTE_AGENTS.md`의 축: "deploy 이후 데스크톱은 substrate 제어 채널을 보유하지 않는다"
  → 원격 에이전트에 대한 보장된 kill switch는 없고, 비활동 자가 종료 타이머가 백스톱.

## 4. 정책·구성에 의한 통제(사전 제약)

- 호출 권한 게이트 `respond_to`: `owner-only` | `allowlist` | `anyone` | `nobody`
  (`crates/buzz-acp/README.md:140`). `owner-only`는 실제로 **owner ∪ NIP-OA 검증된 동일 오너 sibling 에이전트**
  (`desktop/src-tauri/src/managed_agents/access_policy.rs` 헤더 주석).
- 런타임 노브(env, `crates/buzz-acp/src/config.rs`): `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`(steer/queue/interrupt),
  `MAX_TURNS_PER_SESSION`, `MAX_TURN_DURATION`, `TURN_TIMEOUT`, `IDLE_TIMEOUT`, `EXIT_AFTER_INACTIVITY`,
  `PERMISSION_MODE`, `KINDS`, `CHANNELS`, `SUBSCRIBE`, `RESPOND_TO(_ALLOWLIST)`, `ALLOWED_RESPOND_TO` 등.
- 빌드/하네스 상한: `BUZZ_DESKTOP_BUILD_AGENT_ACCESS_OWNER_ONLY`(access_policy.rs),
  하네스별 병렬도 상한 `desktop/src-tauri/src/managed_agents/parallelism.rs`.
- 이 설정들은 **실행 중 바꿀 수 없다** — 다음 바디(재시작/재배포)에서 적용(`VISION_REMOTE_AGENTS.md` "Honest Costs").
  예외가 라이브 `switch_model` 제어 프레임.

## 5. 릴레이/커뮤니티 차원의 강제 수단 (오너가 아니어도, 관리자면 가능 · CLI 존재)

- 모더레이션(`buzz moderation ...`, `crates/buzz-cli/src/lib.rs:1938~`):
  `ban`(9040) / `unban`(9041) / `timeout`(9042, write-block) / `reports` / `resolve`(9044).
  에이전트 pubkey도 그냥 커뮤니티 멤버라서 동일하게 적용된다 — 사실상 가장 강한 차단 레버.
- 정체성 아카이브(NIP-IA): `buzz agents archive|unarchive|archived`(9035/9036/13535).
- 채널 멤버십 제거: ACP가 멤버십 제거를 감지해 큐를 드레인하고 세션을 무효화(`lib.rs:3330` 부근).
- 운영 CLI `buzz-admin`: 멤버 add/remove/list, deletions, 채널 재조정 등(에이전트 전용 기능은 아님).
- 워크플로 승인 게이트: `RequestApproval` 스텝 + 토큰(`crates/buzz-workflow/src/executor.rs:477,736`),
  릴레이 `handle_approval_grant`(`crates/buzz-relay/src/handlers/command_executor.rs:71`),
  CLI `buzz workflows approve --token ...`. 단 executor에는 `TODO (WF-08)` (승인 레코드 DB 생성 / kind:46010 발행) 미완.

## 6. 공백 / 주의 (후속 작업 후보)

1. **도구 호출 단위 사람 승인 없음.** ACP 하네스는 `session/request_permission`을 `allow_once`로
   **자동 승인**한다(`crates/buzz-acp/src/acp.rs:1202,1277,1942`). `buzz-agent`에는 `PermissionBroker`가
   있지만(`crates/buzz-agent/src/agent.rs:152`), Buzz 뒤에서 돌 때는 게이트 역할을 하지 않는다.
   즉 통제는 "무엇을 실행할지 사전 승인"이 아니라 "누가 시킬 수 있는지 + 취소/종료" 축이다.
2. **Job 프로토콜(43001–43006, `KIND_JOB_CANCEL` 포함)은 예약 상태.** 피드/활동 렌더링에만 쓰이고
   (`crates/buzz-db/src/store/feed.rs`, `desktop/src/features/home/*`, `mobile/.../activity_provider.dart`)
   하네스에 생산·소비 구현이 없다. "작업 취소" 정식 프로토콜로 오해하지 말 것.
3. **스레드 단위 옵저버 제어 미구현** (채널 단위만, 다중 스코프면 `ambiguous_target`).
4. **`!cancel`은 루프 차단기가 아님** — 1턴/1에이전트/1채널.
5. 문서 두 곳 stale: `docs/welcome-kickoff-silent-failures.md:408`(§1 참조),
   `docs/nips/NIP-AO.md:125`(구현된 제어 타입 3종 미반영).

## 6-b. 모바일(핸드폰)에서 가능한 통제 수단

### 가능

1. **DM으로 오너 제어 명령 — 폰에서 유효한 유일한 직접 제어 경로.**
   DM 채널은 본문에 `@Name`이 없어도 수신자 `p` 태그가 자동 부착된다
   (`mobile/lib/features/channels/message_mention_pubkeys.dart:17`,
   `mobile/lib/features/channels/send_message_provider.dart:69-95`), 전송 kind는 9
   (`mobile/lib/shared/relay/nostr_models.dart:21`).
   따라서 에이전트와의 1:1 DM에서 본문을 **정확히** `!shutdown` / `!cancel` / `!rotate`만 보내면
   §1의 세 조건(kind 9 + 정확 일치 + 별도 `p` 태그)을 모두 만족한다.
   기본 구독 모드가 `mentions`(`crates/buzz-acp/src/config.rs:331`, 필터는 `:1358`에서 kind 9 + `#p`)
   이므로 DM의 `p` 태그 덕에 이벤트가 하네스에 도달한다.
   - `!shutdown` → 스코프 무관 프로세스 종료. **폰에서 쓸 수 있는 실질적 kill switch**
     (원격/K8s 에이전트의 공식 정지 경로와 동일).
   - `!cancel` / `!rotate` → DM은 항상 conversation 스코프(`crates/buzz-acp/src/scope.rs:132`)이므로
     **DM 대화의 턴만** 취소/회전된다. 다른 채널에서 돌고 있는 턴은 영향 없음.
   - 채널 컴포저로는 불가: 멘션 선택 시 본문에 `@Name `이 삽입되고
     (`compose_bar_widget.dart:388-401`), `p` 태그는 본문에 그 텍스트가 남아 있는 멘션만 부착된다
     (`:472-475` `hasMention` 필터) → 정확 일치 조건이 깨진다.
   - ⚠️ 이 DM 경로는 **코드 경로 정합성**으로 도출한 결론이며, 이를 고정하는 테스트는 찾지 못했다.
     실기 검증 + 회귀 테스트 추가가 후속 작업 후보(§1의 stale 문서 갱신과 함께).
2. **스티어링(가장 실용적).** 기본 `multiple_event_handling = steer`
   (`crates/buzz-acp/src/config.rs:375`, 테스트 `:2724`) → 작업 중인 에이전트에게 평범한 메시지를
   보내면 진행 중 턴을 취소하고 새 지시를 엮어 재디스패치한다. 폰에서 그냥 말을 걸면 된다.
3. **채널에서 에이전트 제거.** `mobile/lib/features/channels/members_sheet.dart:378` →
   `channel_management_actions.dart:303` (kind 9001). ACP가 멤버십 제거를 감지해 큐 드레인 +
   세션 무효화(`crates/buzz-acp/src/lib.rs:3330` 부근).
4. **관찰(읽기 전용).** kind 24200 텔레메트리를 복호화해 라이브 트랜스크립트 표시
   (`mobile/lib/features/channels/agent_activity/observer_subscription.dart:137`,
   `agent_activity_sheet.dart`), working bots 표시(`working_bots_provider.dart`).

### 불가 (폰에 없음)

- **옵저버 제어 프레임 전송**(`cancel_turn` / `switch_model` /
  `publish_project_owner_announcements`) — 모바일에 전송 코드가 없다(`"control"`·`cancel_turn` 그렙 0건).
  데스크톱 전용(`desktop/src/shared/api/agentControl.ts`).
- **런타임 start / stop / restart / deploy** — Tauri 커맨드라 데스크톱 전용.
  즉 **폰에서는 에이전트를 켤 수 없다**(DM `!shutdown`으로 끄는 것만 가능).
- **모더레이션 ban / timeout**(9040/9042) — 모바일 미구현. CLI(`buzz moderation`) 또는 다른 표면 필요.
- **워크플로 승인** — 모바일은 "승인 대기" 알림만 표시(`mobile/lib/features/activity/feed_item.dart:87`,
  kind 46010), 승인 grant(46030) 전송 없음.
- **respond_to / 모델 등 설정 변경** — 모바일에 에이전트 설정 UI 없음.

### 우회
폰에서 터미널(SSH 등)로 `buzz` CLI를 쓸 수 있다면 §1의 `--mention` 경로로 **채널 단위** `!cancel`까지 가능하다.

## 6-c. 웹에서 가능한 통제 수단

**결론: 웹에는 에이전트 통제 UI가 전혀 없다.** 리포의 웹 번들 두 개는 용도가 다르다.

- `web/` — 릴레이가 서빙하는 공개 브라우저 클라이언트. 라우트가
  `index`, `invite.$code`, `repos.$repoId(.blob)` 뿐(`web/src/app/routes/`), 기능은
  `features/repos` + `features/invite`. 채널·메시지·에이전트 UI 자체가 없다.
  `VISION.md:81`도 "browser web client (the repo browser)"로 한정한다.
- `admin-web/` — 운영자 콘솔(`/api/admin/v1`, 관리 호스트에서만 서빙, `router.rs:61,174`).
  NIP-07 확장으로 NIP-98 서명 인증(`admin-web/src/api.ts`). 화면은 모더레이션 리포트 조회 +
  제품 피드백 상태 변경. **에이전트 제어 없음**이며, 집행 액션(`reports/{id}/resolve`의
  delete|kick|ban|timeout, `api/admin/mod.rs:496`)은 릴레이 API에는 있어도 현재 UI에는
  호출이 없다(`App.tsx`에 resolve/mutate 호출 부재).

### 프로토콜은 웹에서 도달 가능 — UI만 없다

- `POST /events`(`crates/buzz-relay/src/router.rs:73`, NIP-98 인증)는 서명 이벤트를 받고,
  **kind 9가 HTTP 허용 스코프에 있다**(`ingest.rs:473` → `Scope::MessagesWrite`).
  → 브라우저에서 NIP-07으로 서명한 `!shutdown`/`!cancel`/`!rotate`(정확 본문 + 에이전트 `p` 태그)를
  보낼 수 있다. 즉 오너 제어 패널을 웹 페이지로 만드는 것은 지금 스택으로 즉시 가능하다.
  모더레이션 ban/timeout(9040/9042)도 같은 경로로 가능(allowlist 테스트 `ingest.rs:3872` 목록).
- **옵저버 제어 프레임(24200)은 HTTP로 불가** — ephemeral kind는 HTTP 스코프 허용 목록에서
  제외되며 그 사실이 테스트로 고정되어 있다(`ingest.rs:3868`
  `ephemeral_kinds_not_in_scope_allowlist`). WebSocket + NIP-42 경로에서만 수락되므로
  (`handlers/event.rs`) 브라우저가 WS로 직접 붙으면 가능하지만, 리포에 그런 웹 클라이언트는 없다.

## 7. 다음 작업자를 위한 재현/검증 경로

- 오너 명령 경로 테스트: `crates/buzz-acp/src/lib.rs`의 `is_owner_control_command` 주변 유닛 테스트,
  `crates/buzz-acp/src/relay.rs:4834`(`!shutdown` 픽스처).
- 릴레이 옵저버 인증/거부: `crates/buzz-relay/src/handlers/event.rs:1229~`의 테스트 모듈.
- 데스크톱 제어 UI 계약: `desktop/src/features/agents/AGENTS.md` "Channel-only runtime controls" 절과
  그 아래 나열된 테스트 목록.
- 라이브 검증은 릴레이+Postgres+Redis 필요(`just relay`, `just test`). 이번 조사는 정적 분석만 수행.
