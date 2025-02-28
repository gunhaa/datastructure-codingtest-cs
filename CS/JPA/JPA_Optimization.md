# JPA 성능 최적화 방법과 시도 순서(김영한 실전 JPA2)

## 엔티티 조회
- 엔티티를 조회해서 그대로 반환: V1 
    - 엔티티가 변경되면 API스펙이 바뀌어서 사용불가
- 엔티티 조회 후 DTO로 변환: V2 
    - 쿼리가 정말 많이 나가서 사용 불가능
- 페치 조인으로 쿼리 수 최적화: V3 
    - 쿼리 한번에 모든것을 반환한다.
    - 하지만 fetch join을 사용한 방식이기 때문에 fetch join 특성상 페이징이 불가능하다.
    - 결과를 받아 온 후 메모리상에서 dto로 변환시킨다.
- 컬렉션 페이징과 한계 돌파: V3.1 
    - ToOne 관계는 fetch 조인으로 쿼리 수 최적화
    - 컬렉션은 fetch 조인 대신에 지연 로딩을 유지하고,  `hibernate.default_batch_fetch_size`, `@BatchSize` 를 이용해서 가져온다.
    - 결과를 받아 온 후 메모리상에서 dto로 변환시킨다.
## DTO 직접 조회로 최적화
- JPA에서 DTO를 직접 조회: V4
    - N+1문제 발생/ 하지만 코드가 간결한다
- 컬렉션 조회 최적화 일대다 관계인 컬렉션은 JPQL의 IN 절을 활용해서 메모리에 미리 조회해서 최적화: V5
    - 쿼리 1+1번으로 최적화 가능, 코드가 길어짐과 반비례해서 높아지는 성능 향상/ 많이 사용하는 방식, 하지만 엔티티의 BatchSize와 사실상 같은 동작 방식이다.
- 플랫 데이터 최적화 - JOIN 결과를 그대로 조회 후 애플리케이션에서 원하는 모양으로 직접 변환: V6
    - 페이징 불가능, 적은 쿼리, 사실 완전히 다른 접근 방식이다.

# 최적화 권장 순서
1. 엔티티 조회 방식으로 우선 접근
    1. 페치조인으로 쿼리 수를 최적화
    2. 컬렉션 최적화
        1. 페이징 필요하면 `hibernate.default_batch_fetch_size` , `@BatchSize` 로 최적화
        2. 페이징 필요없다면 fetch 조인 사용
2. 엔티티 조회 방식으로 해결이 안되면 DTO 조회 방식 사용
3. DTO 조회 방식으로 해결이 안되면 NativeSQL or 스프링 JdbcTemplate

- 엔티티 조회 방식은 페치 조인이나, `hibernate.default_batch_fetch_size` , `@BatchSize` 같이 코드를 거의 수정하지 않고, 옵션만 약간 변경해서, 다양한 성능 최적화를 시도할 수 있다. 반면에 DTO를 직접 조회하는 방식은 성능을 최적화 하거나 성능 최적화 방식을 변경할 때 많은 코드를 변경해야 한다.

- 만약 페치조인, `@BatchSize` 로 최적화가 안된다면 트래픽이 많은 대형 서비스일텐데, 이 경우는 캐시를 써야한다. 캐시를 사용한다면 DTO를 캐시해야한다

- 개발자는 성능 최적화와 코드 복잡도 사이에서 줄타기를 해야 한다. 항상 그런 것은 아니지만, 보통 성능 최적화는 단순한 코드를 복잡한 코드로 몰고간다.

- 엔티티 조회 방식은 JPA가 많은 부분을 최적화 해주기 때문에, 단순한 코드를 유지하면서, 성능을 최적화 할 수 있다.

- 반면에 DTO 조회 방식은 SQL을 직접 다루는 것과 유사하기 때문에, 둘 사이에 줄타기를 해야 한다.

## DTO 조회 방식의 선택지

- DTO로 조회하는 방법도 각각 장단이 있다. V4, V5, V6에서 단순하게 쿼리가 1번 실행된다고 V6이 항상 좋은 방법인 것은 아니다.

- V4는 코드가 단순하다.  특정 주문 한건만 조회하면 이 방식을 사용해도 성능이 잘 나온다. 예를 들어서 조회한 Order 데이터가 1건이면 OrderItem을 찾기 위한 쿼리도 1번만 실행하면 된다.

- V5는 코드가 복잡하다. 여러 주문을 한꺼번에 조회하는 경우에는 V4 대신에 이것을 최적화한 V5 방식을 사용해야 한다. 예를 들어서 조회한 Order 데이터가 1000건인데, V4 방식을 그대로 사용하면, 쿼리가 총 1 + 1000번 실행된다. 여기서 1은 Order 를 조회한 쿼리고, 1000은 조회된 Order의 row 수다. V5 방식으로 최적화 하면 쿼리가 총 1 + 1번만 실행된다. 상황에 따라 다르겠지만 운영 환경에서 100배 이상의 성능 차이가 날 수 있다.

- V6는 완전히 다른 접근방식이다. 쿼리 한번으로 최적화 되어서 상당히 좋아보이지만, Order를 기준으로 페이징이 불가능하다. 실무에서는 이정도 데이터면 수백이나, 수천건 단위로 페이징 처리가 꼭 필요하므로, 이 경우 선택하기 어려운 방법이다. 그리고 데이터가 많으면 중복 전송이 증가해서 V5와 비교해서 성능 차이도 미비하다.