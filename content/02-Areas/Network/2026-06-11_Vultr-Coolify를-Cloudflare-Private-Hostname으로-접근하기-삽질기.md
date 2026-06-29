---
type: research
status: active
publish: true
date: 2026-06-11
tags:
  - type/research
  - topic/dev
  - topic/network
  - topic/infra
topics:
  - cloudflare-zero-trust
  - cloudflare-tunnel
  - warp
  - coolify
  - dns
related:
  - "[[korean-dental-employer-review-platform]]"
sources:
  - https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/connect-private-hostname/
  - https://developers.cloudflare.com/cloudflare-one/traffic-policies/egress-policies/host-selectors/
  - https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/connect-cidr/
  - https://blog.cloudflare.com/tunnel-hostname-routing/
  - https://github.com/cloudflare/cloudflared/blob/master/RELEASE_NOTES
---

# Vultr Coolify를 Cloudflare Private Hostname으로 접근하기 — 삽질기

> [!abstract] 요약
> Vultr VPS의 Coolify 대시보드(`:8000`/`:6001`/`:6002`)를 공인 노출 없이 WARP 전용 사설 호스트네임 `coolify.devdev.vultr`로 접근시키는 작업. Cloudflare Tunnel **hostname routing**(2025-09 GA)을 썼는데, DNS 쿼리가 Gateway까지 도달하고도 `Resolved IPs: None`으로 죽는 증상을 약 5일에 걸쳐 추적. 최종 범인은 둘:
> ① Gateway의 **`host_selector` 토글**(= 합성 IP 100.80.0.0/16 엔진)이 꺼져 있었음 — hostname-routing 가이드가 아니라 **egress host-selectors 문서에만** 적힌 전제조건.
> ② cloudflared의 **virtual DNS service는 `/etc/hosts`를 읽지 않음** — `extra_hosts`로는 절대 해석 안 됨. **dnsmasq 사이드카 + `--dns-resolver-addrs`**가 정답.

## 1. 목표 구성

```
Mac (WARP, Traffic and DNS mode)
  → DoH → Cloudflare Gateway
  → hostname route 매칭 → 터널로 DNS 쿼리 전달
  → cloudflared virtual DNS → dnsmasq(10.0.0.1:53) → "10.0.0.1"
  → Gateway가 합성 IP 100.80.x.x 응답
  → 트래픽: 100.80.x.x → 터널 → 10.0.0.1:8000/6001/6002 (Coolify)
```

- 호스트네임 **1개**로 3개 포트 전부 커버 — hostname routing은 L4라 목적지 포트가 그대로 전달됨 (Public Hostname처럼 포트별 ingress 불필요)
- VPS 인바운드 80/443/8000/6001/6002 전부 닫은 채 유지

## 2. 증상

- 공인 hostname(`api-dev-allchee.pluffycat.com`)은 정상, 사설 hostname만 실패
- `nslookup coolify.devdev.vultr` → NXDOMAIN (WARP 프록시 127.0.2.2 경유)
- Gateway DNS 로그: `Resolved IPs: None` / `Allowed On No Policy Match` / `EDE errors: Other` / `Resolution method: Custom IPs`
- cloudflared 디버그 로그: `dst=2606:4700:cf1:2000::1:53` 세션이 생겼다가 `terminated by edge`로 즉사

## 3. 기각된 가설들 (시간 순)

| # | 가설 | 검증 방법 | 결과 |
|---|------|----------|------|
| 1 | hostname route 미등록 / 다른 터널에 등록 | API `GET cfd_tunnel/{id}/configurations` | 기각 — 올바른 터널, `warp-routing:true` (v9) |
| 2 | vnet 불일치 | 대시보드 확인 | 기각 — 둘 다 default |
| 3 | Split Tunnel에 `100.64.0.0/10` 잔존 | 대시보드 확인 | 기각 — 이미 제거됨 |
| 4 | `extra_hosts: 127.0.0.1` 루프백이라 Gateway가 매핑 거부 | `10.0.0.1`로 교체 후 재시도 | 부분 기각 — 바꿔도 동일 증상 (단 루프백은 어차피 안 됨) |
| 5 | `extra_hosts`가 컨테이너에 미반영 | `docker inspect HostsPath` → 실제 파일 확인 | 기각 — `/etc/hosts`에 정상 기록 |
| 6 | resolver policy가 쿼리를 하이재킹 (`Resolution method: Custom IPs`) | API `GET gateway/rules` + 대시보드 | 기각 — resolver policy 0개 |
| 7 | cloudflared 구버전 | `cloudflare/cloudflared:latest` (2026.x) | 기각 |

> [!info] 디버깅에 결정적이었던 도구
> - **cloudflared `--loglevel debug`**: `dst=…cf1:2000::1:53` 세션 관찰로 "Gateway→터널 전달까지는 됨"을 확정 — 실패 지점을 해석 단계로 좁힘
> - **Cloudflare API 직접 감사** (wrangler OAuth 토큰 + Zero Trust Read 토큰): 대시보드에 안 보이는 `gateway/configuration`의 `host_selector:{enabled:false}`를 발견한 곳

## 4. 진짜 원인 ①: `host_selector` 토글 OFF

- 위치: Zero Trust → **Traffic policies → Traffic settings → Policy settings → "Allow egress policy host selectors"**
- API: `PATCH /accounts/{id}/gateway/configuration` `{"settings":{"host_selector":{"enabled":true}}}`
- 이 토글이 **initial-resolved-IP(100.80.0.0/16 합성 IP) 메커니즘 그 자체**. 꺼져 있으면 hostname route·warp-routing·split tunnel이 전부 정상이어도 Gateway가 합성 IP를 만들 수 없음.

> [!warning] 문서 함정
> 이 전제조건은 [hostname routing 가이드](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/connect-private-hostname/)에는 없고 [egress host-selectors 문서](https://developers.cloudflare.com/cloudflare-one/traffic-policies/egress-policies/host-selectors/)에만 있다. hostname routing만 따라가면 100% 빠지는 구멍.

## 5. 진짜 원인 ②: virtual DNS는 `/etc/hosts`를 안 읽는다

- cloudflared **2025.6.0** "Add virtual DNS service" / **2025.7.0** "Add `--dns-resolver-addrs` flag" (RELEASE_NOTES)
- 문서의 "local system resolver로 해석되면 추가 설정 불필요"라는 문구는 **DNS 클라이언트 → resolv.conf 업스트림** 질의를 의미. OS의 hosts 파일 해석이 아님.
- 따라서 compose `extra_hosts`는 이 경로에서 완전 무효 — 공인 DNS가 `.vultr`를 모르니 NXDOMAIN.
- 해법: 진짜 DNS 서버(dnsmasq)를 호스트 사설 IP에 띄우고 cloudflared에 `--dns-resolver-addrs`로 지정.

## 6. 최종 구성

```yaml
services:
  dnsmasq:
    image: dockurr/dnsmasq:latest
    network_mode: host
    restart: unless-stopped
    entrypoint: ["dnsmasq"]
    command:
      - --keep-in-foreground
      - --log-facility=-
      - --port=53
      - --listen-address=10.0.0.1
      - --bind-interfaces        # 절대 0.0.0.0 금지 (오픈 리졸버)
      - --no-resolv
      - --no-hosts
      - --address=/coolify.devdev.vultr/10.0.0.1
      - --server=1.1.1.1
      - --server=1.0.0.1
      - --cache-size=1000

  cloudflared:
    image: cloudflare/cloudflared:latest
    network_mode: host
    restart: unless-stopped
    depends_on: [dnsmasq]
    command: tunnel --no-autoupdate run --dns-resolver-addrs 10.0.0.1:53 --token ${CLOUDFLARE_TUNNEL_TOKEN}
```

- 레코드는 라우팅 가능한 호스트 IP(`10.0.0.1`)로 — Gateway는 `127.0.0.0/8` 응답을 합성 IP로 매핑하지 않음
- Coolify `.env`: `PUSHER_HOST=coolify.devdev.vultr`, `PUSHER_PORT=6001`
- 검증: `nslookup coolify.devdev.vultr` → **`100.80.40.97`** ✅

### 마지막 함정: Coolify .env 적용은 restart가 아니라 재생성

`PUSHER_HOST`/`PUSHER_PORT`는 soketi 설정이 아니라 **Laravel 앱이 브라우저에게 내려주는 websocket 엔드포인트**다. 그리고 `/data/coolify/source/.env`는 compose `env_file`로 주입되는데, env는 **컨테이너 생성(create) 시점에 박히므로 `docker restart coolify`로는 절대 갱신되지 않는다** (cloudflared `extra_hosts` 때와 똑같은 함정을 같은 날 두 번 밟음):

```bash
sudo nano /data/coolify/source/.env   # PUSHER_HOST / PUSHER_PORT 추가
sudo docker compose \
  --env-file /data/coolify/source/.env \
  -f /data/coolify/source/docker-compose.yml \
  -f /data/coolify/source/docker-compose.prod.yml \
  up -d --force-recreate coolify      # coolify 서비스만 — DB/Redis 무관
sudo docker exec coolify printenv PUSHER_HOST PUSHER_PORT   # 적용 확인
```

- `/data/coolify`는 `700 / uid 9999` 소유라 deploy 계정으로 `cd` 불가 → 절대경로 + sudo로 우회 (`--env-file`을 docker CLI가 직접 읽으므로 compose 명령 자체에 sudo 필요)
- 공식 가이드의 `PUSHER_PORT=443`은 public-hostname/TLS 경유 구성용 — WARP 직결 경로는 `6001`
- 최종 확인: DevTools WS 탭에서 `ws://coolify.devdev.vultr:6001/app/…` **101 Switching Protocols** ✅ (terminal `:6002`도 동일 호스트네임으로 자동 연결)

## 7. 교훈

1. **신기능(GA 1년 미만)의 전제조건은 가이드 본문 밖에 숨어 있을 수 있다.** 관련 기능군(여기선 egress policy) 문서까지 훑거나, API로 계정 설정 원본(`gateway/configuration`)을 덤프해서 `false`인 플래그를 의심하라.
2. **"local system resolver"류의 문서 표현을 구현으로 검증하라.** RELEASE_NOTES의 두 줄("virtual DNS service", "`--dns-resolver-addrs`")이 `/etc/hosts` 가설을 기각시킨 결정타였다.
3. **대시보드가 안 보여주면 API로 직접 본다.** wrangler OAuth 토큰으로 `cfd_tunnel` 설정은 읽혔고, Zero Trust Read 커스텀 토큰으로 나머지를 감사했다. `Resolution method: Custom IPs` 같은 로그 필드 의미도 문서 검색으로 확정하고 움직일 것.
4. **debug 로그의 "error="는 에러가 아닐 수 있다.** `Session terminated … terminated by edge`는 일회성 UDP DNS 세션의 정상 teardown — 성공/실패 모두 같은 모양. 성패는 클라이언트 nslookup으로 판정.
5. **`docker restart`는 env를 갱신하지 않는다.** `env_file`/`extra_hosts` 같은 컨테이너 설정은 생성 시점에 박힌다 — 변경 후엔 반드시 `--force-recreate`(또는 redeploy). 이 함정을 하루에 두 번(cloudflared extra_hosts, Coolify PUSHER env) 밟았다.
6. 단일 레코드면 **CIDR 라우트(`10.0.0.1/32`) + 클라이언트 hosts 파일**이 훨씬 단순한 대안. hostname routing은 사설 서비스가 여럿으로 늘어날 때 빛난다 — dnsmasq에 `--address=` 한 줄 + 대시보드 route 한 줄이면 끝.

## 8. 방화벽 최종 잠금 (2026-06-11)

SSH over WARP까지 검증 후(hostname routing은 L4라 22번 포트도 같은 호스트네임으로 통과 — `ssh deploy@coolify.devdev.vultr` 성공), Vultr Cloud Firewall에서 80/443/8000 인바운드 룰 삭제. 최종 검증:

| 검증 | 결과 |
|---|---|
| 공인 IP 8000/443/80 `nc` | timeout — 외부 완전 차단 ✅ |
| 공개 API `/health` (터널 경유) | `{"status":"ok","db":"ok","redis":"ok"}` — 무영향 ✅ |
| WARP → Coolify `:8000` | 302 ✅ |
| WARP → SSH | ok ✅ |
| ufw | `deny incoming` + 22만 허용 ✅ |

최종 상태: **인바운드는 SSH 22 하나** (WARP SSH 안정성 확인 후 이것도 삭제 예정, 비상 수단 = Vultr 웹 콘솔). 공개 서비스는 전부 터널 아웃바운드, 운영 콘솔은 전부 WARP 사설 호스트네임.

> [!info] ufw 함정
> Docker가 publish한 포트(8000, 6001-6002, 80/443/8080)는 **ufw를 우회**한다 (Docker가 iptables에 직접 룰 삽입). 실질 방어선은 호스트 밖의 Vultr Cloud Firewall이고, ufw는 호스트 프로세스(sshd)용 2차 방어.

## 9. 프로젝트 측 기록

- 운영 절차 SoT: `dental_clinic_app/docs/06-runbook/cloudflare-tunnel.md` → "Private Hostname Access over WARP" 섹션 (이 노트는 여정/맥락 기록)
- 사후 조치: 대화에 노출된 Zero Trust Read API 토큰 **폐기 완료(2026-06-11)** ✅ · 디버그 로깅 제거 **완료(2026-06-19)** ✅ (아래 §10)

## 10. 마무리: 디버그 로깅 제거 (2026-06-19)

추적이 끝났으니 §6 최종 구성에 남아 있던 디버그 플래그를 걷어냄 — cloudflared `--loglevel debug`, dnsmasq `--log-queries`. 둘 다 **컨테이너 생성 시점 설정**이라(§7-5의 그 함정) compose 파일에서 지운 뒤 `--force-recreate`가 필수.

```bash
# tunnel compose(=dnsmasq+cloudflared 묶음)에서 디버그 플래그 라인 삭제 후
sudo docker compose -f <tunnel-compose>.yml up -d --force-recreate

# 적용 확인 — 실행 중 command 에 debug 플래그가 없어야
sudo docker inspect cloudflared --format '{{.Args}}' | grep -q loglevel && echo "STILL DEBUG" || echo "clean ✅"
sudo docker inspect dnsmasq    --format '{{.Args}}' | grep -q log-queries && echo "STILL DEBUG" || echo "clean ✅"
```

- 기능 회귀 없음 확인: `nslookup coolify.devdev.vultr` → `100.80.x.x` 여전히 정상, WARP→`:8000` 302 유지
- 이로써 §6 운영 구성 = 디버그 흔적 0. 추적용 로그는 필요할 때만 한시적으로 다시 켤 것(상시 `--log-queries`는 dnsmasq 디스크/프라이버시 부담).
