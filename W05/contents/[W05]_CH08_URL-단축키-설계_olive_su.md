## URL 단축키 설계

## 문제 이해 및 설계 범위 확정

> Interview Question Exmple: URL 단축키가 어떻게 동작해야하는지 예제를 보여라.

<br/>

- 트래픽 규모
  - 매일 1억개의 단축 URL 생성
- 길이
  - 짧으면 짧을 수록 좋음
- 제한 사항
  - 숫자 (0~9) / 영문자 (a~z, A~Z)
  - 삭제, 갱신 불가

<br/>

- 개략적추정
  - 쓰기 연산: 매일 1억 개의 단축 URL 생성
  - 초당 쓰기 연산: 1억(100million)/24/3600=1160
  - 읽기 연산: 읽기 연산 : 쓰기 연산 = 10:1
  - 초당 읽기 연산: 11,600 (1160 x 10 = 11,600)
  - 10년간 운영: 1억(100million) x 365 x 10 = 3650(365billion) 레코드 보관
  - 축약 전 URL 평균 길이 : 100
  - 10년간 필요 저장 용량 : 3650억(365billion)x 100바이트 = 36.5TB

<br/>

## 설계

### API 엔드포인트

1. URL 단축용 엔드포인트
   1. end-point: POST `/api/v1/data/shorten`
   2. param: long URL(String)
   3. return: short URL

<br/>

1. URL 리디렉션용 엔드포인트
   1. end-point: GET `/api/v1/:shortUrl`
   2. return: 원래 URL

<br/>

### URL 리디렉션

![[./images/01_olive_su.png]]

![[./images/02_olive_su.png]]

- ❓ 응답 코드 301 🆚 302
  - `301 Permanently Moved`: 해당 URL 에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었음을 의미
    따라서 브라우저 상에 캐시를 해두고 같은 요청이 올 시에는 캐시된 원래 URL로 요청 전송
  - `302 Found`: '일시적으로' Location 헤더가 지정하는 URL에 의해 처리
    항상 단축 URL 서버에 먼저 보내진 후 원래 URL로 리디렉션

<br/>

- 그렇다면 어떤 응답 코드를 써야하는가?
  - `301 Permanent Moved`: 서버 부하 감소 시 사용
  - `302 Found`: 클릭 발생률/발생 위치 추적 시 사용

<br/>

## URL 단축의 구현

- 해시 테이블을 사용하여 구현
  - key: 해시값
  - value: <단축 URL, 원래 URL>

<br/>

- 단축 URI - `www.tinyurl.com/{hashValue}`

<br/>

💫 입력 URL이 다른 값이면 해시 값도 달라야한다.
💫 해시값에서 역으로 URL 복원도 가능해야한다.

## URL 단축 상세 설계

### 데이터 모델

- 해시테이블이 아닌 RDB에 저장
- `id`, `shortURL`, `longURL` 컬럼

<br/>

### 해시 함수

- 문자 제한: 숫자 (0~9) / 영문자 (a~z, A~Z)
- 사용 가능 문자: 10 + 26 + 26 = 62개
- $$ 62^{n} >= 365billion $$

![[./images/03_olive_su.png]]

- $$n = 7$$
  - 3.5조 개 생성 가능 (∴ n을 7로 설정)

<br/>

### 해시 충돌 해결

- 해시 함수 사용
  - `CRC32`, `MD5`, `SHA-1` etc..

<br/>

![[./images/04_olive_su.png]]

- 충돌 발생 시에 한번 더 질의를 수행한다.

<br/>

### base-62 변환

- base-62: 사용되는 문자가 62개이기 때문

<br/>

- 차이

![[./images/05_olive_su.png]]

<br/>

### base-62 변환 기법의 플로우

![[./images/06_olive_su.png]]

1. Origin URL 입력
2. 데이터베이스에 해당 URL이 있는지 검사
   1. 데이터베이스에 존재 -> 리턴
   2. 데이터베이스에 미존재 -> 유일 id 생성(데이터베이스 기본키)
3. base-62 변환 적용, ID를 단축 URL 생성
4. ID, 단축 URL, 원래 URL로 새 데이터베이스 레코드 생성 후 -> 단축 URL 리턴

<br/>

⚠️ 전역적으로 유일한 ID를 생성해야함으로, 분산 환경에서의 구현을 주의하자.

<br/>

## URL 리디렉션 상세 설계

![[./images/06_olive_su.png]]

- 캐시에 저장하여 성능 향상 도모

<br/>

1. 단축 URL 클릭
2. 로드밸런서) 요청 웹 서버에 전달
3. 이미 캐시에 단축 URL이 있는 경우) 원래 URL 바로 전달
4. 캐시에 단축 URL이 없는 경우) 데이터베이스 조회
5. URL 캐시 삽입 후 사용자 전달
