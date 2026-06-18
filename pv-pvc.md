# PV(PersistentVolume) / PVC(PersistentVolumeClaim)

## 0 . 참고사항
> kubelet이 등록될 때 hostname 기반으로 생성된다. ex) `kubernetes.io/hostname=<node-name>`
> - Node(MasterNode) :  `kubernetes.io/hostname:k8s-master`
> - Node(WorkerNode1) :  `kubernetes.io/hostname:k8s-water1`
> - Node(WorkerNode2) :  `kubernetes.io/hostname:k8s-water2`

## 1. 필수 값
> PV/PVC 둘다 각각 서로 갖는 필수 값이 있다.
> - PVC가 요청한 용량보다 PV가 커야한다. ( PV는 실제 물리적 용량이기 떄문에 PVC는 더 많은 용량을 요청할 수 없음 )
>   - PVC에서 올바른 Selector로 PV의 labels를 지정해도 연결이 안된다.

- PVC의 요청 용량(resources.requests.storage)은 PV의 capacity.storage **이하이어야 한다**. (PV가 더 크거나 같아야 바인딩 가능)

### 1-1. PV
- capacity : PV가 제공하는 저장소 용량
- accessModes : PV가 지원하는 접근 방식

### 1-2. PVC
- resources : 사용자가 요청하는 저장소 크기
- accessModes : 디스크가 물리적으로 지원할 수 있는 접근 설정.

## 2. PV(PersistentVolume) 
- nodeAffinity : 
  - 해당 PV가 **어떤 노드에 존재하는 저장소인지**를 나타냄.
  - Local PV 사용 시 **필수적으로 설정**
  - 로컬일 경우 `matchExpressions`를 사용하여 지정 : `{key: kubernetes.io/hostname, operator: In, values: [k8s-master]}`
- **Local PV를 사용하는 Pod**는 PV의 **nodeAffinity 조건을 만족하는 노드**에만 생성된다.

### 2-1. hostPath 기능
> hostPath는 노드의 로컬 파일 시스템을 Pod에 직접 연결하는 기능이다. (ReadOnly 권장)
> - 노드 종속성이 생기므로 **운영 환경에서는 사용을 최소화**하고 PVC + 네트워크 스토리지(NFS, Ceph 등)를 권장
#### 사용 용도
- 로그 수집
- 테스트
- Docker Socket 공유
- 설정 파일 공유
- 디버깅
- Device Mount
    
```yaml
volumes:
    - name: files
      hostPath:
        path: /root/k8s-local-volume/1231
nodeSelector:
  kubernetes.io/hostname: k8s-master
```