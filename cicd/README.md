# CI/CD 구성 (Jenkins + Kaniko)

Jenkins를 클러스터 안(cicd 네임스페이스)에 두고,
Kaniko로 이미지를 빌드해 NCR에 push, kubectl로 배포하는 파이프라인.

## 구성 요소

| 요소 | 위치 | 역할 |
|------|------|------|
| Jenkins | cicd 네임스페이스 (Helm) | CI/CD 컨트롤러 |
| Kaniko | 파이프라인 Agent Pod 내 컨테이너 | Docker 데몬 없이 이미지 빌드 |
| Jenkinsfile | 앱 레포 루트 (Pipeline from SCM) | 파이프라인 정의 |

## 구축 순서

### 1. 사전 조건
- StorageClass(cinder-default)에 fsType 선언 필수
  (k8s/storageclass.yaml 참고 — 없으면 Jenkins PVC 권한 오류)

### 2. Jenkins 배포
helm repo add jenkins https://charts.jenkins.io
helm repo update
kubectl create namespace cicd
helm install jenkins jenkins/jenkins -n cicd -f helm-values/jenkins-values.yaml

### 3. 접근 (Bastion 경유)
# Bastion에서
kubectl port-forward -n cicd svc/jenkins 8080:8080 --address 0.0.0.0
# 로컬 PC에서 SSH 터널
ssh -i {key} -L 8080:localhost:8080 ubuntu@{bastion_ip}
# 브라우저: http://localhost:8080 (admin/admin123)

### 4. Jenkins 자격증명 등록 (웹 UI)
- github-cred : Username with password (GitHub 사용자명 + PAT)
- ncr-cred    : Username with password (NCR Access/Secret Key)

### 5. Kaniko용 NCR Secret 생성
kubectl create secret docker-registry kaniko-ncr-secret \
  --namespace cicd \
  --docker-server=f2da3ca6-kr2-registry.container.nhncloud.com \
  --docker-username={ACCESS_KEY} \
  --docker-password={SECRET_KEY}

### 6. 배포 권한 RBAC
kubectl apply -f k8s/jenkins-rbac.yaml
# cicd:default ServiceAccount가 web/was의 Deployment를 배포할 수 있게 함

### 7. Jenkins Pipeline job 생성 (웹 UI)
- New Item → Pipeline
- Definition: Pipeline script from SCM
- SCM: Git, Repository: 앱 레포 URL, Credentials: github-cred
- Branch: */main, Script Path: Jenkinsfile

### 8. 실행
- Build Now (수동 트리거)
- webhook 자동 트리거는 향후 과제 (private 클러스터라 경로 구성 필요)

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| Jenkins Pod Init 권한 오류 | StorageClass에 fsType 없음 | storageclass에 csi.storage.k8s.io/fstype: ext4 |
| Deploy 단계 "process never started" | bitnami/kubectl 이미지에 shell 없음 | alpine/k8s 이미지로 교체 |
| PVC Terminating 멈춤 | 볼륨 참조하는 Pod 잔존 | 해당 Pod 강제 삭제 |

## 파이프라인 구조
cicd/Jenkinsfile.reference 참고. 실제 파일은 앱 레포 루트에 위치.
