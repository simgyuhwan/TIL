## INSERT IGNORE

INSERT IGNORE 옵션을 사용하면 프라이머리 키, 유니크 인덱스와 같이 중복을 허용하지 않는 칼럼을 등록시, 실패를 무시하고 경고만 받으면서 중복된 칼럼을 저장할 수 있게 된다. 뿐만 아니라 대부분의 에러를 무시해준다.

즉, 원래라면 실패할 **에러를 경고만 받고 허용하게 된다.**

만약 IGNORE를 사용하지만 기존의 칼럼의 타입과 전혀 다른 타입을 입력한다면 **경고와 함께 기본 값**으로 저장된다. 

```sql
mysql> INSERT IGNORE INTO salaries VALUES (NULL, NULL, NULL, NULL);
Query OK, 1 row affected, 4 warnings (0.06 sec)

mysql> SELECT * FROM selaries WHERE emp_no=0;

mysql> SELECT * FROM salaries WHERE emp_no=0;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|      0 |      0 | 0000-00-00 | 0000-00-00 |
+--------+--------+------------+------------+
1 row in set (0.00 sec)

mysql> INSERT IGNORE INTO salaries VALUES ('hello', NULL, NULL, NULL);
Query OK, 0 rows affected, 5 warnings (0.01 sec)

mysql> SELECT * FROM salaries WHERE emp_no='hello';
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|      0 |      0 | 0000-00-00 | 0000-00-00 |
+--------+--------+------------+------------+
1 row in set, 1 warning (0.01 sec)
```

이 쿼리를 보면 emp_no은 int 형만 가능하지만 NULL, ‘hello’를 넣었을 때, 0으로 자동 저장되는 것을 확인할 수 있다. 

**INSERT IGNORE**는 의도하지 않는 에러까지 모두 무시하기 때문에 정말 중복된 데이터를 넣고자 할 때만 경고를 확인하고 저장하자.

## INSERT …ON DUPLICATE KEY UPDATE

INSERT 할 때, 만약 중복된 키가 존재할 경우, UPDATE를 진행하는 문장인데. 아래의 예를 살펴보자. 먼저 daily_statistic이라는 테이블을 만들고 데이터를 등록한다.

```sql
mysql> create table daily_statistic(
	target_date DATE NOT NULL, 
	stat_name VARCHAR(10) NOT NULL, 
	stat_value BIGINT NOT NULL DEFAULT 0, 
	PRIMARY KEY(target_date, stat_name)
);

mysql> INSERT INTO daily_statistic(target_date, stat_name, stat_value) VALUES(DATE(NOW()), 'VISIT', 1) ON DUPLICATE KEY UPDATE stat_value=stat_value+1;
Query OK, 1 row affected (0.00 sec)

mysql> SELECT * FROM daily_statistic;
+-------------+-----------+------------+
| target_date | stat_name | stat_value |
+-------------+-----------+------------+
| 2023-11-12  | VISIT     |          1 |
+-------------+-----------+------------+
1 row in set (0.00 sec)

mysql> INSERT INTO daily_statistic(target_date, stat_name, stat_value) VALUES(DATE(NOW()), 'VISIT', 1) ON DUPLICATE KEY UPDATE stat_value=stat_value+1;
Query OK, 2 rows affected (0.01 sec)

mysql> SELECT * FROM daily_statistic;
+-------------+-----------+------------+
| target_date | stat_name | stat_value |
+-------------+-----------+------------+
| 2023-11-12  | VISIT     |          2 |
+-------------+-----------+------------+
1 row in set (0.00 sec)
```

동일한 INSERT를 두 번 했을 경우, stat_value의 값이 1 증가하는 것을 볼 수 있다. 제일 처음 초기화 INSERT문은 뒤에 ON DUPLICATE KEY UPDATE를 무시한다. 그 이후 중복된 키가 존재하면 뒤에 UPDATE문을 실행한다.

## LOAD DATA 명령 주의 사항

데이터를 빠르게 적재할 수 있는 방법으로 LOAD DATA 명령이 자주 소개되지만 다음과 같은 특징 때문에 대용량의 데이터를 저장하기에는 부적절할 수 있다.

- 단일 스레드로 실행
- 단일 트랜잭션으로 실행

단일 스레드 환경에서 대량의 데이터를 저장하게 되면 그만큼 레코드 저장도 하면서 인덱스 저장 처리도 실행되는데, 결과적으로 뒤로 갈수록 성능은 하락하게 될 것이다.

또 단일 트랜잭션으로 실행되면 updo 로그에 처리된 시점을 저장할텐데. 언두 로그를 디스크로 기록하는 부하도 만들고 언두 로그가 쌓이면 레코드를 읽는 쿼리들이 필요한 레코드를 찾는 데 더 많은 오버 헤드를 만든다.

LOAD DATA 명령으로 저장하고 싶다면 파일로 쪼개서 여러 트랜잭션으로 나누어 저장하거나 INSERT … SELECT .. 문장으로 레코드를 쪼개서 저장하는 방식을 사용하자.