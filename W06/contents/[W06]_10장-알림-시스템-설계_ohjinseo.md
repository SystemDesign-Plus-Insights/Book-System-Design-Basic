# 1. 문제 이해 및 설계 범위

- 푸시 알림, SMS 메시지, 이메일을 지원
- 연성 실시간(soft real-time) 시스템
- ios 단말, 안드로이드, 데스크톱 지원
- 클라이언트, 서버 측이 알림 생성
- 사용자는 알림을 받지 않도록 설정 가능해야 함

# 2. 개략적 설계안 제시 및 동의 구하기

## 알림 유형별 지원 방안

### iOS 푸시 알림

iOS에서 푸시 알림을 보내기 위해서는 3가지 컴포넌트가 필요하다.

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/c37cd7b7-50bc-42c4-887e-42b11ca3fd0b)

1. 알림 제공자(provider)
    - 알림 요청(notification request)을 만들어 애플 푸시 알림 서비스(APNS)로 보내는 주체다.
    - 알림 요청을 만들기 위해 다음 데이터가 필요하다.
        - 단말 토큰(device token): 푸시 알림을 수신하는 iOS 디바이스의 고유한 토큰
        - 페이로드(payload): 알림 내용을 담는 JSON 딕셔너리다.
2. APNS(Apple Push Notification Service)
    - 애플이 제공하는 푸시 알림 전송 서비스다.
    - 푸시 알림을 iOS 장치로 보내는 역할을 담당한다.
3. iOS 단말(iOS device)
    - 푸시 알림을 수신하는 사용자 단말이다.

### 안드로이드 푸시 알림

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/b3ab7d09-9483-43aa-8b70-41ad53910cb8)

안드로이드 푸시 알림도 비슷한 절차로 전송되며, APNS 대신 FCM(Firebase Cloud Messaging)을 사용한다는 점만 다르다.

### SMS 메시지

보통 트월리오(Twilio), 넥스모(Nexmo) 같은 서드 파티 서비스를 많이 이용한다.

## 연락처 정보 수집 절차

알림을 보내려면 모바일 단말 토큰, 전화번호, 이메일 주소 등의 정보가 필요하다. 사용자가 앱을 설치하거나 처음 계정 등록을 하면 API 서버는 해당 유저 정보를 수집하여 DB에 저장한다.

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/714eb706-7a7e-4534-bafa-8b231c7eaa5f)

## 알람 전송 및 수신 절차

### 개략적 설계안(초안)

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/2a746ca8-bbe2-4436-9e0e-a61ac3af5285)

- 1부터 N까지 서비스
    - 서비스 각각이 MSA일 수도 있고, 크론잡(cronjob)일 수도 있고, 분산 시스템 컴포넌트일 수도 있다.
- 알림 시스템
    - 1개 서버라 가정하면, 서비스 1~N에 **알림 전송을 위한 API**를 제공해야 한다.
    - 서드 파티에 전달할 **알람 페이로드(payload)**를 만들어 낼 수 있어야 한다.
- 서드 파티 서비스
    - 사용자에게 알림을 실제로 전달하는 역할을 한다.
    - “확장성”을 유의해야 한다. 쉽게 추가/삭제할 수 있어야 한다.

이 설계에는 몇 가지 문제가 있다.

- 알림 시스템 서버 SPOF
- 규모 확장성
- 성능 병목

### 개략적 설계안 (개선된 버전)

- DB와 캐시를 알림 시스템의 주 서버에서 분리한다.
- 알림 서버를 증설하고 자동으로 scale-out될 수 있도록 한다.
- 메시지 큐를 이용해 시스템 컴포넌트 사이 강한 결합을 끊는다.

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/d7c95f9f-175a-411c-8a37-06077aec560a)


- 알림 서버의 기능
    - 알림 전송 API: 스팸 방지를 위해 사내 서비스 또는 인증된 클라이언트만 이용 가능하다.
    - 알림 검증: 이메일 주소, 전화번호 등 기본적 검증 수행
    - DB 또는 캐시 질의: 알림에 포함시킬 데이터를 가져오는 기능
    - 알림 전송: 알림 데이터를 메시지 큐에 넣는다. 하나 이상의 메시지 큐를 사용하므로 알림을 병렬적으로 처리할 수 있다.

<aside>
💡 메시지 큐 도입(시스템 컴포넌트 사이 결합도를 낮춤)
- 동기 → 비동기
- 오류 시 재시도 기능 도입
- 다량의 알림 전송 시 버퍼 역할

</aside>

# 상세 설계

## 안정성

분산 환경에서 안정성 확보를 위한 아래 몇 가지를 반드시 고려해야 한다.

### 데이터 손실 방지

알림 전송 시스템에서 가장 중요한 요구사항 하나는 **어떤 상황에서도 알림이 소실되면 안 된다는 것**이다. 알림이 지연되거나 순서가 틀려도 괜찮지만, 사라지면 곤란하다. 

이 요구사항을 만족하려면 **알림 데이터를 DB에 보관하고 재시도 매커니즘을 구현**해야 한다. 

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/c96e1fd3-c46b-42ce-9389-6a734bc11203)

### 알림 중복 전송 방지

분산 시스템에서 알림 중복을 완전히 막는 것은 가능하지 않다. 그 빈도를 줄이려면 중복 탐지 메커니즘을 도입하고, 오류를 신중하게 처리해야 한다.

보내야 할 알림이 도착하면 그 이벤트 ID를 검사하여 이전에 본 적이 있는 이벤트인지 살핀다.

## 추가로 필요한 컴포넌트 및 고려사항

### 알림 템플릿

템플릿을 사용해 알림 형식을 일관성 있게 유지한다.

### 알림 설정

알림 설정 테이블에 보관되며, 이 테이블은 다음과 같은 필드가 필요할 것이다.

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/38cad388-ebae-42d7-a112-18c5ca7b3f80)

이와 같은 설정을 도입한 뒤에는 특정 종류 알림을 보내기 전에 반드시 해당 사용자가 해당 알림을 켜 두었는지 확인해야 한다.

### 전송률 제한

### 재시도 방법

서드파티 서비스가 알림 전송에 실패하면, 해당 알림을 재시도 전용 큐에 넣는다.

### 푸시 알림과 보안

iOS와 안드로이드 앱의 경우, 알림 전송 API는 appKey와 appSecret을 사용하여 보안을 유지한다. 따라서 인증된, 혹은 승인된 클라이언트만 해당 API를 사용하여 알림을 보낼 수 있다.

### 큐 모니터링

알림 시스템 모니터링에서 중요한 메트릭은 큐에 쌓인 알림의 개수이다. 이 수가 크면 서버들이 이벤트를 빠르게  처리하고 있지 못하다는 뜻으로, 작업 서버를 증설하는 게 바람직하다.

### 이벤트 추적

알림 확인율, 클릭율, 실제 앱 사용으로 이어지는 비율

## 수정된 설계안

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/dced9ae0-90c6-4801-9009-d280608fa9d7)