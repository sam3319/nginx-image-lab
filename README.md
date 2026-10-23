# NGINX Image Lab Helm Chart

사용자가 준비한 NGINX 웹 이미지를 Kubernetes에 배포하기 위한 학습용 Helm 차트입니다. 먼저 초기 이미지가 정상 배포되는지 확인하고, 이후 Argo CD와 Image Updater를 직접 연결하는 실습에 사용할 수 있습니다.

이 저장소에는 Helm 차트만 포함합니다. Dockerfile, Argo CD Application, ImageUpdater 설정, 인증 Secret은 포함하지 않습니다. 이미지 저장소 주소는 예시 값이며, 실제 사용자 계정이나 인증 정보는 저장하지 않습니다.

## 파일 구성

```text
.
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   └── tests/
│       └── test-connection.yaml
├── .helmignore
├── .gitignore
└── README.md
```

| 파일 | 역할 |
| --- | --- |
| `Chart.yaml` | 차트 메타데이터. 차트 버전은 `0.1.0`, 앱 버전은 `1.0.0` |
| `values.yaml` | 이미지 저장소와 태그, Pod 개수, Service 등 설정 |
| `templates/deployment.yaml` | 이미지 설정을 이용해 웹 서버 Pod 관리 |
| `templates/service.yaml` | Pod에 접근할 ClusterIP Service 생성 |
| `templates/_helpers.tpl` | 공통 이름, 라벨, ServiceAccount 이름 생성 |
| `templates/tests/test-connection.yaml` | Helm 직접 설치 후 Service 연결을 검사할 테스트 Pod |

## 사전 준비

- Helm 3 이상
- 배포 시 접근 가능한 Kubernetes 클러스터와 kubectl
- 컨테이너 내부 80번 포트에서 응답하는 웹 이미지
- 레지스트리에 Push된 초기 이미지 태그 `1.0.0`

이미지가 비공개라면 워크로드 네임스페이스에 이미지 Pull Secret을 별도로 준비하고 `imagePullSecrets`로 참조해야 합니다. 토큰이나 비밀번호를 Git에 커밋하지 마세요.

## 1. 이미지 값 설정

`values.yaml`에서 다음 두 값을 실제 이미지에 맞게 변경합니다.

| 값 | 의미 | 예시 |
| --- | --- | --- |
| `image.repository` | 태그를 제외한 이미지 저장소 주소 | `docker.io/your-dockerhub-username/nginx-image-lab` |
| `image.tag` | 초기 이미지 태그 | `"1.0.0"` |

기본 저장소 주소는 배포 전에 변경해야 하는 예시 값입니다. 개인 계정명을 공개 저장소에 남기고 싶지 않다면 로컬에서 `--set-string image.repository=...`를 사용하거나, 별도로 관리하는 Argo CD Application의 Helm 파라미터로 지정할 수 있습니다. 로컬에서만 사용할 values 파일은 `values.local.yaml`로 작성하면 Git에서 제외됩니다.

기본 리소스 설정:

- replicas: 1
- 컨테이너 포트: 80
- Service: ClusterIP, 포트 80
- ServiceAccount: 기본 `default` 계정 사용. 별도 계정은 생성하지 않음
- Ingress / HPA: 템플릿을 포함하지 않아 values 설정만 켜도 해당 리소스가 생성되지는 않음

현재 Deployment의 containerPort는 `service.port`를 참조하고 Service의 targetPort는 80입니다. 이 실습에서는 `service.port: 80`을 유지하세요.

## 2. 배포 전 차트 확인

저장소 루트에서 실행합니다.

```bash
helm lint .
helm template nginx-image-lab .
```

`helm template`은 템플릿에 값을 대입해 최종 YAML을 출력합니다. 기본 실행은 클러스터에 리소스를 생성하지 않습니다. 이미지가 실제로 Pull되는지나 서버의 API 지원 여부까지 검증하는 명령은 아닙니다.

이미지 태그의 연결 여부를 확인할 수 있습니다.

```bash
helm template nginx-image-lab . --set-string image.tag=1.0.1
```

확인할 항목:

1. Deployment 이미지가 지정한 저장소와 태그로 조합되는지
2. Deployment selector, Pod label, Service selector가 일치하는지
3. ServiceAccount 이름이 `default` 또는 미리 만든 계정인지
4. Service가 컨테이너 80번 포트에 연결되는지

## 3. Argo CD로 배포

이번 GitOps 실습에서는 `helm install`을 먼저 실행할 필요가 없습니다. Argo CD Application에서 다음을 지정합니다.

| 설정 | 값 |
| --- | --- |
| Repository URL | 이 저장소의 Git clone 주소 |
| Revision | `main` |
| Path | `.` — 저장소 루트에 Chart.yaml이 있음 |
| Destination | 실습용 클러스터와 네임스페이스 |
| Helm parameter | 필요 시 `image.repository`, `image.tag` 지정 |
| Sync policy | 자동 배포를 원하면 Auto-Sync 활성화 |

공개 Git 저장소는 일반적으로 읽기 인증 없이 연결할 수 있습니다. 이미지 레지스트리 인증과 Git 저장소 인증은 별개입니다.

Argo CD는 Helm 차트를 렌더링한 후 Kubernetes 리소스를 직접 관리합니다. 따라서 이 방식으로 배포한 앱은 일반적인 Helm 릴리스와 달리 `helm list`에 표시되지 않을 수 있습니다.

## 4. 초기 배포 확인

아래 `<namespace>`와 `<service-name>`은 실제 값으로 바꿉니다.

```bash
kubectl get deployments,pods,services -n <namespace>
kubectl port-forward -n <namespace> svc/<service-name> 8080:80
```

명령을 실행한 컴퓨터의 브라우저에서 `http://localhost:8080`에 접속합니다. VM에서 실행했다면 localhost는 VM을 의미하므로 개인 PC에서는 터널 또는 PC의 kubectl을 사용하세요.

## 5. Image Updater를 연결할 때

Image Updater 설정은 직접 작성하는 후속 실습입니다. 이 차트에서 이미지 값의 위치는 다음과 같습니다.

- 저장소 파라미터: `image.repository`
- 태그 파라미터: `image.tag`

Git write-back을 사용할 경우, 새 이미지 버전을 선택한 뒤 지정한 Git 파일에 변경을 커밋하고 Push하도록 설정할 수 있습니다. 실제 write-back 대상이 values.yaml인지 별도의 override 파일인지는 Image Updater 설정에 따라 달라집니다.

먼저 `1.0.0` 배포를 확인한 뒤 레지스트리에 `1.0.1`을 Push해 자동 변경을 관찰하세요. 동일 태그의 이미지 내용을 덮어쓰는 경우에는 semver와 다른 digest 전략을 검토해야 합니다.

## 인증 정보 취급

- 실제 이메일, 실명, 로컬 사용자 경로를 문서에 기록하지 않습니다.
- GitHub PAT, Docker Hub 토큰, AWS 키, kubeconfig를 저장소에 추가하지 않습니다.
- `.gitignore`는 대표적인 로컬 인증/설정 파일을 제외하지만, 커밋 전 변경 사항을 직접 확인해야 합니다.

## 참고

- [Helm 문서](https://helm.sh/docs/)
- [Argo CD Helm 배포](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/)
