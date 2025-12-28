## PostgreSQL insert/count performance
* 운영하면서 PostgreSQL은 insert, count 쿼리가 느리다고 느꼈다.


## PostgreSQL / MySQL / MongoDB Benchmark
* [Database Performance Benchmark: PostgreSQL 17 vs. MySQL 9 vs. MongoDB 8](https://blog.probirsarkar.com/database-performance-benchmark-postgresql-17-vs-mysql-9-vs-mongodb-8-047b3c32d933)
* 테스트 조건
  * 1000개의 Row 
    * 1개씩 Insert
    * Bulk Insert
    * Select
    * Delete
![image.png](../images/db-vendor-benchmark.png

* 벤치마크 결과를 보니, count는 없지만 insert에 대해서는 bulk insert가 느린 것을 알 수 있었다!
  * 마이그레이션 수행 시 bulk insert를 수행했는데, 그래서 느리다고 느꼈던 것 같다.


## Bulk Insert / Count 느린 이유
* MVCC & Write Amplification
  * PostgreSQL의 MVCC는 페이지에서 튜플의 여러 버전을 관리
    * 테이블/인덱스 생성 시 해당 객체에 '페이지' 할당
  * CUD가 발생할 때마다 해당 테이블의 페이지에 Dead Tuple(이전 버전의 Tuple) 발생
    * 특정 수치가 되면 AutoVaccum이 발생하여 I/O를 사용하므로 성능 저하
  * 인덱스가 존재하는 테이블이라면 데이터가 추가될 때마다 인덱스 페이지마다 물리적 위치 정보를 추가해야하므로 Write Disk I/O 추가 발생
    * MySQL 같은 클러스터형 인덱스는 B-Tree에서 실제 데이터 주소가 아닌 PK만 저장하면 됨
    * PostgreSQL은 B-Tree에서 특정 튜플의 주소인 TID를 저장해야함
    * 해당 구조는 Update 시 더 치명적 : MySQL은 PK 값만 변경하면 되지만, PostgreSQL은 TID가 변경되므로 인덱스 전체를 다시 작성해야함
  * Bulk Insert 시 많은 수의 Row가 추가되므로 Dead Tuple이 많이 발생하여 AutoVaccum이 자주 발생

## Count 느린 이유
* Visibility Map(MVCC) & Dead Tuple
  * PostgreSQL은 '현재 테이블에 몇 개의 행이 있는가?'에 대한 답이 Transaction 별로 다르다.
  * 특정 행의 데이터가 보이는지를 체크하기 위한 'Visibility Map'이 있는데, count 시 해당 Map을 전부 뒤져서 보이는 행만 가져와야 한다.
    * 따라서, 해당 flag를 확인하는 O(N) 연산 수행
    * 해당 Visibility는 페이지 단위!
  * 페이지 조회 시 Dead Tuple도 함께 조회되어야 하므로 느림

