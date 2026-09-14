---
type: reference
status: active
publish: true
date: 2026-09-11
tags:
  - type/reference
  - topic/homelab
  - topic/proxmox
  - status/active
topics:
  - proxmox-ve
  - vm-template
  - backup-restore
  - standalone-node
  - vm-rename
related:
  - "[[2026-09-11_Proxmox-9.2-No-valid-subscription-제거]]"
  - "[[2026-06-09_Proxmox-VM-복제-골든템플릿]]"
aliases:
  - Proxmox VM 템플릿 서버 간 복사
  - Proxmox standalone 노드 간 VM 이동
  - Proxmox VM 이름 변경
---

# Proxmox VM 템플릿 — 서로 다른 서버 간 복사 및 이름 변경

## 결론

서로 같은 클러스터에 속하지 않은 Proxmox 서버 간에도 VM 템플릿을 복사해 등록할 수 있다.

권장 흐름:

```text
원본 Template
  → 임시 Full Clone
  → vzdump 백업
  → 대상 서버로 백업 파일 전송
  → qmrestore
  → VM 이름 변경
  → Template 변환
```

템플릿 자체를 직접 복사하거나 디스크 파일만 옮기는 것보다 백업/복원이 안전하다. VM 설정과 디스크가 함께 이동되기 때문이다.

## 1. 원본 서버에서 임시 Full Clone 생성

예시:

- 원본 템플릿 VMID: `9000`
- 임시 VMID: `9900`
- 대상 서버 VMID: `9000`

```bash
qm clone 9000 9900 \
  --name "ubuntu-2404-template-transfer" \
  --full 1
```

`--full 1`이 중요하다. Linked Clone은 원본 템플릿에 종속되므로 서버 간 이동용으로 사용하지 않는다.

복제 상태 확인:

```bash
qm config 9900
```

필요하면 임시 VM을 부팅해 확인한 뒤 종료한다.

```bash
qm start 9900
qm shutdown 9900
```

## 2. 임시 VM 백업

충분한 공간이 있는 디렉터리를 사용한다.

```bash
mkdir -p /mnt/transfer

vzdump 9900 \
  --mode stop \
  --compress zstd \
  --dumpdir /mnt/transfer
```

백업 파일 예시:

```text
vzdump-qemu-9900-YYYY_MM_DD-HH_MM_SS.vma.zst
```

확인:

```bash
ls -lh /mnt/transfer/
```

템플릿 자체를 직접 백업하는 것보다 Full Clone을 일반 VM으로 만든 뒤 백업하는 편이 안전하다. 템플릿 직접 백업은 환경에 따라 실패 사례가 있다.

## 3. 대상 서버로 백업 파일 전송

```bash
scp /mnt/transfer/vzdump-qemu-9900-*.vma.zst \
  root@TARGET_PROXMOX:/mnt/transfer/
```

대용량 파일은 `rsync`를 사용할 수 있다.

```bash
rsync -avh --progress \
  /mnt/transfer/vzdump-qemu-9900-*.vma.zst \
  root@TARGET_PROXMOX:/mnt/transfer/
```

반복 작업이라면 두 Proxmox 서버가 공통으로 접근하는 Proxmox Backup Server를 사용하는 편이 좋다.

## 4. 대상 서버에서 VM 복원

```bash
qmrestore \
  /mnt/transfer/vzdump-qemu-9900-YYYY_MM_DD-HH_MM_SS.vma.zst \
  9000 \
  --storage local-lvm \
  --unique 1
```

`local-lvm`은 대상 서버의 실제 스토리지 ID로 변경한다.

- `9000`: 대상 서버에서 사용할 새 VMID
- `--storage`: 대상 스토리지
- `--unique 1`: MAC 주소 충돌 방지

복원 후 확인:

```bash
qm config 9000
```

필요하면 테스트 부팅한다.

```bash
qm start 9000
qm shutdown 9000
```

## 5. 복원된 VM 이름 변경

### CLI

```bash
qm set 9000 --name "ubuntu-2404-cloud-template"
```

확인:

```bash
qm config 9000
```

다음과 같이 표시된다.

```text
name: ubuntu-2404-cloud-template
```

### GUI

VM 선택 → **Options** → **Name** → **Edit**

## 6. Template으로 변환

```bash
qm template 9000
```

확인:

```bash
qm config 9000
```

다음 항목이 있으면 템플릿이다.

```text
template: 1
```

## GUI 작업 흐름

1. 원본 템플릿 우클릭
2. **Clone** 선택
3. **Full Clone**으로 임시 VM 생성
4. 임시 VM 백업
5. `.vma.zst` 파일을 대상 서버로 전송
6. 대상 서버에서 백업 복원
7. 복원된 VM 이름 변경
8. **Convert to template** 실행

## VM 이름, VMID, 게스트 hostname의 차이

### Proxmox VM 이름

Web GUI에 표시되는 설정 이름이다.

```bash
qm set 9000 --name "ubuntu-template"
```

### VMID

VM 식별자다. 이름을 바꿔도 VMID는 바뀌지 않는다.

```text
9000 → 9100
```

처럼 VMID까지 변경하려면 새 VMID로 Full Clone하거나 백업/복원해야 한다.

### 게스트 OS hostname

Proxmox VM 이름과 별개로 게스트 운영체제 내부에서 관리된다.

Ubuntu 예시:

```bash
hostnamectl set-hostname ubuntu-template
```

Cloud-Init 템플릿은 복제 후 VM마다 hostname을 지정하는 방식을 권장한다.

## 주의사항

- 두 서버가 같은 Proxmox 클러스터일 필요가 없다.
- 대상 서버의 스토리지 ID와 네트워크 브리지 이름이 다르면 복원 후 수정한다.
  - 예: `local-lvm`
  - 예: `vmbr0`
- 템플릿에 SSH host key, 고정 MAC 설정, 사용자 데이터가 남아 있으면 그대로 복사된다.
- 운영용 템플릿은 생성 전에 SSH host key, persistent network MAC 설정, 불필요한 사용자 데이터와 계정을 정리한다.
- VM 디스크 파일만 직접 복사하면 Proxmox VM 설정이 자동으로 등록되지 않는다.

## 공식 문서

- [VM Templates and Clones](https://pve.proxmox.com/wiki/VM_Templates_and_Clones)
- [Backup and Restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
- [qmrestore(1)](https://pve.proxmox.com/pve-docs/qmrestore.1.html)
- [qm(1)](https://pve.proxmox.com/pve-docs/qm.1.html)
