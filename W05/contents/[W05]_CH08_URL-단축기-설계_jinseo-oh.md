**URL 단축기를 사용하는 이유**

- 가독성 향상 되서 마케팅에 도움
    - 기억하기 쉽고, 클릭률 증가
- 링크 클릭 데이터 트래킹 및 통계 분석

# 1. URL shortener 설계 범위

**시스템 기본기능**

1. URL 단축
2. URL redirection
3. 높은 가용성, 규모 확장성, 장애 감내

**개략적 추정**

- 매일 1억 개의 단축 URL 생성
- 쓰기 연산: 초당 1160개 생성
- 읽기 연산: 초당 11,600회 발생
- 10년간 운영한다고 가정하면 3650억개의 레코드를 보관해야 한다.
- 축약 전 URL 평균 길이가 100이라고 가정하면 10년 동안 필요한 저장 공간은 36.5TB이다.(3650억 x 100byte)

# 2. 개략적 설계안 제시

### 1. API Endpoint

URL shortener는 기본적으로 두 개의 엔드포인트를 필요로 한다.

1. 새 단축 URL 생성 엔드포인트
    - POST /api/v1/data/shorten
    - parameter: {longUrl: longURLstring}
    - response: 단축 URL
2. 원래 URL 리디렉션 엔드포인트
    - GET /api/v1/shortUrl
    - response: 원래 URL

### 2. URL Redirection

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/c4d0ae3a-c768-4329-9072-e0461648eee4)

여기서 유의할 점은 301 응답과 302 응답의 차이다.

- 301 Permanently Moved
    - 영구적으로 반환된 URL 사용
    - 브라우저는 이 응답을 캐시하여, 이후 동일한 요청에 캐시된 URL로 직접 접근한다.
    - 서버 부하를 줄일 수 있다.
- 302 Found
    - 일시적으로 반환된 URL 사용
    - 브라우저는 이후에도 원래 URL을 단축 URL 서버로 요청해야 한다.
    - 트래픽 분석을 할 수 있다.

URL Redirection을 구현할 때 가장 직관적인 방법은 해시 테이블을 사용하는 것이다.

### 3. URL 단축

URL 단축 해시 함수는 다음 요구사항을 만족해야 한다.

- 고유성
- 복원 가능성

# 3. 상세 설계

### 데이터 모델

해시 테이블은 메모리 비용이 비싸기 때문에 적합하지 않는다.

<단축URL,원래URL>의 순서쌍을 RDB에 저장하는 것이 더 나은 선택이다. 

근데 매일 1억 개의 단축 URL이 생성, 대규모 읽기 연산된다면 NoSQL이 더 적합하지 않나

- 읽기 성능: no sql은 샤딩으로 병렬 읽기 처리가 가능함
- 데이터 구조: key-value 조회가 더 쉬움
- 데이터 저장: scale out으로 분산 저장할 수 있으며, 확장성도 더 좋음

### 해시 함수

hashValue는 [0-9,a-z,A-Z]로 구성된다. 따라서 사용할 수 있는 문자 개수는 10+26+26=62개다. 

즉, hashValue 길이는 62^n ≥ 3650억인 n의 최솟값을 찾아야 한다. n이 7이면 3.5조 개의 URL을 만들 수 있다.

해시 함수 구현에 쓰일 기술로 두 가지 방법을 살펴보겠다.

1. **해시 후 충돌 해소**

잘 알려진 해시 함수로 CRC32, MD5, SHA-1을 이용하면, 가장 짧은 해시값조차 길이가 7을 넘는다. 이 문제를 해결하기 위해 처음 7개 글자만 이용하는데, 그럼 충돌 확률이 높아진다. 충돌이 발생했을 때는, 충돌이 해소될 때까지 사전에 정한 문자열을 해시값에 덧붙인다.

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/b22801a0-08ee-406b-972c-ea8411817da2)

하지만 이 방법은 단축 URL을 생성할 때 한 번 이상 DB 질의를 해야 하므로 오버헤드가 크다. (DB 대신 불룸 필터를 사용하면 성능을 높일 수 있다.)

1. **base-62 변환**

진법 변환은 수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야 하는 경우에 유용하다.

![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/1a7cf145-ee2c-4c54-9dd5-310246e068e4)

**URL 리디렉션 상세 설계**
![image](https://github.com/SystemDesign-Plus-Insights/Book-System-Design-Basic/assets/62508156/bbc3f378-a55b-44e3-994e-4c76764620ef)
