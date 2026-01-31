# URL 단축기 설계

## 요구 사항 정의
* 동작 방식
  * `https://www.systeminterview.com/q=chatsystem&c=loggedin&v=v3&l=long`
  * `https://tinyurl.com/y7ke-ocwj`
  * 단축 URL을 결과로 제공하고 해당 URL을 접속하면 원래 URL로 리다이렉트 할 수 있어야 한다.
* 트래픽
  * 매일 1억개의 단축 URL을 생성해야 한다.
* 단축 URL
  * 짧을 수록 좋고, 숫자와 영문자만 사용
* 단축 URL을 시스템에서 지우거나 갱신은 할 수 없다.

---
## API 설계
* URL 단축기 API는 다음과 같은 2가지 API가 필요하다
  * `POST /api/v1/data/shorten` : 원래 URL을 단축 URL로 변환
    * Request Body
      ```json
      {
        "originalUrl": "https://www.systeminterview.com/q=chatsystem&c=loggedin&v=v3&l=long"
      }
      ```
    * Response Body
      ```json
      {
        "shortUrl": "https://tinyurl.com/y7ke-ocwj"
      }
      ```
  * `GET /api/v1/{shortUrl}` : 단축 URL을 원래 URL로 리다이렉트


* 단축 URL Get 시 원래 URL로 리다이렉트하는 API의 HttpStatus를 2가지로 선택할 수 있다.
  * 301 Permanently Moved: HTTP 요청 처리 책임이 Location 헤더의 URL로 이동되었다는 의미
    * 영구적으로 책임이 이동된다는 의미로, 브라우저가 해당 원래 URL을 캐시에 저장
    * **이후 같은 요청이 들어오면 브라우저가 캐싱된 URL로 이동하여 서버 부하를 줄인다.**
  * 302 Found: HTTP 요청을 임시로 Location 헤더의 URL이 처리한다는 의미 
    * 임시적으로 처리한다는 의미로, 같은 요청이 와도 항상 서버를 거쳐서 리다이렉트 된다.
    * **리다이렉트 되는 통계나 추적용으로 좋다.**

✈️ 307 Temporary Redirect
* 302와 동작은 비슷하다.
* 302는 명세상으로는 Http Method를 변경하지 않지만, 초기 브라우저들이 리다이렉트는 무조건 GET으로 보내곤 했다.
  * 그래서 302를 받으면 일반적으로 POST 요청이더라도 GET으로 변경해서 요청하는 경우가 많았다.
  * 그래서 POST 시의 사용자 요청 데이터가 유실되는 경우가 있었다.
* 해당 단점을 보완하기 위해 HTTP1.1에서 307이 추가되었다.
* 307의 핵심: "메서드와 Body를 절대로 바꾸지 말고 그대로 다시 보내라."
* 네이버 지도 공유하기에서는 공유 링크를 URL에 입력 시 307을 응답

---
## 단축 해시 설계
* URL을 단축하기 위해서 URL 전체를 해시 함수를 사용한 값으로 매핑한다.

### 해시 충돌 해소 전략
* URL을 해시 함수를 통해 해시 값으로 만든 후 해시 값이 이미 존재하는지 DB에서 확인
* 해시 값이 존재한다면(충돌이라면) 사전에 정한 문자열을 기존 URL에 추가해서 해시
* 충돌이 나지 않을 때까지 반복
-> **충돌 여부 조회를 위해 DB 조회가 필요하므로 오버헤드가 크다.**

### base conversion (진법 변환)
* 기존 URL으로 유일 ID를 생성하고 해당 유일 ID를 진법 변환을 통해 단축 URL로 변환
* 유일 ID이므로 충돌이 발생하지 않는다.


---
### URL 단축이 필요한 필요성
* 이 장에서 말하는 URL이란, 개발자가 아닌 사용자가 사용하는 URL
* 사용자가 사용하는 URL이 길어지게 되면 다음과 같은 문제점이 발생할 수 있다.
  * UX상 부정적 경험
    * `https://example.com/products/category/electronics/item/2024/promotion/discount-final-version-01`
    * `bit.ly/Sale2024`
  * URL 최대 길이 제한
    * 브라우저의 최대 길이 제한은 거의 약 3만자 내외로 길다.
    * 하지만, 웹서버나 미들웨어, SEO(검색 엔진 최적화)에서 에러를 뱉을 수 있다.
  * 링크의 사후 관리
    * 사용자에게 배포할 URL이 잘못되었을 때, 매핑 테이블의 기존 URL을 정상적으로 변경하면 해결할 수 있다.
    * 비즈니스적으로 단축 URL의 사용 기간이 만료되었을 때 링크를 변경하여 비즈니스적으로 사용할 수 있다.
      * 이벤트 페이지를 단축 URL로 사용자들에게 공유
      * 이벤트 종료 후 단축 URL을 회사 메인 페이지나 이벤트 종료 페이지로 매핑 테이블의 기존 URL을 변경하여 리다이렉트






























