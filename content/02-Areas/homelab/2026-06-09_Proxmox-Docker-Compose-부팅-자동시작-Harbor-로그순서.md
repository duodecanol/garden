---
type: reference
status: active
publish: true
date: 2026-06-09
tags:
  - type/reference
  - topic/homelab
  - topic/docker
  - topic/proxmox
  - topic/harbor
  - status/active
topics:
  - docker-compose
  - restart-policy
  - logging-driver
  - boot-ordering
  - systemd
  - harbor
related:
  - "[[2026-06-09_Proxmox-VM-복제-골든템플릿]]"
  - "[[2026-06-09_Linux-Proxmox-VM-디스크-용량-진단-확장]]"
aliases:
  - Docker Compose 자동 재시작 안됨
  - Harbor Exited 128 부팅
  - failed to initialize logging driver
---

# Docker Compose 부팅 자동 시작 & Harbor 로그 드라이버 순서 꼬임

Proxmox VM 재부팅 시 Compose 서비스가 자동으로 안 올라오는 문제. 표면 점검부터 시작해, **로그 드라이버 기동 순서 race** 라는 진짜 원인까지 추적한 트러블슈팅 기록(Harbor 사례).

## 1. 기본 체크리스트 (대부분 여기서 해결)

| 항목 | 처치 |
|---|---|
| 컨테이너 정책 | `docker-compose.yml` 각 서비스에 `restart: always` (또는 `unless-stopped`) → `docker compose up -d` |
| Docker 엔진 | `sudo systemctl enable docker.service containerd.service` (확인: `systemctl is-enabled docker`) |
| VM 자동 부팅 | Proxmox VM → **Options → Start at boot = Yes** |

YAML 일괄 수정 대안: `docker update --restart always <컨테이너>` (실행 중 즉시 적용) / `docker-compose.override.yml` 로 `restart` 만 병합 주입 / `sed` 일괄 주입.

> [!note] `sudo` 로만 기동된다 ≠ 권한이 자동시작을 막는 것
> Docker 엔진은 root 데몬으로 돈다. `sudo` 필요는 사용자가 `docker` 그룹에 없을 뿐(`sudo usermod -aG docker $USER`). 자동시작 실패의 직접 원인은 아니다.

## 2. 진짜 원인 — 부팅 시 로그 드라이버 기동 race

증상: 재부팅하면 **딱 1개 컨테이너만 뜨고** 나머지는 `Exited (128)`. `restart: always` 가 다 걸려 있어도 그렇다.

핵심 로그:
```
failed to create task for container: failed to initialize logging driver:
dial tcp 127.0.0.1:1514: connect: connection refused
```

- 모든 컨테이너가 로그를 `127.0.0.1:1514`(syslog/fluentd/vector 등 **로그 수집 컨테이너**)로 보내도록 설정됨.
- 부팅 시 Docker 가 `restart: always` 로 전부 동시에 깨우는데, **정작 로그 수집기(1514)가 아직 포트를 못 열어** Connection refused.
- Docker 는 **로그 드라이버 초기화 실패 시 컨테이너 기동 자체를 취소**(엔진 레벨 런타임 에러)한다.
- **혼자 뜬 1개 = 로그 수집 컨테이너 자신**(자기 로그는 외부로 안 보냄). Harbor 라면 `harbor-log`.

### 결정적 함정: 부팅 시 `depends_on` 무시
Docker 데몬이 부팅 후 `restart: always` 로 컨테이너를 깨울 때는 **`depends_on` / `condition: service_healthy` 를 전부 무시하고 동시에 올린다.** 이 조건은 `docker compose up` 을 **직접 실행할 때만** 작동한다. → override 에 `depends_on` 만 추가해도 부팅 race 는 안 잡힌다.

### autoheal 도 안 통한다
`willfarrell/autoheal` 은 컨테이너가 **Running 상태에서 unhealthy** 가 됐을 때만 개입. 지금은 Running 진입 전에 `Exited (128)` 로 박히므로 autoheal 의 감시 대상이 아니다.

## 3. 시도 → 실패 기록 (왜 안 됐나)

1. **override 에 `mode: non-blocking` 주입** → 실패. override 의 부분 `logging:` 블록이 원본 `driver: "syslog"`/주소를 **병합이 아니라 대체**해 버려 설정이 날아감 → `Exited (128)`.
2. **override 에 `depends_on: service_healthy`** → 실패. 위의 "부팅 시 depends_on 무시" 때문.
3. **`restart: "no"` + systemd** → 부팅 순서는 잡히지만 **운영 중 컨테이너 장애 시 재시작이 안 됨**(HA 상실). 기각.

## 4. 최종 해법 — 역할 분담 (Dual 구조)

> [!tip] 원칙
> **평소(runtime)**: 원본의 `restart: always` 유지 → 장애 시 Docker 가 즉시 복구(이때 로그 서버는 이미 떠 있어 에러 없음).
> **부팅 시**: 부팅 후 한 번 `docker compose up -d` 를 **정식 실행**해 순서를 교통정리. compose 명령으로 실행될 때만 `depends_on(healthy)` 가 작동하므로 죽었던 컨테이너들이 순서대로 되살아난다.

### 4-1. `docker-compose.override.yml` — 순서만 정의 (restart 는 안 건드림)
```yaml
services:
  registry:
    depends_on: { log: { condition: service_healthy } }
  registryctl:
    depends_on: { log: { condition: service_healthy } }
  postgresql:
    depends_on: { log: { condition: service_healthy } }
  core:
    depends_on:
      log: { condition: service_healthy }
      registry: { condition: service_started }
      redis: { condition: service_started }
      postgresql: { condition: service_started }
  portal:
    depends_on: { log: { condition: service_healthy } }
  redis:
    depends_on: { log: { condition: service_healthy } }
  proxy:
    depends_on:
      log: { condition: service_healthy }
      registry: { condition: service_started }
      core: { condition: service_started }
      portal: { condition: service_started }
```

### 4-2. 부팅 후 정식 기동 — systemd (권장) 또는 root crontab
**systemd** (`/etc/systemd/system/harbor.service`):
```ini
[Unit]
Description=Harbor Container Registry
Requires=docker.service
After=docker.service
[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/user/harbor/harbor   # compose 파일 경로
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload && sudo systemctl enable harbor.service
```

**root crontab 대안** (실측으로 최종 성공한 방식) — systemd 가 번거로울 때. **반드시 root + 절대경로 + 지연**:
```bash
sudo crontab -e
```
```
@reboot sleep 60 && cd /home/user/harbor/harbor && /usr/bin/docker compose up -d
```
> 일반 사용자 crontab 은 PATH 최소화(그냥 `docker` 못 찾음) + 권한 문제로 실패한다. `sudo crontab -e` + `/usr/bin/docker` 절대경로 필수. 디버그: `sudo journalctl -u cron | grep reboot`.

### 부팅 시 일어나는 일
1. Docker 가 `restart: always` 로 전부 동시 기동 → 2. 예상대로 1차 `Exited (128)` 대참사 → 3. systemd/crontab 이 `docker compose up -d` 호출 → 4. compose 가 override 를 읽어 `log` 가 healthy 될 때까지 대기 후 순서대로 재기동 → 5. 전원 `Up (healthy)` → 6. 이후 운영 중 장애는 `restart: always` 가 복구.

**결과**: Harbor 9개 컨테이너 전부 `Up (healthy)` 확인.

---
홈랩 인덱스: [[02-Areas/homelab/index|homelab]]
