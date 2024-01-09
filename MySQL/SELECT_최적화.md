# WHERE, GROUP BY, ORDER BY 인덱스 사용

## 인덱스를 사용하기 위한 기본 규칙

- **인덱스로 사용할 칼럼의 변형 금지 (칼럼의 값 변화 X)**
- **WHERE 절에 사용할 조건의 타입을 일치해야 한다.**

## WHERE 절의 인덱스 사용

다음과 예제를 보자. **A, B, C, B** 는 칼럼의 이름이다.

`결합 인덱스 : A → B → C → D`

`WHERE 순서 : B → C → A → D`

처음 결합 인덱스로 생성한 순서대로 A, B, C, D로 **WHERE** 절도 일치하게 작성해야 할 것 같지만 그렇지 않다. **WHERE** 절은 순서에 상관없이 인덱스에 사용될 수 있다. 오히려 필요한 건 조건이다. 동등 조건인지, 비교 조건인지에 따라 처리가 달라진다.

## GROUP BY 절의 인덱스 사용

`GROUP BY` 절의 조건은 WHERE 보다 까다롭다.

- GROUP BY 절의 칼럼의 순서는 결합 인덱스와 **순서가 일치**해야 한다.
- GROUP BY 절에 넣을 칼럼 중에 앞에 칼럼이 연속해서 있고 뒤에 칼럼이 없는 것은 상관없다. 하지만 **앞에 칼럼부터 없으면** 인덱스를 타지 않는다.
- 결합 인덱스에 포함되지 않는 칼럼이 GROUP BY 에 있으면 인덱스를 타지 않는다.

`결합 인덱스 : A, B, C, D`

결합 인덱스가 위와 같이 생성 됐을 때, 다음 예시는 **인덱스를 타지 못하는 예시**이다.

- `GROUP BY B, A` :  결합 인덱스와 순서가 일치하지 않았다.
- `GROUP BY A, C` : 결합 인덱스의 중간 인덱스가 존재하지 않는다.
- `GROUP BY A, B, C, D, E` : 결합 인덱스에 존재하지 않는 칼럼이 포함됐다

예외적으로 `A, B`가 `WHERE 절`에 사용됐다면 GROUP BY에서 제외해도 인덱스가 가능할 때도 있다. 이유는 선행으로 A, B의 값이 일치하는 순서대로 가져왔기 때문이다. 

## ORDER BY 절의 인덱스 사용

`ORDER BY` 절은 `GROUP BY` 와 기본적인 내용은 비슷하나 약간의 차이가 있다. 인덱스를 생성하면 기본적으로 오름차순으로 정렬된다. 그렇기 때문에 ORDER BY의 칼럼들은 동일하게 **모두 오름차순**이거나 **내림차순**이어야 인덱스를 사용한다.

`결합 인덱스 : A, B, C, D`

결합 인덱스가 위와 같이 생성 됐을 때, 다음 예시는 **인덱스를 타지 못하는 예시**이다.

- `GROUP BY B, A` :  결합 인덱스와 순서가 일치하지 않았다.
- `GROUP BY A, C` : 결합 인덱스의 중간 인덱스가 존재하지 않는다.
- `GROUP BY A, B, C, D, E` : 결합 인덱스에 존재하지 않는 칼럼이 포함됐다
- `GROUP BY A, B DESC, C` : 중간에 B 칼럼이 혼자만 내림차순이다.

## WHERE + ORDER BY(또는 GROUP BY)

`WHERE` 절과 `ORDER` 절이 같이 사용된 쿼리는 다음과 같이 세 가지 방법으로 인덱스가 사용된다. 

- **WHERE, ORDER 모두 인덱스 이용** : `WHERE`, `ORDER BY` 절에 모두 동일하며 정렬된 인덱스로 되어 있고 `WHERE` 절이 `비교 조건`인 경우, 두 곳 모두 인덱스를 사용하기 때문에 다른 두 방법보다 훨씬 빠르다.
- **WHERE 절만 인덱스 이용** : `WHERE` 절만 인덱스 칼럼인데. `ORDER BY` 절을 인덱스 아니면 WHERE 절로 먼저 가져온 뒤, FileSort 를 통해 정렬한다. WHERE 절이 일치하는 건수가 적을 때 적절하다.
- **ORDER BY 절만 인덱스 이용** : `ORDER BY` 절만 인덱스를 사용할 때는 먼저 ORDER BY의 인덱스를 통해서 WHERE 조건절과 일치하지 않는 것들을 걸러낸다. 레코드가 아주 많으면 적절하다.

위의 예시는 동등 비교 조건(=) 일 때의 예시이다. 하지만 범위 비교 조건일 때는 살짝 다르다. 

```sql
mysql> SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_1, COL_2, COL_3;
mysql> SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_2, COL_3;
```

첫 번째는 인덱스를 이용해서 `WHERE`, `ORDER BY`를 처리할 수 있다. 하지만 두 번째는 다르다. 이유는 `COL_1` 이 10보다 큰 데이터를 가져오지만 다시 `COL_2`, `COL_3` 기준으로 정렬해야 하며 또 `COL_1`이 없기 때문에 인덱스를 사용하지 못한다.

## GROUP BY + ORDER BY

GROUP BY + ORDER BY 쿼리에서 인덱스를 사용하려면 **양쪽 모두 칼럼의 순서와 내용이 같아야지만 인덱스를 사용할 수 있다.** 만약 둘 중에 하나라도 안맞으면 둘 모두 인덱스를 사용할 수 없다.

예를 들어, 아래와 같은 예시는 인덱스를 사용할 수 없는 예시이다.

```sql
GROUP BY A, B ORDER BY B;
GROUP BY A, B ORDER BY A, C;
```

## WHERE + ORDER BY + GROUP BY

WHERE + ORDER BY(or GROUP BY), GROUP BY + ORDER BY 에서 설명한 내용 그대로 적용된다.

# WHERE 절의 비교 조건 주의사항

## NULL 비교

다른 DBMS와 다르게 MySQL은 NULL 값이 포함된 레코드도 인덱스로 관리된다. SQL에서 NULL의 정의는 비교할 수 없는 값이다. 그래서 NULL=1 은 NULL이 나올 뿐이다. 대안으로는 `IS NULL` 혹은 `ISNULL()` 를 사용하면 되는데. 

```sql
mysql> SELECT * FRO titles WHERE to_date IS NULL; // 인덱스 O
mysql> SELECT * FRO titles WHERE ISNULL(to_date);  // 인덱스 O
mysql> SELECT * FRO titles WHERE ISNULL(to_date)=1; // 인덱스 X
mysql> SELECT * FRO titles WHERE ISNULL(to_date)=true; // 인덱스 X
```

세 번째, 네 번째 쿼리는 인덱스나 풀 테이블 스캔으로 처리된다. 그러니 첫 번째나 두 번째 방식을 사용하자.

## 날짜 비교

MySQL의 날짜 타입은 다음과 같다.

- **DATE** : 날짜
- **DATETIME,TIMESTAMP** : 날짜, 시간
- **TIME** : 시간

MySQL에는 DATE 타입과 문자열을 비교하면 자동으로 **형변환**을 해준다. 그렇기 때문에 아래의 두 쿼리는 동일하게 인덱스를 사용한다.

```sql
mysql> SELECT COUNT(*)
			 FROM employees
			 WHERE hire_date>STR_TO_DATE('2011-07-23', '%Y-%m-%d');

mysql> SELECT COUNT(*)
			 FROM employees
			 WHERE hire_date>'2011-07-23';
```

하지만 어떤 방식으로 하던 **칼럼을 건드리면 인덱스를 사용하지 못한다**.

```sql
mysql> SELECT COUNT(*) 
			 FROM employees
			 WHERE DATE_FORMAT(hire_date, '%Y-%m-%d') > '2011-07-23';
```

그렇기 때문에 가능하면 상수쪽을 수정하자.

### DATE와 DATETIME 비교

DATE 와 DATETIME을 비교하려면 DATETIME의 시간 값을 버리고 비교하면 된다. 

```sql
mysql> SELECT COUNT(*) FROM employees WHERE hire_date>DATE(NOW());
```

`DATETIME → DATE` 로 변환하지 않고 그대로 **DATE 비교 DATETIME**을 하면 DATE는 DATETIME으로 **형변환**을 하게 된다. 그럼 자연스럽게 뒤에 시간은 00:00:00 시로 추가되고 DATETIME과 맞지 않을 수 있다.

### DATETIME과 TIMESTAMP 비교

칼럼의 타입이 DATETIME이고 TIMESTAMP 값과 비교할 때 다음과 같이하면 안된다.

```sql
mysql> SELECT COUNT(*) 
				FROM employees 
				WHERE hire_date > UNIX_TIMESTAMP('1986-01-01 00:00:00);
```

그 이유는 `UNIX_TIMESTAMP`를 실행하면 단순 숫자 값이 나오기 때문이다.

```sql
mysql> SELECT UNIX_TIMESTAMP('1986-01-01 00:00:00');
+---------------------------------------+
| UNIX_TIMESTAMP('1986-01-01 00:00:00') |
+---------------------------------------+
|                             504889200 |
+---------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT FROM_UNIXTIME(UNIX_TIMESTAMP('1986-01-01 00:00:00'));
+------------------------------------------------------+
| FROM_UNIXTIME(UNIX_TIMESTAMP('1986-01-01 00:00:00')) |
+------------------------------------------------------+
| 1986-01-01 00:00:00                                  |
+------------------------------------------------------+
1 row in set (0.01 sec)
```

그렇기 때문에 아래의 방식처럼 `FROM_UNIXTIME`을 사용해주자.

# JOIN

JOIN이 어떻게 인덱스를 사용하는지 쿼리 패턴별로 자세히 살펴보자. 

## JOIN의 순서와 인덱스

먼저 다음의 쿼리를 살펴보자. 

```sql
mysql> SELECT * FROM employees e, dept_emp de WHERE e.emp_no=de.emp_no;
```

이 쿼리에서 두 개의 테이블 employees, dept_emp 에서 인덱스의 유무에 따라 어떤 테이블이 드리븐이되고 어떤 테이블이 드라이빙이 될까? 

결론부터 말하면 둘 다 인덱스가 존재하면 레코드가 적은 쪽이 드라이빙 테이블이 된다. 

그리고 한쪽만 인덱스가 존재하면 인덱스가 있는 테이블이 드리븐 테이블이 된다.

만약 둘 다 인덱스가 없다면 해시 조인으로 처리된다. 

- **두 칼럼 모두 인덱스가 있는 경우** :  어느 테이블을 드라이빙으로 선택하든 인덱스를 이용하여 검색 작업을 빠르게 처리할 수 있다. 보통의 경우 옵티마이저의 선택이 옳으니 믿고 맡기자.
- **employees.emp_no에만 인덱스가 있는 경우** : employees를 드리븐 테이블로 설정한다. 이유는 뒤에 찾는 테이블에 따라 속도가 달라지기 때문에 인덱스가 있는 쪽을 드리븐 테이블로 설정한다.
- **두 칼럼 모두 인덱스가 없는 경우** : 레코드가 적은 테이블을 드라이빙으로 선택하고 해시 조인으로 처리된다.

## JOIN 칼럼의 데이터 타입

테이블의 조인 조건에도 WHERE 절과 마찬가지로 데이터의 타입은 일치해야 한다. 다음의 예시를 보면

```sql
mysql> CREATE TABLE tb_test1(user_id INT, user_type INT, PRIMARY KEY(user_id));
mysql> CREATE TABLE tb_test2(user_type CHAR(1), type_desc VARCHAR(10), PRIMARY KEY(user_type));

mysql> SELECT * FROM tb_test1 tb1, tb_test2 tb2 WHERE tb1.user_type=tb2.user_type;
```

조인을 사용하여 조회를 하고 있는데. tb1.user_type과 tb2.user_type의 타입은 **INT, CHAR** 로 서로 다른 것을 볼 수 있다. 

만약 타입 같았다면 인덱스 레인지 스캔을 사용했겠지만 이 쿼리는 풀 테이블 스캔과 함께 해시 조인이 사용된다. 

## OUTER JOIN의 성능과 주의사항

대부분의 테이블은 **OUTER JOIN**이 필요 없도록 설계되어 있을 가능성이 높다. 예시를 살펴보자. 

```sql
mysql> EXPLAIN SELECT * FROM employees e LEFT JOIN dept_emp de ON de.emp_no=e.emp_no LEFT JOIN departments d ON d.dept_no=de.dept_no AND d.dept_name='Development';
+----+-------------+-------+------------+--------+---------------------+-------------------+---------+----------------------+--------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys       | key               | key_len | ref                  | rows   | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------------+-------------------+---------+----------------------+--------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | ALL    | NULL                | NULL              | NULL    | NULL                 | 300584 |   100.00 | NULL        |
|  1 | SIMPLE      | de    | NULL       | ref    | ix_empno_fromdate   | ix_empno_fromdate | 4       | employees.e.emp_no   |      1 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | eq_ref | PRIMARY,ux_deptname | PRIMARY           | 16      | employees.de.dept_no |      1 |   100.00 | Using where |
+----+-------------+-------+------------+--------+---------------------+-------------------+---------+----------------------+--------+----------+-------------+
```

**OUTER JOIN**의 사용으로 employees 테이블을 **풀 테이블 스캔**을 한 것을 볼 수 있다. 처음에 말한 것처럼 대부분의 테이블은 이런 식으로 만들어있지 않고 INNER JOIN이 가능한 설계로 되어 있을 것이다. 만약 이 쿼리를 **INNER JOIN**을 사용한다면

```sql
mysql> EXPLAIN SELECT * FROM employees e JOIN dept_emp de ON de.emp_no=e.emp_no JOIN departments d ON d.dept_no=de.dept_no AND d.dept_name='Development';
+----+-------------+-------+------------+--------+---------------------------+-------------+---------+---------------------+-------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys             | key         | key_len | ref                 | rows  | filtered | Extra |
+----+-------------+-------+------------+--------+---------------------------+-------------+---------+---------------------+-------+----------+-------+
|  1 | SIMPLE      | d     | NULL       | ref    | PRIMARY,ux_deptname       | ux_deptname | 162     | const               |     1 |   100.00 | NULL  |
|  1 | SIMPLE      | de    | NULL       | ref    | PRIMARY,ix_empno_fromdate | PRIMARY     | 16      | employees.d.dept_no | 41392 |   100.00 | NULL  |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY                   | PRIMARY     | 4       | employees.de.emp_no |     1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------------------+-------------+---------+---------------------+-------+----------+-------+
```

인덱스를 사용하여 조회하는 쿼리를 대폭 줄일 수 있게 된다.

다음은 **가장 많이 하는 실수**인데. 

```sql
mysql> SELECT FROM employees e LEFT JOIN dept_manager mgr ON mgr.emp_no=e.emp_no WHERE mgr.dept_no='d001';
```

**OUTER JOIN** 에 **WHERE**을 사용하면 INNER JOIN으로 변경되어 실행된다. 그러면 원하는 결과가 나오지 않는다. 

```sql
mysql> SELECT FROM employees e INNER JOIN dept_manager mgr ON mgr.emp_no=e.emp_no WHERE mgr.dept_no='d001';
```

그러니 만약 조건을 사용하고 싶다면 다음과 같이 `AND`를 사용하자.

```sql
mysql> SELECT FROM employees e LEFT JOIN dept_manager mgr ON mgr.emp_no=e.emp_no AND mgr.dept_no='d001';
```

## JOIN과 외래키(FOREIGN KEY)

JOIN을 사용할 때, 외래키가 반드시 필요할까? 

먼저 외래키를 왜 쓰는지 알아야 한다. 일반적으로 **외래키를 사용하는 이유**는 **참조 무결성**을 지키기 위해서이다. 참조 무결성이란, 예를 들어서 사원 테이블과 부서 테이블이 있다고 해보자. 사원 테이블에는 부서 테이블의 키 값을 저장하는데. 이때 부서 테이블에 없는 값을 수 없도록 하기 위한 것이 **외래키**이다. **참조하기 위한 키를 잘못된 값을 넣지 못하는 제약**을 줘서 NULL 값이나 부서 테이블에 존재하는 값만 넣게 한다.

그럼 JOIN 시 외래키가 반드시 필요할까의 대답은 **NO** 이다. 외래키는 참조 무결성의 보장을 위해서 사용되는 것이지. 다른 이유는 없다. 오히려 외래키로 인해 제약이 생겨 빼는 경우도 많다.

## 지연된 조인(Delayed Join)

조인을 사용해서 데이터를 조회하는 쿼리에 GROUP BY, ORDER BY를 사용할 때, 인덱스를 사용하면 최적으로 처리되고 있을 가능성이 높다.

그런데 조인을 할 때마다 레코드 건수는 늘어나고 이렇게 늘어난 레코드를 전체 GROUP BY, ORDER BY를 하면 조인하기 전에 수행하는 것보다 많은 수를 처리해야 한다.

**지연된 조인**은 **GROUP BY, ORDER BY를 조인 전에 처리하는 것**을 의미한다.

먼저 인덱스를 사용하지 못하는 GROUP BY, ORDER BY를 살펴보자.

```sql
mysql> EXPLAIN SELECT e.* FROM salaries s, employees e WHERE e.emp_no=s.emp_no AND s.emp_no BETWEEN 10001 AND 13000 GROUP BY s.emp_no ORDER BY SUM(s.salary) DESC LIMIT 10;
+----+-------------+-------+------------+-------+-------------------+---------+---------+--------------------+------+----------+----------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys     | key     | key_len | ref                | rows | filtered | Extra                                        |
+----+-------------+-------+------------+-------+-------------------+---------+---------+--------------------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY           | PRIMARY | 4       | NULL               | 3000 |   100.00 | Using where; Using temporary; Using filesort |
|  1 | SIMPLE      | s     | NULL       | ref   | PRIMARY,ix_salary | PRIMARY | 4       | employees.e.emp_no |    9 |   100.00 | NULL                                         |
+----+-------------+-------+------------+-------+-------------------+---------+---------+--------------------+------+----------+----------------------------------------------+
2 rows in set, 1 warning (0.01 sec)
```

간단하게 살펴보면 다음과 같이 처리될 것이다.

1. employees 테이블의 인덱스를 사용하여 `e.emp_no=s.emp_no AND s.emp_no BETWEEN 10001 AND 13000` 조건을 먼저 읽는다.
2. 가져온 레코드들을 GROUP BY, ORDER BY로 처리한다.
3. LIMIT 10개만 가져온다.

다음은 지연된 조인이다. 먼저 salaries 테이블에서 필요한 처리를 진행하고 employee 테이블과 조인한다.

```sql
mysql> EXPLAIN SELECT e.* FROM (
										SELECT s.emp_no 
										FROM salaries s 
										WHERE s.emp_no 
												AND s.emp_no BETWEEN 10001 AND 13000 
										GROUP BY s.emp_no 
										ORDER BY SUM(s.salary) DESC 
										LIMIT 10) x, 
										employees e 
										WHERE e.emp_no=x.emp_no;
+----+-------------+------------+------------+--------+-------------------+---------+---------+----------+-------+----------+----------------------------------------------+
| id | select_type | table      | partitions | type   | possible_keys     | key     | key_len | ref      | rows  | filtered | Extra                                        |
+----+-------------+------------+------------+--------+-------------------+---------+---------+----------+-------+----------+----------------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL              | NULL    | NULL    | NULL     |    10 |   100.00 | NULL                                         |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY           | PRIMARY | 4       | x.emp_no |     1 |   100.00 | NULL                                         |
|  2 | DERIVED     | s          | NULL       | range  | PRIMARY,ix_salary | PRIMARY | 4       | NULL     | 56844 |   100.00 | Using where; Using temporary; Using filesort |
+----+-------------+------------+------------+--------+-------------------+---------+---------+----------+-------+----------+----------------------------------------------+
3 rows in set, 1 warning (0.01 sec)
```

먼저 서브쿼리의 결과를 임시 테이블에 저장하고 이를 사용하여 employees 테이블과 조인한다. 임시 테이블에 저장할 레코드는 10건밖에 되지 않으므로 메모리를 이용해 빠르게 처리된다.

실제 테스트를 해보면 3~4배 정도 더 빠르게 실행되는 것을 확인할 수 있다. 잘 튜닝된 지연된 처리는 **몇십 배, 몇백 배** 더 나은 성능을 보일 수 있다.

하지만 잘 모르고 사용할 경우 잘못된 결과가 나올 수 있으니 다음을 주의하자.

- **LEFT (OUTER) JOIN** 인 경우 드라이빙 테이블과 드리븐 테이블은 **1:1 또는 M:1** 관계여야 한다.
- **INNER JOIN인 경우**인 경우 드라이빙 테이블과 드리븐 테이블은 **1:1 또는 M:1** 관계여야 한다. 또 드라이빙 테이블에 있는 레코드는 드리븐 테이블에 존재해야 한다.

## 레터럴 조인(Lateral Join)

MySQL 8.0 버전부터는 레터럴 조인 기능이 추가되었는데. 서브 쿼리에서 외부 칼럼을 조회할 수 있는 기능이다. 

```sql
mysql> EXPLAIN SELECT * FROM employees e LEFT JOIN LATERAL (SELECT * FROM salaries s WHERE s.emp_no=e.emp_no ORDER BY s.from_date DESC LIMIT 2) s2 ON s2.emp_no=e.emp_no WHERE e.first_name='Matt';
+----+-------------------+------------+------------+------+---------------+--------------+---------+--------------------+------+----------+----------------------------+
| id | select_type       | table      | partitions | type | possible_keys | key          | key_len | ref                | rows | filtered | Extra                      |
+----+-------------------+------------+------------+------+---------------+--------------+---------+--------------------+------+----------+----------------------------+
|  1 | PRIMARY           | e          | NULL       | ref  | ix_firstname  | ix_firstname | 58      | const              |  233 |   100.00 | Rematerialize (<derived2>) |
|  1 | PRIMARY           | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0>  | 4       | employees.e.emp_no |    2 |   100.00 | NULL                       |
|  2 | DEPENDENT DERIVED | s          | NULL       | ref  | PRIMARY       | PRIMARY      | 4       | employees.e.emp_no |    9 |   100.00 | Using filesort             |
+----+-------------------+------------+------------+------+---------------+--------------+---------+--------------------+------+----------+----------------------------+
3 rows in set, 2 warnings (0.01 sec)
```

서브쿼리 내부를 살펴보면 외부 employees의 emp_no을 사용하는 것을 볼 수 있다. 이렇게 **LATERAL**을 사용하면 서브쿼리가 후순위로 밀려나고 임시 테이블이 생성되어 조인이 된다. 

만약 **LATERAL**을 붙이지 않으면 에러가 발생한다.

# GROUP BY

GROUP BY는 특정 칼럼의 값으로 레코드를 그룹핑하고, 그룹별로 집계된 결과를 하나의 레코드로 조회할 때 사용된다. 

## WITH ROLLUP

그룹핑으로 그룹화한 레코드들의 합계나 평균같은 처리를 하고 싶을 수 있다. 그때 사용되는 것이 **WITH ROLLUP**이다.

```sql
mysql> SELECT dept_no, COUNT(*) FROM dept_emp GROUP BY dept_no WITH ROLLUP;
+---------+----------+
| dept_no | COUNT(*) |
+---------+----------+
| d001    |    20211 |
| d002    |    17346 |
| d003    |    17786 |
| d004    |    73485 |
| d005    |    85707 |
| d006    |    20117 |
| d007    |    52245 |
| d008    |    21126 |
| d009    |    23580 |
| NULL    |   331603 |
+---------+----------+
10 rows in set (0.11 sec)
```

단순히 GROUP BY에 하나의 칼럼만 있으면 마지막에 NULL 로 총 합계가 표현된다. 만약 GROUP BY의 칼럼이 여러 개라면

```sql
gender	age	AVG(score)
F	20	90
F	24	50
F	NULL	70
M	21	80
M	22	70
M	23	60
M	NULL	70
NULL	NULL	70
```

이런 식으로 표현된다. NULL의 위치로 어느 것의 합계인지 유추해야 한다. 그런데 NULL 이라는 값이 마음에 안들 수 있다. 

```sql
SELECT 
		IF(GROUPING(first_name), 'ALL first_name', first_name) AS first_name, 
		IF(GROUPING(last_name), 'All last_name', last_name) AS last_name, 
		COUNT(*) 
FROM employees 
GROUP BY first_name, last_name 
WITH ROLLUP;
```

**IF(GROUPING())** 을 사용하면 NULL 대신 넣을 값을 지정할 수 있다.

## 레코드를 칼럼으로 변환

### 레코드를 칼럼으로 변환

레코드를 칼럼으로 변환한다는게 무슨 말일까? 

```sql
mysql> SELECT dept_no, COUNT(*) FROM dept_emp GROUP BY dept_no WITH ROLLUP;
+---------+----------+
| dept_no | COUNT(*) |
+---------+----------+
| d001    |    20211 |
| d002    |    17346 |
| d003    |    17786 |
| d004    |    73485 |
| d005    |    85707 |
| d006    |    20117 |
| d007    |    52245 |
| d008    |    21126 |
| d009    |    23580 |
| NULL    |   331603 |
+---------+----------+
10 rows in set (0.11 sec)
```

이 예시처럼 아래로 주르륵 있는 레코드들을 가로로 칼럼별로 나타내는 것을 의미한다. 만약 GROUP BY 로 집계한 쿼리를 가로로 나타내고 싶다면 다음의 예시처럼 **SUM, COUNT, MIN**과 같은 집계함수와 **CASE WHEN**을 사용하면 된다

```sql
mysql> SELECT 
					SUM(CASE WHEN dept_no='d001' THEN emp_count ELSE 0 END) AS count_d001, 
					SUM(CASE WHEN dept_no='d002' THEN emp_count ELSE 0 END) AS count_d002, 
					SUM(CASE WHEN dept_no='d003' THEN emp_count ELSE 0 END) AS count_d003, 
					SUM(CASE WHEN dept_no='d004' THEN emp_count ELSE 0 END) AS count_d004,
					SUM(CASE WHEN dept_no='d005' THEN emp_count ELSE 0 END) AS count_d005,
					SUM(CASE WHEN dept_no='d006' THEN emp_count ELSE 0 END) AS count_d006,
					SUM(CASE WHEN dept_no='d007' THEN emp_count ELSE 0 END) AS count_d007,
					SUM(CASE WHEN dept_no='d008' THEN emp_count ELSE 0 END) AS count_d008,
					SUM(CASE WHEN dept_no='d009' THEN emp_count ELSE 0 END) AS count_d009, 
					SUM(emp_count) AS count_total 
				FROM (SELECT dept_no, COUNT(*) AS emp_count FROM dept_emp GROUP BY dept_no) tb_derived;
+------------+------------+------------+------------+------------+------------+------------+------------+------------+-------------+
| count_d001 | count_d002 | count_d003 | count_d004 | count_d005 | count_d006 | count_d007 | count_d008 | count_d009 | count_total |
+------------+------------+------------+------------+------------+------------+------------+------------+------------+-------------+
|      20211 |      17346 |      17786 |      73485 |      85707 |      20117 |      52245 |      21126 |      23580 |      331603 |
+------------+------------+------------+------------+------------+------------+------------+------------+------------+-------------+
1 row in set (0.11 sec)
```

# ORDER BY

**ORDER BY**는 익히 알고 있듯 정렬을 위해 사용된다. 만약 ORDER BY를 사용하지 않으면 어떻게 될까? 

- **InnoDB**는 클러스터링 인덱스 순서로 레코드를 조회한다.
- 임시 테이블을 거쳐서 조회한다면 순서는 예측할 수 없다.

**“Using fildsort”** 는 디스크에 저장해서 정렬한건가? 싶을 수 있는데. 메모리에서 정렬할 수도 있고 디스크에 정렬할 수 도 있다. 자세한 확인은

`mysql> SHOW STATUS LIKE ‘Sort_%’;`

를 통해서 확인할 수 있다.

## ORDER BY 사용법 및 주의사항

ORDER BY 를 사용할 때, 칼럼을 **숫자**로 표현하는 방법도 사용이 가능하다. 아래의 두 쿼리는 같은 의미를 가진다.

```java
mysql> SELECT first_name, last_name FROM employees ORDER BY 2;
mysql> SELECT first_name, last_name FROM employees ORDER BY last_name;
```

단, ORDER BY에 쌍따움표를 사용하면 문자 리터럴로 인식되기 때문에 무시된다. 다음은 잘못된 표현이다. 

```java
mysql> SELECT first_name, last_name FROM employees ORDER BY "last_name";
```

## 인덱스 정렬

MySQL 8.0 부터는 여러 개의 칼럼을 조합해서 인덱스를 생성할 때, 정렬 순서를 혼용할 수 있다. 아래의 쿼리는 salary를 내림차순으로 from_date를 오름차순으로 정렬한 인덱스를 가진다.

```java
mysql> ALTER TABLE salaries ADD INDEX ix_salary_fromdate (salary DESC, from_date ASC);
```

만약 내림차순으로만 조회할 것같다면 인덱스를 내림차순으로 생성하는 것도 좋은 방법이다. 

```java
mysql> ALTER TABLE salaries ADD INDEX ix_salary_asc (salary ASC);
mysql> ALTER TABLE salaries ADD INDEX ix_salary_asc (salary DESC);
```

## 함수기반 인덱스

MySQL 8.0 이전에는 연산의 결과로 인덱스를 생성하려면 가상 칼럼을 생성해야 했는데. 8.0 이후부터는 함수 기반 인덱스를 지원한다.

```java
mysql> SELECT * FROM salaries ORDER BY COS(salary);
```

# 서브쿼리

서브쿼리는 SELECT, FROM, WHERE 절에서 어디에 위치하는지에 따라 성능 영향도와 최적화 방법이 달라진다. 

## SELECT 절에 사용된 서브쿼리

SELECT 절에 사용된 서브쿼리는 임시 테이블을 만들거나 쿼리를 비효율적으로 실행하게 만들지 않기 때문에 적절히 인덱스를 사용할 수 있다면 크게 주의할 사항은 없다. 그래도 조인을 사용을 권장한다.

SELECT 절에 사용된 서브쿼리는 다음과 같은 조건을 만족해야 한다.

- **반드시 하나의 레코드, 칼럼**을 가진 **스칼라 서브쿼리**여야 한다.

예시를 살펴보자. 

```sql
mysql> SELECT emp_no, (SELECT dept_name FROM departments WHERE dept_name='Sales1') as dept_name FROM dept_emp LIMIT 10;
+--------+-----------+
| emp_no | dept_name |
+--------+-----------+
| 110022 | NULL      |
| 110085 | NULL      |
| 110183 | NULL      |
| 110303 | NULL      |
| 110511 | NULL      |
| 110725 | NULL      |
| 111035 | NULL      |
| 111400 | NULL      |
| 111692 | NULL      |
| 110114 | NULL      |
+--------+-----------+
10 rows in set (0.00 sec)
```

서브쿼리에 있는 dept_name은 일치하는 값이 없기 때문에 NULL 하나만 반환된다. 그래서 위와 같은 결과가 조회된다. 

```sql
mysql> SELECT emp_no, (SELECT dept_name FROM departments) as dept_name FROM dept_emp LIMIT 10;
ERROR 1242 (21000): Subquery returns more than 1 row
```

**서브쿼리의 레코드가 2개 이상**이면 위와 같은 에러가 발생한다.

```sql
mysql> SELECT emp_no, (SELECT dept_name, dept_no FROM departments WHERE dept_name='Sales1') as dept_name FROM dept_emp LIMIT 10;
ERROR 1241 (21000): Operand should contain 1 column(s)
```

**서브쿼리의 반환 칼럼이 2개 이상**이면 위와 같은 에러가 발생한다.

## FROM 절에 사용된 서브쿼리

MySQL 5.7버전부터는 옵티마이저가 FROM 절의 서브쿼리를 외부쿼리와 병합하여 최적화하도록 개선됐다. 

```sql
mysql> EXPLAIN SELECT * FROM (SELECT * FROM employees) y;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 300584 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```

이 쿼리는 서브쿼리를 사용했지만 실제 실행 계획을 살펴보면 하나의 테이블만 생성된 것을 볼 수 있다. 

```sql
mysql> SHOW WARNINGS \G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`employees`.`emp_no` AS `emp_no`,`employees`.`employees`.`birth_date` AS `birth_date`,`employees`.`employees`.`first_name` AS `first_name`,`employees`.`employees`.`last_name` AS `last_name`,`employees`.`employees`.`gender` AS `gender`,`employees`.`employees`.`hire_date` AS `hire_date` from `employees`.`employees`
1 row in set (0.00 sec)
```

서브쿼리를 외부쿼리와 병합하려면 가능하면 다른 조작을 하지 말아야 한다. 아래의 예시는 **병합이 불가능한 조건**이다.

- **집합 함수 사용(SUM(), MIN(), MAX(), COUNT() 등)**
- **DISTINCT**
- **GROUP BY 또는 HAVING**
- **LIMIT**
- **UNION 또는 UNION ALL**
- **SELECT 절에 서브쿼리 사용**
- **사용자 변수 사용**

## WHERE 절에 사용된 서브쿼리

### 1) 동등 또는 크다 작다 비교

MySQL 5.5 이전 버전에는 외부 쿼리에 대해서 인덱스가 없다면 풀 테이블 스캔을 하고 서브쿼리를 체크 조건으로 사용하는 경우가 많아 성능 저하가 있었다. 

MySQL 부터는 서브쿼리는 먼저 상수화 한 뒤, 외부 쿼리를 체크 조건으로 처리한다. 

```sql
mysql> EXPLAIN SELECT * FROM dept_emp de WHERE de.emp_no=(SELECT e.emp_no FROM employees e WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys     | key               | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------+------+----------+-------------+
|  1 | PRIMARY     | de    | NULL       | ref  | ix_empno_fromdate | ix_empno_fromdate | 4       | const |    1 |   100.00 | Using where |
|  2 | SUBQUERY    | e     | NULL       | ref  | ix_firstname      | ix_firstname      | 58      | const |  253 |    10.00 | Using where |
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------+------+----------+-------------+

mysql> EXPLAIN FORMAT=tree SELECT * FROM dept_emp de WHERE de.emp_no=(SELECT e.emp_no FROM employees e WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);
| -> Filter: (de.emp_no = (select #2))  (cost=0.69 rows=1)
    -> Index lookup on de using ix_empno_fromdate (emp_no=(select #2))  (cost=0.69 rows=1)
    -> Select #2 (subquery in condition; run only once)
        -> Limit: 1 row(s)  (cost=236.90 rows=1)
            -> Filter: (e.last_name = 'Facello')  (cost=236.90 rows=25)
                -> Index lookup on e using ix_firstname (first_name='Georgi')  (cost=236.90 rows=253)
 |
```

실행 계획을 살펴보면 서브쿼리부터 인덱스로 조회하고  외부 쿼리를 인덱스를 사용하여 조회한 것을 볼 수 있다. 

단, 다음과 같이 **튜플로 조회**할 경우, 외부 쿼리는 인덱스를 사용하지 못하고 **풀 테이블 스캔을 사용**한다.

```sql
mysql> EXPLAIN SELECT * FROM dept_emp WHERE (emp_no, from_date)=(SELECT emp_no, from_date FROM salaries WHERE emp_no=100001 LIMIT 1);
+----+-------------+----------+------------+------+-------------------+---------+---------+-------+--------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys     | key     | key_len | ref   | rows   | filtered | Extra       |
+----+-------------+----------+------------+------+-------------------+---------+---------+-------+--------+----------+-------------+
|  1 | PRIMARY     | dept_emp | NULL       | ALL  | NULL              | NULL    | NULL    | NULL  | 331143 |   100.00 | Using where |
|  2 | SUBQUERY    | salaries | NULL       | ref  | PRIMARY,ix_salary | PRIMARY | 4       | const |      4 |   100.00 | Using index |
+----+-------------+----------+------------+------+-------------------+---------+---------+-------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```

### 2) IN 비교(세미 조인)

다음 예제와 같이 테이블의 레코드가 다른 테이블의 레코드를 이용한 표현식을 **세미 조인**이라 한다. MySQL 5.5 ~ MySQL 8.0 까지 세미 조인에 대한 최적화가 잘 이루어져있

```sql
mysql> EXPLAIN SELECT * FROM employees e WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
+----+--------------+-------------+------------+--------+-------------------------------+-------------+---------+--------------------+------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys                 | key         | key_len | ref                | rows | filtered | Extra       |
+----+--------------+-------------+------------+--------+-------------------------------+-------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE       | <subquery2> | NULL       | ALL    | NULL                          | NULL        | NULL    | NULL               | NULL |   100.00 | NULL        |
|  1 | SIMPLE       | e           | NULL       | eq_ref | PRIMARY                       | PRIMARY     | 4       | <subquery2>.emp_no |    1 |   100.00 | NULL        |
|  2 | MATERIALIZED | de          | NULL       | ref    | ix_fromdate,ix_empno_fromdate | ix_fromdate | 3       | const              |   57 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+-------------------------------+-------------+---------+--------------------+------+----------+-------------+
3 rows in set, 1 warning (0.01 sec)
```

# 윈도우 함수

**집계 함수**는 그룹별로 **하나의 레코드로 묶어서 출력**한다. 반면 윈도우 함수는 레코드 건수는 **변하지 않고** 그대로 유지한다.

## 쿼리 각 절의 실행 순서

윈도우 함수를 사용하는 쿼리의 결과는 다음의 순서로 실행된다. 

1. **FROM, WHERE, GROUP BY, HAVING** 
2. **윈도우 함수**
3. **SELECT, ORDER BY, LIMIT**

먼저 **데이터를 모은** 다음에 **윈도우 함수가 실행**되고 **레코드를 출력**한다. 이 차이가 무슨 의미가 있을까? 

다음 예시를 보자.

```sql
mysql> SELECT emp_no, from_date, salary, AVG(salary) OVER() AS avg_salary FROM salaries WHERE emp_no=10001 LIMIT 5;
+--------+------------+--------+------------+
| emp_no | from_date  | salary | avg_salary |
+--------+------------+--------+------------+
|  10001 | 1986-06-26 |  60117 | 75388.9412 |
|  10001 | 1987-06-26 |  62102 | 75388.9412 |
|  10001 | 1988-06-25 |  66074 | 75388.9412 |
|  10001 | 1989-06-25 |  66596 | 75388.9412 |
|  10001 | 1990-06-25 |  66961 | 75388.9412 |
+--------+------------+--------+------------+
5 rows in set (0.01 sec)
```

위 쿼리의 레코드는 총 17개가 나온다. 그 중 LIMIT 5로 인해 5개만 가져오게 된다. 17개의 데이터를 가져와서 평균을 계산하기 때문에 평균은 17개의 평균된다.

```sql
mysql> SELECT emp_no, from_date, salary, AVG(salary) OVER() AS avg_salary FROM (SELECT * FROM salaries WHERE emp_no=10001 LIMIT 5) s2;
+--------+------------+--------+------------+
| emp_no | from_date  | salary | avg_salary |
+--------+------------+--------+------------+
|  10001 | 1986-06-26 |  60117 | 64370.0000 |
|  10001 | 1987-06-26 |  62102 | 64370.0000 |
|  10001 | 1988-06-25 |  66074 | 64370.0000 |
|  10001 | 1989-06-25 |  66596 | 64370.0000 |
|  10001 | 1990-06-25 |  66961 | 64370.0000 |
+--------+------------+--------+------------+
5 rows in set (0.00 sec)
```

이 쿼리는 서브쿼리를 통해 제한된 5개만 가져와서 평균을 구한다. 

이렇듯 **윈도우 함수를 사용하려면 쿼리의 실행 순서를 잘 이해하고 사용해야 한다.**

## 윈도우 함수 기본 사용법

```sql
AGGREGATE_FUNC() OVER(<partition> <order>) AS window_func_column
```

기본 문법은 윈도우 함수와 OVER 에 파티션과 정렬 순서를 표현하면 된다. 

다음 예제는 입사 순서를 조회하는 쿼리이다. 

```sql
mysql> EXPLAIN SELECT e.*, RANK() OVER(ORDER BY e.hire_date) AS hire_date_rank FROM employees e;
```

# SELECT 잠금

**SELECT** 에서는 레코드를 조회할 때, 잠금을 걸 수 있는데. `FOR SHARE`, `FOR UPDATE`가 이 방법이다.

```sql
mysql> SELECT * FROM employees WHERE emp_no=10001 FOR SHARE;
mysql> SELECT * FROM employees WHERE emp_no=10001 FOR UPDATE;
```

- `FOR SHARE` : 찾은 레코드를 읽기 잠금을 걸어서, 다른 세션은 읽기만 가능해진다.(shared lock)
- `FOR UPDATE` : 찾은 레코드를 읽기, 쓰기 잠금을 걸어서, 다른 세션은 접근할 수 없게 한다.(exclusive lock)

하지만 이런 레코드 잠금은 같은 `FOR SHARE`, `FOR UPDATE` 쿼리일 때만 잠금이 진행된다. 만약 단순 SELECT 쿼리라면 `FOR SHARE`, `FOR UPDATE` 를 해도 영향을 받지 않고 쿼리가 진행된다.

## 잠금 테이블 선택

만약 JOIN을 여러 번을 해서 많은 테이블을 거칠 수 있는데. 이때 내가 원하는 테이블만 잠금을 걸 수 있다. 이때는 `OF`를 사용하면 된다. 

```sql
mysql> SELECT * 
				FROM employees e
				INNER JOIN dept_emp de ON de.emp_ne.emp_no
				INNER JOIN departments d ON d.dept_no=de.dept_no
			WHERE e.emp_no=10001
			FOR UPDATE OF e
```

## NOWAIT & SKIP LOCKED

하나씩 간단하게 설명하면 `NOWAIT` 은 조회하려는 레코드가 만약 FOR UPDATE 로 잠금 상태에 걸렷다면 바로 실패로 처리한다. 

`SKIP LOCKED` 는 조회하려는 레코드가 FOR UPDATE로 잠금 상태에 걸렸다면 잠금된 레코드를 건너뛴다. 처음에는 SKIP 이 무슨 의미가 있는가 싶었다.

만약 쿠폰을 발행하는 비즈니스가 있다고 해보자. 테이블에는 100개의 쿠폰을 미리 생성해놨고 사용자가 할당되지 않는 쿠폰을 조회해서 update를 한다고 해보자. 

```sql
mysql> SELECT * FROM coupon WHERE owned_user_id=0 ORDER BY coupon_id ASC LIMIT 1 FOR UPDATE;
```

사용자의 요청이 동시에 100명이 온다고 해보자. 그러면 처음 사용자의 처리가 끝날 때까지 이 레코드는 계속 잠금이 되고 다음 99명은 대기 상태로 있게 된다. 

이때 `SKIP LOCKED` 를 추가하면 잠금된 레코드는 건너뛰고 다음 레코드를 조회한다. 이렇게 되면 100명의 사용자가 대기 없이 거의 동시에 처리되게 된다.

어떤 비즈니스냐에 따라 다르겠지만 설계에 따라 동시성 문제를 성능 향상과 함께 손쉽게 처리할 수 있게 된다.