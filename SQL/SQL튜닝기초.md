# SQL 튜닝 기초

### EXPLAIN

Query에 대한 정보를 알려준다. 

```sql
mysql> EXPLAIN select id from member where id = 100000;
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | member | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
```

참고할만한 기준

### select_type

- SIMPLE: 내부 쿼리가 없는 SELECT 문이라는 것 단순한 구문인 경우이다.
- PRIMARY: 서브쿼리가 보함된 SQL
- DERIVED: FROM 절에 작성된 서브쿼리라는 의미 FROM 절의 임시 테이블인 인라인 뷰를 말한다.
- DEPENDENT *:  쿼리가 독립적으로 수행하지 못하고 메인 테이블로 부터 값을 하나씩 공급받는 구조라고 볼 수 있겠다.
- UNCACHEABLE *: 메모리에 상주하여 재활용되어야 할 서브쿼리가 재사용되지 못할때 출력되는 유형이다.

DEPENDENT, UNCACHEABLE을 SQL의 튜닝의 대상으로 불 수 있다. 그러나 모든 상황에 그런건 아니다. 

### type

- system: 테이블에 데이터가 없거나 한개만 있는 성능상 최상의 type
- const: 조회되는 데이터가 단 1건일 경우 출력되는 유형 속도나 리소스 사용 측면에서 유리
- eq_ref: 조인이 수행될 때 고유 인덱스 또는 기본키로 단 1건의 데이터를 조회하는 방식. 조인 할때 성능상 가낭 유리한 경우이다.
- index: 인덱스 풀 스캔을 의미한다.
- all: 테이블 처음부터 끝까지 읽는 테이블 풀 스갠 방식에 해당 되는 유형.

### extra

SQL문을 어떻게 수행할 것인지에 관찬 추가 정보를 보여준다. 

- Using index: 물리적인 데이터 파일을 읽지 않고 인덱스만 읽어서 SQL 문의 요청사항을 처리할 수 있는 경우
- Using filesort: 정렬이 필요한 데이터를 메모리에 올리고 정렬 작업을 수행한다는 의미
    - 인덱스를 사용하면 정렬 작업이 필요 없다. 인덱스를 사용하지 못할 때 정렬을 위해 메모리 영역을 사용하게 된다.
- Using temporary: 데이터의 중간 결과를 저장하고자 임시 테이블을 생성하겠다는 의미
    - 데이터를 가져와 저장한 뒤에 정렬 작업을 수행하거나 중복 제거하는 작업들을 수행한다.
    - `DISTINCT`, `GROUP BY`, `ORDER BY`

## SQL Profiling

SQL문의 병목 지점을 찾고자 사용하는 툴을 말한다. 느린 쿼리나 작성한 SQL의 성능을 파악할 때 사용하면 좋겠다. 

MySql은 기본적으로 비활성화 되어 있다. 

```sql
show variables like 'profiling%'

+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| profiling              | OFF   |
| profiling_history_size | 15    |
+------------------------+-------+
```

SET 키워드로 활성화 시킬 수 있다. **접속한 세션에 한해서만 적용된다.** 다른 세션에는 영향을 미치지 않는다. 

```sql
mysql> set profiling = 'ON';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> show variables like 'profiling%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| profiling              | ON    |
| profiling_history_size | 15    |
+------------------------+-------+
```

이후 간단한 쿼리를 실행해자.

```sql
mysql> select id from member where id = 100000;
+--------+
| id     |
+--------+
| 100000 |
+--------+
1 row in set (0.00 sec)
```

실행했던 쿼리들이 기록되는 것을 볼 수 있었다. 

```sql
mysql> show profiles;
+----------+------------+-----------------------------------------+
| Query_ID | Duration   | Query                                   |
+----------+------------+-----------------------------------------+
|        1 | 0.02793300 | show variables like 'profiling%'        |
|        2 | 0.01751850 | show tables                             |
|        3 | 0.00457275 | select id from member where id = 100000 |
+----------+------------+-----------------------------------------+
3 rows in set, 1 warning (0.01 sec)
```

특정 쿼리 ID에 대해서 프로파일링된 상세 내용을 볼 수 있다. 

```sql
mysql> show profile for query 3;
+--------------------------------+----------+
| Status                         | Duration |
+--------------------------------+----------+
| starting                       | 0.000685 |
| Executing hook on transaction  | 0.000047 |
| starting                       | 0.000059 |
| checking permissions           | 0.000047 |
| Opening tables                 | 0.001734 |
| init                           | 0.000077 |
| System lock                    | 0.000093 |
| optimizing                     | 0.000252 |
| statistics                     | 0.000831 |
| preparing                      | 0.000150 |
| executing                      | 0.000098 |
| end                            | 0.000023 |
| query end                      | 0.000020 |
| waiting for handler commit     | 0.000160 |
| closing tables                 | 0.000073 |
| freeing items                  | 0.000104 |
| cleaning up                    | 0.000123 |
+--------------------------------+----------+
17 rows in set, 1 warning (0.00 sec)
```

Duration값이 높게 나온다면 문제를 의심해 볼만한 구간으로 불 수 있다. 

**일반적인 프로파일링 항목**

| start | 설명 |
| --- | --- |
| starting | SQL 문 시작 |
| checking permissions | 필요 권한 확인 |
| Opening tables | 테이블 열기 |
| After opening tables | 테이블 연 이후 |
| System lock | 시스템 잠금 |
| Table lock | 테이블 잠금 |
| init | 초기화 |
| optimizing | 최적화 |
| statistics | 통계 |
| preparing | 준비 |
| executing | 실행 |
| Sending data | 데이터 보내기 |
| end | 종료 |
| query end | 질의 끝 |
| closing tables | 테이블 닫기 |
| Unlocking tables | 테이블 잠금 해제 |
| freeing items    | 항목 해방 |
| updating status | 상태 업데이트 |
| cleaning up | 정리 |

옵션을 통해 좀더 다양항 출력 정보를 확인 할 수도 있다. 

```sql
mysql> show profile all for query 3;
```

- all: 모든 정보를 표시
- block io: 블록 입력 및 출력 작업 횟수
- context switches: 자발적, 비자발적인 컨텍스트 스위치 수
- cpu: 사용자 및 시스템 CPU 사용 기간을 표시
- ipc: 보내고 받은 메세지의 수
- page faults: 주 페이지 오류 및 부 페이지 오류 수를 표시
- source: 함수가 발행하는 파일 이름과 행 번호와 함께 소스 코드의 함수 이름 표시
- swaps