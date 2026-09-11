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

## 7. 다음 작업자를 위한 재현/검증 경로

- 오너 명령 경로 테스트: `crates/buzz-acp/src/lib.rs`의 `is_owner_control_command` 주변 유닛 테스트,
  `crates/buzz-acp/src/relay.rs:4834`(`!shutdown` 픽스처).
- 릴레이 옵저버 인증/거부: `crates/buzz-relay/src/handlers/event.rs:1229~`의 테스트 모듈.
- 데스크톱 제어 UI 계약: `desktop/src/features/agents/AGENTS.md` "Channel-only runtime controls" 절과
  그 아래 나열된 테스트 목록.
- 라이브 검증은 릴레이+Postgres+Redis 필요(`just relay`, `just test`). 이번 조사는 정적 분석만 수행.
