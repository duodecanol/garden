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
