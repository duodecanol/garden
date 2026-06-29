---
type: reference
status: active
publish: true
date: 2026-04-20
tags:
  - type/resource
  - topic/dev
  - topic/fly-io
  - topic/deployment
  - topic/incident-response
topics:
  - Fly.io
  - Deployment
  - Debugging
  - Blue-Green
related:
  - "[[2026-04-16_Git-Sync-Staging-Alias]]"
sources:
  - fly logs -a oshiz-node-dev (2026-04-20 incident)
  - https://fly.io/docs/launch/deploy/#deployment-strategies
---

# Fly.io 배포 deadlock — 좀비 머신과 숨겨진 release_command machine

## Summary

> [!abstract]
> Fly 배포가 "좀비 머신이 destroy해도 재등장"하며 영원히 막히는 상황의 원인과 복구 절차.
> 핵심 트랩: **release_command 머신은 `fly machines list`에 나타나지 않는다.** Dashboard에서만 보인다.
> 해결 조합: `fly deploy --strategy immediate --ha=false` + `fly.toml`에서 release_command 일시 주석 + Dashboard에서 release_command 머신 수동 destroy.

## 사건 맥락

- 앱: `oshiz-node-dev` (oshiz-backend, Fly.io, SJC region)
- 시점: 2026-04-20
- 표면 증상
  - bluegreen 배포가 "Found 2 different images" pre-check에서 계속 거부됨
  - `fly machine destroy --force`를 해도 같은 머신 ID가 계속 재등장
  - `flyctl secrets deploy`가 `nil pointer dereference`로 panic
  - 로컬에서 `fly scale count 0` + 모든 머신 destroy를 해도 몇 초 뒤에 머신이 되살아남

## 원인 체인 (시간 역순)

1. **앱 코드 crash (진짜 시작점)**
   - `apps/oshiz-backend/src/libs/dm-imagegen/data-bridge.ts`의 `assertCompleteRecord`가 모듈 top-level에서 실행됨
   - `DM_IMAGEGEN_SCHEDULE_TYPES` enum에 `"MOVE"`는 추가됐는데, 데이터 JSON에서는 MOVE entry가 제거된 상태
   - 기동 즉시 `Error: Missing generic activity variant entry for "MOVE"`
   - `Main child exited normally with code: 1` → 10회 재시작 → `host unreachable` 💀

2. **`min_machines_running = 1`이 deadlock을 굳힘**
   - `fly.api.dev.toml`:
     ```toml
     [http_service]
       auto_start_machines = true
       min_machines_running = 1
     [deploy]
       strategy = 'bluegreen'
     ```
   - Fly orchestrator는 "최소 1대 유지" 약속을 강제 → 사용자가 destroy해도 기존 머신 레코드를 **resurrect** (동일 ID로 부활)
   - 그래서 "destroy 명령이 안 먹힌다"고 체감됨

3. **CI가 계속 새 배포를 밀어넣음**
   - GitHub Actions: `on: push` 트리거 + `paths: apps/oshiz-backend/**, packages/oshiz-db/**, packages/data-catalog/**`
   - main에 머지되는 모든 PR이 새 deploy 유발 → 같은 crash → 다시 좀비 축적

4. **숨겨진 release_command 머신이 최종 잔재**
   - bluegreen 실패로 정상 완료되지 못한 release들이 `running` 상태로 남음 (v280, v281 등)
   - 각 running release에 매달린 release_command 머신이 **dashboard에만 노출**
   - `fly machines list -a <app>` 결과에는 `role=app`인 머신만 나와서 CLI로는 찾을 수 없음
   - 이게 pre-check의 "2 different images" 판정을 계속 유발

## 최종 해결 절차

### 1. bluegreen/release_command 우회해서 상태 덮어쓰기

`fly.api.dev.toml`에서 release_command 블록을 **잠시 주석 처리**:

```toml
[deploy]
  strategy = 'bluegreen'
  # release_command = "sh -c \"bun run --filter '@oshiz/db' db:migrate && bun run --filter '@oshiz/db' db:sync\""
  # release_command_timeout = "10m"
  wait_timeout = "10m"
```

그리고 강제 배포:

```bash
fly deploy --strategy immediate --ha=false
```

- `--strategy immediate`: 머신 상태 검사 건너뛰고 즉시 덮어쓰기
- `--ha=false`: HA 설정을 최소로 낮춰 꼬인 머신 수 제약을 해제
- release_command 주석: 새 release 머신 생성 자체를 막음

### 2. 숨겨진 release_command 머신 수동 destroy

Dashboard에서만 찾을 수 있다.

1. Fly Dashboard → 해당 app → **Releases** 탭
2. 각 release (특히 `running`/`failed` 상태)를 열어 상세 페이지 확인
3. release에 매달린 machine id 확인
4. 그 id를 `fly machine destroy --force <id> -a <app>`로 강제 삭제
   - 또는 Dashboard의 해당 machine 상세 페이지에서 직접 destroy 버튼

> **핵심**: 이 머신들은 `fly machines list -a <app>`에 절대 나타나지 않는다. CLI만 붙들고 있으면 끝까지 찾을 수 없다.

### 3. CI 조건 충족시키고 원상 복구

- release_command 주석 해제 (fly.api.dev.toml 원복)
- `min_machines_running = 1` 등 운영 설정 원복
- 원인 fix 커밋 (이번 경우 dm-imagegen에서 MOVE 제거) 머지
- 정상 deploy로 마무리

## 왜 이 조합이 deadlock을 푸는가

| 요소                   | 역할                                                                   |
| ---------------------- | ---------------------------------------------------------------------- |
| release_command 주석   | 새 release 머신 생성 차단 — **좀비 공급원 자체를 끊음**                |
| `--strategy immediate` | bluegreen pre-check(2 different images) 우회                           |
| `--ha=false`           | `min_machines_running` 등 HA constraint 축소 — **resurrect loop 차단** |
| Dashboard 수동 destroy | CLI에 안 보이는 release 머신을 없애 pre-check 조건 정리                |

세 가지 제약이 동시에 걸린 상태(bluegreen / HA constraint / release_command lock)를 **동시에** 풀어야 한다. 하나라도 남으면 loop로 다시 빠진다.

## Preventive Checklist

- Fly 배포 문제가 꼬일 때 제일 먼저 **머신이 진짜 죽는 원인**을 `fly logs -a <app>`로 확인. 인프라 문제로 보이는 대부분의 증상이 실제로는 앱 crash에서 시작된다.
- `fly machines list`에 안 나오는 머신이 존재할 수 있음을 기억. Dashboard Release 탭을 항상 병행 확인.
- `min_machines_running`은 복구 시 일시적으로 `0`으로 내리면 좀비 재등장 차단에 결정적.
- CI가 계속 새 배포를 푸시하는 중이면 복구 작업 전에 `gh workflow disable <name>`로 반드시 중지.

## Key Takeaways

1. **`host unreachable` 💀는 Fly 인프라 장애 신호가 아니라 재시작 max 초과 후 상태일 수 있다.** 로그부터 봐라.
2. **Fly CLI 기본 출력은 `role=app` 머신만 보여준다.** release_command 머신 같은 특수 role은 Dashboard로만 접근.
3. **bluegreen + `min_machines_running` + failing release 조합은 self-healing이 안 된다.** 수동 개입 필수.
4. **`fly deploy --strategy immediate --ha=false` + release_command 주석**은 꼬인 상태를 돌파하는 Swiss Army knife 조합.

## Sources

- `fly logs -a oshiz-node-dev` (2026-04-20, MOVE crash 로그)
- 실험적으로 재현한 destroy-resurrect 루프 (같은 machine ID `90803d2ecdd387`이 반복 재등장)
- Fly docs: https://fly.io/docs/reference/configuration/#the-deploy-section
- Fly docs: https://fly.io/docs/launch/deploy/#deployment-strategies
