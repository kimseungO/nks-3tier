# 재구축 가이드 (Rebuild Guide) v3

NKS 클러스터 삭제 후 전체 재구축 절차.
콘솔 작업과 kubectl/helm 작업을 순서대로 정리했다.

> **레포 역할 분리**
> - **앱 레포 (suwon-daytour-3tier)**: 앱 코드 + 배포 매니페스트 (was.yaml, web.yaml, web-configmap.yaml) + Jenkinsfile
> - **인프라 레포 (nks-3tier, 이 레포)**: 인프라 공통 설정 (Ingress, StorageClass, RBAC, 모니터링, Jenkins values)
>
> 앱 배포 매니페스트는 앱 레포에 있고 CI/CD 파이프라인이 apply한다.
> 이 가이드의 앱 배포 부분은 파이프라인이 자동 처리하므로, 수동 재현 시에만 앱 레포의 k8s/를 참고한다.

---

## 전제 조건 (유지되는 것)

| 리소스 | 상태 | 비고 |
|--------|------|------|
| VPC, Subnet, Peering, 라우팅 | 유지 | 무과금 |
| IGW, NAT GW | 유지 | |
| Security Group | 유지 | bastion-sg, nks-sg, rds-sg, monitoring-sg |
| Bastion Host | 유지 | 파일/kubeconfig 보관처 |
| RDS for MySQL | 유지 | appdb 데이터 보존 |
| Monitoring Instance | 중지 | 재시작 시 Grafana 설정 보존 |
| NCR 이미지 | 유지 | suwon-web, suwon-was |
| Object Storage (lab-obj) | 유지 | 배너 + 리뷰 사진, public |

**삭제한 것: NKS 클러스터, 공인 LB, Floating IP**

---

## 네트워크 구성

```
lab_vpc (10.1.0.0/16)
├─ public_subnet  (10.1.0.0/24)   IGW
├─ private_subnet (10.1.1.0/24)   NAT GW, NKS 워커
└─ db_subnet      (10.1.2.0/24)   RDS

mgt_vpc (10.0.0.0/16)
├─ mgt_public_subnet  (10.0.0.0/24)   Bastion
└─ mgt_private_subnet (10.0.1.0/24)   Monitoring (10.0.1.84)

Peering 연결됨
```

---

## 1. NKS 클러스터 생성 (콘솔)

```
콘솔 → Container → NKS → 클러스터 생성
```

### 클러스터 설정

| 항목 | 값 |
|------|-----|
| 이름 | lab-nks |
| K8s 버전 | v1.34.3 |
| 키페어 | nks_key |
| VPC | lab_vpc |
| **서브넷** | **private_subnet (10.1.1.0/24)** |
| K8s 서비스 네트워크 | 10.254.0.0/16 |
| 파드 네트워크 | 10.100.0.0/16 |
| K8s API 엔드포인트 | Private |

> 서브넷을 private으로 지정 → LB가 사설로 생성됨(정상). 외부 노출은 섹션 4에서 수동 LB로 해결.

### 노드 그룹

| 항목 | 값 |
|------|-----|
| 인스턴스 타입 | m2.c2m4 |
| 노드 수 | 2 |
| 오토 스케일러 | 사용 안 함 |
| 보안 그룹 | nks-sg |

### ★ Add-ons 설치 (중요 — 빠뜨리기 쉬움)

클러스터 생성 후 Add-Ons 탭에서 설치한다.

| 애드온 | 필요 이유 |
|--------|-----------|
| **cinder-csi-plugin** | PVC(블록 스토리지). Jenkins에 필수 |
| **metrics-server** | HPA(Pod 오토스케일링)에 필요 |

> cinder-csi-plugin은 **드라이버만 설치하고 StorageClass는 자동 생성하지 않는다.**
> 섹션 3에서 StorageClass를 직접 만들어야 한다.

---

## 2. Bastion에서 클러스터 연결

```
콘솔 → NKS → lab-nks → 기본 정보 → 설정 파일 다운로드
```

```bash
scp -i {key.pem} kubeconfig.yaml ubuntu@{bastion_ip}:~/.kube/config
kubectl get nodes
kubectl get nodes -o wide   # 워커 노드 INTERNAL-IP 메모 (섹션 4 LB 멤버용)
```

> `~/.kube/config`는 파일이어야 한다. 디렉토리로 만들지 말 것.

---

## 3. StorageClass 생성 (★ fsType 필수)

```bash
kubectl apply -f k8s/storageclass.yaml
kubectl get storageclass   # cinder-default (default) 확인
```

> **storageclass.yaml에는 반드시 `csi.storage.k8s.io/fstype: ext4`가 있어야 한다.**
> 없으면 Cinder CSI의 `ReadWriteOnceWithFSType` 정책이 fsGroup을 적용하지 않아,
> non-root로 실행되는 워크로드(Jenkins 등)가 볼륨 쓰기 권한 오류로 기동 실패한다.

---

## 4. Ingress Controller (NodePort) + 수동 공인 LB

### 4-1. Ingress Controller 설치

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f helm-values/ingress-values.yaml
```

`EXTERNAL-IP: <none>`, NodePort 30080/30443이면 정상.

### 4-2. 공인 LB 생성 (콘솔)

```
콘솔 → Network → Load Balancer → 생성
```

| 항목 | 값 |
|------|-----|
| 이름 | web-public-lb |
| 서브넷 | **public_subnet** |
| 리스너 포트 | 80 |
| 멤버 그룹 포트 | **30080** |
| 헬스체크 URL | **/healthz** |
| 멤버 | 워커 노드 IP 2개, 포트 30080 |

> 헬스체크 URL을 `/`로 두면 404 → 멤버 INACTIVE. 반드시 `/healthz`.
> 워커 노드 IP는 재생성 시 바뀌므로 새로 확인.

### 4-3. Floating IP 연결

```
LB 선택 → 플로팅 IP 관리 → 연결
```

**연결된 공인 IP 메모** (섹션 8 blackbox 설정에 필요).

---

## 5. Namespace 및 Secret 생성

### Namespace

```bash
kubectl create namespace web
kubectl create namespace was
kubectl create namespace cicd
kubectl create namespace monitoring
```

### Secret (실제 값은 별도 관리)

```bash
# RDS 접속
kubectl create secret generic was-secret \
  --namespace was \
  --from-literal=DATABASE_URL="mysql://{USER}:{PW}@{RDS_ENDPOINT}:3306/appdb"

# S3 (Object Storage 리뷰 사진) — S3 API 전용 키
kubectl create secret generic s3-secret \
  --namespace was \
  --from-literal=S3_ACCESS_KEY={S3_ACCESS_KEY} \
  --from-literal=S3_SECRET_KEY={S3_SECRET_KEY}

# NCR 이미지 pull (web, was 각각)
kubectl create secret docker-registry ncr-secret \
  --namespace was \
  --docker-server={REGISTRY_URI}/lab-ncr \
  --docker-username={ACCESS_KEY} --docker-password={SECRET_KEY}

kubectl create secret docker-registry ncr-secret \
  --namespace web \
  --docker-server={REGISTRY_URI}/lab-ncr \
  --docker-username={ACCESS_KEY} --docker-password={SECRET_KEY}
```

> S3 키는 콘솔의 "API 비밀번호"(Swift용)와 **다른** S3 API 전용 키다.
> Object Storage → S3 API 자격 증명에서 별도 발급.
>
> S3 비민감 값(ENDPOINT, REGION, BUCKET, PUBLIC_BASE)은 Secret이 아니라
> 앱 레포의 was.yaml에 평문 env로 있다. 참고값:
> - S3_ENDPOINT: https://kr2-api-object-storage.nhncloudservice.com
> - S3_REGION: kr2
> - S3_BUCKET: lab-obj
> - S3_PUBLIC_BASE: https://kr2-api-object-storage.nhncloudservice.com/v1/AUTH_a995e426e16c45eea712c899280d95d2/lab-obj

---

## 6. CI/CD 구축 (Jenkins + Kaniko)

### 6-1. Jenkins 배포

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm install jenkins jenkins/jenkins \
  --namespace cicd -f helm-values/jenkins-values.yaml
```

`jenkins-0`이 2/2 Running이면 성공.
(StorageClass fsType이 없으면 여기서 권한 오류로 실패 — 섹션 3 확인)

### 6-2. 접근 (Bastion 경유)

```bash
# Bastion
kubectl port-forward -n cicd svc/jenkins 8080:8080 --address 0.0.0.0
# 로컬 PC
ssh -i {key} -L 8080:localhost:8080 ubuntu@{bastion_ip}
# 브라우저: http://localhost:8080 (admin/admin123)
```

### 6-3. Jenkins 자격증명 (웹 UI)

- `github-cred`: Username with password (GitHub 사용자명 + PAT)
- `ncr-cred`: Username with password (NCR Access/Secret Key)

### 6-4. Kaniko용 NCR Secret

```bash
kubectl create secret docker-registry kaniko-ncr-secret \
  --namespace cicd \
  --docker-server=f2da3ca6-kr2-registry.container.nhncloud.com \
  --docker-username={ACCESS_KEY} --docker-password={SECRET_KEY}
```

### 6-5. 배포 권한 RBAC

```bash
kubectl apply -f k8s/jenkins-rbac.yaml
```

> cicd:default ServiceAccount에 web/was의 deployments, configmaps, services 권한 부여.
> 파이프라인이 `kubectl apply`로 이 리소스들을 배포하므로 create/patch/update 권한 필요.

### 6-6. Pipeline job 생성 (웹 UI)

- New Item → Pipeline
- Definition: Pipeline script from SCM
- SCM: Git, Repository: 앱 레포 URL, Credentials: github-cred
- Branch: */main, Script Path: Jenkinsfile

### 6-7. 실행

- Build Now (수동 트리거)
- 파이프라인이 was/web 빌드 → NCR push → 매니페스트 태그 치환 후 apply
- webhook 자동 트리거는 향후 과제 (private 클러스터라 경로 구성 필요)

> **배포 방식:** 파이프라인은 앱 레포의 k8s/was.yaml, web.yaml의 이미지 태그
> `PLACEHOLDER`를 빌드 번호로 sed 치환한 뒤 `kubectl apply`한다.
> `kubectl set image`가 아니라 apply 방식이라, env·ConfigMap 변경도 함께 반영되고
> 매니페스트가 진실의 원천이 된다.

---

## 7. 동작 검증

```bash
# 내부: web → was
kubectl exec -n web $(kubectl get pod -n web -l app=web -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://localhost/api/health

# 외부
curl http://{공인_IP}/
curl http://{공인_IP}/api/health

# 앱 메트릭 (was)
kubectl exec -n was $(kubectl get pod -n was -l app=was -o jsonpath='{.items[0].metadata.name}') -- \
  wget -qO- http://localhost:4000/metrics | head

# 리뷰 API
kubectl exec -n was $(kubectl get pod -n was -l app=was -o jsonpath='{.items[0].metadata.name}') -- \
  wget -qO- http://localhost:4000/api/reviews
```

---

## 8. 모니터링 재연결

Monitoring Instance를 중지만 했다면 시작 후 컨테이너 자동 기동
(`restart: unless-stopped`). 삭제했다면 섹션 9 참고.

### 8-1. Helm 저장소

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 8-2. kube-state-metrics + Alloy

```bash
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace monitoring

helm install alloy grafana/alloy \
  --namespace monitoring \
  -f helm-values/alloy-values.yaml
```

> alloy-values.yaml은 노드/컨테이너/kube-state-metrics/was 앱 메트릭을 수집해
> 10.0.1.84:9090으로 remote_write(push)한다.

### 8-3. 확인

```bash
kubectl get pods -n monitoring   # alloy(DaemonSet), kube-state-metrics Running
```

Prometheus(localhost:9091)에서:
```
node_memory_MemTotal_bytes
kube_pod_status_phase
http_requests_total          # was 앱 메트릭
```

### 8-4. blackbox 대상 IP 갱신 (★ LB 재생성 시 필수)

LB 재생성 → 공인 IP 변경. Monitoring Instance의 prometheus.yml 수정:

```bash
cd ~/monitoring
vi prometheus.yml   # targets의 IP를 새 공인 IP로
docker compose restart prometheus
```

> 설정 변경 후 반드시 restart. `up -d`만으로는 마운트된 설정 파일이 반영 안 됨.

```
probe_success   → 두 대상 모두 1이면 정상
```

---

## 9. Monitoring Instance 구축 (삭제한 경우)

중지만 했다면 건너뛴다.

### 9-1. 인스턴스 생성 (콘솔)

| 항목 | 값 |
|------|-----|
| 이미지 | Ubuntu Server 22.04 |
| 타입 | m2.c1m2 |
| 서브넷 | mgt_private_subnet |
| Floating IP | 붙이지 않음 |
| 보안 그룹 | monitoring-sg |

### 9-2. Docker 설치

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
# 로그아웃 후 재접속
```

### 9-3. 모니터링 스택 실행

레포의 monitoring/ 파일 사용 (docker-compose.yml, prometheus.yml).

```bash
cd monitoring
docker compose up -d
```

> docker-compose.yml의 Prometheus에 `--web.enable-remote-write-receiver` 필수.
> 없으면 Alloy의 push(remote_write)를 Prometheus가 거부한다.

### 9-4. SSH 터널 접근

```powershell
# Windows PowerShell
ssh -i C:\path\key.pem -L 3000:{monitoring_ip}:3000 -L 9091:{monitoring_ip}:9090 ubuntu@{bastion_ip}
```
Grafana: localhost:3000 (admin/admin), Prometheus: localhost:9091

> Windows에서 로컬 9090이 예약돼 있으면 `bind: Permission denied`.
> 다른 로컬 포트(9091)로 매핑.

### 9-5. Grafana 설정

- Data source: Prometheus, URL `http://prometheus:9090` (localhost 아님, 컨테이너 이름)
- Dashboard import: ID 315 (Kubernetes cluster monitoring)

---

## 트러블슈팅 메모

| 증상 | 원인 | 해결 |
|------|------|------|
| LB EXTERNAL-IP가 사설 IP | 클러스터가 private_subnet 기준 생성 | 수동 공인 LB (섹션 4) |
| LB 멤버 INACTIVE | 헬스체크 URL `/` → 404 | `/healthz`로 변경 |
| Jenkins Pod Init 권한 오류 | StorageClass에 fsType 없음 | fstype: ext4 추가 (섹션 3) |
| PVC Terminating 멈춤 | 볼륨 참조 Pod 잔존 | 해당 Pod 강제 삭제 |
| Deploy "process never started" | bitnami/kubectl에 shell 없음 | alpine/k8s 이미지 사용 |
| kubectl apply Forbidden | RBAC에 해당 리소스 권한 없음 | jenkins-rbac에 리소스 추가 |
| apply 시 이미지 롤백 | 매니페스트에 옛 태그 고정 | PLACEHOLDER + sed 치환 방식 |
| ImagePullBackOff | ncr-secret 네임스페이스 누락 | web/was 각각 생성 |
| /api 504 | proxy_pass가 옛 서비스명 | was-svc로 수정 후 Pod 재시작 |
| S3 Region is missing | S3 env 미주입 | was.yaml env + s3-secret 확인 |
| NCR push Swift 오류 | Object Storage 백엔드 일시 장애 | 재시도 |
| Prometheus 설정 반영 안 됨 | 컨테이너 재시작 안 함 | docker compose restart prometheus |
| Alloy push해도 데이터 없음 | remote-write-receiver 미활성화 | 플래그 추가 |
| Grafana 대시보드 N/A | 대시보드 기대 메트릭 미수집 | Alloy에 node-exporter/ksm 추가 |

---

## 예상 소요 시간

| 단계 | 시간 |
|------|------|
| NKS 생성 + Add-ons | 15분 |
| StorageClass + Ingress + LB | 15분 |
| Secret + CI/CD 구축 | 20분 |
| 파이프라인 실행 (앱 배포) | 5분 |
| 모니터링 재연결 | 5분 |
| **합계** | **약 60분** |

Monitoring Instance 신규 구축 시 20분 추가.