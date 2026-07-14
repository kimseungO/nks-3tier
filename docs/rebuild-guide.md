# 재구축 가이드 (Rebuild Guide)

NKS 클러스터 삭제 후 재구축하는 전체 절차.
콘솔 작업과 kubectl/helm 작업을 순서대로 정리했다.

---

## 전제 조건 (삭제하지 않고 유지되는 것)

아래는 삭제하지 않았으므로 재생성 불필요하다.

| 리소스 | 상태 | 비고 |
|--------|------|------|
| VPC, Subnet, Peering, 라우팅 | 유지 | 무과금 |
| Internet Gateway, NAT Gateway | 유지 | |
| Security Group | 유지 | bastion-sg, nks-sg, rds-sg, monitoring-sg |
| Bastion Host | 유지 | 설정 파일, kubeconfig 보관처 |
| RDS for MySQL | 유지 | appdb 데이터 보존 |
| Monitoring Instance | 유지 | Prometheus, Grafana, blackbox |
| NCR 컨테이너 이미지 | 유지 | suwon-web:1.0, suwon-was:1.0 |

**삭제한 것: NKS 클러스터, 그리고 그에 딸린 공인 LB**

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
> **사설 IP만** 받는다. 이건 정상 동작이다. 외부 노출은 아래 3번(수동 공인 LB)으로 해결한다.
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
```

워커 노드 2개가 Ready로 나오면 성공.

```bash
kubectl get nodes -o wide
```

**워커 노드의 INTERNAL-IP를 메모해둔다.** 3번에서 LB 멤버로 등록해야 한다.

---

## 3. Ingress Controller 설치 (NodePort)

자동 LB를 쓰지 않고 NodePort로만 노출한다. 그 앞에 수동 공인 LB를 붙일 것이다.

### Helm 저장소 추가

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### 설치

`helm-values/ingress-values.yaml` 사용.

```yaml
controller:
  service:
    type: NodePort
    nodePorts:
      http: 30080
      https: 30443
```

```bash
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

> `EXTERNAL-IP: <none>`이 정상이다. 외부 노출은 다음 단계 수동 LB가 담당한다.
> NodePort를 30080으로 **고정**하는 이유는 LB 멤버 포트와 맞추기 위함이다.

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

> **서브넷을 반드시 public_subnet으로** 지정해야 공인 IP를 받을 수 있다.

### 리스너 설정 (외부에서 받는 포트)

| 항목 | 값 |
|------|-----|
| 이름 | listener-1 |
| 프로토콜 | HTTP |
| 로드 밸런서 포트 | **80** |

### 멤버 그룹 설정 (워커 노드로 보내는 포트)

| 항목 | 값 |
|------|-----|
| 이름 | memberGroup-1 |
| 프로토콜 | HTTP |
| **포트** | **30080** |
| 로드 밸런싱 방식 | ROUND_ROBIN |

> **포트를 30080으로 지정해야 한다.** 기본값 80으로 두면 워커 노드의 80 포트로 가는데
> 거기엔 아무것도 없어서 실패한다.

### 상태 확인(헬스 체크) 설정 — 반드시 수정

| 항목 | 값 |
|------|-----|
| 상태 확인 프로토콜 | HTTP |
| 상태 확인 포트 | 멤버 포트 |
| HTTP 메서드 | GET |
| **URL** | **/healthz** |
| HTTP 상태 코드 | 200 |

> **기본값 `/`로 두면 멤버가 INACTIVE가 된다.**
> Ingress Controller는 `/`로 요청하면 라우팅 규칙이 없어 404를 반환하기 때문이다.
> `/healthz`는 Ingress Controller의 헬스 체크 전용 경로로 항상 200을 반환한다.

### 멤버 목록 (워커 노드 등록)

`kubectl get nodes -o wide`에서 확인한 INTERNAL-IP를 등록한다.

| IP 주소 | 멤버 포트 |
|---------|-----------|
| 10.1.1.x (worker-node-0) | 30080 |
| 10.1.1.x (worker-node-1) | 30080 |

> 워커 노드 IP는 클러스터를 재생성하면 **바뀐다.** 반드시 새로 확인해서 등록할 것.

### 생성 후 확인

```
LB 상세 → 멤버 그룹 → 멤버 탭
→ 두 노드가 ACTIVE인지 확인 (30초 정도 소요)
```

INACTIVE면 헬스 체크 URL이 `/healthz`인지 다시 확인한다.

### Floating IP 연결

```
LB 선택 → "플로팅 IP 관리" → Floating IP 연결
```

> 직접 만든 LB는 Floating IP 수동 연결이 가능하다.
> (NKS 자동 생성 LB는 이게 막혀 있다.)

**연결된 공인 IP를 메모해둔다.** 나중에 blackbox 모니터링 설정에 필요하다.

---

## 5. 애플리케이션 배포

### Namespace 생성

```bash
kubectl create namespace web
kubectl create namespace was
```

### Secret 생성 (실제 값은 별도 관리)

`k8s/secret.yaml.example` 참고. 실제 값을 채워서 적용한다.

```bash
# RDS 접속 정보
kubectl create secret generic was-secret \
  --namespace was \
  --from-literal=DATABASE_URL="mysql://{DB_USER}:{DB_PASSWORD}@{RDS_ENDPOINT}:13306/appdb"

# NCR 인증 (web, was 양쪽 네임스페이스에 각각 필요)
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

### ConfigMap, Deployment, Service, Ingress 적용

```bash
cd ~/nks-3tier

kubectl apply -f k8s/web-nginx-conf.yaml
kubectl apply -f k8s/web.yaml
kubectl apply -f k8s/was.yaml
kubectl apply -f k8s/ingress.yaml
```

### 확인

```bash
kubectl get pods -n web
kubectl get pods -n was
kubectl get ingress -n web
```

모두 Running이어야 한다.

---

## 6. 동작 검증

### 클러스터 내부 검증 (web → was)

```bash
kubectl exec -n web $(kubectl get pod -n web -l app=web -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://localhost/api/health
```

`{"status":"ok","tier":"was"}` 가 나오면 web nginx의 프록시가 정상이다.

### 외부 검증

```bash
curl http://{공인_IP}/
curl http://{공인_IP}/api/health
```

- `/` → 정적 페이지 HTML
- `/api/health` → `{"status":"ok","tier":"was"}`

---

## 7. 모니터링 재연결

Monitoring Instance는 삭제하지 않았으므로 Prometheus/Grafana는 그대로 살아 있다.
다만 **클러스터가 새로 생겼으므로 Alloy를 다시 배포**해야 한다.

### Helm 저장소 추가

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

`helm-values/alloy-values.yaml` 사용. remote_write 엔드포인트가
Monitoring Instance IP(`10.0.1.84:9090`)로 되어 있는지 확인한다.

```bash
helm install alloy grafana/alloy \
  --namespace monitoring \
  -f ~/nks-3tier/helm-values/alloy-values.yaml
```

### 확인

```bash
kubectl get pods -n monitoring
```

alloy Pod가 워커 노드 수만큼(DaemonSet), kube-state-metrics Pod 1개가 Running이어야 한다.

Prometheus에서 메트릭 확인:

```
node_memory_MemTotal_bytes
kube_pod_status_phase
```

### blackbox 대상 IP 갱신 (필요 시)

LB를 재생성하면 **공인 IP가 바뀐다.** Monitoring Instance의
`prometheus.yml`에서 blackbox 대상 URL을 새 IP로 수정해야 한다.

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
probe_success
```

두 대상 모두 값이 1이면 정상.

---

## 트러블슈팅 메모 (겪었던 문제들)

| 증상 | 원인 | 해결 |
|------|------|------|
| LB EXTERNAL-IP가 사설 IP(10.1.1.x) | 클러스터가 private_subnet 기준으로 생성됨 | 수동 공인 LB 구성 (4번) |
| LB 멤버가 INACTIVE | 헬스 체크 URL이 `/`라서 404 | URL을 `/healthz`로 변경 |
| ImagePullBackOff | ncr-secret이 해당 네임스페이스에 없음 | web/was 양쪽에 각각 생성 |
| ImagePullBackOff (secret 있는데도) | 이미지 태그 불일치 (`latest` vs `1.0`) | 실제 존재하는 태그로 수정 |
| web Pod CrashLoopBackOff | nginx.conf의 upstream `was`를 못 찾음 (크로스 네임스페이스) | proxy_pass를 FQDN으로 (`was-svc.was.svc.cluster.local`) |
| `/api` 504 Gateway Timeout | proxy_pass가 옛 Service 이름(`was`)을 가리킴 | 현재 이름(`was-svc`)으로 수정 후 Pod 재시작 |
| `localhost:8080 connection refused` | kubeconfig를 못 읽음 | `~/.kube/config` 경로/권한 확인 |
| Prometheus 설정 변경이 반영 안 됨 | 컨테이너 재시작 안 함 | `docker compose restart prometheus` |

---

## 예상 소요 시간

| 단계 | 시간 |
|------|------|
| NKS 클러스터 생성 | 10분 |
| Ingress Controller 설치 | 2분 |
| 공인 LB 생성 및 설정 | 10분 |
| 애플리케이션 배포 | 5분 |
| 모니터링 재연결 | 5분 |
| **합계** | **약 30분** |
