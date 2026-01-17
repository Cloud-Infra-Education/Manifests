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
├── ingress.yaml                     # Ingress 리소스
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
- **기능**:
  - FastAPI 기반 REST API
  - Keycloak 통합 (JWT 인증)
  - Keycloak 사용자 자동 생성 기능
  - Meilisearch 연동
  - S3 연동 (IRSA 사용)

### User Service
- **Deployment**: `ott-users`
- **Service**: `user-service`
- **이미지**: `087730891580.dkr.ecr.ap-northeast-2.amazonaws.com/y2om-user-service:v4`
- **포트**: 8000
- **Replicas**: 1
- **기능**:
  - 사용자 관리 서비스
  - 데이터베이스 연동

### Keycloak
- **Deployment**: `keycloak`
- **Service**: `keycloak-service`
- **포트**: 8080
- **기능**: 인증 및 인가 서버

### Meilisearch
- **Deployment**: `meilisearch`
- **Service**: `meilisearch-service`
- **포트**: 7700
- **기능**: 검색 엔진

## 🔧 ArgoCD 연동

### Application 설정

- **Repository**: `https://github.com/Cloud-Infra-Education/Manifests.git`
- **Branch**: `feat/#1`
- **Path**: `base`
- **Namespace**: `formation-lap`
- **Auto Sync**: 활성화됨

### 동기화 방법

1. **자동 동기화**: Auto sync가 활성화되어 있어 Git에 푸시하면 자동으로 배포됩니다.
2. **수동 동기화**: ArgoCD 웹 UI에서 "SYNC" 버튼 클릭

## 📝 매니페스트 수정 방법

1. 로컬에서 매니페스트 파일 수정
2. 변경사항 커밋 및 푸시:
   ```bash
   git add .
   git commit -m "feat: 변경 내용"
   git push origin feat/#1
   ```
3. ArgoCD가 자동으로 감지하여 배포

## 🔐 환경 변수 및 시크릿

### ConfigMap
- `db-config`: 데이터베이스 연결 정보
- `backend-config`: Backend API 설정

### Secret
- `db-secret`: 데이터베이스 비밀번호
- `backend-secrets`: Backend API 시크릿 정보

## 🌐 네트워크

- **Namespace**: `formation-lap`
- **Service Type**: ClusterIP
- **Ingress**: ALB를 통한 외부 접근

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

## 📚 참고 사항

- 모든 매니페스트는 Kustomize를 사용하여 관리됩니다.
- `base/kustomization.yaml`에서 모든 리소스를 참조합니다.
- ArgoCD는 Git 저장소를 모니터링하여 변경사항을 자동으로 배포합니다.

## 🛠️ 문제 해결

### Pod가 시작되지 않는 경우    
1. ArgoCD 웹 UI에서 Application 상태 확인
2. Pod 로그 확인: `kubectl logs -n formation-lap <pod-name>`
3. 이벤트 확인: `kubectl describe pod -n formation-lap <pod-name>`

### 이미지 Pull 실패
- ECR 이미지 경로 확인
- ECR 접근 권한 확인
- 이미지가 ECR에 존재하는지 확인

### 동기화 실패
- ArgoCD Application의 저장소 연결 상태 확인
- `base/kustomization.yaml` 파일의 리소스 경로 확인
- 저장소에 모든 파일이 푸시되었는지 확인

