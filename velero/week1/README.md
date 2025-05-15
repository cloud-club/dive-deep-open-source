# Week 1 - Velero 집중탐구

## 1. 개요

Velero(구 Heptio Ark)는 쿠버네티스 클러스터의 리소스 및 퍼시스턴트 볼륨을 백업하고 복원할 수 있는 오픈소스 도구입니다. 주로 다음 용도로 사용됩니다:

- 클러스터 전체 백업 및 장애 발생 시 복원
- 클러스터 간 리소스 마이그레이션
- 프로덕션 상태를 개발/테스트 클러스터로 복제

Velero는 다음 두 가지 구성 요소로 이루어져 있습니다:

- **클러스터 내 서버(Operator)**
- **로컬에서 사용하는 CLI 클라이언트**

---

## 2. 주요 구성 요소

Velero는 Kubernetes의 Custom Resource Definition(CRD)을 활용하여 운영됩니다. 또한, Velero는 이러한 커스텀 리소스를 처리하여 백업, 복원 및 관련된 모든 작업을 수행하는 컨트롤러를 포함합니다.

### 2.1 CRD(CustomResourceDefinition)

Velero는 백업·복원·스케줄·저장소 위치 설정을 위한 여러 CRD를 추가합니다:

https://github.com/vmware-tanzu/helm-charts/tree/main/charts/velero/crds

- `Backup` (`Backups.velero.io`) : 온디맨드 또는 스케줄 백업 정의
    
    ```yaml
    apiVersion: velero.io/v1
    kind: Backup
    metadata:
      namespace: velero
      name: backup-1
    spec:
      includedNamespaces:
        - wordpress
      ttl: 72h0m0s
      snapshotVolumes: true
    ```
    
- `Restore` (`Restores.velero.io`) : 특정 백업을 기반으로 복원 작업 정의
    
    ```yaml
    apiVersion: velero.io/v1
    kind: Restore
    metadata:
      namespace: velero
      name: restore-1
    spec:
      backupName: backup-1
      includedNamespaces:
        - wordpress
    ```
    
- `Schedule` (`Schedules.velero.io`) : 반복 백업 스케줄 정의
    
    ```yaml
    apiVersion: velero.io/v1
    kind: Schedule
    metadata:
      namespace: velero
      name: daily-backup
    spec:
      schedule: "0 3 * * *"
      template:
        includedNamespaces:
          - wordpress
        ttl: 168h0m0s # 7일
        snapshotVolumes: true
    ```
    
- `BackupStorageLocation` : 오브젝트 스토리지(S3/GCS 등) 설정
    
    ```yaml
    apiVersion: velero.io/v1
    kind: BackupStorageLocation
    metadata:
      name: default
      namespace: velero
    spec:
      backupSyncPeriod: 2m0s
      provider: aws
      objectStorage:
        bucket: myBucket
      credential:
        name: secret-name
        key: key-in-secret
      config:
        region: us-west-2
        profile: "default"
    ```
    
- `VolumeSnapshotLocation` : 볼륨 스냅샷 대상(CSI, 클라우드 벤더) 설정
    
    ```yaml
    apiVersion: velero.io/v1
    kind: VolumeSnapshotLocation
    metadata:
      name: aws-default
      namespace: velero
    spec:
      provider: aws
      credential:
        name: secret-name
        key: key-in-secret
      config:
        region: us-west-2
        profile: "default"
    ```
    

이 CRD들은 **etcd**에 저장되며, Velero 컨트롤러가 이를 감시(watch)하며 동작을 트리거합니다.

### 2.2 컨트롤러(Operator)

Velero 서버는 쿠버네티스 내 `velero` 네임스페이스에 Deployment로 배포되며, 주요 컨트롤러는 다음과 같습니다:

- **BackupController**: `Backup` CR을 처리하여 API 서버에서 리소스를 조회하고 tarball 생성 후 스토리지에 업로드
- **RestoreController**: `Restore` CR을 처리하여 백업 데이터를 다운로드하고 리소스를 재생성
- **ScheduleController**: `Schedule` CR을 처리하여 주기적 백업 생성
- **BackupSyncController**: 외부 스토리지에 남은 tarball 목록을 스캔해 클러스터 내 `Backup` CR과 동기화(자동 생성/삭제)
- **GarbageCollectionController**: TTL이 지난 백업을 정리

---

## 3. 백업 워크플로우

![image.png](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRGty7uC33lh6TEbXVm-5SDos2sX5EbSVdKLg&s)

### 3.1 수동 백업

```
velero backup create <백업이름> [--flags]
```

1. CLI가 `Backup` CR을 생성
2. BackupController가 CR 검증 후 대상 리소스를 API 서버에서 조회
3. 리소스를 `resources/` 디렉터리 구조로 JSON/YAML 직렬화
4. tarball(`backup-<이름>-resources.tar.gz`)로 묶어 오브젝트 스토리지에 업로드
5. `-snapshot-volumes=true`(기본) 시 볼륨 스냅샷 생성(CSI/클라우드 API)

> 참고: 클러스터 상태는 원자적이지 않아, 동시에 변경된 리소스는 일관성 보장이 제한됩니다.
> 

### 3.2 스케줄 백업

Cron 표현식으로 스케줄을 정의:

```
velero schedule create <스케줄이름> --schedule="0 */6 * * *"
```

- 각 실행 결과는 `<스케줄이름>-<타임스탬프>` 형태의 백업으로 저장
- 다음 예약 시점에 첫 실행

---

## 4. 복원 워크플로우

```
velero restore create --from-backup <백업이름> [--flags]
```

1. CLI가 `Restore` CR 생성
2. RestoreController가 CR 검증
3. tarball 다운로드·압축 해제 후 매니페스트 전처리(API 버전 호환성 확인)
4. 리소스 재생성 순서: CRD → 네임스페이스 → 클러스터 스코프 → 네임스페이스 스코프
5. 볼륨 스냅샷 복원(CSI 스냅샷 리소스 생성 또는 Restic/Kopia Pod 실행)
- 기본적으로 기존 리소스는 건드리지 않으며(`existing-resource-policy=Skip`), `-existing-resource-policy=Update` 옵션으로 업데이트 가능

---

## 5. 저장소 위치 설정

### 5.1 BackupStorageLocation

- S3/GCS/Azure 등을 위한 버킷, 경로(prefix), 인증 정보(region, 서비스어카운트) 등 설정
- Velero는 이 위치를 전용으로 사용하며, tarball 및 백업 메타데이터를 저장

### 5.2 VolumeSnapshotLocation

- CSI 드라이버, AWS EBS, GCP PD 등 스냅샷 대상에 대한 설정
- 여러 위치를 정의해 멀티 리전 백업 또는 온프레미스/클라우드 혼합 시 사용 가능

---

## 6. 추가 기능 및 설정

### 6.1 백업 만료(TTL)

- `-ttl <기간>` 옵션으로 백업 수명 지정(기본 30일)
- TTL 만료 후 GarbageCollectionController가 CR, tarball, 스냅샷을 정리

### 6.2 오브젝트 스토리지 동기화

- BackupSyncController가 스토리지와 클러스터 CR을 비교
- 스토리지에만 있는 tarball을 자동으로 `Backup` CR로 생성

### 6.3 플러그인

- 다양한 스토리지 및 스냅샷 제공자를 위한 인트리(in-tree) 및 커스텀 플러그인 지원
- 인증, API 연동 등의 기능 확장

### 6.4 병렬 백업

- 파일 업로드를 병렬로 구성하여 백업 속도를 높일 수 있습니다.

```bash
velero backup create <BACKUP_NAME> --include-namespaces <NAMESPACE> --parallel-files-upload <NUM> --wait
```

### 6.5 파일 시스템 백업

https://velero.io/docs/v1.15/file-system-backup/

Velero는 Kubernetes 볼륨을 파일 시스템에서 백업하고 복원하는 기능을 제공하며, 이를 파일 시스템 백업(FSB) 또는 Pod Volume 백업이라고 합니다. 데이터 이동은 restic 및 kopia 같은 오픈 소스 백업 도구의 모듈을 사용하여 수행됩니다.

- 클라우드 공급자의 스냅샷 API에 의존하지 않음
- 블록 스토리지 백업을 지원하지 않는 스토리지 시스템에서도 사용 가능

---

## 7. 모범 사례

- **etcd 스냅샷**을 주기적으로 별도 관리하여 전체 클러스터 복구 준비
- **복원 테스트**를 개발/테스트 환경에서 미리 수행해 백업 무결성 검증
- **스케줄 + TTL**을 적절히 조합해 스토리지 비용 최적화
- **별도 시크릿**으로 각 저장소 위치 인증 정보 안전하게 관리
- **로그 모니터링 및 알림**을 설정해 백업/복원 실패 즉시 대응
