## 재고시스템으로 알아보는 동시성이슈 해결방법

### 문제점
- **레이스 컨디션**
  - 둘 이상의 스레드가 공유 데이터에 액세스 할 수 있고, 동시에 변경하려고 할 때 발생하는 문제
  - **해결방법**
    - 하나의 스레드만 데이터에 액세스 할 수 있도록 한다.


### synchronized
- 동작 원리
  - **synchronized 메서드**는 동기화된 객체(인스턴스 또는 클래스) 레벨에서 락을 걸어, 한 번에 하나의 스레드만 해당 메서드를 실행하도록 한다.
- 사용 목적
  - 멀티스레드 환경에서 공유 자원에 대한 동시 접근을 제어하여 데이터의 일관성을 유지하고 충돌을 방지한다.
- 문제점
  - **@Transactional** 이 프록시 객체를 사용하므로, **synchronized** 키워드가 원본 메서드에만 적용되어 의도한 동기화가 제대로 작동하지 않을 수 있다.


### Mysql 활용
- **Pessimistic Lock (exclusive lock)**
  - 다른 트랜잭션이 특정 row의 Lock을 얻는 것을 방지한다.
    - A 트랜잭션이 끝날 때까지 기다렸다가 B 트랜잭션이 lock 을 획득한다.
  - 특정 row를 update 하거나 delete 할 수 있다.
  - 일반 select는 별다른 lock이 없기 때문에 조회는 가능하다.
  - **장점**
    - 충돌이 빈번하게 일어난다면 **Optimistic Lock**보다 성능이 좋을 수 있다.
    - 락을 통해 업데이트를 제어하기 때문에 데이터 정합성이 보장된다.
  - **단점**
    - 별도의 락을 잡기 때문에 성능 감소가 있을 수 있다.

- **Optimistic Lock**
  - lock을 걸지 않고, 문제가 발생 할 때 처리한다.
  - 대표적으로 version column 을 만들어서 해결하는 방법이 있다.
  - **장점**
    - 별도의 락을 잡지 않으므로 **Pessimistic Lock**보다 성능상 이점이 있다.
  - **단점**
    - 업데이트가 실패했을 때 재시도 로직을 개발자가 직접 작성해 주어야 하는 번거로움이 발생한다.

- **Pessimistic Lock**과 **Optimistic Lock**의 용도
  - 충돌이 빈번하게 일어난다면 **Pessimistic Lock**을 추천, 빈번하게 일어나지 않는다면 **Optimistic Lock**을 추천

- **Named Lock**
  - 이름과 함께 lock을 획득한다.
  - 해당 lock은 다른 세션에서 획득 및 해제가 불가능하다.
  - 주로 분산락을 구현할 때 사용한다.
  - 데이터 삽입 시에 정합성을 맞춰야 하는 경우에도 사용한다.
  - **장점**
    - **Pessimistic Lock**은 타임아웃 구현하기 힘들지만, **Named Lock**은 타임아웃을 구현하기 쉽다.
  - **단점**
    - 트랜잭션 종료 시에 락 해제, 세션 관리를 잘 해줘야 하기 때문에 주의해서 사용해야 하고, 구현 방법이 복잡할 수 있다.


### Redis 활용
- **Lettuce**
  - 구현이 간단하다.
  - Spring Data Redis 를 이용하면, **lettuce**가 기본이기 때문에 별도의 라이브러리를 사용하지 않아도 된다.
  - **Spin lock** 방식이기 때문에, 동시에 많은 스레드가 **lock** 획득 대기 상태라면 **redis**에 부하가 갈 수 있다.
    - 그래서 락 획득 간에 텀을 주어야 한다
    - `예) Thread.sleep(100);`

- **Redisson**
  - 락 획득 재시도를 기본으로 제공한다.
  - **pub-sub** 방식으로 구현이 되어 있기 때문에, **lettuce**와 비교를 했을 때 **redis**에 부하가 덜 간다.
    - 대신 **lettuce** 보다 구현이 어렵다.
  - 별도의 라이브러리를 사용해야 한다.
  - **lock**을 라이브러리 차원에서 제공해주기 때문에 사용법을 공부해야 한다.

- **실무에서는**
  - 재시도가 필요하지 않은 **lock**은 **lettuce** 활용
  - 재시도가 필요한 경우에는 **redisson**을 활용


### Mysql 과 Redis 의 특징
- **Mysql**
  - 이미 **Mysql**을 사용하고 있다면, 별도의 비용 없이 사용이 가능하다.
  - 어느 정도의 트래픽까지는 문제없이 활용이 가능하다.
  - **Redis** 보다는 성능이 좋지 않다.
- **Redis**
  - 활용 중인 **Redis**가 없다면, 별도의 구축 비용과 인프라 관리 비용이 발생한다.
  - **Mysql** 보다 성능이 좋다.