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
Trivy 검사에서 NGINX, MariaDB, WordPress Deployment에 각각 3건씩 총 9건의 보안 설정 문제(Misconfiguration)가 발견되었습니다.

각 Deployment에 seccomp 프로파일을 적용하고, 권한 상승을 제한했으며, 컨테이너의 루트 파일시스템을 읽기 전용으로 설정했습니다.

읽기 전용 파일시스템 적용으로 서비스 실행에 필요한 경로에 쓰기 문제가 발생하지 않도록 별도의 `emptyDir` 볼륨을 구성했습니다.

#### 1. 서비스별 보안 설정 변경

| 서비스 | 주요 변경 사항 | 코드 변경 내역 |
|---|---|---|
| NGINX | seccomp 적용, 권한 상승 차단, 루트 파일시스템 읽기 전용, 임시 볼륨 추가 | [커밋 f7dc2f2](https://github.com/cojisang/argocd-test/commit/f7dc2f2) |
| MariaDB | seccomp 적용, 권한 상승 차단, 루트 파일시스템 읽기 전용, 실행·임시 디렉터리 분리, Recreate 전략 | [커밋 ec7d403](https://github.com/cojisang/argocd-test/commit/ec7d403) |
| WordPress | seccomp 적용, 권한 상승 차단, 루트 파일시스템 읽기 전용, Apache 실행 경로 및 권한 설정, Recreate 전략 | [커밋 415ad68](https://github.com/cojisang/argocd-test/commit/415ad68) |

#### 2. 주요 보안 설정

**seccompProfile: RuntimeDefault**
- 컨테이너에서 사용할 수 있는 시스템 콜을 제한하도록 기본 seccomp 프로파일 적용

**allowPrivilegeEscalation: false**
- 컨테이너 프로세스의 추가 권한 상승 제한

**readOnlyRootFilesystem: true**
- 컨테이너 루트 파일시스템을 읽기 전용으로 설정
- 서비스 실행에 필요한 쓰기 경로는 별도의 볼륨으로 분리

**emptyDir 볼륨**
- NGINX: `/var/cache/nginx`, `/var/run`, `/tmp`
- MariaDB: `/run/mysqld`, `/tmp`
- WordPress: Apache 실행 디렉터리 및 `/tmp`

MariaDB의 `/var/lib/mysql` 데이터 저장 경로는 기존 PVC 연결을 유지했습니다.

※ MariaDB와 WordPress에 추가한 `Recreate`는 보안 설정이 아닌 Deployment 배포 전략입니다.

#### 3. Trivy 검사 결과

| 검사 대상 | 수정 전 | 수정 후 |
|---|---:|---:|
| `deployment.yaml` | 3건 | 0건 |
| `wordpress/mariadb-deployment.yaml` | 3건 | 0건 |
| `wordpress/wordpress-deployment.yaml` | 3건 | 0건 |
| **합계** | **9건** | **0건** |

- 검사 도구: Trivy Configuration Scan
- 검사 명령: `trivy config .`
- 검사 대상: Kubernetes 설정 파일 12개
- 검증 일자: 2026년 10월 17일

#### 4. GitHub Actions 실행 증빙

**수정 전 — Trivy 검사 실패 (9건)**

[Trivy 검사 실패 로그 확인](https://github.com/cojisang/argocd-test/actions/runs/39029021458/job/117385575201)

![Trivy 검사 실패](docs/images/trivy-before.png)

**수정 후 — Trivy 검사 성공 (0건)**

[Trivy 검사 성공 로그 확인](https://github.com/cojisang/argocd-test/actions/runs/39034712811/job/117402111548)

![Trivy 검사 성공](docs/images/trivy-after.png)

## ImagePullBackOff 장애 재현 및 GitOps 복구

### 1. 테스트 목적

GitHub Actions의 Kubernetes Manifest 검증과
Trivy 보안 설정 검사를 통과하더라도 실제
Kubernetes 배포 과정에서 오류가 발생할 수
있는지 확인하기 위해 장애를 재현했습니다.

### 2. 이미지 태그 변경 — PR #5

NGINX Deployment의 이미지 태그를
존재하지 않는 테스트용 태그로 변경했습니다.

**변경 전**
`nginx:latest`

**변경 후**
`nginx:devsecops-rollback-test-invalid-20261017`

**코드 변경 증빙**
- [PR #5 - 장애 재현](https://github.com/cojisang/argocd-test/pull/5/files)
- [장애 재현 커밋](https://github.com/cojisang/argocd-test/commit/151bca4)

### 3. ImagePullBackOff 발생 및 원인 확인

CI 검사 이후 Kubernetes에서 잘못된 이미지
태그를 가져오지 못하는 문제가 발생했습니다.

Pod 이벤트에서 다음 오류를 확인했습니다.

- `Failed to pull image`
- `not found`
- `ErrImagePull`
- `ImagePullBackOff`

Deployment에 설정된 이미지 태그가
레지스트리에 존재하지 않는 것이 원인이었습니다.

**실제 장애 발생 화면**

![ImagePullBackOff 발생](docs/images/imagepullbackoff.png)

### 4. Git Revert를 통한 설정 복구 — PR #6

Git Revert를 이용해 잘못된 이미지 태그를
기존 태그로 되돌렸습니다.

**변경 전**
`nginx:devsecops-rollback-test-invalid-20261017`

**복구 후**
`nginx:latest`

**코드 변경 증빙**
- [PR #6 - Git Revert](https://github.com/cojisang/argocd-test/pull/6/files)
- [Git Revert 커밋](https://github.com/cojisang/argocd-test/commit/60d6387)

### 5. Pod 상태 확인

NGINX Pod 상태를 확인한 화면에서는
Pod 2개가 `1/1 Running`, `RESTARTS 0`으로
표시되었습니다.

**NGINX Pod Running 확인 화면**

![NGINX Pod Running](docs/images/nginx-running.png)

### 6. 검증 결과

| 검증 항목 | 결과 |
|---|---|
| 잘못된 이미지 태그 적용 | PR #5에서 확인 |
| 이미지 다운로드 실패 | ErrImagePull 확인 |
| ImagePullBackOff | Pod 이벤트에서 확인 |
| Git Revert | PR #6에서 확인 |
| 이미지 태그 원복 | nginx:latest 확인 |
| 정상 상태 Pod | Running 2개 확인 |

## Secret 관리

MariaDB와 WordPress 연결에 필요한 인증정보는
Kubernetes Secret을 통해 관리하도록 구성했습니다.

초기 실습에서는 Secret Manifest에 인증정보를
직접 작성했으나, 공개 저장소에 인증정보가
포함되지 않도록 해당 파일을 제거했습니다.

현재 저장소에는 실제 인증정보 대신
예제 파일만 포함하고 있습니다.

- [MariaDB Secret 예제](docs/examples/mariadb-secret.example.yaml)

실습 환경을 다시 구성할 경우 실제 Secret은
Git 저장소 외부에서 별도로 생성해야 합니다.

이 저장소의 예제 파일은 구성 참고용이며,
실제 인증정보를 포함하지 않습니다.
