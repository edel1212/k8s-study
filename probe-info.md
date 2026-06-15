# Kubernetes Probe 정리
> 애플리케이션 내 Probe의 응답을 받을 엔드포인트 경로(Path)를 제공해야 한다.
> - Spring Boot 2.3 이상을 사용 시 Spring Boot Actuator를 활용 권장
> 이 설정은 `Deployment.spec.template.spec.containers` 하위에 작성된다.

- **독립된 경로 사용 권장**
    - 3개의 Probe에 같은 내부 경로를 사용해도 구동 자체는 가능하지만 권고하지 않는다.
    - 각각의 Probe 목적에 맞는 전용 경로를 구성하고 점검 로직을 분리하는 것이 좋다.
- **역할 분리 예시**
    - **startupProbe**: 애플리케이션 초기화 프로세스 완료 여부 (Spring 부트스트래핑, Jar 실행 완료 시점 등)
    - **readinessProbe**: 유저 트래픽을 받을 준비가 되었는지 여부 (내부 초기 데이터 로딩 완료 등)
    - **livenessProbe**: 애플리케이션이 살아있는지 여부 (데드락 검증 등 단순 헬스체크)

> ⚠️ **주의 사항**
> `readinessProbe`와 `livenessProbe`는 애플리케이션이 종료될 때까지 **무한 반복 실행**된다.
> - 타임아웃을 유발하지 않도록 매우 가볍고 단순한 검증 로직만 포함해야 한다. 
> - 특히 외부 DB나 서드파티 API 상태를 여기에 직접 연동하면 연쇄 장애의 원인이 될 수 있음 

## 1. startupProbe (기동 확인)
- 대상 애플리케이션이 최초 기동 중일 때, 성공 여부를 확인하기 위해 사용한다.
- 애플리케이션이 완전히 뜨기 전부터 쿠버네티스가 호출하므로, 성공 전까지는 실패 횟수가 누적된다.
- **단 한 번만 성공 기준에 도달 하면** 이후로는 다시 호출되지 않고 다음 단계 프로브로 제어권을 넘긴다.

```yaml
# [1단계] 앱 기동 확인: 정상 동작(200 OK) 전까지 트래픽 및 다른 Probe 대기. 실패 시 재기동
  startupProbe:
    # 확인할 요청 방식 및 path
    httpGet:
      path: "/startup"
      port: 8080
    # 5초 마다 요청
    periodSeconds: 5
    # 성공으로 간주할 횟수 (1회 성공 시 다음 단계로 넘어  ) 
    successThreshold: 1
    # 최대 36회 실패 허용 (총 180초 동안 기동 시간을 벌어줌)
    failureThreshold: 36
```

## 2. readinessProbe (서비스 연결 가능 여부 확인)
- startupProbe가 **성공한 이후부터 계속해서 실행**된다.
- 애플리케이션이 정상적으로 동작하여 외부 유저의 접근(Service 오브젝트를 통한 트래픽)을 **허용하거나 차단하는 역할**을 한다.
- 실패 횟수에 도달하면 **해당 Pod를 서비스 라우팅 대상에서 제외**(트래픽 차단)한다.

```yaml
  # [2단계] 서비스 투입 확인: 정상일 때만 유저 트래픽(요청)을 이 Pod로 보내줌. 실패 시 트래픽 차단
  readinessProbe:
    # 확인할 요청 방식 및 path
    httpGet:
      path: "/readiness"
      port: 8080
    periodSeconds: 10
    successThreshold: 1
    # 3회 연속 실패 시 서비스 그룹에서 일시 격리
    failureThreshold: 3 
```

## 3. livenessProbe (생존 유지 확인) 
- startupProbe가 **성공한 이후부터 계속해서 실행**된다.
- readinessProbe의 성공 여부와 상관없이 동시에 **완전히 독립적으로 실행**된다.
- 애플리케이션이 데드락 같은 **복구 불가능한 상태에 빠졌는지 주기적으로 체크**한다.
- 실패 횟수에 도달하면 애플리케이션에 문제가 있다고 판단하여 **Pod를 강제로 재시작(Restart)** 시킨다.
- ✅ 일시적인 장애일 수도 있는데 앱이 재시작되면 작업이 사라지면 문제가 될 수있다.
  - **적정한 요청 주기** 및 **실패 횟수**를 설정하는 것이 중요 (스스로 회복 할 수 있는 시간을 주는것)

```yaml
  # [3단계] 생존 유지 확인: 운영 중 앱이 뻗으면(데드락 등) Pod를 강제 재시작시켜 버림 
  readinessProbe:
    httpGet:
      path: "/liveness"
      port: 8080
    periodSeconds: 10
    successThreshold: 1
    # 3회 연속 실패 시 Pod 강제 재시작 (일시적 지연을 감안해 적절한 횟수 설정 필요)
    failureThreshold: 3
```