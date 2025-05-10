# BNS Deploy (Blog, News, Shop)

**BNSdeploy**는 GitHub Webhook을 이용해 정적 HTML 기반의 `nginx` 웹사이트(`main`, `blog`, `news`, `shop`)를 자동으로 Docker 이미지로 빌드하고, Kubernetes 클러스터에 배포하는 Helm Chart입니다.

개발자는 GitHub에 `main.html`, `blog.html`, `news.html`, `shop.html` 파일만 업로드하면 됩니다.  
자동으로 이미지가 생성되며 각각 다음 경로로 접근할 수 있습니다:

- `/` → main.html  
- `/blog` → blog.html  
- `/news` → news.html  
- `/shop` → shop.html

---

## 🔧 사전 준비 사항

다음 환경이 사전에 준비되어 있어야 합니다:

- Kubernetes 클러스터 구축
- HTML 파일을 저장할 GitHub 저장소
- NFS 서버 구축 및 PersistentVolume 접근 가능
- DNS 서버를 통한 도메인 및 외부 접근용 IP 설정

---

## 📁 Templates 구성

```
📁
├── calico.yaml # Calico 네트워크 플러그인 설치
├── deployment.yaml # 앱 배포용 Deployment 리소스
├── service.yaml # ClusterIP 기반 서비스 설정
├── pvc.yaml # Persistent Volume Claim 정의
├── nfs-csi.yaml # NFS 접근을 위한 CSI 드라이버 설치
├── keda.yaml # KEDA 자동 스케일링 구성
├── metallb.yaml # MetalLB 설정 및 외부 IP 풀 정의
├── ingress.yaml # Ingress Controller 및 Ingress Rule 설정
└── webhook-server.yaml # GitHub Webhook 트리거로 Docker 이미지 자동 빌드 및 재배포
```
---

## ⚙️ 사용 방법

### 1. `webhook-server` 이미지 생성

1. [BNS_Webhook_Server_Image 저장소](https://github.com/jongseo22/BNS_Webhook_Server_Image.git)를 클론합니다.
2. `app.py` 파일 내 다음 항목을 수정합니다:

```python
APPS = ["main", "blog", "news", "shop"]
DOCKER_USER = "jongseo22" # 사용할 DockerHub 사용자명으로 수정
IMAGE_TAG = "latest"

REPO_URL = "https://github.com/jongseo22/BNSdeploy.git" # 사용할 GitHub URL로 수정
REPO_PATH = "/tmp/repo"
```

3. 수정 후 다음 명령어로 이미지를 빌드하고 DockerHub에 푸시합니다:

```bash
docker build -t "DockerHub 사용자명"/webhook-server:latest .
docker push "DockerHub 사용자명"/webhook-server:latest
```

---

### 2. Helm 저장소 추가 및 차트 설치

```bash
helm repo add BNSdeploy https://jongseo22.github.io/BNSdeployChart/
helm repo update
helm install my-BNS BNSdeploy/BNSdeployChart --namespace BNS --create-namespace
```

---

### 3. `values.yaml` 설정 항목

```yaml
apps.image.repository     # DockerHub에 업로드된 이미지 경로
storageClass.server       # NFS 서버 IP 주소
metallb.ipRange           # MetalLB에서 사용할 외부 IP 범위
ingress.host              # 사용할 도메인 주소 (예: www.example.com)
webhook.image             # webhook 이미지를 업로드한 DockerHub 사용자명
```

> 📄 기타 설정은 사용자의 환경에 맞게 입력하면 됩니다.

---

### 4. GitHub Webhook 설정

GitHub 저장소에서 아래와 같이 Webhook을 등록하세요:

| 항목 | 값 |
|------|-----|
| **Payload URL** | `http://<외부에서 접근 가능한 IP 또는 도메인>:8080/webhook` |
| **Content type** | `application/json` |
| **Trigger event** | `Just the push event` |

> ⚠️ 주의 : `Payload URL`에는 단순 Node IP가 아닌, 외부에서 접근 가능한 주소를 입력해야 합니다.
> (아래 중 하나를 사용하세요.)
>
> - MetalLB로 할당된 공인/외부 IP
> - 도메인 기반 Ingress (예: `http://webhook.example.com/webhook`)