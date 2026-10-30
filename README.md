# Kubernetes GitOps & DevSecOps CI

Kubernetes Manifest를 Git으로 관리하고, Pull Request 단계에서 설정 오류와 보안 문제를 검사한 뒤 Argo CD로 변경 사항을 배포하는 실습 프로젝트입니다.

## 1. 프로젝트 목적

- 배포 전에 Kubernetes Manifest의 구조와 보안 설정을 검사
- 검증을 통과한 변경 사항을 `main` 브랜치에 반영하고 GitOps 방식으로 배포
- 배포 중 오류가 발생했을 때 원인을 확인하고 Git Revert로 이전 상태 복구

## 2. 사용 기술

| 기술 | 역할 |
| --- | --- |
| Kubernetes (kubeadm) | NGINX, WordPress 서비스 실행 환경 |
| GitHub Actions | Pull Request 시 CI 실행 |
| Kubeconform | Kubernetes Manifest 스키마 검증 |
| Trivy | Manifest 보안 설정 검사 (HIGH, CRITICAL) |
| Argo CD | Git에 기록된 변경 사항을 Kubernetes에 동기화 |
| NGINX | WordPress 앞단 Reverse Proxy |

## 3. CI/CD 흐름

```text
개발 브랜치에서 Manifest 수정
        ↓
Pull Request → main
        ↓
GitHub Actions
  ├─ Kubeconform: Manifest 스키마 검증
  └─ Trivy: HIGH/CRITICAL 설정 검사
        ↓
검증 결과 확인 후 main에 병합
        ↓
Argo CD에서 변경 사항 감지 및 동기화
        ↓
Kubernetes에 적용
```

- 워크플로 파일: [kubernetes-ci.yml](.github/workflows/kubernetes-ci.yml)
- CI는 `main`을 대상으로 하는 Pull Request 또는 수동 실행(`workflow_dispatch`) 시 동작합니다.
- Trivy에서 지정한 심각도의 문제가 탐지되면 CI 검사가 실패합니다.
- **CI 실패가 병합을 자동 차단하려면 GitHub 브랜치 보호 및 필수 상태 검사 설정이 별도로 필요합니다.**
- Argo CD의 동기화 방식(자동/수동)은 클러스터의 Argo CD Application 설정에 따릅니다.

## 4. 저장소 구조

```text
argocd-test/
├── .github/workflows/
│   └── kubernetes-ci.yml   # Manifest / 보안 검사
├── wordpress/              # WordPress 관련 Manifest
├── configmap.yaml          # NGINX Reverse Proxy 설정
├── deployment.yaml         # NGINX Deployment
├── namespace.yaml          # 네임스페이스 정의
└── service.yaml            # NGINX NodePort Service
```

`service.yaml`에는 NGINX NodePort `30080`이 설정되어 있으며, `configmap.yaml`에는 WordPress Service를 향한 Reverse Proxy 설정이 있습니다.

## 5. 검증 및 장애 복구 사례

### Trivy 보안 설정 검사

Kubernetes Manifest를 검사했을 때 보안 설정 관련 문제 9건을 확인했습니다. 설정을 수정한 뒤 같은 검사를 다시 실행해 0건으로 줄어든 것을 확인했습니다.

### 이미지 태그 오류로 인한 배포 실패

1. CI 검사는 통과했지만 존재하지 않는 컨테이너 이미지 태그로 인해 Pod가 `ErrImagePull` / `ImagePullBackOff` 상태가 되었습니다.
2. Pod 이벤트와 Deployment의 이미지 태그를 확인해 원인을 찾았습니다.
3. GitHub에서 변경 사항을 Revert하고 Argo CD로 다시 동기화했습니다.
4. 정상 이미지가 적용되고 Rollout이 완료된 것을 확인했습니다.

## 6. 현재 구성의 한계와 개선할 점

- 현재 CI는 Manifest의 스키마 및 보안 설정을 검사하지만 **컨테이너 이미지 태그가 레지스트리에 실제 존재하는지까지는 검증하지 않습니다.**
- 검증한 컨테이너 이미지의 태그 또는 digest를 고정해 배포의 재현성을 높일 필요가 있습니다.
- 실제 배포 후 상태를 자동 확인하는 검사와 장애 발생 시 대응 절차를 추가로 개선할 수 있습니다.

이 저장소는 **Kubernetes Manifest, CI 검사, GitOps 배포 흐름**에 대한 자료입니다. Ansible, Prometheus, Loki, Longhorn, Velero, MinIO 등 다른 인프라 구성 전체의 코드를 포함한 저장소는 아닙니다.
