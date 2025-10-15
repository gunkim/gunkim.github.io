---
layout: post
title: MySQL InnoDB Locking
category: mysql
---

## Shared Lock (S-Lock, 공유 잠금) & Exclusive Lock (X-Lock, 배타 잠금)
행(Record) 수준의 Lock인 것은 동일하나 **Shared Lock**은 다른 트랜잭션에서의 읽기를 허용하고, **Exclusive Lock**은 읽기를 불가능하게 한다는 차이가 있다.

**Shared Lock**은 `SELECT ... FOR UPDATE`를 사용하고, **Exclusive Lock**은 `SELECT ... FOR SHARE` 구문을 사용한다.

## Intention Locks (의도 잠금)
**S-Lock**과 **X-Lock**을 테이블 수준에서 설정하는 경우를 **Intention Lock**이라고 표현한다. 예를 들어, S-Lock을 설정할 의도가 있는 경우 이를 **Intention Shared Lock (IS Lock)** 이라고 부르며, X-Lock을 설정할 의도가 있는 경우 이를 **Intention Exclusive Lock (IX Lock)** 이라고 부른다.

**Intention Lock**의 동작은 S, X Lock을 획득하기 위해 더 강한 Intention Lock을 획득해야 한다.
- 트랜잭션이 테이블의 레코드에 대해 S-Lock을 획득하기 전에 먼저 테이블에 대한 IS-Lock 또는 그보다 강한 Lock을 획득해야 한다.
- 트랜잭션이 테이블의 레코드에 대해 X-Lock을 획득하기 전에 먼저 테이블에 대한 IX-Lock을 획득해야 한다.

|     | X (Exclusive) | IX (Intention Exclusive) | S (Shared) | IS (Intention Shared) |
| --- | ------------- | ------------------------ | ---------- | --------------------- |
| X   | 충돌            | 충돌                       | 충돌         | 충돌                    |
| IX  | 충돌            | 호환                       | 충돌         | 호환                    |
| S   | 충돌            | 충돌                       | 호환         | 호환                    |
| IS  | 충돌            | 호환                       | 호환         | 호환                    |

Lock 획득 요청이 기존 Lock과 호환(Compatible)될 경우 요청된 트랜잭션이 Lock을 획득하지만, 기존 Lock과 충돌(Conflict)할 경우에는 획득할 수 없다. 트랜잭션은 충돌하는 기존 Lock이 해제될 때까지 대기해야 하며, 만약 Lock 획득 요청이 기존 Lock과 충돌하고 Dead Lock(교착 상태)을 발생시킬 수 있는 경우에는 오류가 발생한다.

Intention Lock은 전체 테이블 요청(예: `LOCK TABLES ... WRITE`)을 제외하고는 어떤 것도 차단하지 않는다. Intention Lock의 주요 목적은 누군가가 테이블의 레코드를 잠그거나 잠글 예정임을 나타내는 것이다.

Lock 상태 확인을 해보면 아래와 같이 나온다.
```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

## Record Lock (레코드 잠금)
Record Lock은 Index 레코드에 대한 Lock이다. S-Lock, X-Lock을 통해 설정된다.

예를 들어, X-Lock을 건다고 가정한다면 `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;` 쿼리는 다른 트랜잭션이 `t.c1 = 10`인 레코드를 삽입, 업데이트, 삭제하지 못하도록 한다. 읽기는 S, X Lock 여부에 따라 달라진다.

Lock 상태 확인을 해보면 아래와 같이 나온다. S-Lock과 X-Lock이 Record Lock에 해당하기 때문에 Record Lock이라고 표현되며 S-Lock과 X-Lock이 추가로 표현된다.
```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` trx id 10078 lock_mode X locks rec but not gap Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0 0: len 4; hex 8000000a; asc ;; 1: len 6; hex 00000000274f; asc 'O;; 2: len 7; hex b60000019d0110; asc ;;
```

## Gap Lock (간격 잠금)
**Gap Lock**은 인덱스의 특정 간격이나 그 사이에 대한 잠금이다. 주로 **X-Lock** 을 통해 설정되며, **S-Lock** 과 함께 사용되기도 한다. **Gap Lock**의 주된 목적은 **순수한 방지(purely inhibitive)** 로, 트랜잭션 중에 **레코드 간격(갭)에 새로운 레코드가 삽입되는 것을 방지**하는 것이다.

예를 들어, `SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20 FOR UPDATE;` 쿼리를 실행하면, `c1` 값이 10에서 20 사이에 있는 레코드들은 물론, 그 사이의 **빈 공간(갭)** 에도 잠금이 걸리기 때문에 다른 트랜잭션이 열 `t.c1`에 값 15를 삽입하지 못하게 된다.

**Gap Locking**은 **Unique Index**를 사용하는 **단일 레코드 조회** 시에는 필요하지 않으며, 이 경우에는 **Record Lock**만 걸린다. 예를 들어, `id`가 **PK(Primary Key)** 이거나 **Unique Index**일 때 `SELECT * FROM child WHERE id = 100 FOR UPDATE;`는 `id = 100`에만 **Record Lock**을 걸며, `id = 99`나 `id = 101`과 같은 값의 삽입은 허용된다. 이는 고유 인덱스가 특정 레코드를 정확히 지정할 수 있기 때문에, 추가적인 범위 잠금(Gap Lock)이 필요하지 않기 때문이다.

반면, **Non-Unique Secondary Index**나 **인덱스가 없는 경우**에는 항상 **Next-Key Lock**이 적용된다. 이 경우 **Record Lock**과 **Gap Lock**이 결합된 **Next-Key Lock**이 걸리기 때문에, 해당 범위 내에서 새로운 레코드를 삽입하는 것이 차단된다. 예를 들어, `SELECT * FROM t WHERE c1 = 100 FOR UPDATE;`와 같은 쿼리는 `c1 = 100`인 레코드와 그 앞뒤의 간격에 **Next-Key Lock**이 걸려, 100 이하의 값에 대한 새로운 레코드 삽입이 차단된다.

하지만 **Gap Lock**은 **공존**할 수 있다. **S Gap Lock**은 같은 구간에 대해 여러 개가 존재할 수 있지만, **X Gap Lock**이 같은 구간에 걸리게 되면, 다른 트랜잭션에서 동일한 구간에 **Gap Lock**을 걸 수 없다. 즉, **S Gap Lock**은 중복 허용이 가능하지만, **X Gap Lock**은 독점적이다.

또한, `READ COMMITTED` 격리 수준에서는 **Gap Locking**이 비활성화된다. 이 격리 수준에서는 검색 및 인덱스 스캔 시 **Gap Lock**이 적용되지 않으며, 오직 **외래 키 제약 조건**이나 **중복 키 검사** 시에만 사용된다. 이는 **REPEATABLE READ** 격리 수준과 차이가 있으며, `READ COMMITTED`에서는 팬텀 리드를 방지하지 않기 때문이다.

그리고 `READ COMMITTED` 격리 수준에서 InnoDB는 `WHERE` 조건에 맞지 않는 레코드에 대해 설정된 **Record Lock**을 조건을 확인한 후 바로 해제한다. 예를 들어, `UPDATE`를 수행할 때 InnoDB는 **반일관성(semi-consistent)** 읽기를 통해 가장 최근에 커밋된 데이터를 MySQL에 제공하고, MySQL은 그 데이터를 이용해 해당 레코드가 `WHERE` 조건과 일치하는지 확인한 후, 조건에 맞지 않는 레코드에 걸린 잠금을 해제한다.
## Next-Key Locks (다음 키 잠금)
**Next-Key Lock**은 Index 레코드에 대한 Record Lock과 그 Index 레코드 앞 간격에 대한 Gap Lock의 조합이다.

InnoDB는 테이블의 인덱스를 검색하거나 스캔할 때, 각 인덱스 레코드에 S-Lock 또는 X-Lock을 걸어 행 수준의 잠금을 수행한다. 즉, 행 수준의 잠금은 실제로 Index Record Lock이다. 또한, Next-Key Lock은 인덱스 레코드뿐만 아니라 그 앞의 간격에도 영향을 미친다. 다시 말해, Next-Key Lock은 인덱스 레코드 잠금과 해당 레코드 앞의 간격에 대한 Gap Lock을 포함한다. 한 트랜잭션이 인덱스에서 레코드 R에 S-Lock 또는 X-Lock을 걸고 있다면, 다른 트랜잭션은 인덱스 순서상 R 앞의 간격에 새로운 인덱스 레코드를 삽입할 수 없다.

id가 10, 11, 13, 20인 4개의 인덱스 레코드가 있을 때, InnoDB의 Next-Key Lock이 설정될 수 있는 구간은 다음과 같다. 각 구간은 새로운 레코드가 삽입될 수 없는 범위를 의미한다. 원형 괄호는 구간 끝점을 제외하고, 대괄호는 끝점을 포함한다.
```
(-∞, 10]  // 음의 무한대부터 10까지
(10, 11]  // 10과 11 사이
(11, 13]  // 11과 13 사이
(13, 20]  // 13과 20 사이
(20, ∞)   // 20 이후부터 양의 무한대까지
```
마지막 간격의 Next-Key Lock은 인덱스의 가장 큰 값 위의 간격을 잠그고, 실제로 존재하는 어떤 값보다 큰 가상의 레코드인 초과(superum)를 잠근다. 초과는 실제 인덱스 레코드가 아니기 때문에, 이 Next-Key Lock은 효과적으로 가장 큰 인덱스 값 다음의 간격만 잠근다고 보면 된다.

기본적으로 InnoDB는 트랜잭션 격리 레벨은 **REPEATABLE READ**이다. 이 경우 InnoDB는 검색 및 Index 스캔에 대해 Next-Key Lock을 사용하여 팬텀 리드를 방지한다.

Lock 상태 확인을 해보면 아래와 같이 나온다.
```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## Insert Intention Locks (의도 잠금 삽입)
**Insert Intention Lock**은 Insert할 때 설정되는 Gap Lock의 일종이다. 삽입하려는 레코드의 Index 범위에 Gap Lock이 걸려 있다면 Gap Lock이 해제될 때까지 Insert는 대기해야 한다. 이 때 대기할 때 여러 트랜잭션이 충돌될 수 있으니 Gap Lock이 해지될 때 Insert될 순서를 조율하기 위해 사용되는 Lock이다.

다음 예시는 Insert Intention Lock을 설정한 후에 삽입된 레코드에 대해 X-Lock을 얻는 트랜잭션을 보여준다. 이 예시에는 두 클라이언트 A와 B가 포함된다.

클라이언트 A는 두 개의 Index 레코드(90과 102)를 포함하는 테이블을 생성하고, 그 후 ID가 100보다 큰 Index 레코드에 대해 X-Lock을 설정하는 트랜잭션을 시작한다. 이 X-Lock은 레코드 102 앞의 Gap Lock도 포함된다.
```sql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) VALUES (90), (102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```
클라이언트 B는 갭에 레코드를 삽입하는 트랜잭션을 시작한다. 이 트랜잭션은 **Insert Intention Lock**을 설정하면서 **X-Lock**을 얻기 위해 대기한다.
```sql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

Lock 상태 확인을 해보면 아래와 같이 나온다.
```
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;
```

## AUTO-INC Locks (자동 증가 잠금)
**AUTO-INC Lock**은 **AUTO_INCREMENT** 열을 가진 테이블에 삽입하는 트랜잭션에 의해 설정되는 특별한 테이블 수준의 잠금이다. 가장 단순한 경우, 하나의 트랜잭션이 테이블에 값을 삽입하고 있다면, 다른 트랜잭션은 해당 테이블에 자신들의 삽입을 위해 대기해야 한다. 이는 첫 번째 트랜잭션이 삽입한 행들이 연속적인 기본 키 값을 받도록 하기 위함이다.

**innodb_autoinc_lock_mode** 변수는 AUTO-INC Lock에 사용되는 알고리즘을 제어한다. 이 변수는 예측 가능한 자동 증가 값의 순서와 삽입 작업의 최대 동시성 간의 균형을 선택할 수 있게 해준다.

자세한 내용은 [Section 17.6.1.6, “AUTO_INCREMENT Handling in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-auto-increment-handling.html)를 참고.

## Predicate Locks for Spatial Indexes (공간 인덱스를 위한 술어 잠금)
InnoDB는 공간 데이터가 포함된 열에 대해 **SPATIAL** 인덱싱을 지원한다. (자세한 내용은 [Section 13.4.9, “Optimizing Spatial Analysis”](https://dev.mysql.com/doc/refman/8.4/en/optimizing-spatial-analysis.html) 를 참고)

**SPATIAL** 인덱스를 포함하는 작업에 대한 잠금을 처리하기 위해, Next-Key Lock은 **REPEATABLE READ** 또는 **SERIALIZABLE** 트랜잭션 격리 수준을 지원하는 데 잘 작동하지 않는다. 다차원 데이터에는 절대적인 순서 개념이 없기 때문에, 어떤 것이 "다음" 키인지 명확하지 않다.

**SPATIAL** 인덱스를 가진 테이블의 격리 수준을 지원하기 위해, InnoDB는 **Predicate locks**를 사용한다. **SPATIAL** 인덱스는 최소 경계 사각형 (MBR) 값을 포함하므로, InnoDB는 쿼리에 사용된 MBR 값에 대해 **Predicate lock**을 설정하여 인덱스에서 일관된 읽기를 강제한다. 다른 트랜잭션은 쿼리 조건과 일치하는 행을 삽입하거나 수정할 수 없다.

## Lock 상태 확인 방법
```sql
SHOW ENGINE INNODB STATUS; // innoDB 상세 정보

SELECT * FROM performance_schema.data_locks; // 걸려잇는 락 확인
```
## 이 글의 작성 기준
- [MySQL InnoDB Locking 8.4 Document](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)에 소개된 공식 문서 내용을 토대로 이해한 내용을 정리한다.
- 어색한 영단어는 의역해서 본인이 이해하기 쉬운 방향으로 정리한다.
- 영어 명사는 왠만해선 있는 그대로 표현한다.