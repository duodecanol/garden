---
type: reference
status: active
publish: true
date: 2026-03-27
tags:
  - type/reference
  - topic/dev
  - topic/obsidian
  - topic/cloud-storage
topics:
  - Obsidian Know-how
  - Image Management
  - Cloud Storage
  - CDN
related:
  - "[[hello-world]]"
  - "[[2026-03-28_Obsidian-Image-Upload-Setup]]"
---

# Obsidian Image Cloud Upload Tools Research

Obsidian vault에 붙여넣은 이미지를 클라우드 스토리지에 자동 업로드하고, 로컬 참조(`![[image.png]]`)를 원격 URL(`![](https://...)`)로 교체한 뒤, 로컬 파일을 삭제하는 워크플로우에 대한 리서치 결과.

---

## 1. Obsidian Community Plugins

### 1.1 Image Auto Upload (PicGo)

- **Plugin ID**: `obsidian-image-auto-upload-plugin`
- **Repo**: [renmu123/obsidian-image-auto-upload-plugin](https://github.com/renmu123/obsidian-image-auto-upload-plugin)
- **How it works**: PicGo / PicList / PicGo-Core를 백엔드로 사용하여 클립보드 이미지를 붙여넣기 시 자동 업로드. "Upload all images" 커맨드로 노트 내 모든 로컬 이미지 일괄 업로드 가능.
- **Supported providers**: PicGo가 지원하는 60+ 서비스 (AWS S3, Cloudflare R2, Imgur, GitHub, Aliyun OSS, Tencent COS, Qiniu 등). PicGo S3 플러그인으로 R2 연동 가능.
- **Git/multi-device**: Desktop only. PicGo 앱이 별도로 실행되어야 함. 각 기기에 PicGo 설정 필요.
- **Limitations**: PicGo 데스크톱 앱 또는 PicGo-Core가 별도 설치 필요. Mobile 미지원. Frontmatter `image-auto-upload: false`로 개별 파일 제어 가능.
- **Key strength**: 가장 오래되고 널리 사용되는 플러그인. PicGo 생태계(60+ providers) 활용.

### 1.2 Custom Image Auto Uploader

- **Plugin ID**: `custom-image-auto-uploader`
- **Repo**: [haierkeys/obsidian-custom-image-auto-uploader](https://github.com/haierkeys/obsidian-custom-image-auto-uploader)
- **How it works**: Desktop, iOS, Android 모두 지원. 이미지 일괄 다운로드/업로드/크롭/리사이즈 가능. 자체 API 서버 또는 클라우드 스토리지에 직접 연결.
- **Supported providers**: ==Alibaba Cloud OSS, Amazon S3, Cloudflare R2, MinIO, WebDAV, 자체 서버/NAS==
- **Git/multi-device**: 모바일 포함 크로스 플랫폼 지원. Git 기반 vault에서 사용 가능.
- **Limitations**: 자체 API 서버 설정이 필요할 수 있음 (직접 S3/R2 연결도 가능).
- **Key strength**: ==모바일 지원 + R2 직접 지원== + 이미지 크롭/리사이즈 기능. Hugo/Hexo 블로그 사용자에게 특히 유용.

### 1.3 Image Upload Toolkit

- **Plugin ID**: `image-upload-toolkit`
- **Repo**: [addozhang/obsidian-image-upload-toolkit](https://github.com/addozhang/obsidian-image-upload-toolkit)
- **How it works**: "Publish page" 커맨드로 노트 내 모든 로컬 이미지를 업로드하고, 이미지 URL이 교체된 마크다운을 클립보드에 복사. 원본 vault 파일은 로컬 이미지 참조 유지.
- **Supported providers**: Imgur, Aliyun OSS, ImageKit, AWS S3, Tencent COS, Qiniu Kodo, ==Cloudflare R2==
- **Git/multi-device**: Desktop only.
- **Limitations**: 기본 동작은 vault 원본을 수정하지 않고 클립보드에 복사. 정적 사이트 퍼블리싱 용도에 최적화. 로컬 이미지 자동 삭제 기능은 없음.
- **Key strength**: 퍼블리싱 워크플로우에 적합. 원본 보존 방식.

### 1.4 S3 Image Uploader

- **Plugin ID**: `s3-image-uploader`
- **Repo**: [jvsteiner/s3-image-uploader](https://github.com/jvsteiner/s3-image-uploader)
- **How it works**: 이미지를 붙여넣거나 드래그 앤 드롭하면 S3에 업로드하고 마크다운 링크 삽입. YAML frontmatter 설정 지원.
- **Supported providers**: AWS S3 및 ==모든 S3 호환 서비스 (Cloudflare R2 포함)==
- **Git/multi-device**: Desktop only. 각 기기에 S3 credentials 설정 필요.
- **Limitations**: 기존 로컬 이미지 일괄 업로드 기능은 제한적. 붙여넣기 시점에 업로드하는 방식.
- **Key strength**: 외부 앱 불필요. S3 API 직접 호출. 비디오, 오디오, PDF도 지원.

### 1.5 Image Uploader For Note

- **Plugin ID**: `image-uploader-for-note`
- **Repo**: [yy4382/obsidian-image-upload](https://github.com/yy4382/obsidian-image-upload)
- **How it works**: 리본 버튼 또는 커맨드 팔레트로 현재 노트의 이미지 업로드. S3 링크로 교체. ==해당 노트에서만 사용되는 이미지는 vault에서 삭제==.
- **Supported providers**: S3 및 S3 호환 서비스. Custom JS 플러그인으로 커스텀 업로더 작성 가능.
- **Git/multi-device**: Desktop only.
- **Limitations**: 자동 업로드가 아닌 수동 트리거 방식.
- **Key strength**: ==로컬 이미지 자동 삭제 기능 내장== (해당 노트에서만 사용되는 경우). 가장 원하는 워크플로우에 근접.

### 1.6 Cloudflare Image Auto Uploader

- **Plugin ID**: `cloudflare-image-auto-uploader`
- **Repo**: [youngseokyoon/obsidian-cloudflare-plugin](https://github.com/youngseokyoon/obsidian-cloudflare-plugin)
- **How it works**: 이미지 붙여넣기 시 ==Cloudflare R2에 자동 업로드==. 로컬 복사본 유지 옵션. 이미지 크기 조정 모드 (Fixed, Percentage, Auto).
- **Supported providers**: ==Cloudflare R2 전용==
- **Git/multi-device**: Desktop. "Keep local copy" 옵션으로 로컬 fallback 가능.
- **Limitations**: Cloudflare R2에만 특화. 비교적 새로운 플러그인 (2025년 커뮤니티 등록).
- **Key strength**: Cloudflare R2 네이티브 지원. 업로드 실패 시 로컬 fallback.

### 1.7 Cloudinary Uploader

- **Plugin ID**: `obsidian-cloudinary-uploader`
- **Repo**: [jordanhandy/obsidian-cloudinary-uploader](https://github.com/jordanhandy/obsidian-cloudinary-uploader)
- **How it works**: 클립보드에서 붙여넣기 시 Cloudinary에 자동 업로드. 이미지, 비디오, 오디오, raw 파일 지원.
- **Supported providers**: ==Cloudinary 전용==
- **Git/multi-device**: Desktop only.
- **Limitations**: Cloudinary 무료 플랜의 월별 transformation 제한. 기존 로컬 이미지 일괄 마이그레이션 기능 없음.
- **Key strength**: Unsigned upload으로 API secret 불필요. URL 기반 이미지 transformation (리사이즈, 크롭, 필터).

### 1.8 Imgur Plugin

- **Plugin ID**: `obsidian-imgur-plugin`
- **Repo**: [gavvvr/obsidian-imgur-plugin](https://github.com/gavvvr/obsidian-imgur-plugin)
- **How it works**: 이미지 붙여넣기 시 Imgur에 업로드. Anonymous 또는 OAuth 인증 모드.
- **Supported providers**: ==Imgur 전용==
- **Git/multi-device**: Desktop only.
- **Limitations**: 시간당 50장 업로드 제한. 일별 API 호출 제한. 비디오 미지원. 기존 이미지 일괄 업로드 미지원 (feature request 존재).
- **Key strength**: 가장 간단한 설정. 무료. 별도 클라우드 계정 불필요.

### 1.9 NotePix (GitHub Image Upload)

- **Repo**: [AyushParkara/NotePix](https://github.com/AyushParkara/NotePix)
- **How it works**: 이미지를 GitHub 리포지토리에 업로드하고 `![[image.png]]`를 `![](https://raw.githubusercontent.com/...)` URL로 교체.
- **Supported providers**: ==GitHub (Public/Private repos)==
- **Git/multi-device**: Desktop + Mobile (Android/iOS). PAT는 AES-GCM으로 암호화 저장.
- **Limitations**: GitHub repo 크기 제한 (권장 1GB, hard limit 5GB). 큰 vault에는 부적합할 수 있음.
- **Key strength**: 별도 클라우드 스토리지 불필요. GitHub 생태계 내에서 완결. 모바일 지원.

### 1.10 WebDAV Image Uploader

- **Plugin ID**: `webdav-image-uploader`
- **Repo**: [Koishiiko/obsidian-webdav-image-uploader](https://github.com/Koishiiko/obsidian-webdav-image-uploader)
- **How it works**: 붙여넣기/드래그 시 WebDAV 서버에 업로드. 일괄 업로드/다운로드. 업로드 후 로컬 파일 보존 여부 설정 가능.
- **Supported providers**: ==모든 WebDAV 서버== (NextCloud, Synology, 자체 서버 등)
- **Git/multi-device**: Desktop only.
- **Limitations**: WebDAV 서버 필요. 이미지를 수동으로 fetch하여 표시하므로 Reading mode에서 간헐적 문제.
- **Key strength**: 업로드 후 로컬 파일 삭제 옵션. PDF 지원. 자체 인프라 사용 가능.

---

## 2. CLI Tools & Scripts

### 2.1 PicGo / PicList (Standalone)

- **Repo**: [Molunerfinn/PicGo](https://github.com/Molunerfinn/PicGo)
- **Description**: 독립 실행형 이미지 업로드 데스크톱 앱. 60+ 클라우드 서비스 지원.
- **Cloudflare R2**: S3 플러그인으로 지원. Endpoint, Access Key, Secret Key 설정.
- **Obsidian 연동**: Image Auto Upload 플러그인의 백엔드로 동작. CLI 모드(PicGo-Core)로도 사용 가능.
- **장점**: 가장 넓은 클라우드 지원. 플러그인 생태계.
- **단점**: 별도 앱 설치/실행 필요.

### 2.2 Cloudflare Image Upload Script (Python)

- **Repo**: [bigsk1/cloudflare-image-upload](https://github.com/bigsk1/cloudflare-image-upload)
- **Description**: Python 스크립트. 이미지를 Cloudflare Images에 벌크 업로드하고, 업로드된 이미지의 Markdown 파일 생성.
- **사용법**: 이미지 디렉토리 지정 → 벌크 업로드 → MD 파일에 URL 목록 생성. 중복 처리, 처리 완료 이미지 이동.
- **제한**: Obsidian vault와 직접 통합은 아님. 마크다운 내 참조 교체는 별도 스크립트 필요.

### 2.3 Custom Python Script (S3 Upload + Replace)

- **Obsidian Forum**: [Upload obsidian image to S3](https://forum.obsidian.md/t/upload-obsidian-image-to-s3/77908)
- **Description**: 커뮤니티에서 공유된 Python 스크립트. vault 내 마크다운 파일을 스캔하여 로컬 이미지 참조를 찾고, S3에 업로드 후 링크를 교체.
- **장점**: 완전 커스터마이즈 가능. Git hook이나 CI/CD에 통합 가능.
- **단점**: 직접 작성/유지보수 필요.

---

## 3. GitHub Actions / CI/CD Approaches

### 3.1 Git Pre-commit Hook + S3 Upload

커스텀 스크립트 기반 접근:

1. Git commit 시 pre-commit hook 실행
2. 변경된 `.md` 파일에서 로컬 이미지 참조 파싱
3. 이미지 파일을 S3/R2에 업로드
4. 마크다운 내 참조를 CDN URL로 교체
5. 로컬 이미지 파일 삭제 후 `.gitignore`에 추가

> [!warning] 주의사항
> 현재 이 워크플로우를 완전히 자동화하는 공개된 GitHub Action은 발견되지 않았음. 커스텀 구현 필요.

### 3.2 GitHub Actions Workflow (DIY)

```yaml
# 개념적 워크플로우 예시
on:
  push:
    paths:
      - '**.png'
      - '**.jpg'
      - '**.jpeg'
jobs:
  upload-images:
    steps:
      - uses: actions/checkout@v4
      - name: Upload images to R2
        # aws s3 cp or rclone to R2
      - name: Replace references in markdown
        # sed/python script to replace ![[image]] with ![](url)
      - name: Remove local images and commit
        # git rm images, git commit, git push
```

---

## 4. Comparison Matrix

| Plugin | Auto Upload | Batch Upload | Delete Local | R2 Support | Mobile | External App |
|---|---|---|---|---|---|---|
| Image Auto Upload (PicGo) | Yes | Yes | No | Via PicGo | No | PicGo 필요 |
| Custom Image Auto Uploader | Yes | Yes | No | ==Direct== | ==Yes== | Optional |
| Image Upload Toolkit | No | Yes | No | ==Direct== | No | No |
| S3 Image Uploader | Yes | Limited | No | ==S3 compat== | No | No |
| Image Uploader For Note | No (manual) | Yes | ==Yes== | S3 compat | No | No |
| Cloudflare Image Auto Uploader | Yes | No | Optional | ==Native== | No | No |
| Cloudinary Uploader | Yes | No | No | No | No | No |
| Imgur | Yes | No | No | No | No | No |
| NotePix | Yes | No | Yes (implicit) | No | ==Yes== | No |
| WebDAV Image Uploader | Yes | Yes | ==Yes== | No | No | No |

---

## 5. Recommended Approaches

### Best for Cloudflare R2 Specific

1. **Cloudflare Image Auto Uploader** -- R2 네이티브 지원, 자동 업로드
2. **Custom Image Auto Uploader** -- R2 직접 지원 + 모바일 + 배치 기능
3. **S3 Image Uploader** -- S3 호환 API로 R2 연결

### Best for "Upload + Replace + Delete Local" Workflow

1. ==**Image Uploader For Note**== -- S3 업로드 + 링크 교체 + 로컬 삭제 (단독 사용 노트에 한해)
2. **WebDAV Image Uploader** -- WebDAV 서버 필요하지만 로컬 삭제 옵션 내장
3. **Custom Image Auto Uploader** -- 가장 다기능이지만 로컬 삭제는 수동

### Best for Git-based Multi-device Setup

1. **Custom Image Auto Uploader** -- 유일하게 Desktop + Mobile 모두 지원하면서 R2 직접 연결
2. **NotePix** -- GitHub 기반으로 Git vault와 자연스럽게 통합
3. **PicGo + Image Auto Upload** -- 가장 넓은 provider 지원이지만 각 기기에 PicGo 설치 필요

### DIY / Maximum Control

- **Python script + GitHub Actions** -- 완전한 커스터마이즈. Cloudflare R2 API 직접 호출. Git hook으로 자동화.

---

> [!tip] 구축 기록
> 이 리서치를 바탕으로 한 실제 구축 내용은 [[2026-03-28_Obsidian-Image-Upload-Setup]] 참조.
