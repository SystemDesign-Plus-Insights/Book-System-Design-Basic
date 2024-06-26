> 계략적 규모 추정을 효과적으로 해 내려면 규모 확장성을 표현하는 데 필요한 기본기에 능숙해야 한다. 2의 제곱수나 응답지연(latency) 값, 그리고 가용성에 관계된 수치들을 기본적으로 잘 이해하고 있어야 한다.

---

### 응답지연 값

1. 메모리와 디스크의 속도 차이
    - 메모리 : 데이터 접근 속도가 빠르며, 나노초(ns) 단위로 측정됨
    - HDD : 데이터 접근 속도가 느리며, 디스크 탐색(seek) 시간이 밀리초(ms) 단위로 측정됨
2. 디스크 탐색(Seek) 피하기
    - 디스크 탐색 : 디스크에서 특정 데이터를 찾기 위해 Head가 이동하는 과정으로, 시간 소모가 큼
3. 압축 알고리즘
    - 단순한 압축 알고리즘 : 데이터 크기를 줄여 전송 및 전송 효율성을 높이는 데 사용됨.
        
        ex) Run-Length Encoding(RLE)
        
    - 데이터 전송 전 압축 : 네트워크 대역폭을 절약하고 전송 속도를 높이기 위해 데이터 전송 전에 압축. ex) gzip
4. 데이터 센터의 지역 분산
    - 데이터 전송 지연 : 지역 간 데이터 전송 시 latency가 발생할 수 있음

---

### 가용성에 관계된 수치들

1. High Availability (HA)
    - 지속적 운영 : 시스템이 오랜 시간 동안 중단 없이 운영될 수 있는 능력
    - 구현 방법 : 중복 시스템, 자동 장애 조치(failover), 로드 밸런싱 등
2. Service Level Agreement(SLA)
    - 정의 : 서비스 사업자와 고객 사이에 맺어진 서비스 제공에 대한 공식적인 합의
    - 가용시간(Uptime) : SLA에는 서비스가 제공될 것으로 기대되는 시간 비율이 명시된다. ex) 99.9% 가용성은 8.76시간의 다운 타임을 의미한다.

---

### 트위터 QPS와 저장소 요구량 추정

**가정**

1. **MAU (Monthly Active Users)**: 3억명.
2. **DAU (Daily Active Users)**: 전체 사용자의 50%가 매일 사용. (1.5억명)
3. **트윗 빈도**: 각 사용자는 하루에 평균 2개의 트윗을 작성.
4. **미디어 포함 트윗**: 전체 트윗의 10%가 미디어를 포함.
5. **데이터 보관 기간**: 5년.

**QPS (Queries Per Second) 추정**

1. **DAU 계산**: 3억 x 50% = 1.5억
2. **QPS 계산**: 1.5억 x 2트윗 / 24시간 / 3600초 ≈ 3500
3. **최대 QPS (Peak QPS)**: 2 x QPS ≈ 7000

### 규모 측정 문제에서 주로 다루는 요소들

- QPS : 초당 처리할 수 있는 쿼리 수
- 최대 QPS : 피크 시간대의 초당 최대 쿼리 수
- 저장소 요구량 : 데이터 보관에 필요한 저장소 크기
- 캐시 요구량 : 효율적인 데이터 접근을 위한 캐시 크기
- 서버 수 : 요구되는 성능을 제공하기 위해 필요한 서버의 수