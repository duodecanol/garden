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
sudo cloud-init clean --logs
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo apt clean
sudo poweroff   # → Convert to Template
```
## 4. 개발용 VM 도구 선설치 — Cloud-Init으로 가능

**결론: 가능하다.** 다만 세 도구의 설치 성격이 다르다.

| 도구 | 설치 위치 | Cloud-Init 적합도 | 주의점 |
|---|---|---:|---|
| Docker Engine | 시스템 전역 | 높음 | 공식 Docker APT 저장소를 먼저 등록해야 함 |
| Homebrew on Linux (구 Linuxbrew) | 특정 일반 사용자 | 중간 | root 설치 불가·단일 사용자 설치·초기 `sudo` 필요 |
| mise | 특정 일반 사용자 | 높음 | root가 아닌 대상 사용자의 `HOME`으로 설치해야 함 |

### 4-1. 권장 분리

- **골든 템플릿에 미리 설치**: `cloud-init`, `qemu-guest-agent`, `curl`, `git`, `build-essential`, Docker, Homebrew, mise처럼 모든 VM이 공통으로 쓸 도구.
- **클론별 Cloud-Init**: hostname, SSH 키, IP, 역할별 도구와 설정.
- Docker는 시스템 서비스이므로 템플릿에 선설치하기 쉽다.
- Homebrew와 mise는 사용자별 상태이므로 템플릿의 실제 개발 사용자(`dev` 등)를 정하고 그 사용자로 설치한다. 여러 사용자가 로그인할 VM이면 사용자별 설치가 필요하다.
- 매 클론 첫 부팅마다 외부 저장소에서 설치하면 부팅 시간이 길고 네트워크·저장소 상태에 의존한다. 재현 가능한 골든 템플릿이 목적이면 도구 설치를 1회 수행한 뒤 `cloud-init clean` 하고 봉인하는 편이 낫다.

### 4-2. Proxmox custom user-data 예시 (Ubuntu 22.04/24.04)

Proxmox Cloud-Init 탭은 기본 사용자·키·네트워크 설정 중심이다. `runcmd`/`write_files`를 쓰려면 snippets 스토리지의 custom user-data를 `cicustom`으로 연결한다.

아래 예시는 `dev` 사용자가 **이미 존재하고 passwordless sudo가 가능하다**는 전제다. `cicustom user`를 지정하면 Proxmox가 자동 생성하던 user-data를 대체하므로, 사용자·SSH 키 설정도 custom 파일 안에서 관리해야 한다.

```yaml
#cloud-config
package_update: true
package_upgrade: false
packages:
  - ca-certificates
  - curl
  - file
  - git
  - build-essential
  - procps
  - qemu-guest-agent

write_files:
  - path: /usr/local/sbin/install-dev-tools
    permissions: '0755'
    content: |
      #!/usr/bin/env bash
      set -euxo pipefail

      TARGET_USER=dev
      TARGET_HOME="$(getent passwd "$TARGET_USER" | cut -d: -f6)"
      test -n "$TARGET_HOME"

      # Docker 공식 APT 저장소
      install -m 0755 -d /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
        -o /etc/apt/keyrings/docker.asc
      chmod a+r /etc/apt/keyrings/docker.asc
      . /etc/os-release
      SUITE="${UBUNTU_CODENAME:-$VERSION_CODENAME}"
      cat >/etc/apt/sources.list.d/docker.sources <<EOF
      Types: deb
      URIs: https://download.docker.com/linux/ubuntu
      Suites: ${SUITE}
      Components: stable
      Architectures: $(dpkg --print-architecture)
      Signed-By: /etc/apt/keyrings/docker.asc
      EOF

      apt-get update
      DEBIAN_FRONTEND=noninteractive apt-get install -y \
        docker-ce docker-ce-cli containerd.io \
        docker-buildx-plugin docker-compose-plugin
      systemctl enable --now docker
      usermod -aG docker "$TARGET_USER"

      # Homebrew on Linux: 공식 설치기는 일반 사용자로 실행해야 한다.
      # 대상 사용자는 passwordless sudo가 가능해야 한다.
      curl -fsSL \
        https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh \
        -o /tmp/install-homebrew.sh
      chown "$TARGET_USER:$TARGET_USER" /tmp/install-homebrew.sh
      runuser -u "$TARGET_USER" -- env \
        HOME="$TARGET_HOME" NONINTERACTIVE=1 \
        /bin/bash /tmp/install-homebrew.sh

      # mise를 Homebrew로 설치하고 대상 사용자의 Bash에 활성화한다.
      runuser -u "$TARGET_USER" -- env HOME="$TARGET_HOME" \
        /bin/bash -lc '
          eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
          HOMEBREW_NO_AUTO_UPDATE=1 brew install mise
          grep -qxF '\''eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"'\'' "$HOME/.bashrc" ||
            echo '\''eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"'\'' >> "$HOME/.bashrc"
          grep -qxF '\''eval "$(mise activate bash)"'\'' "$HOME/.bashrc" ||
            echo '\''eval "$(mise activate bash)"'\'' >> "$HOME/.bashrc"
        '
      rm -f /tmp/install-homebrew.sh
      systemctl enable --now qemu-guest-agent

runcmd:
  - [bash, /usr/local/sbin/install-dev-tools]
```

```bash
# snippets를 지원하는 스토리지에 dev-tools.yaml을 올린 뒤
qm set <VMID> --cicustom user=local:snippets/dev-tools.yaml
```

`docker` 그룹 추가는 다음 로그인부터 적용된다. 따라서 설치 직후 현재 Cloud-Init 프로세스에서 검증하기보다, SSH를 새로 연결한 뒤 `docker run hello-world` 또는 `docker version`으로 확인한다.

### 4-3. 운영상 주의

- `package_upgrade: true`는 템플릿마다 패키지 버전과 재부팅 여부가 달라지므로 골든 이미지에는 보통 사용하지 않는다.
- Homebrew 설치 URL의 `HEAD`는 최신 스크립트를 가리키므로 완전한 재현성은 없다. 장기 운영 템플릿은 설치 시점을 고정하거나, 이미지 빌드 단계에서 검증한 결과를 봉인한다.
- `curl | bash` 대신 예시처럼 파일로 내려받아 실행하면 실패 시 `/tmp`의 설치기를 먼저 확인할 수 있다. 외부 설치기가 변경되면 다시 검토한다.
- 기존 3-4의 `apt-daily*` 타이머 비활성화는 유지한다. 첫 부팅 때 Cloud-Init과 apt 자동 업데이트가 동시에 실행되면 APT lock 경합이 날 수 있다.
- Docker가 UFW/firewalld 포트를 우회할 수 있으므로 Docker를 설치하는 VM은 기존 방화벽 정책과 `DOCKER-USER` 체인을 함께 점검한다.
- `cicustom`에 사용하는 snippets 스토리지는 `snippets` 콘텐츠 타입을 지원해야 하며, 클러스터에서 VM을 옮길 모든 노드가 해당 파일을 읽을 수 있어야 한다.
- 모든 노드가 Docker를 쓸 필요가 없으면 Docker/Homebrew/mise를 공통 템플릿에 넣지 말고, 역할별 custom user-data로 분리한다.

공식 문서: [Proxmox Cloud-Init Support](https://pve.proxmox.com/wiki/Cloud-Init_Support), [Cloud-Init modules](https://docs.cloud-init.io/en/latest/reference/modules.html), [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux), [mise Getting Started](https://mise.jdx.dev/getting-started.html), [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).

---
홈랩 인덱스: [[02-Areas/homelab/index|homelab]]

