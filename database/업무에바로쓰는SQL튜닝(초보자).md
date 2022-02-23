# SQL튜닝 초보자

### 인덱스를 통한 테이블 조회를 하자.

```sql
select * from member
where substring(id, 1,4) = 1100
and length(id) = 5;
```

- 이건 Full Scan을 하게 되는 방식이된다.

```sql
select * from member
where id between 11000 and 11009
```

- 이렇게 사용하면 index range scan을 하게 되므로 훨씬 시간이 줄어 든다.
- id를 그대로 사용하는 대신 조작해서 사용하면 테이블을 Full scan을 수행하게 된다.

### 데이터의 값을 확인하여 불필요하게 함수를 사용하고 있지 않은지 확인하자

```sql
select ifnull(성별, 'NO DATA') as sex, count(1) 건수 
from member
group by ifnull(성별, 'NO DATA');
```

- 성별이라는 column이 null을 포함하고 있지 않은 데이터들로 구성되어 있다면 ifnull 함수는 불필요하게 함수를 사용하고 있다는 것이다.

### 형변환을 하면 인덱스를 활용하지 못하게 된다.

```sql
mysql> EXPLAIN select count(1) from salary where is_use = 1;
+----+-------------+--------+------------+-------+---------------+----------+---------+------+--------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key      | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+----------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | salary | NULL       | index | I_is_use      | I_is_use | 4       | NULL | 486780 |    10.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+----------+---------+------+--------+----------+--------------------------+
```

- 이상함을 느낄수 없을 수 있다.
- `EXPLAIN` 을 확인해보면 `filtered`항목이 10이라고 나온다.
    - 스토리지 엔진 → MySQL 엔진 → 10% 만 최종출력이라는의미
- 0, 1 `‘’` or `null` 이 포함되었나?
    - 실제 데이터는 0, 1로만 구성되어 있음.

```sql
mysql> desc salary;
+------------+---------+------+-----+---------+-------+
| Field      | Type    | Null | Key | Default | Extra |
+------------+---------+------+-----+---------+-------+
| id         | int     | NO   | PRI | NULL    |       |
| income     | int     | NO   |     | NULL    |       |
| start_date | date    | NO   | PRI | NULL    |       |
| end_date   | date    | NO   |     | NULL    |       |
| is_use     | char(1) | YES  | MUL |         |       |
+------------+---------+------+-----+---------+-------+
5 rows in set (0.01 sec)
```

- `is_use`의 type은 `char(1)` 이다. 숫자를 넣으면 내부적으로 묵시적 형변환이 발생하는 것 이다.
- 1 이 아닌 ‘1’를 사용해 보면

```sql
mysql> EXPLAIN select count(1) from salary where is_use = '1';
+----+-------------+--------+------------+------+---------------+----------+---------+-------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key      | key_len | ref   | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+----------+---------+-------+-------+----------+-------------+
|  1 | SIMPLE      | salary | NULL       | ref  | I_is_use      | I_is_use | 4       | const | 51134 |   100.00 | Using index |
+----+-------------+--------+------------+------+---------------+----------+---------+-------+-------+----------+-------------+
```

- 형변환이 발생하지 않았다.