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


## Trivy 보안 설정 검사 및 개선 결과

GitHub Actions에서 Trivy를 이용해 Kubernetes Manifest 보안 설정을 검사했습니다.

최초 검사에서 3개 Deployment 파일에 총 9건의 Misconfiguration이 발견되어 CI가 실패했습니다. 이후 해당 Manifest의 보안 설정을 수정하고 동일한 검사 대상으로 재검사를 진행한 결과, 탐지 건수가 0건으로 줄어들고 CI가 정상적으로 완료되었습니다.

### 1. 검사 결과 비교

| 검사 대상 | 수정 전 | 수정 후 |
|---|---:|---:|
| `deployment.yaml` | 3건 | 0건 |
| `wordpress/mariadb-deployment.yaml` | 3건 | 0건 |
| `wordpress/wordpress-deployment.yaml` | 3건 | 0건 |
| **총 탐지 건수** | **9건** | **0건** |

- 검사 도구: Trivy (Configuration Scan)
- 검사 명령: `trivy config .`
- 검사 대상: Kubernetes 설정 파일 12개
- 검증 일자: 2026년 10월 17일

### 2. GitHub Actions 실행 증빙

**수정 전 — Trivy 검사 실패**

[Trivy Security Scan 실패 로그 확인](https://github.com/cojisang/argocd-test/actions/runs/39029021458/job/117385575201)

`Run Trivy` 단계에서 Misconfiguration 9건이 탐지되었으며, 해당 검사 단계가 실패했습니다.

**수정 후 — Trivy 검사 성공**

[Trivy Security Scan 성공 로그 확인](https://github.com/cojisang/argocd-test/actions/runs/39034712811/job/117402111548)

동일한 12개 설정 파일을 검사한 결과 Misconfiguration 0건이 확인되었고, 검사 작업이 정상적으로 완료되었습니다.
