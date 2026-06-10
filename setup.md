# 설치 관련 내용

## 네트워크 관련
> 중요 : 내 Host Ip 대역대 와 VM Ip 대역이 겹치면 안된다.
- virtualbox는 Host-Only Network라는 내 Host에서만 접근 가능한 네트워크 망을 만들 수 있다.
  - ex) `master.vm.network "private_network" ip:"192.168.56.30"
- virtualbox에서 기본적으로 만들어주는 네트워크 또한 존재하며, **"NAT"**라 부르며  VM을 **외부 인터넷이랑 연결** 시켜준다.
  - Mac의 경우 NAT이 아닌 "Shared Network"를 사용함

## 자원
- Memory : 서로 할당된 공간을 침범하면 안되기에, 메모리 할당 시 Host에서 할당된 값 만큼 VM을 끄지 않는 이상 점유된다.
- CPU :  필요로 하는 순간에 서로 나눠 쓰는 자원이므로 가진 전체를 할당해줘도 필요에 맞춰 동적으로 나눠서 사용
  - 둘 다 CPU가 필요한 상황이라면 이 전체 core 자원을 나눠 사용
  - k8s는 최소 2 이상의 CPU를 요구한다.


## swap 비활성화 이유
- **메모리 관리 일관성**: Kubernetes는 노드의 물리적 메모리만을 기준으로 리소스를 할당하며, 스왑 메모리를 포함하지 않음, 스왑이 활성화된 경우 메모리 부족 상황을 정확히 감지하지 못할 경우가 생길 수 있기 때문이다.
- **안정성**: 스왑 사용으로 인해 컨테이너의 메모리 사용량이 증가하면, Kubernetes가 이를 적절히 관리하지 못할 수 있으며, 이는 전체 클러스터의 안정성에 부정적인 영향을 미칠 수 있다.


## iptables 세팅
### cat /etc/modules-load.d/k8s.conf
```text
overlay         # 컨테이너 런타임(containerd 등..)이 사용하는 OverlayFS 지원, 컨테이너 이미지 레이어를 효율적으로 관리
br_netfilter    # 리눅스 브리지 트래픽을 iptables가 처리할 수 있게 함, Kubernetes Pod 네트워크(CNI)가 정상 동작하는 데 필수 설정
```

### cat /etc/sysctl.d/k8s.conf
```text
net.bridge.bridge-nf-call-iptables = 1     # 브리지 네트워크를 통과하는 패킷도 iptables 규칙 적용 - Pod ↔ Pod 파드 간 통신 제어 가능
net.bridge.bridge-nf-call-ip6tables = 1    # IPv6 환경에서 위와 동일한 역할
net.ipv4.ip_forward = 1                    # 패킷 포워딩 허용 - Pod ↔ 외부 네트워크 통신 허용 시 필요
```

## 각각의 역할
- **kubeadm** :	클러스터 생성
- **kubelet** :	노드에서 실제 Pod 실행
- **kubectl** :	사용자/관리자가 클러스터를 제어(운영/관리)하는 CLI

## kubeadm
> kubeadm만으로는 쿠버네티스 클러스터가 동작하지 않기 때문에 kubelet, kubectl까지 따로 각각 설치가 필요하다.
> - 서로 의존하는 구조
> - 운영 표준 패턴 : “kubeadm + kubelet + kubectl 세트”
> - Kubernetes는 버전 호환이 엄격하므로 3개의 **버전은 항상 맞춰 주자**
- Kubernetes 클러스터를 손쉽게 설치하고 초기화하기 위한 **공식 부트스트래핑(bootstrap) 도구**
- 여러 서버에 Kubernetes를 구성할 때 컨트롤 플레인(Control Plane)과 워커 노드(Worker Node)를 자동으로 설정해 주는 도구

### 주요 기능
- Kubernetes 컨트롤 플레인 초기화
- 클러스터 인증서 생성 및 관리
- 워커 노드 조인(join) 토큰 생성
- 클러스터 업그레이드 지원
- 고가용성(HA) 클러스터 구축 지원


## kubelet이 하는 일
> cgroup 설정 중요 이유는 "상태 보고" 때문이다.
- Pod 생성 
  - 생성 / 삭제 / 재시작
- 상태 보고 (API Server에 보고)
  - Node(서버) 상태
  - CPU 사용량
  - 메모리 사용량
- 컨테이너 런타임 제어
  - Pod 실행 / 종료
  - 이미지 다운로드
- Health Check

## 컨테이너 런타임 (CRI활성화)
> Kubernetes에서는 kubelet과 컨테이너 런타임이 **동일한** cgroup driver를 사용해야 한다.
> - systemd 기반 OS(RHEL, Rocky, AlmaLinux, CentOS, Ubuntu 등)에서는 systemd 사용을 권장한다.

- containerd, kubelet(configmap) , kubelet 모두 CGroup이 true 이며 systemd를 사용하도록 한다.
  - **containerd** :
    - 명령어 : `cat /etc/containerd/config.toml`
    - 확인 : `SystemdCgroup = true`
  - **kubelet ConfigMap** :  (목적: 클러스터 전체 kubelet 설정 관리)
    - 명령어 : `kubectl get -n kube-system cm kubelet-config -o yaml`
    - 확인 : `cgroupDriver: systemd`
  - **kubelet cgroup** : (목적: kubelet 실행 시 설정 적용)
    - 명령어 : `cat /var/lib/kubelet/config.yaml`
    - 확인 : `cgroupDriver : systemd`