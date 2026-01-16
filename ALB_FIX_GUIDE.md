# ALB 생성 문제 해결 가이드

## 문제 상황
- ALB가 생성되지 않음
- Kubernetes Ingress가 배포되어야 ALB가 자동 생성됨

## 원인
1. Ingress에 `/api` 경로가 없어서 backend-api-service로 라우팅되지 않음
2. ArgoCD에서 Ingress를 배포해야 ALB가 생성됨

## 해결 방법

### 1. Ingress 수정 완료 ✅
- `/root/Manifests/base/ingress.yaml`에 `/api` 경로 추가
- 서울/오레곤 리전별 ALB 이름 설정

### 2. ArgoCD에서 Ingress 배포

ArgoCD에서 Ingress Application을 배포해야 합니다:

```bash
# ArgoCD 접속 (포트 포워딩)
kubectl port-forward -n argocd svc/argocd-server 8080:443

# 브라우저에서 접속
# https://localhost:8080
# admin / (초기 비밀번호는 argocd-secret에서 확인)
```

**ArgoCD Application 생성:**

서울 리전:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-seoul
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <GitHub Repository URL>
    targetRevision: main
    path: Manifests/seoul
  destination:
    server: https://kubernetes.default.svc
    namespace: formation-lap
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

오레곤 리전:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-oregon
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <GitHub Repository URL>
    targetRevision: main
    path: Manifests/oregon
  destination:
    server: <Oregon EKS Cluster API Server>
    namespace: formation-lap
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3. ALB 생성 확인

Ingress가 배포되면 ALB가 자동으로 생성됩니다:

```bash
# 서울 리전 ALB 확인
aws elbv2 describe-load-balancers --region ap-northeast-2 \
  --query "LoadBalancers[?contains(LoadBalancerName, 'matchacake-alb-test-seoul')]"

# 오레곤 리전 ALB 확인
aws elbv2 describe-load-balancers --region us-west-2 \
  --query "LoadBalancers[?contains(LoadBalancerName, 'matchacake-alb-test-oregon')]"

# ALB 태그 확인 (AWS Load Balancer Controller가 추가)
aws elbv2 describe-tags \
  --resource-arns <ALB_ARN> \
  --region ap-northeast-2 \
  --query "TagDescriptions[0].Tags"
```

### 4. Ingress 상태 확인

```bash
# 서울 리전
kubectl get ingress -n formation-lap msa-ingress

# 오레곤 리전
kubectl get ingress -n formation-lap msa-ingress \
  --context <oregon-context>
```

Ingress의 `ADDRESS` 필드에 ALB DNS 이름이 표시되어야 합니다.

### 5. AWS Load Balancer Controller 확인

ALB가 생성되지 않으면 AWS Load Balancer Controller가 제대로 설치되어 있는지 확인:

```bash
# Controller Pod 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Controller 로그 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

## 수정된 파일

1. `/root/Manifests/base/ingress.yaml`
   - `/api` 경로 추가 (backend-api-service)

2. `/root/Manifests/seoul/ingress.yaml` (신규)
   - 서울 리전 ALB 이름: `matchacake-alb-test-seoul`

3. `/root/Manifests/oregon/ingress.yaml` (신규)
   - 오레곤 리전 ALB 이름: `matchacake-alb-test-oregon`

4. `/root/Manifests/seoul/kustomization.yaml`
   - ingress.yaml 오버레이 추가

5. `/root/Manifests/oregon/kustomization.yaml` (신규)
   - 오레곤 리전 kustomization 설정

## 다음 단계

1. ✅ Ingress YAML 수정 완료
2. ⏳ ArgoCD에서 Ingress Application 배포
3. ⏳ ALB 생성 확인
4. ⏳ Global Accelerator 적용 (08-domain-ga)
