# OpenFaaS 구축 가이드

이 문서는 OKE 클러스터에 **OpenFaaS(Community Edition)** 를 배포하는 방법과
배포 후 사용법을 설명합니다.

## 개요

OpenFaaS는 쿠버네티스 위에서 FaaS(함수형 서버리스)를 제공합니다.
이 저장소에서는 ArgoCD를 통해 GitOps 방식으로 배포하며, Gateway UI 를
`openfaas.simproject.kr` 도메인으로 TLS(Let's Encrypt)와 함께 노출합니다.

## 구성 요소

| 컴포넌트 | 역할 | Namespace |
|---------|------|-----------|
| Gateway | 함수 배포/호출 API + 웹 UI | `openfaas` |
| faas-netes (operator) | 함수 CRD를 Deployment로 변환 | `openfaas` |
| Prometheus | 스케일링/메트릭 수집 | `openfaas` |
| AlertManager | 경고 처리 | `openfaas` |
| queue-worker (+ NATS Streaming) | 비동기(asynchronous) 호출 | `openfaas` |

함수는 별도 네임스페이스 `openfaas-fn` 에 배포됩니다.

## 배포 구조

```
ArgoCD (k8s/argocd-apps/infra.yaml)
 ├── infra-openfaas          → OpenFaaS helm chart (openfaas/openfaas 15.0.11)
 └── infra-openfaas-ingress  → k8s/infra/openfaas (Ingress)
```

변경 사항은 `main` 브랜치에 push 되면 ArgoCD가 자동 동기화합니다.

## 파일 구조

```
k8s/infra/openfaas/
├── kustomization.yaml
└── ingress.yaml        # Gateway UI Ingress + TLS (openfaas.simproject.kr)
terraform/dns.tf        # Cloudflare A 레코드 (openfaas.simproject.kr)
```

## 배포 후 접속

### 1. Gateway UI

- URL: `https://openfaas.simproject.kr`
- **Cloudflare Access** (Google 로그인)으로 보호됩니다.
- 인증 흐름: `Browser -> Cloudflare Access(Google login) -> OpenFaaS gateway`
- 허용 이메일: `terraform/access.tf` 의 `cloudflare_allowed_emails` (기본 `srfsrf0103@gmail.com`)

### 2. 로그인

OpenFaaS 자체 basic-auth는 비활성화되어 있습니다. Cloudflare Access가 로그인을
대신 처리합니다. 브라우저로 접속하면 Cloudflare 로그인 화면이 뜨고,
허용 이메일로 로그인하면 Gateway UI에 접속됩니다.

### 3. faas-cli 설정 (CLI)

> OpenFaaS basic-auth가 꺼져 있으므로, CLI는 Gateway를 직접 호출하는 경우가
> 아니라면 UX를 위해선 UI를 사용하세요. CLI에서 함수를 관리하려면
> Cloudflare Access를 경유하는 설정이 별도로 필요합니다.

```bash
# faas-cli 설치 (macOS)
brew install faas-cli

# Gateway 접속 설정
export OPENFAAS_URL=https://openfaas.simproject.kr
```

## 함수 테스트

```bash
# 함수 목록 확인
faas-cli list

# helloworld 함수 배포 (기본 템플릿)
faas-cli new hello
faas-cli deploy -f hello.yml

# 호출
echo -n "OpenFaaS!" | faas-cli invoke hello
```

## 주의사항 / 트러블슈팅

### 인증 방식 (basic-auth → Cloudflare Access)
이 저장소에서는 OpenFaaS `basic_auth`를 **비활성화**하고, 대신 **Cloudflare Access**가
인증을 담당합니다. (ArgoCD와 동일한 패턴)
- `basic_auth: false`, `generateBasicAuth: false`
- Cloudflare Access가 `openfaas.simproject.kr`을 보호하므로, 허용되지 않은
  이메일은 Gateway에 도달하기 전에 차단됩니다.
- 만약 OpenFaaS를 사내망/다른 인프라에 그대로 노출하는 경우에는 `basic_auth: true`로
  되돌려야 안전합니다.

### 인증서 발급 확인
```bash
kubectl get certificate -n openfaas
# READY: True 가 아니면 Let's Encrypt http-01 challenge 실패를 의미
```

### DNS 레코드
`openfaas.simproject.kr` A 레코드는 `terraform/dns.tf`의
`cloudflare_record.openfaas` 리소스로 관리됩니다.
Terraform 적용 후 DNS 전파까지 수 분이 걸릴 수 있습니다.

### 리소스 (OCI Always Free 고려)
Gateway/controller/queue-worker 등 각 컴포넌트의 리소스 request를
낮게 설정해 두었습니다. prometheus는 PVC를 사용하지 않으므로
파드 재시작 시 메트릭이 초기화됩니다(치명적이지 않음).