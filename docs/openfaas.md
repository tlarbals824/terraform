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
- 위에서 basic auth로 보호되므로 브라우저가 계정/비밀번호를 요구합니다.

### 2. 기본 인증 정보 확인

OpenFaaS 설치 시 `basic-auth` Secret에 admin 계정/비밀번호가 생성됩니다.

```bash
# 비밀번호 조회
kubectl -n openfaas get secret basic-auth -o jsonpath='{.data.basic-auth-password}' | base64 --decode; echo

# (선택) 로그인 정보를 파일로 저장
kubectl -n openfaas get secret basic-auth -o jsonpath='{.data.basic-auth-password}' | base64 --decode > /tmp/faas-pass
```

- 기본 아이디: `admin`
- 기본 비밀번호: 위 명령어로 출력된 값

### 3. faas-cli 설정

```bash
# faas-cli 설치 (macOS)
brew install faas-cli

# Gateway 접속 설정
export OPENFAAS_URL=https://openfaas.simproject.kr
cat ~/.faas-pass  # 또는 위에서 뽑은 비밀번호 파일 경로
faas-cli login --username admin --password-stdin < /tmp/faas-pass
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

### basic_auth 비활성화 금지
이 저장소에서는 `basic_auth: true`, `generateBasicAuth: true` 로 설정합니다.
basic_auth를 끄면 Gateway/API가 무인증 노출되어 심각한 보안 위험이 있으므로
비활성화하지 마세요.

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