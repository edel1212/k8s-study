# ConfigMap, Secret

## 1. ConfigMap (일반 설정 관리)
> 애플리케이션이 사용하는 **비민감성 환경 변수나 설정 파일**을 저장하고 주입하는 Object

## 1-1. 핵심 메커니즘 및 주의점
- **애플리케이션 준비**: App 내부(application.yml 등)에서 **환경 변수를 받아 처리할 수 있도록 구조가 잡혀** 있어야 함.
- **보안 리스크**: **암호화되지 않은 평문**이므로 **민감한 정보(비밀번호, API 키)는 넣지 않는 것**을 권장.
  - Pod 내부에서 env 명령어로 **상시 확인 가능**.
  - 파이프라인이나 애플리케이션 기동 로그(stdout)에 그대로 노출될 위험.
- ✅ 업데이트 반영 시점 : 주입 방식에 따라 다르게 동작.
  - 환경 변수 (env, envFrom) 주입: 
    - Pod 기동 시점에 **딱 한 번** 프로세스에 주입 
    - ConfigMap을 변경해도 **Pod를 재시작하기 전에는 절대 반영되지 않음**
  - 볼륨 마운트 주입: 
    - 쿠버네티스가 **파일 변경을 감시**하므로 컨테이너 내부의 파일은 실시간(1~2분 내)으로 **자동 업데이트 진행**. 
    - ✨ Spring Boot가 기동 중에 파일을 다시 읽는 로직이 없다면 마찬가지로 **메모리에는 반영되지 않음**. (JVM에 올라간건 반영 X)

#### 1-1-1. 환경 변수 (env, envFrom) 주입 방식 예시
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  # ✅ configMapRef에서 찾을 이름
  name: app-env-config
data:
  # Key-Value 형태로 저장
  SERVER_PORT: "8080"
  WELCOME_MESSAGE: "Hello Dev Team!"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-app-deployment
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: spring-app
          image: my-spring-app:1.0
          # envFrom을 쓰면 ConfigMap의 모든 Key가 OS 환경변수로 등록됨
          envFrom:
            - configMapRef:
                name: app-env-config
```

#### 1-1-2. 볼륨 마운트 (volumeMounts) 주입 방식 예시
> 만약 ConfigMap을 플랫(Flat)한 Key-Value로 적었을 경우 잘못된 형식의 파일이 생성됨
> - service.notice.banner 라는 이름의 파일 (내용: Under Maintenance)
> - service.notice.level 라는 이름의 파일 (내용: WARN)
> 그렇기에 volumes 방식 사용 시 **파일로 떨궈줘**야한다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  # ✅ volumes에서 찾을 이름
  name: app-file-config
data:
  # 파일명: 파일내용 형태로 저장
  extra-config.properties: |
    service.notice.banner=Under Maintenance
    service.notice.level=WARN
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-app-deployment
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: spring-app
          image: my-spring-app:1.0
          # 2. 볼륨 마운트 지정 (컨테이너 내부 경로)
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
      # 1. ConfigMap을 디스크(볼륨)처럼 정의
      volumes:
        - name: config-volume
          configMap:
            name: app-file-config
```

### 1-2. 아키텍처 고민: 모든 환경 설정을 ConfigMap으로 빼야 할까?
> 결론 : 아니다, 변하는 것과 변하지 않는 것을 분리해야 함.
- 유지 대상: 로깅 포맷 설정, 타임아웃 기본값 등 환경이 바뀌어도 **절대 변경될 일이 없는 애플리케이션 고유 상수**. 유지
- 추출 대상: DB 접속 URL, 프로파일, 외부 API 주소 등 배포 환경(dev/prod)에 따라 **동적으로 달라지는 설정**. 

## 2. Secret (보안 데이터 관리)
> DB 패스워드, 인증 토큰 등 민감한 중요 데이터를 관리하는 Object

### 2-1. 핵심 메커니즘 및 주의점
- stringData 속성: 
  - YAML을 작성할 때 사람이 읽을 수 있는 평문으로 작성하게 해주는 편의성 필드 (작성후 내용이 yaml로 저장).
  - 저장 시에는 알아서 Base64로 인코딩되어 data 필드에 저장. (Base64는 암호화가 아닌 단순 인코딩이므로 노출 시 **쉽게 디코딩된다**.)
- 자체 암호화 한계 해결법:
  - GitOps 외부 관리: 매니페스트를 Git에 올리지 않고 **클러스터 내에서 관리자가 직접 생성 및 관리**.
  - 가벼운 도구 활용: Sealed Secrets나 Sops 같은 오픈소스를 활용해 **Git에는 암호화된 파일로 올리고, 클러스터 배포 시에만 복호화**. 
  - 전문 솔루션 연동: 전사적 보안 관리가 필요할 땐 HashiCorp Vault 같은 **외부 서드파티와 연동**.


// TODO 타입 관련 테스트 진행 및 확인 필요
// - secret 환경 변수 처럼 쓸수 있는지
// - secret 파일 저장 형식이 아니여도 되는지

- 중요 데이터를 관리하는게 목적임
  - type이라는 속성이 있음 근데 어따쓰는건지는 모르겠   
    - docker-registry 라는 타입도 있다는데 이게 뭐
      

### 흐름
- 1 ) pod 내 volumes.mountPath 로 컨테이너 내 매핑된 볼륨 디렉토리 가 생심
- 2 ) volume 과 secret이 매칭이 되서 "1" 위치(컨테이너 내)에 파일이 생성됨
- 3 ) ConfigMap에 적용된 환경 변수 값과 매칭되어 해당 값을 읽어 옴


