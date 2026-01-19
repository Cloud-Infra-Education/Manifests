# Manifests

Kubernetes 매니페스트 저장소입니다. ArgoCD를 통해 자동으로 배포됩니다.

## 📋 개요

이 저장소는 `formation-lap` 네임스페이스에 배포되는 모든 Kubernetes 리소스의 매니페스트를 관리합니다.

## 🏗️ 디렉토리 구조

```
base/
├── namespace.yaml                    # formation-lap 네임스페이스
├── kustomization.yaml               # Kustomize 설정 파일
├── configmap/
│   └── db-config.yaml              # 데이터베이스 ConfigMap
├── secret/
│   └── db-secret.yaml              # 데이터베이스 Secret
├── ingress.yaml                     # Ingress 리소스 (ALB, HTTPS)
├── backend-api/                     # Backend API 서비스
│   ├── namespace.yaml
│   ├── serviceaccount.yaml         # IRSA 설정
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── user-service/                    # User Service
    ├── deployment.yaml
    └── service.yaml
```

## 🚀 배포된 서비스

### Backend API
- **Deployment**: `backend-api`
- **Service**: `backend-api-service`
- **이미지**: `087730891580.dkr.ecr.ap-northeast-2.amazonaws.com/backend-api:latest`
- **포트**: 8000
- **Replicas**: 2
- **외부 접근 경로**:
  - `https://api.exampleott.click/api/v1/*` - API 엔드포인트
  - `https://api.exampleott.click/docs` - API 문서
  - `https://api.exampleott.click/api/docs` - OpenAPI 문서
  - `https://api.exampleott.click/api/openapi.json` - OpenAPI JSON
- **기능**:
  - FastAPI 기반 REST API
  - Keycloak 통합 (JWT 인증)
  - Keycloak 사용자 자동 생성 기능
  - Meilisearch 연동
  - S3 연동 (IRSA 사용)

### User Service
- **Deployment**: `ott-users`
- **Service**: `user-service`
- **이미지**: `404457776061.dkr.ecr.ap-northeast-2.amazonaws.com/user-service:latest`
- **포트**: 8000
- **Replicas**: 1
- **외부 접근 경로**:
  - `https://api.exampleott.click/users/*` - 사용자 서비스 엔드포인트
- **기능**:
  - 사용자 관리 서비스
  - 데이터베이스 연동

### Keycloak
- **Deployment**: `keycloak`
- **Service**: `keycloak-service`
- **포트**: 8080
- **외부 접근 경로**:
  - `https://api.exampleott.click/keycloak/*` - Keycloak 관리 콘솔 및 API
- **기능**: 인증 및 인가 서버

### Meilisearch
- **Deployment**: `meilisearch`
- **Service**: `meilisearch-service`
- **포트**: 7700
- **기능**: 검색 엔진 (내부 네트워크 전용)

## 🌐 네트워크 및 Ingress

### Ingress 설정
- **Ingress Controller**: AWS Load Balancer Controller (ALB)
- **Load Balancer**: `matchacake-alb-test-seoul`
- **HTTPS**: 활성화 (포트 443)
- **SSL 인증서**: ACM 인증서 (api.exampleott.click)
- **HTTP → HTTPS 리다이렉트**: 활성화

### 외부 접근 경로

| 경로 | 서비스 | 설명 |
|------|--------|------|
| `/api/v1/*` | backend-api-service | Backend API 엔드포인트 |
| `/docs` | backend-api-service | FastAPI 문서 |
| `/api/docs` | backend-api-service | OpenAPI 문서 |
| `/api/openapi.json` | backend-api-service | OpenAPI JSON 스키마 |
| `/users/*` | user-service | 사용자 서비스 |
| `/keycloak/*` | keycloak-service | Keycloak 관리 콘솔 |

### 도메인
- **API 도메인**: `api.exampleott.click`
- **Global Accelerator**: 활성화됨

## 🔧 ArgoCD 연동

### Application 설정

- **Repository**: `https://github.com/Cloud-Infra-Education/Manifests.git`
- **Branch**: `feat/#1`
- **Path**: `base`
- **Namespace**: `formation-lap`
- **Auto Sync**: 활성화됨
- **Sync Policy**:
  - Automated sync enabled
  - Prune: true
  - Self-heal: true

### 동기화 방법

1. **자동 동기화**: Auto sync가 활성화되어 있어 Git에 푸시하면 자동으로 배포됩니다.
2. **수동 동기화**: ArgoCD 웹 UI에서 "SYNC" 버튼 클릭
3. **하드 리프레시**: ArgoCD가 변경사항을 감지하지 못할 경우:
   ```bash
   kubectl patch application manifest-management-test -n argocd \
     --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
   ```

## 📝 매니페스트 수정 방법

1. 로컬에서 매니페스트 파일 수정
2. 변경사항 커밋 및 푸시:
   ```bash
   git add .
   git commit -m "feat: 변경 내용"
   git push origin feat/#1
   ```
3. ArgoCD가 자동으로 감지하여 배포 (약 1-2분 소요)

## 🔐 환경 변수 및 시크릿

### ConfigMap
- `db-config`: 데이터베이스 연결 정보
- `backend-config`: Backend API 설정
  - Keycloak URL, Realm, Client ID
  - Meilisearch URL
  - S3 Bucket 정보
  - 데이터베이스 연결 정보

### Secret
- `db-secret`: 데이터베이스 비밀번호
- `backend-secrets`: Backend API 시크릿 정보
  - Keycloak Admin 계정 정보
  - 데이터베이스 연결 문자열
  - Meilisearch API Key

## 📊 리소스 제한

### Backend API
- CPU: 200m (request) / 1000m (limit)
- Memory: 512Mi (request) / 1Gi (limit)

### User Service
- CPU: 250m (request) / 500m (limit)
- Memory: 256Mi (request) / 512Mi (limit)

## 🔍 Health Checks

모든 서비스는 다음 Health Check를 사용합니다:
- **Readiness Probe**: `/health` 또는 `/api/v1/health` 엔드포인트
- **Liveness Probe**: `/health` 또는 `/api/v1/health` 엔드포인트
- **Initial Delay**: 10-30초
- **Period**: 10-30초

## 🛠️ 문제 해결

### Pod가 시작되지 않는 경우    
1. ArgoCD 웹 UI에서 Application 상태 확인
2. Pod 로그 확인: `kubectl logs -n formation-lap <pod-name>`
3. 이벤트 확인: `kubectl describe pod -n formation-lap <pod-name>`

### 이미지 Pull 실패
- ECR 이미지 경로 확인
- ECR 접근 권한 확인
- 이미지가 ECR에 존재하는지 확인
- 다른 AWS 계정의 ECR 이미지를 사용하는 경우 접근 권한 설정 필요

### 동기화 실패
- ArgoCD Application의 저장소 연결 상태 확인
- `base/kustomization.yaml` 파일의 리소스 경로 확인
- 저장소에 모든 파일이 푸시되었는지 확인
- ArgoCD repo-server Pod 재시작:
  ```bash
  kubectl delete pod -n argocd -l app.kubernetes.io/name=argocd-repo-server
  ```

### 외부 접속 불가
- Ingress 리소스 상태 확인: `kubectl get ingress -n formation-lap`
- ALB 리스너 확인 (HTTP 80, HTTPS 443)
- Route53 DNS 레코드 확인
- Security Group 규칙 확인

### ArgoCD Degraded 상태
- Degraded 상태인 리소스 확인
- Pod 상태 확인: `kubectl get pods -n formation-lap`
- ImagePullBackOff 또는 CrashLoopBackOff 확인
- 사용하지 않는 리소스 제거

## 📚 참고 사항

- 모든 매니페스트는 Kustomize를 사용하여 관리됩니다.
- `base/kustomization.yaml`에서 모든 리소스를 참조합니다.
- ArgoCD는 Git 저장소를 모니터링하여 변경사항을 자동으로 배포합니다.
- HTTPS는 ALB를 통해 제공되며, HTTP 요청은 자동으로 HTTPS로 리다이렉트됩니다.
- Backend API는 IRSA(IAM Roles for Service Accounts)를 사용하여 S3에 접근합니다.

