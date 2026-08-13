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

<img width="1937" height="969" alt="image" src="https://github.com/user-attachments/assets/c9120b94-a5de-4811-99bc-c61a59ece293" />

