# 재구축 가이드 (Rebuild Guide)

NKS 클러스터 삭제 후 재구축하는 전체 절차.
콘솔 작업과 kubectl/helm 작업을 순서대로 정리했다.

---

## 전제 조건 (삭제하지 않고 유지되는 것)

| 리소스 | 상태 | 비고 |
|--------|------|------|
| VPC, Subnet, Peering, 라우팅 | 유지 | 무과금 |
| Internet Gateway, NAT Gateway | 유지 | |
| Security Group | 유지 | bastion-sg, nks-sg, rds-sg, monitoring-sg |
| Bastion Host | 유지 | 설정 파일, kubeconfig 보관처 |
| RDS for MySQL | 유지 | appdb 데이터 보존 |
| Monitoring Instance | **중지** | 재시작하면 Grafana 설정 보존됨 |
| NCR 컨테이너 이미지 | 유지 | suwon-web:1.0, suwon-was:1.0 |

**삭제한 것: NKS 클러스터, 공인 LB, Floating IP**

> Monitoring Instance를 **삭제**한 경우에는 섹션 8(Monitoring Instance 구축)을 참고한다.
> **중지**만 한 경우에는 인스턴스를 시작하고 섹션 7(모니터링 재연결)로 바로 간다.

---

## 네트워크 구성 (참고)

```
lab_vpc (10.1.0.0/16)          ← 서비스 VPC
├─ public_subnet  (10.1.0.0/24)   IGW 연결
├─ private_subnet (10.1.1.0/24)   NAT GW 연결, NKS 워커 노드
└─ db_subnet      (10.1.2.0/24)   RDS

mgt_vpc (10.0.0.0/16)          ← 관리 VPC
├─ mgt_public_subnet  (10.0.0.0/24)   Bastion
└─ mgt_private_subnet (10.0.1.0/24)   Monitoring Instance (10.0.1.84)

두 VPC는 Peering으로 연결됨 (양쪽 라우팅 테이블에 상대 대역 등록)
```

---

## 1. NKS 클러스터 생성 (콘솔)

```
콘솔 → Container → NKS → 클러스터 생성
```

### 클러스터 설정

| 항목 | 값 |
|------|-----|
| 클러스터 이름 | lab-nks |
| K8s 버전 | v1.34.3 |
| 키페어 | nks_key |
| VPC | lab_vpc (10.1.0.0/16) |
| **서브넷** | **private_subnet (10.1.1.0/24)** |
| K8s 서비스 네트워크 | 10.254.0.0/16 |
| 파드 네트워크 | 10.100.0.0/16 |
| 파드 서브넷 크기 | 24 |
| K8s API 엔드포인트 | **Private** |

> **중요:** 서브넷을 private_subnet으로 지정하면 `type: LoadBalancer` 서비스가
> **사설 IP만** 받는다. 이건 정상 동작이다. 외부 노출은 섹션 4(수동 공인 LB)로 해결한다.
> public_subnet으로 만들면 자동 공인 LB가 되지만, 워커 노드가 Public에 노출되므로
> 보안상 private을 유지하고 수동 LB를 쓴다.

### 노드 그룹 설정

| 항목 | 값 |
|------|-----|
| 인스턴스 타입 | m2.c2m4 (2 vCPU / 4GB) |
| 노드 수 | 2 |
| 플로팅 IP 자동 할당 | 사용 안 함 |
| 오토 스케일러 | 사용 안 함 |
| 보안 그룹 | nks-sg |

생성까지 약 10분 소요.

---

## 2. Bastion에서 클러스터 연결

### kubeconfig 다운로드 및 배치

```
콘솔 → NKS → lab-nks → 기본 정보 → 설정 파일 다운로드
```

로컬 PC에서 Bastion으로 전송한다.

```bash
scp -i {key.pem} kubeconfig.yaml ubuntu@{bastion_floating_ip}:~/.kube/config
```

> `~/.kube/config`는 **파일**이어야 한다. `mkdir`로 디렉토리를 만들지 말 것.

### 연결 확인

```bash
kubectl get nodes
kubectl get nodes -o wide
```

**워커 노드의 INTERNAL-IP를 메모해둔다.** 섹션 4에서 LB 멤버로 등록해야 한다.

---

## 3. Ingress Controller 설치 (NodePort)

자동 LB를 쓰지 않고 NodePort로만 노출한다. 그 앞에 수동 공인 LB를 붙일 것이다.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  -f ~/nks-3tier/helm-values/ingress-values.yaml
```

### 확인

```bash
kubectl get svc -n ingress-nginx
```

```
NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
ingress-nginx-controller   NodePort   10.254.x.x     <none>        80:30080/TCP,443:30443/TCP
```

> `EXTERNAL-IP: <none>`이 정상이다. NodePort 30080 고정은 LB 멤버 포트와 맞추기 위함이다.

---

## 4. 공인 LoadBalancer 생성 (콘솔) — 핵심

```
콘솔 → Network → Load Balancer → 로드 밸런서 생성
```

### 기본 설정

| 항목 | 값 |
|------|-----|
| 이름 | web-public-lb |
| VPC | lab_vpc |
| **서브넷** | **public_subnet (10.1.0.0/24)** |
| IP 할당 | 자동 할당 |

### 리스너 (외부에서 받는 포트)

| 항목 | 값 |
|------|-----|
| 프로토콜 | HTTP |
| 로드 밸런서 포트 | **80** |

### 멤버 그룹 (워커 노드로 보내는 포트)

| 항목 | 값 |
|------|-----|
| 프로토콜 | HTTP |
| **포트** | **30080** |
| 로드 밸런싱 방식 | ROUND_ROBIN |

> **포트를 30080으로 지정해야 한다.** 기본값 80으로 두면 워커 노드의 80 포트로 가는데
> 거기엔 아무것도 없어서 실패한다.

### 상태 확인(헬스 체크) — 반드시 수정

| 항목 | 값 |
|------|-----|
| 프로토콜 | HTTP |
| **URL** | **/healthz** |
| HTTP 상태 코드 | 200 |

> **기본값 `/`로 두면 멤버가 INACTIVE가 된다.**
> Ingress Controller는 `/`로 요청하면 라우팅 규칙이 없어 404를 반환하기 때문이다.

### 멤버 목록

`kubectl get nodes -o wide`의 INTERNAL-IP를 등록한다. 각 멤버 포트는 **30080**.

> 워커 노드 IP는 클러스터를 재생성하면 **바뀐다.** 반드시 새로 확인할 것.

### Floating IP 연결

```
LB 선택 → "플로팅 IP 관리" → Floating IP 연결
```

**연결된 공인 IP를 메모해둔다.** 섹션 7의 blackbox 설정에 필요하다.

---

## 5. 애플리케이션 배포

```bash
kubectl create namespace web
kubectl create namespace was
```

### Secret 생성 (실제 값은 별도 관리)

```bash
kubectl create secret generic was-secret \
  --namespace was \
  --from-literal=DATABASE_URL="mysql://{DB_USER}:{DB_PASSWORD}@{RDS_ENDPOINT}:13306/appdb"

kubectl create secret docker-registry ncr-secret \
  --namespace was \
  --docker-server={REGISTRY_URI}/lab-ncr \
  --docker-username={ACCESS_KEY} \
  --docker-password={SECRET_KEY}

kubectl create secret docker-registry ncr-secret \
  --namespace web \
  --docker-server={REGISTRY_URI}/lab-ncr \
  --docker-username={ACCESS_KEY} \
  --docker-password={SECRET_KEY}
```

> **주의 1:** Secret은 네임스페이스를 넘나들지 못한다. web과 was 양쪽에 각각 만들어야 한다.
> **주의 2:** `--docker-server`에 레지스트리 이름(`/lab-ncr`)까지 포함해야 한다.

### 매니페스트 적용

```bash
cd ~/nks-3tier

kubectl apply -f k8s/web-nginx-conf.yaml
kubectl apply -f k8s/web.yaml
kubectl apply -f k8s/was.yaml
kubectl apply -f k8s/ingress.yaml
```

---

## 6. 동작 검증

### 클러스터 내부 (web → was)

```bash
kubectl exec -n web $(kubectl get pod -n web -l app=web -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://localhost/api/health
```

`{"status":"ok","tier":"was"}` 가 나오면 정상.

### 외부

```bash
curl http://{공인_IP}/
curl http://{공인_IP}/api/health
```

---

## 7. 모니터링 재연결 (Monitoring Instance가 살아 있는 경우)

Monitoring Instance를 중지만 했다면, 시작 후 컨테이너가 자동으로 뜬다
(`restart: unless-stopped` 설정).

```bash
# Monitoring Instance에서 확인
docker compose ps
```

클러스터는 새로 생겼으므로 **Alloy와 kube-state-metrics를 다시 배포**해야 한다.

### Helm 저장소 추가 (Bastion)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### kube-state-metrics 배포

```bash
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace monitoring --create-namespace
```

### Alloy 배포

`helm-values/alloy-values.yaml`의 remote_write 엔드포인트가
Monitoring Instance IP(`10.0.1.84:9090`)인지 확인한다.

```bash
helm install alloy grafana/alloy \
  --namespace monitoring \
  -f ~/nks-3tier/helm-values/alloy-values.yaml
```

### 확인

```bash
kubectl get pods -n monitoring
```

alloy Pod가 워커 노드 수만큼(DaemonSet), kube-state-metrics 1개가 Running이어야 한다.

Prometheus에서:

```
node_memory_MemTotal_bytes
kube_pod_status_phase
```

### blackbox 대상 IP 갱신 (필수)

LB를 재생성하면 **공인 IP가 바뀐다.** Monitoring Instance의 `prometheus.yml`에서
blackbox 대상 URL을 새 IP로 수정한다.

```bash
# Monitoring Instance에서
cd ~/monitoring
vi prometheus.yml   # targets의 IP를 새 공인 IP로 변경
docker compose restart prometheus
```

> **설정 파일을 바꾸면 반드시 컨테이너를 재시작해야 한다.**
> `docker compose up -d`만으로는 마운트된 설정 파일 변경이 반영되지 않는다.

확인:

```
probe_success    → 두 대상 모두 1이면 정상
```

---

## 8. Monitoring Instance 구축 (인스턴스를 삭제한 경우)

인스턴스를 중지만 했다면 이 섹션은 건너뛴다.

### 8-1. 인스턴스 생성 (콘솔)

```
콘솔 → Compute → Instance → 인스턴스 생성
```

| 항목 | 값 |
|------|-----|
| 이름 | monitoring |
| 이미지 | Ubuntu Server 22.04 LTS |
| 인스턴스 타입 | m2.c1m2 (1 vCPU / 2GB) |
| VPC | mgt_vpc |
| **서브넷** | **mgt_private_subnet (10.0.1.0/24)** |
| 키페어 | Bastion과 동일한 키 사용 |
| 보안 그룹 | monitoring-sg |
| Floating IP | **붙이지 않음** (Bastion 경유 접근) |

> private subnet에 두는 이유: 모니터링 서버는 외부 노출이 불필요하고,
> 운영 지표를 담고 있어 노출 시 위험하다. Bastion을 통해서만 접근한다.

### 8-2. Bastion 경유 접속

```bash
# Bastion에서
ssh -i {key.pem} ubuntu@{monitoring_private_ip}
```

### 8-3. Docker 설치

private subnet이므로 NAT Gateway를 통해 인터넷에 나간다. 먼저 아웃바운드를 확인한다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://download.docker.com
```

정상 응답이 오면 설치를 진행한다.

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
```

그룹 적용을 위해 로그아웃 후 재접속한다.

```bash
docker --version
docker compose version
docker ps          # sudo 없이 되는지 확인
```

### 8-4. Prometheus + Grafana + blackbox 실행

레포에서 설정 파일을 가져온다.

```bash
git clone https://github.com/{사용자명}/{레포명}.git
cd {레포명}/monitoring
```

또는 파일을 직접 복사한다. 필요한 파일은 두 개다.

**docker-compose.yml** — Prometheus, Grafana, blackbox-exporter를 컨테이너로 정의

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-remote-write-receiver'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

  blackbox:
    image: prom/blackbox-exporter:latest
    container_name: blackbox
    ports:
      - "9115:9115"
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

> **`--web.enable-remote-write-receiver` 플래그가 핵심이다.**
> Prometheus는 기본적으로 "긁어오는(pull)" 서버라, 외부에서 밀어 넣는(push) 걸 받으려면
> 이 플래그로 수신 기능을 켜야 한다. 이게 없으면 Alloy가 remote_write로 아무리 보내도
> Prometheus가 거부한다.
>
> `command`를 직접 지정하면 기본값이 사라지므로,
> `--config.file`과 `--storage.tsdb.path`도 함께 명시해야 한다.

> `volumes`(prometheus-data, grafana-data)는 **데이터 영속성**을 위한 것이다.
> 컨테이너가 재시작돼도 수집한 메트릭과 Grafana 대시보드 설정이 보존된다.

**prometheus.yml** — 수집 설정 (blackbox 대상 포함)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 블랙박스: 외부에서 서비스 접속 확인
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://{공인_LB_IP}/
          - http://{공인_LB_IP}/api/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

> blackbox는 Prometheus가 직접 요청하는 게 아니라,
> **blackbox-exporter에게 "이 URL 확인해줘"라고 시키는** 구조다.
> `relabel_configs`가 그 파라미터 전달을 담당한다.

실행:

```bash
docker compose up -d
docker compose ps
```

Prometheus, Grafana, blackbox 세 컨테이너가 running이어야 한다.

**remote_write 수신 확인:**

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:9090/api/v1/write
```

`400`이 나오면 수신 기능이 켜진 것이다(빈 요청이라 400, 엔드포인트는 존재).
`404`가 나오면 플래그가 적용되지 않은 것이다.

### 8-5. SSH 터널링으로 접근

Monitoring Instance는 private subnet이라 브라우저 직접 접근이 불가능하다.
Bastion을 경유하는 SSH 터널을 만든다.

**로컬 PC에서 실행** (Linux/Mac):

```bash
ssh -i {key.pem} \
  -L 3000:{monitoring_private_ip}:3000 \
  -L 9091:{monitoring_private_ip}:9090 \
  ubuntu@{bastion_floating_ip}
```

**Windows (PowerShell)**:

```powershell
ssh -i C:\path\to\key.pem -L 3000:{monitoring_private_ip}:3000 -L 9091:{monitoring_private_ip}:9090 ubuntu@{bastion_floating_ip}
```

> **Windows에서 로컬 9090 포트가 예약되어 있어 `bind: Permission denied`가 날 수 있다.**
> 그 경우 로컬 포트를 9091 등으로 바꾼다(원격은 9090 그대로).
> 예약 포트 확인: `netsh interface ipv4 show excludedportrange protocol=tcp`

터널을 연 상태에서 브라우저로 접속:

```
Grafana    : http://localhost:3000   (admin / admin, 첫 접속 시 비밀번호 변경)
Prometheus : http://localhost:9091
```

### 8-6. Grafana 초기 설정

**데이터소스 추가**

```
좌측 메뉴 → Connections → Data sources → Add new data source → Prometheus

Prometheus server URL: http://prometheus:9090
```

> **URL이 `localhost:9090`이 아니라 `prometheus:9090`이다.**
> Grafana가 Docker 컨테이너 안에서 돌기 때문에, 컨테이너 입장에서 `localhost`는
> 자기 자신(Grafana)이다. Docker Compose는 같은 네트워크의 컨테이너를
> **서비스 이름**으로 접근하게 해주므로 `prometheus`로 연결한다.

**Save & test** → "Successfully queried the Prometheus API" 확인.

**대시보드 import**

```
좌측 메뉴 → Dashboards → New → Import
→ Import via grafana.com 에 ID 입력

315  : Kubernetes cluster monitoring (노드 지표)
```

> 대시보드 315는 node-exporter 지표를 사용한다.
> Alloy가 내장 node-exporter(`prometheus.exporter.unix`)로 수집하므로 정상 표시된다.
> Alloy 배포 전에는 패널이 N/A로 나오는 것이 정상이다.

---

## 트러블슈팅 메모 (실제로 겪었던 문제들)

| 증상 | 원인 | 해결 |
|------|------|------|
| LB EXTERNAL-IP가 사설 IP(10.1.1.x) | 클러스터가 private_subnet 기준으로 생성됨 | 수동 공인 LB 구성 (섹션 4) |
| LB 멤버가 INACTIVE | 헬스 체크 URL이 `/`라서 404 | URL을 `/healthz`로 변경 |
| ImagePullBackOff | ncr-secret이 해당 네임스페이스에 없음 | web/was 양쪽에 각각 생성 |
| ImagePullBackOff (secret 있는데도) | 이미지 태그 불일치 (`latest` vs `1.0`) | 실제 존재하는 태그로 수정 |
| web Pod CrashLoopBackOff | nginx.conf의 upstream `was`를 못 찾음 (크로스 네임스페이스) | proxy_pass를 FQDN으로 (`was-svc.was.svc.cluster.local`) |
| `/api` 504 Gateway Timeout | proxy_pass가 옛 Service 이름을 가리킴 | 현재 이름(`was-svc`)으로 수정 후 Pod 재시작 |
| `localhost:8080 connection refused` | kubeconfig를 못 읽음 | `~/.kube/config` 경로/권한 확인 |
| Prometheus 설정 변경이 반영 안 됨 | 컨테이너 재시작 안 함 | `docker compose restart prometheus` |
| Alloy가 push해도 Prometheus에 데이터 없음 | remote-write-receiver 미활성화 | `--web.enable-remote-write-receiver` 플래그 추가 |
| Grafana 대시보드가 전부 N/A | 대시보드가 기대하는 메트릭 미수집 | Alloy에 node-exporter/kube-state-metrics 추가 |
| SSH 터널 `bind: Permission denied` (Windows) | 로컬 포트가 시스템 예약됨 | 다른 로컬 포트로 매핑 (9090 → 9091) |

---

## 예상 소요 시간

| 단계 | 시간 |
|------|------|
| NKS 클러스터 생성 | 10분 |
| Ingress Controller 설치 | 2분 |
| 공인 LB 생성 및 설정 | 10분 |
| 애플리케이션 배포 | 5분 |
| 모니터링 재연결 (인스턴스 유지 시) | 5분 |
| **합계** | **약 30분** |

Monitoring Instance를 처음부터 구축하는 경우 20분 추가.