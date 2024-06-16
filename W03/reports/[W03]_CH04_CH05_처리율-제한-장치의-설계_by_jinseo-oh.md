- 발표자 : 김수경
- 서기 : 오진서

# Summary

### rate limit 알고리즘

- 토큰 버킷 알고리즘
- 누출 버킷 알고리즘
- 고정 윈도 카운터 알고리즘
- 이동 윈도 로깅 알고리즘

# Insight

Throttling : 기기의 전압을 낮추는 기능

Q. Lock 사용시 왜 성능이 저하? → 동시성이 낮아짐

Q. Lock에 의한 성능 저하 최소화하는 방안

- 원자성을 보장하는 선에서 T를 가능한 짧게 정의

> 이미 시스템에 API Gateway가 포함되있다면 rate limiter 또한 포함시켜야 할 수도 있다.
> 

⇒ 실제 필요에 의해 기술을 도입해야 함. 

# Debate

### 실무에서는 어떻게 알고리즘을 써서 적용할까?

- rate limiter 사용 이유는 dos공격, 해킹 때문

### rate limiter를 두는 위치 (치윤)

- 클라이언트 : 서버에 도달하는 트래픽을 줄일 수 있지만, 클라이언트 요청은 쉽게 위변조가 가능해지므로 고려X

만약 Spring Boot 환경이라면

- 서버 : 서버 애플리케이션 로직에 두는 것
- 미들웨어 : 서버와 클라이언트 사이 중간 레이어 (인터셉터/필터 or 서드파티 미들웨어)

### **미들웨어란? (노아)**

- 서버와 클라이언트 사이에 있는 요청에 관한 공통적인 로직을 처리해주는 Layer.
- 여러 애플리케이션 간의 요청을 다루며, BS와는 별도로 동작
- 요청의 처리율을 관리하는 알고리즘을 포함할 수 있음.

### rate limit 알고리즘 (치윤)

- 토큰, 누출 알고리즘은 기본적으로 정적이다. 버킷 크기를 동적으로 변경할 수 없기 때문에 정적인 특성을 가지므로 크기를 정하는게 힘들다.
- 이동 윈도와 고정 윈도 로깅 알고리즘은 시간 단위로 커스텀할 수 있어 보다 동적인 제어가 가능하다.
- Redis와 같은 중앙 저장소를 사용하여 분산된 서버 환경에서도 일관된 rate limiting 알고리즘을 구현할 수 있다.

### 실무에서는 어떤 상황 어떤 알고리즘? (수경)

- API를 넘겨주는 상황, 일일 처리량 1000개를 관리하는 상황이라면 토큰 버킷 알고리즘을 쓴게 아니었나
- Dynamo를 사용하여 rate liming 상태를 중앙에서 관리, 토큰 버킷 알고리즘을 통해 모든 API 요청을 단일 버킷(하나의 기업)으로 관리하는 것 같음
- 누출 버킷 알고리즘일 수도 있지 않나

### 고정 윈도, 이동 윈도 메모리 효율 차이가 나는 이유? (치윤)

- 이동 윈도는 요청마다 로그를 계속 저장해야 하므로 메모리 사용량 증가함

### 버린 요청은 어떻게? (치윤)

메시지 큐를 사용하여 버린 요청을 저장하고, 버킷에 여유가 생기면 다시 처리

- 버린 요청들을 메시지 큐에 넣어서 버킷 여유가 있을 때 요청을 처리할 수 있다.
- Rabbit MQ, Kafka
- 배민은 주문이 몰렸을 때, 은행은 이체가 몰린다면? ⇒ 메시지 큐에 요청을 임시 저장하고, 시스템이 처리할 수 있을 때 요청을 처리

### 로그를 왜 Redis에 저장할까? (수경)

- 정렬 집합(sorted set)에 요청 로그를 타임 스탬프와 함께 저장하고, 시간에 따라 정렬할 수 있음
- TTL기능을 제공하여, 요청 로그를 자동으로 만료시킬 수 있음.
    - 만료된건 지우고, 요청 reject된건 메시지 큐에 넣는다

# ref

[https://blogs.halodoc.io/taming-the-traffic-redis-and-lua-powered-sliding-window-rate-limiter-in-action/#:~:text=Sliding Window Algorithm as a,a smoother distribution of requests](https://blogs.halodoc.io/taming-the-traffic-redis-and-lua-powered-sliding-window-rate-limiter-in-action/#:~:text=Sliding%20Window%20Algorithm%20as%20a,a%20smoother%20distribution%20of%20requests)

https://github.com/Spring-Lab-s-Class/Sliding-Window-Log-Algorithm/blob/main/src/main/java/com/systemdesign/slidingwindowlog/service/SlidingWindowLogService.java

[Redis Lua Script를 이용해서 API Rate Limiter개발](https://dev.gmarket.com/69)

[고 처리량 분산 비율 제한기](https://engineering.linecorp.com/ko/blog/high-throughput-distributed-rate-limiter)

# 좀 더 논의 해봐야될 내용
그림 4-10 - reject된 요청 어떻게 처리되는지