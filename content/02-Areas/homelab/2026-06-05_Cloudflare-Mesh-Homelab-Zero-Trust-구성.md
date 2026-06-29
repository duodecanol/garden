---
type: research
status: active
publish: true
date: 2026-06-05
tags:
  - type/research
  - topic/cloudflare
  - topic/zero-trust
  - topic/networking
  - topic/homelab
  - topic/proxmox
  - status/active
topics:
  - cloudflare-mesh
  - cloudflared-tunnel
  - warp-connector
  - zero-trust
  - internal-dns
  - private-network
  - proxmox
related:
  - "[[01-Projects/oshiz-data-insight/2026-05-08_Proxmox-CT-Ubuntu26-Cloudflare-WARP-설정-여정|Proxmox CT WARP 설정 여정]]"
  - "[[02-Areas/homelab/index|Homelab Index]]"
---

# Cloudflare Mesh 홈랩 Zero Trust 구성

> [!abstract] 한 줄 요약
> LAN DNS 없는 LG 공유기 + Proxmox VM/CT 홈랩을, **단일 connector 박스 하나**로 Cloudflare Zero Trust 에 묶어 ① 가상 hostname ② 무에이전트 전체 LAN 접근 ③ 특정 포트 public HTTPS 노출을 모두 해결한다. 세 요구사항은 각각 다른 Cloudflare 제품에 매핑된다.

## 배경 / 목표

기존에 [[01-Projects/oshiz-data-insight/2026-05-08_Proxmox-CT-Ubuntu26-Cloudflare-WARP-설정-여정|db-middleman CT]] 에서 이미 같은 Zero Trust **Team org** 을 운영 중이다 (WARP 설치, `engineering` vnet, `eevee.local`/`vivident.lan` fallback domain). 이 노트는 그 org 를 홈랩 전체로 확장하는 설계다.

요구사항 3가지:

1. **LAN DNS 부재 보완** — LG 공유기에 local DNS 기능이 없어, LAN 호스트에 가상 hostname (`vm1.home.internal` 등) 부여 필요
2. **Proxmox VM/CT 전체 접근** — 모든 VM/CT 에 WARP(Cloudflare One Client) 를 깔지 **않고** 원격 접근
3. **특정 머신 포트 → public HTTPS** — 포트포워딩/고정 IP 없이 public HTTPS 도메인 부여

## 핵심 결론 (TL;DR)

> [!success] 제품 매핑
> | 요구 | Cloudflare 제품 | 무엇이 어디 깔리나 | 플랜 |
> |---|---|---|---|
> | ① 가상 hostname | **Local Domain Fallback** → 로컬 DNS 서버 (또는 Internal DNS + Resolver Policy) | connector 박스에 dnsmasq 1개 | 무료 (Internal DNS 는 Enterprise) |
> | ② 무에이전트 전체 LAN | **cloudflared Private Network — CIDR route** | connector 박스 1개에만 cloudflared | 무료 |
> | ③ 포트 public HTTPS | **Tunnel Public Hostname** | 같은 cloudflared | 무료 |
> | (확장) 양방향 site-to-site | **Cloudflare Mesh node** (구 WARP Connector) | mesh node 박스 | 무료 50 nodes/50 users |

connector 박스는 **전용 LXC 1개** 권장. 접속하는 *내 기기* 만 Cloudflare One Client(WARP) 필요, *대상* VM/CT 는 무에이전트.

## 중요한 네이밍 변경 (2026)

> [!warning] WARP Connector → Cloudflare Mesh
> 2026 리브랜딩으로 용어가 바뀌었다. 옛 문서/블로그를 읽을 때 혼동 주의.
> - **WARP Connector** → **Cloudflare Mesh node** ("Mesh nodes are your servers, VMs, and containers. They run a headless version of Cloudflare One Client and get a Mesh IP.")
> - **WARP Client** → **Cloudflare One Client**
> - peer-to-peer connectivity → **Cloudflare Mesh**
> - GA + 무료 티어: **"50 nodes and 50 users free"** (계정당 포함)

즉 사용자가 부른 "Cloudflare Mesh" 는 실제 제품명과 정확히 일치한다.

## 아키텍처

```
                  Cloudflare Edge (Zero Trust Team org)
                   │            │              │
       public HTTPS│   private  │CIDR route   DNS│policy
        (Public    │   (192.168 │.x.0/24)   (Local Domain
         Hostname) │                          Fallback)
                   │            │              │
        ┌──────────┴────────────┴──────────────┴──────────┐
        │   LAN 내 단일 connector LXC                       │
        │   cloudflared + dnsmasq                          │
        └──────────────────────┬──────────────────────────┘
                               │ LAN 192.168.x.0/24
            ┌──────────────┬───┴───┬──────────────┐
          VM1            CT1     CT2     ...    (무에이전트)
```

## 요구사항별 상세

### ① 가상 hostname — Local Domain Fallback (무료 경로)

> [!note] 플랜 게이트가 결정적
> **Resolver Policies / Internal DNS 는 Enterprise 전용**. 홈랩(Free/Team)에서는 **Local Domain Fallback** 이 정답.
> - Local Domain Fallback: "Available on all plans"
> - Resolver Policies: "Only available on Enterprise plans"

**구성:**
1. connector LXC 에 `dnsmasq` 설치, `vm1.home.internal → 192.168.1.21` 식 정적 매핑
2. Zero Trust → Settings → WARP Client → **Local Domain Fallback** 에 도메인 `home.internal` + DNS server = connector LXC 의 LAN IP 등록
3. dnsmasq 가 CIDR route(②) 안에 있으면 WARP 터널 통해 도달 가능 — "Connecting it to Cloudflare if inside the WARP tunnel"
4. 이미 운영 중인 `eevee.local → 10.0.0.2`, `vivident.lan` fallback 과 같은 방식 (기존 노트에서 검증됨)

> [!tip] Enterprise 라면
> Internal DNS zone (`.local` 등 TLD 제약 없음) + DNS view + Resolver Policy("Use Internal DNS") 로 디바이스 설정 없이 중앙 관리. 단 "Gateway configuration must exist within the same Cloudflare account where the internal zone exists." 홈랩에는 보통 과함.

### ② 무에이전트 전체 LAN — cloudflared CIDR route

VM/CT 마다 Cloudflare One Client 를 깔지 않고, **connector 가 gateway 역할**로 서브넷 전체를 광고한다.

> [!quote] 공식 동작
> "To make other devices on the subnet behind the node reachable — servers, databases, printers, IoT devices that cannot run the Cloudflare One Client — add CIDR routes. ... traffic destined for the advertised CIDR is forwarded to the node, which delivers it to the appropriate host on the local network."

**구성 (Dashboard):**
1. Zero Trust → Networks → Connectors → Cloudflare Tunnels → 터널 생성/편집 → **CIDR** 탭
2. `192.168.1.0/24` 입력 (IP 충돌 시 virtual network 분리)
3. **클라이언트 측 Split Tunnel 필수** — WARP 는 기본적으로 RFC1918 을 exclude 하므로:
   - **Include 모드**: include 리스트에 `192.168.1.0/24` 추가 (홈랩 권장, 디버깅 쉬움)
   - **Exclude 모드**: 해당 대역 포함하는 광역 RFC1918 route 삭제 후 비겹침 subdivision 재추가

> [!caution] RFC1918 default exclude 함정
> "By default, WARP excludes traffic bound for RFC 1918 space." Split Tunnel 설정을 안 하면 CIDR route 를 등록해도 클라이언트가 그쪽으로 트래픽을 안 보낸다. ②가 안 되면 99% 여기.

기존 `engineering` vnet 의 AWS/GCP 접근과 홈 LAN 대역이 **겹치지 않는지** 확인 필요. 겹치면 별도 vnet 으로 분리.

### ③ 특정 포트 public HTTPS — Tunnel Public Hostname

> [!success] 이것도 이제 무료
> "Connect and secure any private or public app by hostname, not IP — free for everyone in Cloudflare One."

같은 cloudflared 에 public hostname route 추가:
1. Dashboard → 터널 → **Public Hostname** 탭 → Add a public hostname
2. Hostname: `app.yourdomain.com`, Service URL: `http://192.168.1.30:8080` (또는 `http://localhost:port`)
3. Cloudflare 가 자동으로 `app.yourdomain.com → <UUID>.cfargotunnel.com` DNS 레코드 생성
4. public HTTPS 종단 = Cloudflare edge, connector 가 내부 origin(HTTP 가능)으로 전달 → 포트포워딩/고정 IP 불필요

origin 이 self-signed HTTPS 면 cert name mismatch 주의 — 첫 셋업은 내부 HTTP 가 단순. 방문자는 어차피 Cloudflare 까지 public HTTPS.

## cloudflared vs Cloudflare Mesh — 무엇을 쓸까

> [!info] 공식 비교
> | | cloudflared (Tunnel) | Cloudflare Mesh (node) |
> |---|---|---|
> | 방향 | **단방향** (outbound-only) | **양방향** TCP/UDP/ICMP |
> | OSI | L7 | L3 |
> | 프로토콜 | HTTP/S, TCP, SSH, RDP, SMB | TCP, UDP, ICMP |
> | TCP | "terminated and re-established at Cloudflare" | "End-to-end — preserves long-lived TCP" |
> | 용도 | hostname 으로 앱 publish, user→app | 서버발 트래픽(VoIP/AD/SCCM), site-to-site, source IP 보존 |

**홈랩 권장:** ②③ 는 **cloudflared** 로 충분. 다음 경우에만 **Mesh node** 로 격상:
- LAN ↔ 다른 site(사무실 등) 양방향 mesh
- 서버가 먼저 connection 여는 트래픽 (VoIP/SIP, DB replication)
- source IP 보존 필요
- "Both methods can be used together" — 혼용 가능: 기본은 Tunnel, 양방향 필요 지점만 Mesh node.

## 주의점 / 함정 모음

> [!danger] 운영 주의
> - **ingress authoritative**: cloudflared config(IaC)가 ingress 리스트 전체를 덮어쓴다. Dashboard 수동 rule 은 config push 시 wiped. → [[reference_cloudflare_tunnel_ingress_authoritative]] 패턴 동일. 홈랩도 config.yml 단일 소스로 관리 권장.
> - **connector 위치**: Proxmox host 직접보다 **전용 LXC**. host 재부팅/AMI 회전과 tunnel lifecycle 분리.
> - **LXC TUN device**: WARP/Mesh node 는 `/dev/net/tun` 필요. LXC 에서 unprivileged 면 호스트에서 device passthrough 설정 필요 (기존 db-middleman 노트에서 확인된 전제).
> - **public/private 혼재 보안 경계**: ②(private)와 ③(public)을 같은 터널에 섞으면 노출면 관리가 흐려짐. 노출 앱 늘면 public 전용 / private 전용 터널 분리 고려.
> - **vnet 대역 충돌**: 기존 `engineering` vnet 과 홈 LAN CIDR 겹치면 virtual network 분리 필수.

## 다음 액션 (구현 시)

- [ ] 전용 LXC 생성 (Ubuntu, `/dev/net/tun` passthrough)
- [ ] cloudflared 설치 + Named Tunnel 생성, config.yml 단일 소스화
- [ ] CIDR route `192.168.x.0/24` 등록 + WARP Split Tunnel Include
- [ ] dnsmasq + Local Domain Fallback `home.internal`
- [ ] 노출 대상별 Public Hostname route
- [ ] (옵션) 양방향 필요 시 Mesh node 평가

## 출처

- [Private networks · Cloudflare One docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/)
- [Connect an IP/CIDR · Cloudflare One docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/connect-cidr/)
- [Choose a connection method (cloudflared vs Mesh) · Learning Paths](https://developers.cloudflare.com/learning-paths/replace-vpn/connect-private-network/connection-methods/)
- [Cloudflare Mesh - Private networking · Cloudflare One docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/warp-connector/)
- [Introducing Cloudflare Mesh (blog)](https://blog.cloudflare.com/mesh/)
- [Introducing WARP Connector (blog)](https://blog.cloudflare.com/introducing-warp-connector-paving-the-path-to-any-to-any-connectivity-2/)
- [Resolve private DNS (Local Domain Fallback vs Resolver Policies) · Learning Paths](https://developers.cloudflare.com/learning-paths/replace-vpn/configure-device-agent/private-dns/)
- [Resolver policies · Cloudflare One docs](https://developers.cloudflare.com/cloudflare-one/traffic-policies/resolver-policies/)
- [Internal DNS get started · Cloudflare DNS docs](https://developers.cloudflare.com/dns/internal-dns/get-started/)
- [Connect and secure any app by hostname, free (blog)](https://blog.cloudflare.com/tunnel-hostname-routing/)
- [Routing · Cloudflare Tunnel docs](https://developers.cloudflare.com/tunnel/routing/)
