---
type: reference
status: active
publish: true
date: 2026-06-09
tags:
  - type/reference
  - topic/homelab
  - topic/proxmox
  - topic/linux
  - status/active
topics:
  - proxmox-ve
  - vm-clone
  - cloud-init
  - machine-id
  - golden-template
  - serial-console
related:
  - "[[2026-06-09_Proxmox-8to9-업그레이드]]"
  - "[[2026-06-09_Linux-Proxmox-VM-디스크-용량-진단-확장]]"
  - "[[2026-06-09_Proxmox-Docker-Compose-부팅-자동시작-Harbor-로그순서]]"
aliases:
  - Proxmox Linked vs Full Clone
  - VM machine-id 중복
  - Cloud-Init 골든 템플릿
---

# Proxmox VM 복제 & 골든 템플릿

템플릿을 복사해 VM 을 찍어낼 때의 클론 유형 선택, 그리고 **"클론의 역습"**(중복 정체성으로 인한 장애) 회피 + Cloud-Init 골든 템플릿 제작법.

## 1. Linked Clone vs Full Clone

| | Linked Clone | Full Clone |
|---|---|---|
| 생성 속도 | 즉시 (<1s) | 느림 (디스크 크기 비례) |
| 초기 용량 | 매우 작음 (delta 만) | 원본 100% |
| 의존성 | **템플릿 필수** (없으면 깨짐) | 완전 독립 |
| 디스크 동작 | template 읽기 + diff 쓰기 (CoW) | 자체 디스크 R/W |
| 성능 | 쓰기 시 CoW 페널티 | 네이티브 (최상) |
| 적합 | 랩·테스트·임시 VM·VDI | 운영·DB·장기 서버 |

- **Linked**: 50GB 템플릿 + 클론 10개 ≈ 추가 0GB. 단 **부모 템플릿이 SPOF** — 삭제/손상 시 즉시 깨짐. 장기간 업데이트로 diff 가 커지면 용량 이점이 사라지면서 CoW 오버헤드만 남는다.
- **Full**: 완전 격리·이식성·write-heavy(DB) 유리. 대신 프로비저닝 느리고 OS 파일 중복 저장.
- Linked → Full 전환("flatten")도 가능.

> [!tip] 운영 DB / 장기 서버는 Full Clone. 홈랩 클러스터(K8s·Ceph·Swarm) 다중 노드처럼 용량이 빠듯한 실험은 Linked.

## 2. 클론의 역습 — 중복 정체성 3대 증상

같은 템플릿을 복사하면 OS 고유 식별자가 그대로 복제되어 기괴한 장애가 난다. `hostnamectl set-hostname` 은 커널 호스트네임만 바꿀 뿐 근본 원인을 안 건드린다.

### 증상 A — `sudo` 가 느려짐
- 원인: `/etc/hosts` 의 `127.0.1.1 ubuntu-template` 이 옛 이름 그대로. `sudo` 가 자기 호스트네임 IP 를 못 찾아 DNS 타임아웃까지 대기.
- 처치: `/etc/hosts` 의 loopback 라인을 새 호스트네임으로 수정.

### 증상 B — VM X 로 SSH 했는데 Y 로 접속 / IP 충돌
- 원인: **`/etc/machine-id` 중복**. systemd 는 DHCP DUID 를 machine-id 기반으로 생성 → 라우터가 X·Y 를 "이름만 다른 같은 기계"로 인식, 같은 IP 배정·라우팅 꼬임.
- machine-id 가 비어 있으면 부팅 시 새로 생성되지만, **값이 있으면 그대로 박제**되어 복제된다.

### "왜 예전 클론은 멀쩡하고 오늘 것만?"
유력 가설: 과거엔 설치 직후 곧바로 템플릿화(또는 machine-id 초기화)했으나, 이번엔 **템플릿 VM 을 잠깐 켜서 `apt upgrade` 등을 한 뒤 machine-id 초기화 없이 다시 템플릿으로 변환** → ID 가 박제된 채 복제됨.

## 3. 골든 템플릿 만들기 (Best Practice)

### 3-1. 정체성 봉인 (템플릿화 직전 필수)
```bash
# machine-id 비우기 (삭제가 아니라 truncate)
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# SSH 호스트키 삭제 (부팅 시 재생성)
sudo rm -f /etc/ssh/ssh_host_*

# Cloud-Init 캐시 삭제
sudo cloud-init clean
```

### 3-2. Cloud-Init 적용
1. VM 에 `cloud-init` 설치 (`sudo apt install cloud-init -y`)
2. Proxmox Hardware → **Add → CloudInit Drive** (VM 디스크와 같은 스토리지)
3. **Cloud-Init 탭**에서 기본 User/Password/**SSH public key**, IP Config(보통 DHCP) 설정
4. VM 우클릭 → **Convert to Template**
5. 복제(Full Clone 권장) 후 새 VM 의 Cloud-Init 탭에서 Hostname/IP 만 바꾸고 **Regenerate Image** → Start

→ 부팅 시 hostname·`/etc/hosts`·machine-id 가 자동으로 올바르게 주입된다. 더 이상 터미널 노가다 불필요.

### 3-3. Serial Console 활성화 (xterm.js 복붙 가능)
Proxmox Hardware 에서 Display 만 `Serial terminal 0` 으로 바꾸면 화면이 안 뜬다 — **Guest OS GRUB 설정**도 같이 해야 한다.
```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet console=tty0 console=ttyS0,115200"
# (선택) GRUB 메뉴도 시리얼로
GRUB_TERMINAL=serial
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```
```bash
sudo update-grub   # 후 poweroff → Proxmox Display=serial0 → Start
```

> [!warning] `error: prohibited by secure boot policy`
> 커널 파라미터/GRUB 변경을 Secure Boot 가 차단. 개발·테스트 VM 이면 끈다: VM 부팅 시 ESC 연타 → UEFI → **Device Manager → Secure Boot Configuration → Attempt Secure Boot 체크 해제(Space)** → F10 저장. 같이 뜨는 `snd_hda_intel: no codecs found` 는 Hardware 탭에서 Audio Device **Remove** 하면 됨(서버엔 불필요).

### 3-4. 템플릿용 타이머 정리 (지뢰 제거)
새 VM 부팅 직후 자동 `apt` 가 락을 잡아 `Waiting for cache lock` 을 유발한다. 끄는 게 정신건강에 좋다.
```bash
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer  # 가장 중요
sudo systemctl disable --now motd-news.timer        # 로그인 시 광고/뉴스 fetch
sudo systemctl disable --now fwupd-refresh.timer    # 베어메탈 펌웨어용, VM엔 불필요
```
**남겨둘 것**: `fstrim.timer`(thin provisioning 공간 회수·SSD 수명), `logrotate.timer`(로그 비대 방지), `sysstat-collect.timer`(성능 기록).

### 3-5. 최종 봉인
```bash
sudo cloud-init clean
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo poweroff   # → Convert to Template
```

---
홈랩 인덱스: [[02-Areas/homelab/index|homelab]]
