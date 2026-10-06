---
title: Homelab Index
type: index
date: 2026-06-05
tags:
  - topic/homelab
  - topic/networking
publish: true
---

# Homelab

홈랩(Proxmox + LAN) 운영을 종료 없이 지속 관리하는 영역. 네트워킹(Cloudflare Zero Trust/Mesh), 가상화, 셀프호스팅 서비스 설계를 모은다.

> [!note] 관련 프로젝트
> 같은 Cloudflare Zero Trust Team org 의 업무용 맥락은 [[01-Projects/oshiz-data-insight/index|oshiz-data-insight]] (db-middleman CT) 에 있다. 이 영역은 홈랩 사적 인프라 전반을 다룬다.
- [[Ubuntu-Ventoy-USB-운영-가이드|Ubuntu에서 Ventoy USB 운영 가이드]] — Ubuntu 기반 Ventoy 부팅 USB 운영 및 장애 대응
- [[2026-10-01_Proxmox-9.2-Ventoy-Gigabyte-BIOS-GPU-설정-비교|Proxmox 9.2 Ventoy 설치 전 Gigabyte BIOS·GPU 설정 비교]] — Ventoy 설치 전 BIOS·GPU 설정 비교
- [[2026-10-01_Proxmox-9.2-Ventoy-설치화면-진입실패-원인-해결|Proxmox 9.2 Ventoy 설치화면 진입 실패 원인과 해결]] — NVIDIA/NovaCore·Ventoy 부팅 멈춤 진단 및 우회

```dataview
TABLE WITHOUT ID
  file.link AS "Note",
  date AS "Date",
  status AS "Status",
  topics AS "Topics"
FROM "02-Areas/homelab"
WHERE file.name != "index"
SORT date DESC
```
