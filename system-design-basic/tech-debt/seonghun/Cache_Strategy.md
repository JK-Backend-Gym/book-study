## Cache Strategy
* [Caching Strategies and How to Choose the Right One](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/)

> Your caching strategy depends on the data and data access patterns. In other words, how the data is written and read.
> * is the system write heavy and reads less frequently? (e.g. time based logs)
> * is data written once and read multiple times? (e.g. User Profile)
> * is data returned always unique? (e.g. search queries)

* 캐싱 전략은 Data Access Pattern(데이터가 어떻게 쓰여지고 읽히는지)에 따라 결정해야 한다.


### 1. Cache-Aside
![image.png](../images/cache-aside.png)
* Cache가 Application-DB의 바깥 Side에 존재
* Cache - DB 사이의 Connection이 없다.
* Cache/DB로의 모든 Operation은 Application이 수행

* Flow
  * 데이터 조회 시 Application 
    * Cache 조회 후 있으면 Cache에서 반환
    * Cache에 없으면 DB에서 조회 후 Cache에 저장!
    * @Cacheable

**UseCase / Pros & Cons**
* **read가 많은 데이터에 적합**
* Cache Miss/Failure에 resilient하다.
  * Cache 시스템에 장애가 발생해도 DB에서 서빙 가능
  * ⚠️Cache Miss로 인해 DB 부하가 증가할 수 있다. 
* **DB의 데이터 모델과 Cache의 데이터 모델이 다를 수 있다는 장점**
  * 여러 DB 데이터를 합친 API Response를 Cache에 저장할 수 있다.

Cache-Aside는 DB에만 Write를 하기 때문에 캐시와 데이터 일관성이 깨질 수 있다.
* 캐시의 TTL 동안 이전의 데이터를 서빙
* 실시간 데이터 일관성이 중요하다면 Application에서 DB Write 시 해당 Cache Invalidate
  * @CacheEvict

---

### 2. Read-Through
![img.png](../images/read-through.png)
* Cache가 Application-DB 사이에 존재


* Flow
  * 데이터 조회 시 Application은 Cache만 바라본다.
  * Cache Miss 시 Cache Provider가 DB에 데이터를 저장하고 Cache에도 저장한다.
  * Application에는 저장된 데이터를 Cache가 반환한다.
  * @Cacheable


🤔Cache-Aside / Read-Through
* Cache-Aside는 Application이 DB에 데이터를 가져와서 Cache에 저장하는 책임을 가진다.
* Read-Through는 Application은 단순히 Cache로부터 데이터를 가져오기만 하고 데이터를 저장하는 역할은 Cache가 DB를 읽어서 한다.
  * 따라서, Cache-Aside와 다르게 캐시에 저장되는 데이터는 DB 모델과 다르면 안된다.


**UseCase / Pros & Cons**
* **read가 많은 데이터에 적합**
  * 첫 요청 시에 DB에서 데이터를 캐시에 저장하는 latency가 단점
    * 해당 문제를 해결하기 위해 해당 데이터를 pre-fetching(warming, pre-heating) 하는 방법도 사용한다.
  * 캐시에 저장되는 데이터는 DB 모델과 다르면 안된다는 단점
  * 캐시와 DB 사이의 데이터 일관성이 깨질 수 있다.
    * cache-aside와 동일하게 Cache가 남아 있을 때 DB Write 시에 일관성이 깨진다.

---

### 3. Write-Through
![img.png](../images/write-through.png)
* Cache가 Application-DB 사이에 존재
* Write 시 Application은 Cache에만 저장하고, Cache가 DB에 저장한다.
* Read-Through 전략과 함께 사용 시 캐시와 DB 사이의 데이터 일관성이 보장된다.
  * 데이터 R/W를 모두 Cache에서 수행하므로 일관성이 보장된다.


**UseCase / Pros & Cons**
* Write 시 Latency가 발생한다. 
  * Cache에 저장 후 DB에 저장해야 하므로 Latency 발생
* Read-Through와 세트로 사용 시 Cache 무효화를 하지 않고도 데이터 일관성을 보장할 수 있다.
* example : DAX (DynamoDB Accelerator)
  * Read-Through / Write-Through를 전략으로 사용하여 Application과 DynamoDB 사이에 DAX가 위치
  * 해당 DAX로 캐시를 사용하면서 데이터 일관성을 보장한다.

---

### 4. Write-Around
* 데이터를 Write할 때 DB에 직접 Write를 수행하고 Cache에는 저장 X
* 해당 데이터를 처음 Read 할 때 Cache에 저장하는 방식
* Write-Through와 비교하여 데이터 저장 후 해당 데이터를 저장할 때만 캐시에 저장하므로 메모리 절약
  * 따라서, 채팅 메시지나 로그처럼 Write 후 Read가 거의 없는 데이터에 적합


---

### 5. Write-Back (Write-Behind)
![img_1.png](../images/writer-back.png)
* 데이터 Write 시 Cache -> DB로 저장되는 건 Write-Through와 동일
* **하지만, Cache -> DB로 저장할 때 동기가 아닌 비동기로 저장**
  * 따라서, 쓰기 성능을 높일 수 있다.

**UseCase / Pros & Cons**
* **write가 많은 데이터에 적합**
* DB Down 시 resilient하다. (캐시에 Write 데이터 존재)
* 캐시가 Down 되었을 경우 데이터가 영구적으로 유실될 수 있다.
* 대부분의 RDB의 Storage Engine (e.g - MySQL InnoDB)에서 내부 엔진 캐시 방식을 Write-Back으로 사용하고 있다.
  * 내부 메모리 캐시에 먼저 저장해놓고 디스크에 비동기로 저장


---

<br>
<br>

## 📝 캐시 전략별 실무 적용 예시 (Summary)

캐시 전략은 데이터의 **읽기/쓰기 비중**과 **정합성 요건**에 따라 선택해야 합니다. 현재 우리 프로젝트의 **Modular Monolith(api/domain)** 구조에서 적용 가능한 시나리오들입니다.

---

### 1. Cache-Aside (Look-Aside)
*가장 범용적이며, 캐시 장애 시에도 서비스가 유지되어야 하는 핵심 도메인에 사용합니다.*

* **이커머스 상품 상세 정보:** 상품명, 상세 설명 등 조회 빈도는 매우 높지만 Redis 장애 시에도 DB(Domain Repository)를 통해 조회가 가능해야 하는 경우.
* **복합 API 응답(Aggregation):** 여러 테이블을 조인하거나 외부 API 결과를 조합하여 만든 '무거운' DTO 데이터를 통째로 캐싱할 때.

---

### 2. Read-Through
*애플리케이션 코드를 간결하게 유지하고, 데이터 로딩 로직을 캐시 계층에 위임할 때 사용합니다.*

* **시스템 공통 코드:** 국가 코드, 통화 단위, 약관 내용 등 DB 모델과 캐시 모델이 1:1로 일치하는 마스터 데이터.
* **읽기 전용 정적 데이터:** 도서 카탈로그나 영화 메타데이터처럼 한 번 등록되면 변경이 거의 없고 대량의 읽기만 발생하는 데이터.

---

### 3. Write-Through
*데이터 정합성이 가장 중요한 금융 거래나 보안 설정 등에 사용합니다.*

* **사용자 포인트 및 잔액:** 포인트 충전/사용 시 캐시와 DB에 동시에 기록하여, 조회 시 항상 최신 상태의 잔액을 보장해야 하는 경우.
* **실시간 차단 리스트(Blacklist):** 관리자가 특정 IP나 사용자를 차단했을 때, 즉시 캐시와 DB 모두에 반영되어 보안 필터가 작동해야 하는 경우.

---

### 4. Write-Around
*캐시 메모리를 효율적으로 사용해야 하며, 쓰기 직후 조회가 거의 없는 데이터에 적합합니다.*

* **고객 상담 내역 및 감사 로그:** 상담 내역은 실시간으로 저장되지만, 나중에 특정 사건이 터져서 조회하기 전까지는 다시 읽을 일이 거의 없는 데이터.
* **SNS 게시글 등록:** 사용자가 글을 올린 직후보다, 팔로워들의 피드에 노출되어 실제 '조회'가 일어나는 시점에 캐시에 올라오도록 유도.

---

### 5. Write-Back (Write-Behind)
*극단적인 쓰기 부하를 견뎌야 하거나 응답 속도가 매우 빨라야 하는 서비스에 사용합니다.*

* **실시간 조회수 및 '좋아요' 수:** 대형 이벤트 시 초당 수만 건의 클릭을 DB에 직접 반영하지 않고, 캐시에서 먼저 합산한 뒤 5~10분 단위로 DB에 Batch 업데이트.
* **게임 캐릭터 위치 정보:** 유저의 실시간 좌표 이동을 매번 DB에 기록하지 않고 캐시에만 유지하다가, 일정 주기마다 체크포인트로 DB에 영속화.
