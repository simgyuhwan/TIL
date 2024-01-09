## 형 변환 성능 문제

```sql
SELECT * FROM tab_test WHERE number_column='10001';
SELECT * FROM tab_test WHERE string_column=10001;
```

MySQL에서는 자동으로 타입을 변환하는데. 만약 number_column이 숫자 타입이라면 첫 번째 ‘10001’은 자동으로 숫자 타입으로 변환해준다. 

하지만 두 번째 string_column은 타입이 문자열이라면 10001이 아닌 **내부 타입들을 숫자 타입으로 변환해서 비교해주기 때문에 인덱스를 활용하지 못하거나 쿼리가 실패**한다.(성능 하락)

## <=> 연산자

```sql
mysql> SELECT 1 <=> 1, NULL <=> NULL, 1 <=> NULL;
+---------+---------------+------------+
| 1 <=> 1 | NULL <=> NULL | 1 <=> NULL |
+---------+---------------+------------+
|       1 |             1 |          0 |
+---------+---------------+------------+
1 row in set (0.01 sec)

mysql> SELECT 1 = 1, NULL = NULL, 1 = NULL;
+-------+-------------+----------+
| 1 = 1 | NULL = NULL | 1 = NULL |
+-------+-------------+----------+
|     1 |        NULL |     NULL |
+-------+-------------+----------+
1 row in set (0.00 sec)
```

= 연산자는 NULL 값을 비교할 수 없지만, <=> 연산자는 IS NULL과 동일하게 NULL 값을 비교가 가능하다.

## 정규 표현식(Reguler Expression)

```sql
mysql> SELECT 'abcz' REGEXP '^[x-z]';
+------------------------+
| 'abcz' REGEXP '^[x-z]' |
+------------------------+
|                      0 |
+------------------------+
1 row in set (0.02 sec)

mysql> SELECT 'zabc' REGEXP '^[x-z]';
+------------------------+
| 'zabc' REGEXP '^[x-z]' |
+------------------------+
|                      1 |
+------------------------+
1 row in set (0.00 sec)
```

‘^[x-z]’ 문자열이 x, y, z 로 시작하는지 확인하는 정규표현식

REGEXP 연산자는 WHERE 조건절에 **단독으로 사용하면 인덱스 레인지 스캔을 사용할 수 없어서 성능상 좋지 않다**.  가능하다면 데이터 조회 범위를 줄일 수 있는 조건과 함께 사용하라

## BEWEEN, IN

```sql
SELECT * FROM dept_emp WHERE dept_no BETWEEN 'd003' AND 'd005' AND emp_no=10001
```

**BETWEEN**을 사용하면 > , < 를 사용하는 것과 동일하다. 그 사이의 값을 선형 탐색을 하게 돼서 상당히 많은 데이터를 읽는데. 만약 **BETWEEN** 에 사용되는 범위가 작다면 **IN**을 사용하자

```sql
SELECT * FROM dept_emp WHERE dept_no IN ('d003', 'd004', 'd005') AND emp_no=10001;
```

IN을 사용함으로써 동등 비교를 여러번 수행하는 것이기 때문에 인덱스(dept_no, emp_no)를 최적으로 사용할 수 있게 된다. 

```sql
mysql> EXPLAIN SELECT * FROM dept_emp USE INDEX(PRIMARY) WHERE dept_no BETWEEN 'd003' AND 'd005' AND emp_no=10001;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | dept_emp | NULL       | range | PRIMARY       | PRIMARY | 20      | NULL | **165571** |     0.00 | Using where |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT * FROM dept_emp USE INDEX(PRIMARY) WHERE dept_no IN ('d003', 'd004', 'd005') AND emp_no=10001;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | dept_emp | NULL       | range | PRIMARY       | PRIMARY | 20      | NULL |    **3** |   100.00 | Using where |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

rows 수를 비교해보면 165571, 3 의 차이를 확연히 확인할 수 있다. 물론 **USE INDEX(PRIMARY)**를 사용하지 않으면 둘 다 다른 인덱스를 사용하기 때문에 rows는 1이다.

## InnoDB의 잠금

InnoDB의 잠금은 **레코드를 잠그는 것이 아니라 인덱스를 잠근다**. 이것이 무슨 차이일까?? 먼저 아래의 쿼리를 보면 first_name 이 ‘Georgi’ 인 데이터는 **253**개이다. 그리고 first_name이 ‘Georgi’이면서 last_name 이 ‘Klassen’ 인 것은 **하나**이다.

```sql
mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi';
+----------+
| COUNT(*) |
+----------+
|      253 |
+----------+
1 row in set (0.01 sec)

mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi' AND last_name='Klassen';
+----------+
| COUNT(*) |
+----------+
|        1 |
+----------+
1 row in set (0.06 sec)
```

그런데 이 데이터를 Update 한다고 해보자.

```sql
mysql> UPDATE employees SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='klassen';
Query OK, 1 row affected, 1 warning (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 1
```

그러면 first_name 에만 **인덱스로 처리되어 있기 때문에 253개의 레코드가 잠금**이 된다. 그리고 1개를 변경한다.

만약 **인덱스가 어느 곳에도 없다면 모든 레코드가 잠금**이 된다. 

이렇게 때문에 MySQL에서 InnoDB에서 인덱스 설계가 중요하다.