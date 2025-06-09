# Week 5 - PR 분석

@https://github.com/vmware-tanzu/helm-charts/issues/628 

### 목표
Velero helm chart의 repositoryMaintenanceJob에 ttlSecondsAfterFinished 파라미터를 추가하여 Completed 상태의 maintenance job 파드들을 자동으로 정리할 수 있도록 하기

### 배경
이슈: Maintenance Job이 자주 실행되어 Completed 상태의 파드가 계속 누적됨
원인: Kubernetes Job에서 ttlSecondsAfterFinished 파라미터가 설정되지 않아 완료된 파드가 자동 삭제되지 않음
Kubernetes 지원: ttlSecondsAfterFinished는 Kubernetes 1.23+에서 stable 기능
Velero 현황: Velero v1.14+부터 maintenance job을 별도 k8s Job으로 실행

### 현재 아키텍처
1. values.yaml의 repositoryMaintenanceJob 섹션에는 requests, limits, latestJobCount 옵션만 존재
2. deployment.yaml 템플릿에서 이 설정들이 Velero server의 CLI argument로 전달됨
3. Velero server가 실제 maintenance job을 생성할 때 이 파라미터들을 사용

### 구현
deployment에 ttlSecondsAfterFinished 추가

### 이슈
velero server에서 --maintenance-job-ttl-seconds-after-finished 플래그가 구현되어있는지 확인 필요

### next Week
필요하면 server 코드 수정해야할 듯?
