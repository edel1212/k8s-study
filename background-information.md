# Linux 와 오케스트레이컨

## 1. Linux OS 흐름
- 최초에 OS로 unix가 있었고, 한참 시간이 지나고 linux가 나왔다.
- Linux 배포반은 크게 debian이랑 redhat 계열이 있다.

### Debian
- 시간이 지나 ubuntu로 발전 했으며, 무료기에 점유율이 올랐다.

### Redhat
- 최초에는 fedora linux라고 해서 새로운 기능을 개발하는 버전 [무료]  
- fedora에서 안정화가 되면 redhat linux( RHEL이라고도 부름)로 이름을 바꿔서 릴리즈 [유료]
- redhat linux를 **복사**해서 만든 게 centOS [무료 - 2024년 종료 ] 
- fedora에서 RHEL로 넘어가기 전 테스트용 centOS Stream이 생겼지만 테스트 배포판이기에 안정적이지 않음 [무료 - 안정적이지 못함]
  - 구조 : fedora -> centOS Stream -> RHEL
- CentOS의 마이그레이션으로 Roky/Alma Linux가 탄생 [무료 - Roky가 인지도가 더 높음]


## 2. Container 흐름

### 초기(최초)의 컨테이너
- Linux가 발전해나가면서 생긴 코어기술 중 하나
- LXC (linux container) : 최초의 컨테이 (Docker 또한 해당 기술을 토대로 만들어짐)
  - chroot : 파일 격리
  - cgroup : cpy/mem/blkio(디스크 입출력)/net_cls(네트워크 트레픽) 자원 격리
  - namespace : 프로세스 격리

### 쿠버네티스에서 컨테이너 문제사항
- 시간이 지날 수록 Dokcer와의 호환성 좋지 않아 걸림돌이 되는 상황 발생
- 쿠버네티스에 사용될 인터페이스가 잘맞는 컨테이너를 더 선호하게 됨

### rkt (rocket) 컨테이너
- 과거 docker는 보안에 안 좋은 점이 좀 있었고, 이 부분을 공략을 하면서 더 안정적인 컨테이너로 나온 대체안
  - root 권한으로 설치하고 실행을 강제함 -> 현재는 rootless 설치 모드가 생겨서 보안이 강화되었음
  - 보안 강화로 다시 docker가 우위를 점령함

### cli-o
- Redhat이 인수 후 밀고 있는 컨테이너

### containerd 
- docker에서 컨테이너를 만들어주는 기능이 분리된 것
  - docker는 엄청나게 많은 기능들이 녹아진 엔진이다.
-  CNCF에 기부 되었음

### docker
- mirantis에 인수 이후 kubernetes 인터페이스를 잘 맞추려고 하고 있기에 쿠버네티스에서 docker는 빠지진 않게 됨

## 3. Container Orchestration과 Container 흐름
> K8S는 컨테이너를 하나 만달라는 명령어는 없다. pad 생성 후 그 안에 컨테이너를 하나 이상 명시 가능
### kube-apiserver
- k8s에서 보내지는 **모든 API 명령**을 여기서 받음
  - ex) pad 하나에 3개의 컨테이너 올려줘

### kubelet
- "kube-apiserver"에서 전달 받은 pad 생성 내용은 "kubelet"로 넘어옴
- pod 내용을 확인하고 컨테이너가 2개 필요한 것을 확인하고 해당 내용을 "container runtime"에게 **요청을 2번 보냄**

#### CLI (Container Runtime Interface)
> container runtime와 호출하는 Interface를 지칭
- kubelet가 containers와 소통 시 사용하는 컨테이너 종류(containerd, rkd ...)가 늘어남에 따른 **인터페이스 방식**으로 변경
  - **오픈 소스 형태**이므로 **각각의 컨테이너들이 컨트리뷰트** 하는 방식으로 개발 진행

### container runtime
- 컨테이너를 생성하는 주체

#### OCI
- 컨테이너런타임이 컨테이터를 만들때 지켜야하는 표준 규약을 관리하는 단체
  - 규약 덕분에 **모든 이미지들은 호환이 가능**하다.



