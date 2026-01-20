# ECR 계정 ID 하드코딩 문제 해결 가이드

## 문제
Manifests의 kustomization.yaml에 AWS 계정 ID가 하드코딩되어 있습니다:
```yaml
- name: 570795932693.dkr.ecr.ap-northeast-2.amazonaws.com/user-service
```

## 해결 방법

### 현재 상태
- Kustomize는 변수를 직접 지원하지 않음
- 주석으로 개선 필요 사항 명시 완료

### 권장 해결 방법

#### 방법 1: Terraform에서 Kubernetes Manifest 생성 (권장)
Terraform에서 Kubernetes Deployment를 직접 생성하여 ECR URL을 동적으로 주입:

```hcl
data "aws_caller_identity" "current" {}

resource "kubernetes_deployment" "user_service" {
  metadata {
    name      = "user-service"
    namespace = "formation-lap"
  }
  spec {
    template {
      spec {
        container {
          image = "${data.aws_caller_identity.current.account_id}.dkr.ecr.ap-northeast-2.amazonaws.com/user-service:${var.image_tag}"
        }
      }
    }
  }
}
```

#### 방법 2: Helm 사용
Helm을 사용하여 변수 주입:

```yaml
# values.yaml
image:
  repository: {{ .Values.awsAccountId }}.dkr.ecr.ap-northeast-2.amazonaws.com/user-service
  tag: {{ .Values.imageTag }}
```

#### 방법 3: Kustomize replacements 사용 (Kustomize 4.0+)
```yaml
replacements:
  - source:
      kind: ConfigMap
      name: aws-config
      fieldPath: data.accountId
    targets:
      - select:
          kind: Deployment
        fieldPaths:
          - spec.template.spec.containers.[name=user-service].image
```

### 현재 임시 조치
- 주석으로 개선 필요 사항 명시
- Terraform ECR outputs에 user-service repository URL 추가

## 다음 단계
1. Terraform에서 Kubernetes Deployment 생성으로 전환 검토
2. 또는 Helm으로 마이그레이션 검토
