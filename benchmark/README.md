# fio 벤치마크 측정 조건

self-managed 환경에서 동일 조건으로 재현하여 비교한다.

## 공통
- direct=1, ioengine=libaio, size=2G, runtime=60s, time_based
- PVC 10Gi

## 측정 종류
| 파일 | rw | bs | iodepth | numjobs | 측정 목적 |
|------|-----|-----|---------|---------|-----------|
| randwrite-4k | randwrite | 4k | 32 | 4 | 랜덤 쓰기 IOPS |
| randread-4k | randread | 4k | 32 | 4 | 랜덤 읽기 IOPS |
| seqwrite-1m | write | 1M | 16 | 1 | 순차 쓰기 처리량 |
| seqread-1m | read | 1M | 16 | 1 | 순차 읽기 처리량 |

## NKS 측정 환경
- storageClass: cinder-default
- 측정일: 2026-07-27
