# Video Processing Kubernetes 배포 문서

## 개요
Video Processing 기능을 위한 Kubernetes 배포 설정 및 관련 리소스에 대한 문서입니다.

## 구현 내용

### 1. Backend API 배포

#### 1.1 Deployment (`base/deployment/backend-api-deployment.yaml`)
- **이미지**: ECR에서 가져온 Backend API Docker 이미지
- **환경 변수**: ConfigMap 및 Secret에서 주입
- **리소스**: CPU/Memory 제한 설정
- **Health Check**: Liveness 및 Readiness Probe

#### 1.2 Service (`base/services/backend-api-service.yaml`)
- **타입**: ClusterIP
- **포트**: 8000
- **Selector**: `app: backend-api`

#### 1.3 Ingress (`base/ingress.yaml`)
- **도메인**: `api.matchacake.click`
- **TLS**: SSL 인증서 사용
- **경로**: `/api/v1/*` → Backend API Service

### 2. ConfigMap 및 Secret

#### 2.1 ConfigMap (`base/configmap.yaml`)
- **APP_NAME**: 애플리케이션 이름
- **APP_VERSION**: 버전 정보
- **DEBUG**: 디버그 모드
- **ENVIRONMENT**: 환경 설정
- **HOST**, **PORT**: 서버 설정
- **KEYCLOAK_URL**, **KEYCLOAK_REALM**, **KEYCLOAK_CLIENT_ID**: Keycloak 설정
- **JWT_ALGORITHM**: JWT 알고리즘
- **MEILISEARCH_URL**: Meilisearch URL
- **DB_PORT**, **DB_NAME**: 데이터베이스 설정
- **S3_BUCKET_NAME**, **S3_REGION**, **CLOUDFRONT_DOMAIN**: S3 및 CloudFront 설정

#### 2.2 Secret (`base/secret.yaml`)
- **KEYCLOAK_CLIENT_SECRET**: Keycloak 클라이언트 시크릿
- **KEYCLOAK_ADMIN_USERNAME**, **KEYCLOAK_ADMIN_PASSWORD**: Keycloak 관리자 정보
- **MEILISEARCH_API_KEY**: Meilisearch API 키
- **DATABASE_URL**: 데이터베이스 연결 문자열 (RDS Proxy 포함)

### 3. 네임스페이스

#### 3.1 Namespace (`base/namespace.yaml`)
- **이름**: `formation-lap`
- **용도**: Backend API 및 관련 서비스 배포

### 4. Kustomization

#### 4.1 Base Kustomization (`base/kustomization.yaml`)
- 리소스 목록:
  - namespace.yaml
  - configmap.yaml
  - secret.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml

### 5. 배포 프로세스

#### 5.1 Backend API 배포
```bash
# 1. Docker 이미지 빌드 및 ECR 푸시
cd /root/Backend
docker build -t backend-api:latest .
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin \
  <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com
docker tag backend-api:latest <ECR_REPO>:latest
docker push <ECR_REPO>:latest

# 2. Kubernetes 배포
kubectl apply -k /root/Manifests/base
```

#### 5.2 배포 확인
```bash
# Pod 상태 확인
kubectl get pods -n formation-lap -l app=backend-api

# 로그 확인
kubectl logs -n formation-lap -l app=backend-api --tail=50

# Service 확인
kubectl get svc -n formation-lap backend-api-service

# Ingress 확인
kubectl get ingress -n formation-lap
```

### 6. 접근 경로

#### 6.1 외부 접근
- **API Base URL**: `https://api.matchacake.click`
- **Swagger UI**: `https://api.matchacake.click/docs`
- **Health Check**: `https://api.matchacake.click/api/v1/health`

#### 6.2 클러스터 내부 접근
- **Service 이름**: `backend-api-service.formation-lap.svc.cluster.local:8000`
- **단축 이름**: `backend-api-service:8000` (같은 네임스페이스 내)

### 7. 사용 기술

- **Kubernetes**: 컨테이너 오케스트레이션
- **Docker**: 컨테이너 이미지
- **AWS ECR**: 컨테이너 레지스트리
- **AWS ALB**: Application Load Balancer (Ingress Controller)
- **Kustomize**: Kubernetes 리소스 관리

### 8. 주요 파일

- `base/namespace.yaml`: 네임스페이스 정의
- `base/configmap.yaml`: ConfigMap 정의
- `base/secret.yaml`: Secret 정의
- `base/deployment/backend-api-deployment.yaml`: Backend API Deployment
- `base/services/backend-api-service.yaml`: Backend API Service
- `base/ingress.yaml`: Ingress 설정
- `base/kustomization.yaml`: Kustomize 설정

### 9. 환경 변수 매핑

| ConfigMap/Secret | 환경 변수 | 설명 |
|-----------------|----------|------|
| `APP_NAME` | `APP_NAME` | 애플리케이션 이름 |
| `KEYCLOAK_URL` | `KEYCLOAK_URL` | Keycloak 서버 URL |
| `MEILISEARCH_URL` | `MEILISEARCH_URL` | Meilisearch 서버 URL |
| `DATABASE_URL` | `DATABASE_URL` | 데이터베이스 연결 문자열 |
| `S3_BUCKET_NAME` | `S3_BUCKET_NAME` | S3 버킷 이름 |
| `CLOUDFRONT_DOMAIN` | `CLOUDFRONT_DOMAIN` | CloudFront 도메인 |

### 10. 트러블슈팅

#### 10.1 이미지 Pull 실패
- **문제**: `ImagePullBackOff` 오류
- **해결**: ECR에 이미지가 푸시되었는지 확인, IAM 역할 권한 확인

#### 10.2 데이터베이스 연결 실패
- **문제**: `Access denied` 오류
- **해결**: RDS Proxy 엔드포인트 확인, Secrets Manager 비밀번호 확인

#### 10.3 Ingress 접근 불가
- **문제**: 외부에서 API 접근 불가
- **해결**: ALB Ingress Controller 확인, 도메인 DNS 설정 확인
