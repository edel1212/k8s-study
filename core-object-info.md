# Object 설정 (중요)

## Namespace
- k8s의 가장 큰 묶음으로 **Object들을 그루핑** 해주는 기능을 함
- 해당 name-space 기준으로 deployment를 묶는다.
- Namespace를 삭제할 경우 안에 있던 하위 모든 Object들이 삭제된다.
```yaml
apiVersion: v1
# Object의 구분
kind: Namespace
metadata:
  # name space의 이름
  name: anotherclass-123
  # name space의 라벨 설명
  labels:
    # 전체 시스템 이름 - 논리적 그룹
    part-of: k8s-anotherclass
    # 생성된 곳 (optional)
    managed-by: dashboard
```

## Deployment (비유 : 매니저)
> pod(비유 : 직원)를 만들고 업그레이드를 해주는 역할
- `metadata.namespace`에 적힌 값을 기준으로 만들어둔 Namespace에 소속됨 
- Namespace에서 Deployment의 `metadata.name`은 **중복❌**
  - 쿠버네티스에서 네임스페이스(Namespace)는 일종의 **'논리적으로 분리된 방'**
  - 하나의 네임스페이스 안에서 Deployment의 name은 고유한 식별자
  - **다른 종류**의 Object는 **이름 중복 🙆**
- Deployment의 인스턴스명이 중복 ❌
  - labels에서의 중복은 시스템적인 에러는 나지 않음 (라벨은 단순 태그에 불과하기 때문)
    - 단 중복 없이 진행해주는 것을 권장
  - [☠️ 진짜 문제 ] `spec.selector`의 인스턴스가 중복일 경우 
    - 디플로이먼트는 자신이 띄운 pod를 기억하지 못함
    - 오직 클러스터 전체에서 `instance: api-tester-1231`이라는 명찰을 달고 있는 pod를 세어서 행동하기에 무한 경쟁이 일어남
- template를 찾을 때 `spec.selector`가 찾는 기준의 개수에만 맞으면된다.
  - template에 나열된 라벨의 정보가 더 많아도 문제가 되지 않음

```yaml
apiVersion: apps/v1
# Object 종류
kind: Deployment
metadata:
  # 그룹으로 묶일 Namespace 지정
  namespace: anotherclass-123
  # Deployment 이름 지정 (namespace 내 중복 이름 ❌)
  name: api-tester-1231
  # 라벨 설정 (kubectl 조회 시 용이함)
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    # 인스턴스명 (중복 ❌)
    instance: api-tester-1231
    version: 1.0.0
    managed-by: dashboard
    
# 스팩 설정    
spec:
  # (중요) Deployment가 관리할 Pod를 찾는 기준표 (아래 template의 labels와 일치해야 함)
  # - ✅ Selector와 Template의 라벨을 동일하게 가는것이 중요
  selector:
    matchLabels:
      part-of: k8s-anotherclass
      component: backend-server
      name: api-tester
      instance: api-tester-1231
      
  # 항상 유지할 Pod의 개수  
  replicas: 2
  
  # (핵심 기능) 배포 업데이트 방식 (RollingUpdate: 무중단 순차 배포)
  strategy:
    type: RollingUpdate
    
  # 실제 생성될 Pod의 붕어빵 틀 (템플릿)
  template:
    metadata:
      # 생성될 Pod에 붙일 명찰 (이 명찰을 보고 Deployment가 찾아옴)
      labels:
        part-of: k8s-anotherclass
        component: backend-server
        name: api-tester
        instance: api-tester-1231
        version: 1.0.0
        
    # Pod 내부 스펙 정보    
    spec:
      # Pod를 띄울 특정 노드(서버)를 지정할 때 사용
      nodeSelector:
        kubernetes.io/hostname: k8s-master
        
      # Pod 내부에서 돌아갈 컨테이너들 (여러 개 가능)    
      containers:
        - name: api-tester-1231
          # 실행할 도커 이미지
          image: 1pro/api-tester:v1.0.0
          ports:
          - name: http
            containerPort: 8080
            
          # 환경 변수
          envFrom:
            # ConfigMap에 정의된 변수들을 통째로 가져와서 주입
            - configMapRef:
                name: api-tester-1231-properties
                
          # [1단계] 앱 기동 확인: 정상 동작(200 OK) 전까지 트래픽 및 다른 Probe 대기. 실패 시 재기동
          startupProbe:
            httpGet:
              path: "/startup"
              port: 8080
            periodSeconds: 5
            failureThreshold: 36
            
          # [2단계] 서비스 투입 확인: 정상일 때만 유저 트래픽(요청)을 이 Pod로 보내줌. 실패 시 트래픽 차단
          readinessProbe:
            httpGet:
              path: "/readiness"
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
            
          # [3단계] 생존 유지 확인: 운영 중 앱이 뻗으면(데드락 등) Pod를 강제 재시작시켜 버림
          livenessProbe:
            httpGet:
              path: "/liveness"
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
            
          # 컨테이너 자원 할당량 설정
          resources:
            # 보장(최소) 자원: 이만큼 여유가 있는 서버에만 Pod를 배치함
            requests:
              memory: "100Mi"
              cpu: "100m"
            # 제한(최대) 자원: 이 이상 쓰려고 하면 강제 종료(OOMKilled) 시킴
            limits:
              memory: "200Mi"
              cpu: "200m"
              
          # 아래 'volumes'에서 정의한 볼륨을 컨테이너 내부의 실제 폴더 경로에 연결(Mount)
          volumeMounts:
            # volumes와 매핑될 이름
            - name: files
              # 실제 pod 내 생성될 마운트 경로
              mountPath: /usr/src/myapp/files/dev
            - name: secret-datasource
              mountPath: /usr/src/myapp/datasource
              
      # Pod에서 사용할 디스크(볼륨) 정의
      volumes:
        - name: files
          # PVC( Persistent Volume Claim)
          # 영구적으로 보존되어야 할 저장소(Volume)를 요청(Claim)하는 속성 
          persistentVolumeClaim:
            claimName: api-tester-1231-files
        - name: secret-datasource
          # 민감한 정보(Secret)를 안전하게 Pod에 주입하는 속성
          secret:
            secretName: api-tester-1231-postgresql
```

## Service
> Pod한테 트래픽을 연결해주는 역할을 함

```yaml
apiVersion: v1
# Object 종류
kind: Service
# 메타 데이터
metadata:
  # 묶일 namespace
  namespace: anotherclass-123
  # 서비스의 이름 (오브젝트 종류가 다르므로 디플로이먼트 이름과 중복 가능 ➔ 관리상 통일 권장)
  name: api-tester-1231
  # 라벨 정보
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231
    version: 1.0.0
    managed-by: dashboard

# 스팩 정보    
spec:
  # (중요) 이 서비스가 트래픽을 전달할 목적지 Pod들의 명찰 조건을 지정
  # 디플로이먼트의 template.metadata.labels에 적힌 명찰들과 일치해야 함
  selector:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231
  
  # 트래픽을 연결해줄 포트  
  ports:
    # 1. 클러스터 내부에서 이 서비스를 호출할 때 쓰는 포트
    - port: 80
      
      # 2. 목적지 Pod 내부에서 앱이 실제로 구동 중인 포트 (디플로이먼트에 정의한 포트 이름 'http' = 8080을 가리킴)
      targetPort: http

      # 3. 실제 외부 사용자가 노드 IP를 치고 들어올 포트 번호 (30000~32767 사이 지정)
      nodePort: 31231
      
  # 외부에 서비스를 노출하는 방식 지정
  # - ClusterIP (기본값): 클러스터 내부에서만 연결하는 내부 전용 포트 (외부 접속 불가)
  # - NodePort: 서버 IP와 지정한 포트를 통해 외부에서 직접 접속 가능 (지금 선택하신 것)
  # - LoadBalancer: 클라우드(AWS, GCP 등) 환경에서 실제 공인 IP(External IP)를 받아와 외부로 노출할 때 사용
  type: NodePort
```

## Configmap, Secret
> 설정 데이터 : ConfigMap // 민감 데이터 : Secret  
- Secret 와 Configmap에 설정된 name은 Deployment의 template 하위 volume에서 불러와 사용중
  - ConfigMap : claimName
  - Secret : secretName

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: anotherclass-123
  name: api-tester-1231-properties
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231
    version: 1.0.0
    managed-by: dashboard

# 환경 데이터로 들어갈 값    
data:
  spring_profiles_active: "dev"
  application_role: "ALL"
  postgresql_filepath: "/usr/src/myapp/datasource/postgresql-info.yaml"
---
apiVersion: v1
kind: Secret
metadata:
  namespace: anotherclass-123
  name: api-tester-1231-postgresql
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231
    version: 1.0.0
    managed-by: dashboard

# 지정된 이름 기준으로 파일이 생성된다.    
stringData:
  # 디스크에 마운트될 때 만들어질 실제 파일 이름
  postgresql-info.yaml: |
    driver-class-name: "org.postgresql.Driver"
    url: "jdbc:postgresql://postgresql:5431"
    username: "dev"
    password: "dev123"
```

## PVC, PV
> PersistentVolume에는 namespace가 없다.
> - 그 이유는 namespace 보다 더큰 개념인 cluster에 묶여 있기 때문이다. (cluster -> "namespace , PersistentVolume" 공간 구조)
> - PV : PVC ➔ 무조건 1 : 1 관계

```text
[물리 디스크]           [요청서]             [실제 실행될 앱들]
   
   +----+              +-----+              +-----------+
   | PV |  (1:1 매칭)   | PVC |   (1:N 공유)  | Pod (앱1) |
   |    | <==========> |     |   <----------+-----------+
   +----+              +-----+              | Pod (앱2) |
                                            +-----------+
                                            | Pod (앱3) |
                                            +-----------+
```

- **PersistentVolume (PV / 퍼시스턴트 볼륨)**:
  - 시스템 관리자가 미리 **준비해 둔 '실제 외장 하드(예시 - 토지)' 자원**
  - "우리 서버 인프라에 2GB짜리 하드디스크가 준비되어 있어"라고 선언하는 물리적인 자원 정보
- **PersistentVolumeClaim (PVC / 퍼시스턴트 볼륨 클레임)**: 
  - template에서 보낸 **'요청서(에시 - 임대 계약서)'** 
  - Pod에 2GB 공간이 필요하니까 조건에 맞는 외장 하드 하나만 연결을 쿠버네티스에게 요청 하는 것.

### 부동산 예시
- **PV** = 실제 존재하는 '단 하나의 전세방'
- **PVC** = 그 방을 빌리기 위한 '임대 계약서'
- **Pod** = 그 방에 들어가 살 '세입자(사람)'

```yaml
apiVersion: v1
# 오브젝트 종류가 '저장소 요청서(PVC)'임을 선언
kind: PersistentVolumeClaim
metadata:
  namespace: anotherclass-123
  name: api-tester-1231-files
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231
    version: 1.0.0
    managed-by: kubectl
spec:
  resources:
    requests:
      # "최소 2 기가바이트(2G) 공간이 필요해" 라고 신청
      storage: 2G
      
  # 저장소에 접근하는 방식을 지정합니다.    
  accessModes:
    # "여러 서버(노드)에 뜬 pod들이 동시에 읽고 쓸 수 있도록 요청" (RWX 모드) 
    - ReadWriteMany

  # 대기 중인 PV들 중 어떤 것을 고를지 기준을 정합니다.
  selector:
    matchLabels:
      part-of: k8s-anotherclass
      component: backend-server
      name: api-tester
      instance: api-tester-1231-files
---
apiVersion: v1
# 오브젝트 종류가 '실제 물리 저장 공간(PV)'임을 선언
kind: PersistentVolume
metadata:
  name: api-tester-1231-files
  # PVC가 찾아올 수 있도록 작성된 라벨.
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231-files
    version: 1.0.0
    managed-by: dashboard
spec:
  capacity:
    storage: 2G
  # 디스크를 통째로 쓰지 않고, 일반적인 폴더/파일 구조(Filesystem)로 포맷해서 제공합니다.
  volumeMode: Filesystem
  # 디스크가 물리적으로 지원할 수 있는 접근 설정.
  accessModes:
    - ReadWriteMany
    
  # nodeAffinity에 지정된 볼륨으로 사용할 노드 디렉토리 패스
  # - 사전에 해당 경로의 디렉토리가 없을 경우 믄제가 될 수 있으니 사전 생성 필요
  local:
    path: "/root/k8s-local-volume/1231"
    
  # 대상 노드 지정  
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - {key: kubernetes.io/hostname, operator: In, values: [k8s-master]}
```

## HPA (HorizontalPodAutoscaler)
> 부하에 따라 Pod를 늘려주고 줄여주는 역할을 함

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  namespace: anotherclass-123
  name: api-tester-1231-default
  labels:
    part-of: k8s-anotherclass
    component: backend-server
    name: api-tester
    instance: api-tester-1231
    version: 1.0.0
    managed-by: dashboard
    
spec:
  # [1] 스케일링을 적용할 대상 타겟 지정 (디플로이먼트의 셀렉터 역할을 합니다.)
  scaleTargetRef:
    apiVersion: apps/v1
    # 대상 Object는 Deployment
    kind: Deployment
    # 대상 Deployment의 name
    name: api-tester-1231
    
  # [2] 자동 확장할 Pod 개수의 하한선과 상한선 지정
  minReplicas: 2
  maxReplicas: 4

  # [3] 스케일링을 유발할 기준(메트릭) 설정
  metrics:
    - type: Resource
      resource:
        # CPU 사용량을 기준
        name: cpu
        target:
          # 퍼센트(%) 방식으로 계산
          type: Utilization
          # 대상 디플로이먼트 'requests.cpu'의 60%를 넘으면 스케일 아웃 시작
          averageUtilization: 60
          
  # 스케일링 속도 조절 브레이크        
  behavior:
    # 트래픽이 튀어도 즉시 늘리지 말고, 최근 120초 동안의 추이를 보며 올릴 수 있도록 함
    scaleUp:
      stabilizationWindowSeconds: 120
```