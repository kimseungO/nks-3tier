> ### Managed vs Self-managed Kubernetes 플랫폼 비교 프로젝트
>
> 동일한 3-tier 워크로드를 관리형 플랫폼(NHN Cloud NKS)과
> 자체 구축 플랫폼(OpenStack + Ceph)에 각각 올려,
> 관리형 플랫폼이 추상화하는 계층을 직접 측정·비교합니다.
>
> | 구성 | 역할 | 상태 |
> |---|---|---|
> | [suwon-daytour-3tier](https://github.com/kimseungO/suwon-daytour-3tier) | 애플리케이션 · K8s 매니페스트 | 완료 |
> | [nks-3tier](https://github.com/kimseungO/nks-3tier) | NKS 운영 환경 (CI/CD · 모니터링) | 완료 |
> | [nks-storage-layer-analysis](https://github.com/kimseungO/nks-storage-layer-analysis) | 스토리지 성능 계층 분석 | 완료 |
> | [openstack-private-cloud](https://github.com/kimseungO/openstack-private-cloud) | Self-managed 인프라 구축 | 진행중 |


이 레포는 프로젝트의 **관리형 플랫폼(NKS) 운영 계층**을 담당합니다.
앱 변경과 인프라 변경의 배포 주기가 달라 애플리케이션 레포와 분리했습니다.


# NKS 기반 3-Tier 웹 서비스 인프라

한국 여행 상품을 판매하는 중소규모 업체의 클라우드 마이그레이션을 가정하고, **NHN Cloud NKS 위에 3-Tier 웹 서비스 인프라를 설계·구축·운영한 프로젝트**입니다.

단순 배포에 그치지 않고, 고객이 요구한 네 가지 조건(비용·안정성·보안·확장성)을 각각 어떤 아키텍처로 해결했는지에 초점을 맞췄습니다. 모니터링, CI/CD, 오토스케일링까지 운영에 필요한 체계를 갖췄으며, 모든 구성은 파일로 관리해 재현 가능합니다.

<img width="1937" height="962" alt="image" src="https://github.com/user-attachments/assets/a5f5eeb6-d36f-4acb-ba1e-e36296a5fa40" />


### 한눈에 보기

| | |
|---|---|
| **무엇을** | 3-Tier 웹 서비스(Nginx + Node.js + MySQL)를 Managed Kubernetes 위에 구축 |
| **어떻게** | 워커 노드를 사설망에 격리하고, 관리 VPC에서 별도로 관측하는 구조 |
| **운영 체계** | 크로스 VPC 모니터링 · Jenkins 기반 CI/CD · CPU 기반 오토스케일링 |
| **역할** | 인프라 설계, 구축, 운영 체계 수립 전 과정 단독 수행 |
| **기간** | 약 2개월 |

---

## 목차

1. [고객 요구사항과 해결 방식](#1-고객-요구사항과-해결-방식)
2. [아키텍처](#2-아키텍처)
3. [기술 스택](#3-기술-스택)
4. [구현 상세](#4-구현-상세)
5. [설계 결정과 근거](#5-설계-결정과-근거)
6. [트러블슈팅](#6-트러블슈팅)
7. [저장소 구조](#7-저장소-구조)
8. [재구축](#8-재구축)
9. [다음 단계](#9-다음-단계)

---

## 1. 고객 요구사항과 결정 사항

### 고객 상황

> **한국의 버스 데이투어 여행업체**
>
> 여행 수요가 늘면서 소규모에서 중소규모로 확장되었습니다. 단체 예약이 잦아지며 서버가 느려지거나 다운되는 일이 발생했고, MSP를 통한 클라우드 마이그레이션을 결정했습니다.

### 요구사항 → 구현

고객의 네 가지 요구사항이 이 프로젝트의 설계 기준입니다.

#### 비용 효율성
> *"이제 막 궤도에 오른 사업이라 현금 흐름이 넉넉하지 않습니다."*

- 워커 노드 2대(m2.c2m4)로 최소 구성, 필요할 때만 HPA로 확장
- Cluster Autoscaler 대신 Pod 단위 스케일링을 선택해 노드 증설 비용 억제
- 모니터링 스택을 관리형 서비스 대신 단일 인스턴스에 Docker Compose로 구성

#### 안정성
> *"이용자가 갑자기 늘어도 안정적이어야 하고, 그걸 지켜볼 수 있어야 합니다."*

- 관리 VPC에서 서비스 VPC를 관측하는 크로스 VPC 모니터링
- 내부 지표(화이트박스)와 외부 접속 확인(블랙박스)을 함께 수집
- RDS Active-Standby 구성으로 DB 이중화
- 서비스 관제 대시보드로 생존·트래픽·지연·리소스를 한 화면에서 확인

#### 보안성
> *"고객 정보를 일정 기간 보관해야 해서 외부 노출이 걱정됩니다."*

- 워커 노드를 Private Subnet에 배치, 외부 노출 지점을 LB 하나로 제한
- DB를 별도 서브넷에 격리, 클러스터 내부에서만 접근
- 3-Tier 계층 분리로 브라우저가 애플리케이션 서버의 존재를 알 수 없음
- Kubernetes API 엔드포인트를 Private으로 설정, Bastion 경유로만 접근

#### 확장성 및 개발 편의성
> *"이용자가 늘 것에 대비하고, 타지역 상품도 개발할 예정입니다."*

- CPU 기반 HPA로 트래픽 증가 시 자동 확장 (부하 테스트로 검증)
- Jenkins + Kaniko CI/CD로 코드 push부터 배포까지 자동화
- 매니페스트 기반 배포로 환경 변경도 코드로 관리

---

## 2. 아키텍처

### 네트워크 구성

<img width="1937" height="962" alt="image" src="https://github.com/user-attachments/assets/305201ae-f708-45e1-8939-960cbb254d73" />


서비스망과 관리망을 VPC 단위로 분리하고 Peering으로 연결했습니다.

```
lab_vpc (10.1.0.0/16) — 서비스 VPC
├─ public_subnet  (10.1.0.0/24)   Internet Gateway, 공인 LoadBalancer
├─ private_subnet (10.1.1.0/24)   NAT Gateway, NKS 워커 노드
└─ db_subnet      (10.1.2.0/24)   RDS for MySQL (Active-Standby)

mgt_vpc (10.0.0.0/16) — 관리 VPC
├─ mgt_public_subnet  (10.0.0.0/24)   Bastion Host
└─ mgt_private_subnet (10.0.1.0/24)   Monitoring Instance
```

### 요청 흐름

```
사용자
  ↓
공인 LoadBalancer (Floating IP)
  ↓
워커 노드 NodePort 30080
  ↓
Ingress Controller
  ↓
web (Nginx)  ──/api──▶  was (Node.js)  ──▶  RDS for MySQL
   정적 파일               비즈니스 로직          데이터
```

워커 노드가 Private Subnet에 있어 자동 생성 LoadBalancer는 사설 IP만 받습니다. Ingress Controller를 NodePort로 노출하고 그 앞에 공인 LB를 직접 구성하여, **노드를 외부에 노출하지 않으면서 서비스를 공개**했습니다.

---

## 3. 기술 스택

| 구분 | 기술 |
|------|------|
| **오케스트레이션** | NHN Cloud NKS (Kubernetes v1.34.3), Calico |
| **네트워크** | VPC Peering, NAT Gateway, LoadBalancer, Ingress-NGINX |
| **스토리지** | Cinder CSI, Object Storage (S3 호환 API) |
| **데이터베이스** | RDS for MySQL (Active-Standby) |
| **애플리케이션** | Nginx, Node.js + Express, Prisma, React |
| **CI/CD** | Jenkins, Kaniko, NHN Container Registry |
| **모니터링** | Prometheus, Grafana, Grafana Alloy, kube-state-metrics, blackbox-exporter |
| **오토스케일링** | HPA, metrics-server |

---

## 4. 구현 상세

### 4-1. 크로스 VPC 모니터링

<!-- 📷 이미지 삽입: 모니터링 아키텍처 (Alloy → remote_write → Prometheus → Grafana) -->

관리 VPC의 Prometheus가 서비스 VPC를 긁어오는 Pull 방식 대신, 클러스터 내부의 Grafana Alloy가 메트릭을 모아 밀어 보내는 **Push(remote_write) 방식**을 택했습니다.

```
[서비스 VPC — NKS]
Grafana Alloy (DaemonSet)
├─ node-exporter        노드 CPU · 메모리 · 디스크
├─ cAdvisor             컨테이너 리소스
├─ kubelet              kubelet 지표
├─ kube-state-metrics   Pod · Deployment 상태
└─ was /metrics         애플리케이션 지표
        │
        │  remote_write — 경계를 넘는 단일 아웃바운드 연결
        ▼
[관리 VPC — Monitoring Instance]
Prometheus  →  Grafana
blackbox-exporter  →  외부에서 서비스 생존 확인
```

**Push를 택한 이유**는 두 가지입니다. VPC 경계를 넘는 연결이 하나로 줄어 노출면이 최소화되고, 노드가 늘어나도 Alloy가 클러스터 내부에서 자동 발견하므로 수집 대상을 수동 관리할 필요가 없습니다.

### 4-2. 화이트박스 + 블랙박스

내부 지표만으로는 *"Pod는 정상인데 사용자는 접속하지 못하는"* 상황을 잡을 수 없습니다. 실제로 프록시 설정 오류로 504가 발생했을 때 모든 내부 지표는 정상이었습니다.

관리 VPC의 blackbox-exporter가 공인 IP로 실제 HTTP 요청을 보내 생존을 확인합니다. **감시 대상과 분리된 위치**에 있으므로, 서비스 VPC 전체가 중단되어도 감지할 수 있습니다.

### 4-3. 서비스 관제 대시보드


<img width="1291" height="924" alt="image" src="https://github.com/user-attachments/assets/f5920849-5c5e-497c-a2a1-62e6e4cc8d98" />
<img width="1305" height="924" alt="image" src="https://github.com/user-attachments/assets/46015ce8-5841-438c-bce2-a9bcbbefa07b" />


커뮤니티 대시보드를 가져오는 대신, 이 서비스를 관제하는 데 실제로 필요한 지표를 직접 선별했습니다.

| 관제 질문 | 패널 | 지표 |
|-----------|------|------|
| 사용자가 접속할 수 있는가 | 서비스 생존 | `probe_success` |
| 트래픽은 어느 정도인가 | 초당 요청 수 | `rate(http_requests_total[1m])` |
| 어떤 API에 몰리는가 | 엔드포인트별 분포 | `sum by (route) (...)` |
| 사용자가 체감하는 지연은 | p95 응답시간 | `histogram_quantile(0.95, ...)` |
| 부하에 어떻게 대응했는가 | Pod 개수 | `kube_deployment_status_replicas` |
| 리소스는 충분한가 | CPU · 메모리 | `container_cpu_usage_seconds_total` |

대시보드 정의는 JSON으로 저장소에 보관하여 재구축 시 복원됩니다.

### 4-4. CI/CD 파이프라인

<!-- 📷 이미지 삽입: Jenkins 파이프라인 실행 화면 -->

```
GitHub push
  ↓
Jenkins (클러스터 내부, cicd 네임스페이스)
  ↓
Agent Pod 동적 생성 — Kaniko + kubectl 컨테이너
  ↓
Kaniko 이미지 빌드 → NCR push
  ↓
매니페스트 태그 치환 → kubectl apply
  ↓
롤링 업데이트 (무중단 배포)
```

**Kaniko**를 사용해 Docker 데몬 없이, 특권 컨테이너 없이 이미지를 빌드합니다.

**배포는 `apply` 방식**입니다. `kubectl set image`는 이미지만 교체하므로 환경변수나 ConfigMap 변경이 반영되지 않습니다. 매니페스트의 태그 플레이스홀더를 빌드 번호로 치환한 뒤 전체를 apply하여, 매니페스트가 항상 배포 상태를 정확히 표현하도록 했습니다.

**Jenkins에는 최소 권한만** 부여했습니다. 필요한 리소스(Deployment, ConfigMap, Service, HPA)를 명시적으로 나열한 Role을 대상 네임스페이스에만 바인딩했습니다.

### 4-5. 애플리케이션 메트릭

`prom-client`로 애플리케이션이 직접 메트릭을 노출합니다.

```
http_requests_total              엔드포인트 · 상태코드별 요청 수
http_request_duration_seconds    응답 시간 히스토그램
nodejs_*                         이벤트 루프, 힙 등 프로세스 지표
```

`/metrics`는 애플리케이션 포트에만 노출되며 Alloy가 Pod를 직접 스크레이프합니다. Nginx를 거치지 않으므로 외부에 노출되지 않습니다.

### 4-6. HPA 오토스케일링

<!-- 📷 이미지 삽입: HPA 동작 그래프 (Pod 개수 + CPU 사용률) -->

단체 예약으로 트래픽이 몰리는 상황에 대응하기 위해 CPU 기반 HPA를 적용하고, 부하를 주입해 실제 동작을 검증했습니다.

| 시점 | CPU | Replicas | 동작 |
|------|----:|---------:|------|
| 평상시 | 5% | 1 | — |
| 부하 시작 | 85% | 1 → 2 | 임계 초과, 확장 시작 |
| 부하 증가 | 190% | 2 → 4 | 초과 폭에 비례해 한 번에 2개 추가 |
| 부하 지속 | 78% | 4 → 5 | 상한 도달, 더 이상 증설 없음 |
| 부하 제거 | 5% | 5 | 약 5분간 유지 (안정화 대기) |
| 안정화 후 | 4% | 5 → 1 | 축소 |

부하가 클수록 여러 Pod를 한 번에 추가하는 **비례 제어**, `maxReplicas` 상한 준수, 스케일 인 시 **약 5분의 안정화 대기**가 모두 확인되었습니다. 확장은 빠르고 축소는 느린 비대칭 동작은 반복적인 증감(플래핑)을 방지하기 위한 설계입니다.

### 4-7. Object Storage 연동

데이터 성격에 따라 접근 정책을 구분했습니다.

| 데이터 | 성격 | 처리 |
|--------|------|------|
| 배너 사진 | 운영자가 관리하는 정적 콘텐츠 | Public URL 직접 참조 |
| 리뷰 사진 | 사용자가 런타임에 업로드 | 애플리케이션이 S3 호환 API로 업로드 |
| 고객 · 예약 정보 | 보관이 필요한 민감 데이터 | RDS(Private Subnet)에 격리 |

모든 데이터를 잠그는 것이 보안은 아니라고 판단했습니다. 공개가 목적인 콘텐츠는 공개하고, 실제로 보호가 필요한 데이터는 네트워크 계층에서 격리했습니다.

---

## 5. 설계 결정과 근거

<details>
<summary><b>Private Subnet + 수동 공인 LoadBalancer</b> — 노드 격리와 서비스 공개를 동시에</summary>

<br>

NKS는 클러스터가 위치한 서브넷에 따라 LoadBalancer의 IP 유형이 결정됩니다. Private Subnet에 클러스터를 두면 자동 생성 LB가 사설 IP만 받습니다.

Public Subnet으로 옮기면 자동으로 공인 LB를 얻지만, 워커 노드가 외부에 노출됩니다. 고객 정보를 보관해야 하는 요구사항을 고려해 노드 격리를 우선하고, Ingress Controller를 NodePort로 노출한 뒤 공인 LB를 직접 구성했습니다.

LB 운영은 MSP가 담당하므로 고객 관점의 운영 부담은 늘지 않습니다.

</details>

<details>
<summary><b>Jenkins</b> — GitHub Actions 대신 클러스터 내부 CI/CD</summary>

<br>

Kubernetes API 엔드포인트가 Private이므로, 외부에서 실행되는 GitHub Actions 러너는 클러스터에 접근할 수 없습니다.

우회 방법으로 Self-hosted 러너를 내부에 두거나 API를 Public으로 여는 방법이 있지만, 전자는 GitHub Actions의 장점(설치 불필요)을 상쇄하고 후자는 격리 원칙을 훼손합니다.

Jenkins를 클러스터 내부에 두면 이 문제가 발생하지 않으며, 폐쇄망 환경에서도 동일하게 동작합니다.

</details>

<details>
<summary><b>Kaniko</b> — Docker-in-Docker 대신 특권 없는 빌드</summary>

<br>

Jenkins가 Pod로 동작하므로 Docker 데몬이 없습니다. DinD는 특권 컨테이너를 요구하는데, 이는 컨테이너가 호스트에 광범위하게 접근할 수 있다는 의미이므로 보안 요구사항과 충돌합니다.

Kaniko는 Dockerfile을 사용자 공간에서 실행해 특권 없이 이미지를 빌드하고, 완성된 이미지를 레지스트리로 직접 push합니다.

</details>

<details>
<summary><b>Pod 오토스케일링</b> — Cluster Autoscaler를 도입하지 않은 이유</summary>

<br>

노드 오토스케일링은 노드 증설로 이어져 비용이 증가합니다. 비용 효율성이 중요한 고객 조건에서, 현재 트래픽 규모라면 Pod 단위 확장으로 충분하다고 판단했습니다.

또한 수동으로 구성한 공인 LB의 멤버 목록이 워커 노드 IP로 등록되어 있어, 노드가 자동 증감하면 멤버 목록 관리가 복잡해지는 문제도 고려했습니다.

</details>

---

## 6. 트러블슈팅

구축 과정에서 겪은 주요 문제와 해결 과정입니다. 상세 기록은 [기술 블로그](https://velog.io/@rtd7878)에 정리했습니다.

| 증상 | 원인 | 해결 |
|------|------|------|
| LoadBalancer가 사설 IP만 할당 | 클러스터가 Private Subnet 기준으로 생성됨 | NodePort + 수동 공인 LB 구성 |
| LB 멤버가 INACTIVE 상태 | 헬스체크 URL이 `/`라 Ingress가 404 반환 | 헬스체크 경로를 `/healthz`로 변경 |
| Jenkins Pod 볼륨 권한 오류 | StorageClass에 fsType 미선언으로 CSI가 fsGroup 미적용 | `csi.storage.k8s.io/fstype: ext4` 추가 |
| PVC가 Terminating에서 정지 | 볼륨을 참조하는 Pod가 남아 finalizer 유지 | 해당 Pod 강제 삭제 |
| 배포 단계 프로세스 미실행 | `bitnami/kubectl` 이미지에 shell 부재 | `alpine/k8s` 이미지로 교체 |
| `kubectl apply` 권한 거부 | Role에 해당 리소스 권한 없음 | RBAC에 ConfigMap · Service · HPA 추가 |
| 배포 후 이미지가 이전 버전으로 롤백 | 매니페스트에 옛 태그가 고정됨 | 태그 치환 후 apply 방식으로 전환 |
| 크로스 네임스페이스 프록시 실패 (504) | Nginx upstream이 짧은 이름 사용 | FQDN(`was-svc.was.svc.cluster.local`)으로 변경 |
| Push해도 메트릭 미수집 | Prometheus remote-write 수신 비활성 | `--web.enable-remote-write-receiver` 추가 |

---

## 7. 저장소 구조

이 저장소는 **인프라 공통 설정**을 관리합니다. 애플리케이션 코드와 배포 매니페스트는 별도 저장소에 있으며, CI/CD 파이프라인이 이를 배포합니다.

```
nks-3tier/
├── k8s/
│   ├── ingress.yaml            Ingress 정의
│   ├── storageclass.yaml       Cinder StorageClass (fsType 포함)
│   ├── jenkins-rbac.yaml       CI/CD 배포 권한
│   └── secret.yaml.example     Secret 구조 예시
├── helm-values/
│   ├── ingress-values.yaml     Ingress Controller (NodePort)
│   ├── alloy-values.yaml       메트릭 수집 및 remote_write
│   └── jenkins-values.yaml     Jenkins 배포 설정
├── monitoring/
│   ├── docker-compose.yml      Prometheus · Grafana · blackbox
│   ├── prometheus.yml          수집 설정
│   └── dashboards/             Grafana 대시보드 JSON
├── cicd/
│   ├── Jenkinsfile.reference   파이프라인 구조 참고본
│   └── README.md               CI/CD 구축 절차
├── benchmark/                  스토리지 성능 측정 결과
└── docs/
    └── rebuild-guide.md        전체 재구축 가이드
```

> 자격증명, 키, kubeconfig 등 민감 정보는 저장소에 포함하지 않으며 생성 절차만 문서화되어 있습니다.

---

## 8. 재구축

클러스터를 삭제한 뒤 전체를 다시 구축하는 절차를 [`docs/rebuild-guide.md`](docs/rebuild-guide.md)에 정리했습니다. 콘솔 작업과 명령 기반 작업을 순서대로 포함하며, 예상 소요 시간은 약 60분입니다.

다음 항목은 누락 시 후속 단계가 실패하므로 가이드에서 별도로 강조했습니다.

- `cinder-csi-plugin` 애드온 설치 — 미설치 시 PVC 사용 불가
- StorageClass의 `fstype` 선언 — 미선언 시 non-root 워크로드 기동 실패
- LoadBalancer 헬스체크 경로 `/healthz` — 기본값 사용 시 멤버 INACTIVE
- 워커 노드 IP와 공인 IP는 재생성 시 변경되므로 LB 멤버·모니터링 대상 갱신 필요

---

## 9. 다음 단계

이 프로젝트는 Managed Kubernetes 환경에서의 인프라 구축과 운영을 다뤘습니다. 다음 단계로 **동일한 서비스를 Self-managed Kubernetes(OpenStack + Ceph) 위에 구축하여 두 환경을 비교**할 예정입니다.

Managed 환경에서는 스토리지 성능을 측정할 수는 있지만, 그 수치가 왜 나오는지는 알 수 없습니다. 백엔드 구성, 복제 정책, 물리 디스크 상태에 접근할 수 없기 때문입니다. Self-managed 환경에서는 동일한 측정값이 어느 계층에서 병목이 되는지 물리 디스크까지 추적할 수 있습니다.

비교의 기준선으로 이 환경에서 측정한 스토리지 성능 데이터를 [`benchmark/`](benchmark/)에 보관해 두었습니다.

---

## 관련 링크

| | |
|---|---|
| 애플리케이션 저장소 | [suwon-daytour-3tier](https://github.com/kimseungO/suwon-daytour-3tier) |
| Self-managed 스토리지 분석 | [ceph-openstack-performance-analysis](https://github.com/kimseungO/ceph-openstack-performance-analysis) |
| 기술 블로그 | [velog.io/@rtd7878](https://velog.io/@rtd7878) |
