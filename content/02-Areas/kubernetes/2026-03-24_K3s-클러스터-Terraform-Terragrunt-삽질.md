---
type: research
status: active
publish: true
date: 2026-03-24
tags:
  - type/research
  - topic/kubernetes
  - topic/k3s
  - topic/terraform
  - topic/iac
  - status/active
topics:
  - terraform
  - terragrunt
  - kubernetes-provider
  - reloader
  - cert-manager
  - crd-lifecycle
related:
  - "[[2026-01-26_K3s-MetalLB-Tailscale-MTU-Flannel-트러블슈팅]]"
  - "[[2026-01-15_K3s-노드-토큰형식-TLS-SAN]]"
  - "[[2026-01-28_K8s-NFS-마운트-Operation-not-permitted-서브넷-Tailscale]]"
aliases:
  - terraform 삽질기
  - kubernetes_manifest reloader env tfstate
  - CRD purge on destroy
---

# K3s 클러스터 IaC — Terraform/Terragrunt 삽질 모음

gpu-intranet K3s 클러스터(imagegen ComfyUI GPU)를 Terraform/Terragrunt + Helm으로 구성하며 누적한 스크래치 + 트러블슈팅. (모듈: metallb · cert_manager · traefik · reflector · reloader · gpu_operator · data_layer)

## 참고 링크 모음 (구성 레퍼런스)

**k3d / k3s**
- [Using Config Files - k3d](https://k3d.io/stable/usage/configfile/) · [k3d config schema.json](https://github.com/k3d-io/k3d/blob/main/pkg/config/v1alpha5/schema.json) · [rjsf playground](https://rjsf-team.github.io/react-jsonschema-form/) · [k3d multiserver.sh](https://github.com/k3d-io/k3d-demo/blob/main/scripts/multiserver.sh)
- [K3s Configuration Options](https://docs.k3s.io/installation/configuration#configuration-file) · [Advanced / nvidia-container-runtime](https://docs.k3s.io/advanced#nvidia-container-runtime)
- ❗NVIDIA: [gpu-operator failing on rke2 #406](https://github.com/NVIDIA/gpu-operator/issues/406#issuecomment-1251509924)
- [K3s cluster setup - The Hotel Hero](https://thehotelhero.com/k3s) · [kubectl 치트시트](https://kubernetes.io/ko/docs/reference/kubectl/cheatsheet/)

**GPU (NVIDIA driver / container toolkit / operator)**
- [Ubuntu NVIDIA drivers](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/) · [NVIDIA Container Toolkit 설치](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) · [GPU Operator 설치](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html)
- ⭐ K3s 중요 세팅: [gpu-operator #406](https://github.com/NVIDIA/gpu-operator/issues/406#issuecomment-1251509924) · [GPU Operator containerd 설정](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html#specifying-configuration-options-for-containerd)

**MetalLB / Ingress / 부트스트랩**
- [MetalLB + Traefik + Longhorn on-prem 가이드](https://medium.com/@kalyanishah86/setting-up-metallb-traefik-ingress-and-longhorn-storage-for-on-premises-kubernetes-c7f39750f243)
- Terraform: [helm-charts (CloudPirates)](https://github.com/CloudPirates-io/helm-charts) · [Ingress vs IngressRoute](https://stackoverflow.com/questions/60177488/) · [remote-exec 가이드](https://scalr.com/learning-center/terraform-remote-exec-a-concise-guide/) · [sudo in terraform](https://stackoverflow.com/questions/37847273/)
- Machine Bootstrapping: [k3s-io/k3s-ansible](https://github.com/k3s-io/k3s-ansible) · [timothystewart6/k3s-ansible](https://github.com/timothystewart6/k3s-ansible) (kube-vip, MetalLB 포함 HA) · [K3s Zero to Hero - 설정](https://blog.alphabravo.io/part-3-k3s-zero-to-hero-mastering-k3s-configuration-from-yaml-to-cli/) · [HA Embedded etcd](https://docs.k3s.io/datastore/ha-embedded)

---

## 1. `reloader` env-var 주입이 `kubernetes_manifest` tfstate를 오염

### 증상
```
Invalid value: "": may not be specified when `value` is not empty
...
Deployment.apps "comfyui-worker" is invalid:
  spec.template.spec.containers[0].env[0].valueFrom.fieldRef.fieldPath: Required value,
  spec.template.spec.containers[0].env[0].valueFrom: Invalid value: "":
    may not be specified when `value` is not empty
```

### 원인 (체인)
1. terraform 배포로 환경변수가 설정됨(worker Deployment, env[0] = `valueFrom`).
2. `stakater/reloader`의 개입을 위한 `annotation`이 붙은 pod에 reloader의 환경변수가 추가 주입됨 — 그것도 **0번에 삽입**.
3. `terraform.state`에 반영됨.
4. 다음 변경 후 `terraform apply`.
5. 특정 deployment 정의의 env와 `terraform.state`의 env 정의가 안 맞음. 원래 0번은 `value` 없고 `valueFrom`이 있었는데, 저장된 state에는 0번 이름도 다르고 상태도 반대.
6. 오류 출력, apply 불가.

### 시도 기록
- **try-01: `kubernetes_manifest` computed_fields (array of objects)** → 안 통함.
- **try-02: field_manager** → 안 통함.
- **try-03: `kubernetes_env`** → 문서 첫 줄을 안 읽은 죄. `> This resource provides a way to manage environment variables in resources that were created **outside of Terraform**.` (Terraform 밖에서 만든 리소스용이라 부적합)
  ```
  Error: Deployment.apps "comfyui-worker" is invalid: [spec.selector: Required value,
    spec.template.metadata.labels: ... does not match template `labels`,
    spec.selector: Invalid value: "null": field is immutable]
  ```
- **try-04 (해결): `stakater/reloader`의 reload strategy 변경 `env-var` → `annotation`.**
  - reload strategy: `env-var`(기본) | `annotation`.
  - `env-var`: 파드 env를 편집 → 위 충돌 유발. pod env에 prepend됨: `STAKATER_<NAME>_[SECRET|CONFIGMAP]`.
  - `annotation`: pod annotation에 append됨 → env 불변 → tfstate 안 깨짐.
    ```
    reloader.stakater.com/last-reloaded-from={"type":"CONFIGMAP","name":"worker-config",
      "namespace":"imagegen-gpu","hash":"e96b...","containerRefs":["comfyui-worker"],...}
    ```
  - 참고: [Reloader How-it-works](https://github.com/stakater/Reloader/blob/master/docs/How-it-works.md) · [Reloader vs k8s-trigger-controller](https://github.com/stakater/Reloader/blob/master/docs/Reloader-vs-k8s-trigger-controller.md) · [reload behavior](https://github.com/stakater/Reloader?tab=readme-ov-file#1--reload-behavior)

> `kubernetes_manifest`는 manifest를 **고정 리스트**로 취급해 "없던 env가 들어왔다"고 화낸다. 반면 `Deployment` 컨트롤러는 자동 계산해 피해준다. → [Terraform Registry: manifest computed-fields](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/manifest#computed-fields)

## 2. MetalLB CRD: `kubernetes_manifest` → `kubectl_manifest`

```
Terraform plan error: API did not recognize GroupVersionKind from manifest (CRD may not be installed)
```
`kubernetes_manifest`는 **plan 시점에 CRD가 이미 설치되어 있어야** 한다(chicken-egg). MetalLB CRD처럼 같은 apply에서 만드는 리소스는 [`gavinbunney/kubectl`의 `kubectl_manifest`](https://registry.terraform.io/providers/gavinbunney/kubectl/latest/docs/resources/kubectl_manifest)로 우회.
- 이슈: [Ignore CRD check flag in kubernetes_manifest #2187](https://github.com/hashicorp/terraform-provider-kubernetes/issues/2187)
- 예시: [hideForming metallb/main.tf](https://github.com/youhide/hideForming/blob/d57d967/terragrunt/kubernetes/metallb/main.tf#L7)

## 3. 배포 중 cluster connection refused → 모듈 의존성

```
Kubernetes cluster unreachable: Get "https://10.0.0.7:6443/version":
  dial tcp 10.0.0.7:6443: connect: connection refused
```
모듈 간 dependency를 제대로 안 걸어서 그럴 수 있다. 예: **Traefik이 제대로 ingress하려면 MetalLB의 IP L2 Advertisement + cert-manager의 인증서 생성이 모두 완료되어야 한다.** (실제 plan 로그상 traefik `helm_release`가 cert-manager 완료 직후 떴다가 connection refused로 실패.) 추가로 `kubernetes_namespace` → `kubernetes_namespace_v1` deprecation 경고도 동반.

## 4. `terraform destroy` 후 CRD 잔존

`These resources were kept due to the resource policy: CustomResourceDefinition` — **`helm uninstall`은 CRD를 안 지운다.**
```bash
helm uninstall traefik -n traefik-system --cascade foreground --wait --ignore-not-found --debug
```
destroy 시 `local-exec` provisioner로 명시 삭제:
```hcl
resource "helm_release" "metallb" {
  name       = "metallb"
  repository = "https://metallb.github.io/metallb"
  chart      = "metallb"
  version    = "0.15.3"
  namespace  = kubernetes_namespace_v1.this.metadata[0].name
  values = [ yamlencode({
    controller = { image = { pullPolicy = "IfNotPresent" } }
    speaker    = { logLevel = "info" }
    crds       = { enabled = true }
  }) ]
  provisioner "local-exec" {
    when    = destroy
    command = <<-EOF
    kubectl get crds -oname | grep metallb.io | sed 's|.*/||' \
      | xargs -n1 --no-run-if-empty kubectl delete crd --cascade=foreground --ignore-not-found --wait
    EOF
  }
}
```
- 고통을 함께하는 자들: [agones #1422](https://github.com/googleforgames/agones/issues/1422) · [Helm: tell Helm not to uninstall a resource](https://helm.sh/docs/howto/charts_tips_and_tricks/#tell-helm-not-to-uninstall-a-resource) · [spacelift: destroy terraform resources](https://spacelift.io/blog/how-to-destroy-terraform-resources)

## 5. cert-manager: 인증서 네임스페이스 분리 → `reflector` 필요

cert-manager로 발급한 Secret(인증서)은 **다른 네임스페이스 파드가 기본적으로 못 본다.** [emberstack/kubernetes-reflector](https://github.com/emberstack/kubernetes-reflector)로 필요한 네임스페이스마다 복제.

증상: Traefik이 LetsEncrypt cert 대신 self-signed로 응답.
```
# 잘못된 상태
*  subject: CN=TRAEFIK DEFAULT CERT
*  issuer:  CN=TRAEFIK DEFAULT CERT
# 올바른 상태
*  subject: CN=*.intranet.example.internal
*  issuer:  C=US; O=Let's Encrypt; CN=R12
*  SSL certificate verify ok.
```
- 참고: [cert-manager Cloudflare DNS01](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/) · [cert-manager ACME](https://cert-manager.io/docs/configuration/acme/)

## 6. `sudo -S` provisioner — command injection 주의

`remote-exec`/`local-exec`로 노드 비밀번호를 넘길 때 `$`·백틱·`!`·공백이 섞이면 깨지거나 의도치 않은 코드 실행 + `ps`에 노출. here-string/here-document으로 셸 해석 차단:
```bash
sudo -S sh /tmp/k8s_conf.sh <<< '${var.node_passwords[count.index]}'
```

## 7. K8s probe / 재시작 메커니즘 메모

- **readiness** probe: Service에서 pod 제거(요청 안 들어오게). **liveness** probe: pod 재시작.
- stop signal 기본 `SIGTERM` → `terminationGracePeriodSeconds`(기본 30s) 후 `SIGKILL`.
  - [Pod Lifecycle - termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-stop-signals) · [Configure Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- ConfigMap/Secret 변경 시 자동 롤링: [`stakater/reloader`](https://github.com/stakater/Reloader) (위 1번 전략 주의) · [데모 영상](https://www.youtube.com/watch?v=iuwJb0GFzuE)

## 8. DNS 캐시 확인 스니펫

```bash
resolvectl query remote.k8s.intranet.example.internal
#   첫 요청:  -- Data from: network
#   두번째:   -- Data from: cache network   ← 캐시 적중
```
> [!warning] DNS cache로 변경된 주소를 못 찾음
> 출력의 `Data from: network` → `cache network` 변화로 캐시 적중을 알 수 있다. 네트워크 세팅이 변경된 때엔 캐시 제거: `sudo resolvectl flush-caches`.
```bash
curl -k -v https://gateway.imagegen-gpu.intranet.example.internal/health
curl -k -v -H "Host: gateway.imagegen-gpu.intranet.example.internal" https://10.0.0.2/docs
```

## 9. 툴 환경

- **VSCode Terraform LS 깨짐** (`helm_release set ... not expected here`) → 확장 **v2.37.6 → v2.37.1** 다운그레이드 (terraform-ls 0.38.2 → 0.37.0). [vscode-terraform #2063](https://github.com/hashicorp/vscode-terraform/issues/2063) · [Official Packaging Guide](https://www.hashicorp.com/en/official-packaging-guide)
- **Terragrunt LS**: [gruntwork-io/terragrunt-ls](https://github.com/gruntwork-io/terragrunt-ls) · [hcl-lsp 확장](https://marketplace.visualstudio.com/items?itemName=BahramJoharshamshiri.hcl-lsp) · [Terragrunt LS 이슈 #2779](https://github.com/gruntwork-io/terragrunt/issues/2779)
- `brew install terraform terragrunt`
- Terragrunt + MetalLB 예시 검색: [code search](https://github.com/search?q=terragrunt+metallb&type=code) · [100rd/platform-design (eks-cilium-istio-karpenter)](https://github.com/100rd/platform-design/blob/b00c76e/docs/eks-cilium-istio-karpenter-terragrunt-manual.md?plain=1#L168) · [youhide/hideForming](https://github.com/youhide/hideForming/tree/d57d967/terragrunt) · [jinglemansweep/kube-homelab root.hcl](https://github.com/jinglemansweep/kube-homelab/blob/main/terragrunt/root.hcl)

## 관련 노트
- 네트워크/MTU: [[2026-01-26_K3s-MetalLB-Tailscale-MTU-Flannel-트러블슈팅]]
- 노드 부트스트랩 토큰·TLS SAN: [[2026-01-15_K3s-노드-토큰형식-TLS-SAN]]
- NFS PV: [[2026-01-28_K8s-NFS-마운트-Operation-not-permitted-서브넷-Tailscale]]
- Proxmox VM template hostname/machine-id 함정: [[02-Areas/homelab/2026-06-09_Proxmox-VM-복제-골든템플릿]]
