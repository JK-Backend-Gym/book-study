## Cache Strategy
* [Caching Strategies and How to Choose the Right One](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/)

> Your caching strategy depends on the data and data access patterns. In other words, how the data is written and read.
> * is the system write heavy and reads less frequently? (e.g. time based logs)
> * is data written once and read multiple times? (e.g. User Profile)
> * is data returned always unique? (e.g. search queries)

* 캐싱 전략은 Data Access Pattern(데이터가 어떻게 쓰여지고 읽히는지)에 따라 결정해야 한다.


### 1. Cache-Aside
![image.png](../images/cache-aside.png)
* Cache가 Application-DB Side에 존재
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
* 
