---
published: true   
title:  "스프링 부트와 JPA"
excerpt: "스프링 부트와 JPA 활용"

categories:
- Spring

tags:
- [Spring Boot, JPA]

toc: true
toc_sticky: true

date: 2022-08-10
last_modified_at: 2022-08-10
---


### 기능 목록
- 회원 기능
  - 회원 등록
  - 회원 조회
- 상품 기능
  - 상품 등록
  - 상품 수정
  - 상품 조회
- 주문 기능
  - 상품 주문
  - 주문 내역 조회
  - 주문 취소
- 기타 요구사항
  - 상품은 재고 관리가 필요하다.
  - 상품의 종류는 도서, 음반, 영화가 있다.
  - 상품을 카테고리로 구분할 수 있다.
  - 상품 주문시 배송 정보를 입력할 수 있다.
  
### 회원 테이블 분석
![ERD](https://user-images.githubusercontent.com/62706198/146130131-8e78b579-537d-49c0-bfe0-d6e83704b5a5.PNG)


### 연관관계 매핑 분석

- 회원과 주문: 일대다 , 다대일의 양방향 관계다. 따라서 연관관계의 주인을 정해야 하는데, 
외래 키가 있는 주문을 연관관계의 주인으로 정하는 것이 좋다. 
그러므로 Order.member를 ORDERS.MEMBER_ID 외래 키와 매핑한다.

- 주문상품과 주문: 다대일 양방향 관계다. 외래 키가 주문상품에 있으므로 주문상품이 연관관계의 주인이다.
그러므로 OrderItem.order 를 ORDER_ITEM.ORDER_ID 외래 키와 매핑한다.
- 주문상품과 상품: 다대일 단방향 관계다. OrderItem.item 을 ORDER_ITEM.ITEM_ID 외래 키와 매핑한다.
- 주문과 배송: 일대일 양방향 관계다. Order.delivery 를 ORDERS.DELIVERY_ID 외래 키와 매핑한다.
- 카테고리와 상품: @ManyToMany 를 사용해서 매핑한다.(실무에서 @ManyToMany는 사용하지 말자. 
여기서는 다대다 관계를 예제로 보여주기 위해 추가했을 뿐이다)

### 엔티티 설계시 주의점
1. 엔티티에서는 가급적 Setter를 사용하지 말자
    - Setter가 모두 열려있다면 변경 포인트가 너무 많아서, 유지보수가 어렵다. 나중에 리펙토링으로 Setter를 제거하자
2. 모든 연관관계는 지연로딩으로 설정
    - 즉시로딩(EAGER)은 예측이 어렵고, 어떤 SQL이 실행될지 추측하기 어렵다. 특히 JPQL을 실행할 때 N+1 문제가 자주 발생한다.
    - 실무에서 모든 연관관계는 지연로딩(LAZY)으로 설정해야 한다.
    - 연관된 엔티티를 함께 DB에서 조회해야 하면, fetch join 또는 엔티티 그래프 기능을 사용한다.
    - @XtoOne(OneToOne, ManyToOne)관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정해야 한다.
3. 컬렉션은 필드에서 초기화 하자.
    - 컬렉션은 필드에서 바로 초기화하는 것이 안전하다.
    - null문제에서 안전하다.
    - 하이버네이트는 엔티티를 영속화 할 때, 컬랙션을 감싸서 하이버네이트가 제공하는 내장 컬렉션으로 변경한다. 
만약 getOrders() 처럼 임의의 메서드에서 컬력션을 잘못 생성하면 하이버네이트 내부 메커니즘에 문제가 발생할 수 있다. 따라서 필드레벨에서 생성하는 것이 가장 안전하고, 코드도 간결하다.

### 어플리케이션 아키텍쳐
![아키텍쳐](https://user-images.githubusercontent.com/62706198/146131284-2693b483-6cb8-450c-9891-5493681df786.PNG)
- 계층형 구조 사용
  - controler, web: 웹 계층
  - service: 비즈니스 로직, 트랜잭션 처리
  - repository: JPA를 직접 사용하는 계층, 엔티티 매니저 사용
  - domain: 엔티티가 모여 있는 계층, 모든 계층에서 사용
- 개발 순서: 서비스, 리포지토리 계층을 개발하고, 테스트 케이스를 작성해서 검증, 마지막에 웹 계층에 적용

### 어노테이션
- @Repository: 스프링 빈으로 등록, JPA 예외를 스프링 기반 예외로 예외 변환
- @PersistenceContext: 엔티티 메니저(EntityManager) 주입
- @PersistenceUnit: 엔티티 매니저 팩토리(EntityManagerFactory) 주입
- @Transactional: 트랜잭션, 영속성 컨텍스트
  - readonly=true: 데이터의 변경이 없는 읽기 전용 메서드에 사용, 영속성 컨텍스트를 플러시 하지 않으므로 약간의 성능 향상(읽기 전용에는 다 적용)
  - 데이터베이스 드라이버가 지원하면 DB에서 성능 향상
- @Autowired: 생성자 Injection 많이 사용, 생성자가 하나면 생략 가능
- @RunWith(SpringRunner.class): 스프링과 테스트 통합
- @SpringBootTest: 스프링 부트 띄우고 테스트(이게 없으면 @Autowired 실패)
- @Transactional: 반복 가능한 테스트 지원, 각각의 테스트를 실행할 때마다 트랜잭션을 시작하고 테스트가 끝나면 트랜잭션을 강제로 롤백

### 도메인 모델 패턴
- 주문 서비스의 주문과 주문 취소 메서드를 보면 비즈니스 로직 대부분이 엔티티에 있다. 서비스계층은
단순히 엔티티에 필요한 요청을 위임하는 역할을 한다. 이처럼 엔티티가 비즈니스 로직을 가지고 객체 지향의 특성을 적극 활용하는 것을 도메인 모델 패턴이라 한다.
- 반대로 엔티티에는 비즈니스 로직이 거의 없고 서비스 계층에서 대부분의 비즈니스 로직을 처리하는 것을 트랜잭션 스크립트 패턴이라 한다.

### 폼 객체 vs 엔티티 직접 사용
- 요구사항이 정말 단순할 때는 폼 객체(MemberForm)없이 엔티티(Member)를 직접 등록과 수정 화면에서 사용해도 된다.
하지만 화면 요구사항이 복잡해지기 시작하면, 엔티티에 화면을 처리하기 위한 기능이 점점 증가한다.
- 결과적으로 엔티티는 점점 화면에 종속적으로 변하고, 이렇게 화면 기능 때문에 지저분해진 엔티티는 결국 유지보수하기 어려워진다.
- 실무에서 엔티티는 핵심 비즈니스 로직만 가지고 있고, 화면을 위한 로직은 없어야 한다.
화면이나 API에 맞는 폼 객체나 DTO를 사용하자. 그래서 화면이나 API 요구사항을 이것들로 처리하고, 엔티티는 최대한 순수하게 유지하자.

### 변경 감지와 병합(merge)
- 준영속 엔티티
  - 영속성 컨텍스트가 더는 관리하지 않는 엔티티를 말한다.
  - itemService.saveItem(book)에서 수정을 시도하는 Book 객체. Book 객체는 이미 DB에 한번 저장되어서 식별자가 존재한다. 
이렇게 임의로 만들어낸 엔티티도 기존 식별자를 가지고 있으면 준영속 엔티티로 볼 수 있다.
- 준영속 엔티티를 수정하는 2가지 방법
  - 변경 감지 기능 사용
  - 병합(merge)사용
- 변경 감지 기능 사용
  - 영속성 컨텍스트에서 엔티티를 다시 조회한 후에 데이터를 수정하는 방법
  - 트랜잭션 안에서 엔티티를 다시 조회, 변경할 값 선택 -> 트랜잭션 커밋 시점에 변경 감지(Dirty Checking)이 동작해서 데이터베이서에 UPDATE SQL 실행
  ```
  @Transactional
  void update(Item itemParam){ //itemParam: 파라미터로 넘어온 준영속 상태의 엔티티
  Item findItem = em.find(Item.class, itemParam.getId()); //같은 엔티티를 조회한다.
  findItem.setPrice(itemParam.getPrice()); //데이터를 수정한다.
  }
  ```
- 병합 사용
  - 병합은 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능이다.
  ```
    @Transactional
    void update(Item itemParam){ //itemParam: 파라미터로 넘어온 준영속 상태의 엔티티
    Item mergeItem = em.merge(item);
  ```
### 병합: 기존에 있는 엔티티  
![병합](https://user-images.githubusercontent.com/62706198/146719795-ef5f237f-acda-42c1-ae25-f03ec352efc0.PNG)
- 병합 동작 방식
  1. merge()를 실행한다.
  2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티를 조회한다. 
  (만약 1차 캐시에 엔티티가 없으면 데이터베이스에서 엔티티를 조회하고, 1차 캐시에 저장한다.)
  3. 조회한 영속 엔티티(mergeMember)에 member엔티티의 값을 채워 넣는다.
     (member 엔티티의 모든 값을 mergeMember에 밀어 넣는다. 이때 mergeMember의 "회원1"이라는 이름이 "회원명변경"으로 바뀐다.)
  4. 영속 상태인 mergeMember를 반환한다.
- 병합 시 동작 방식을 간단히 정리
  1. 준영속 엔티티의 식별자 값으로 영속 엔티티를 조회한다.
  2. 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체한다. (병합한다.)
  3. 트랜잭션 커밋 시점에 변경 감지 기능이 동작해서 데이터베이스에 UPDATE SQL이 실행
- 주의: 변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, 병합을 사용하면 모든 속성이 변경된다.
변합시 값이 없으면 null로 업데이트 할 위험도 있다.(병합은 모든 필드를 교체한다.)

### 가장 좋은 방법
- 엔티티를 변경할 때는 항상 변경 감지를 사용하라.
  - 컨트롤러에서 어설프게 엔티티를 생성하지 않는다.
  - 트랜잭션이 있는 서비스 계층에 식별자(id)와 변경할 데이터를 명확하게 전달한다.(파라미터 or dto)
  - 트랜잭션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고, 엔티티의 데이터를 직접 변경한다.
  - 트랜잭션 커밋 시점에 변경 감지가 실행된다.

### 회원 조회 API
- 응답 값으로 엔티티를 직접 외부에 노출할 경우 문제점
  - 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
  - 기본적으로 엔티티의 모든 값이 노출된다.
  - 응답 스펙을 맞추기 위해 로직이 추가된다. (@JsonIgnore, 별도의 뷰 로직 등등)
  - 실무에서는 같은 엔티티에 대해 API가 용도에 따라 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기는 어렵다.
  - 엔티티가 변경되면 API 스펙이 변한다.
  - 추가로 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어렵다. (별도의 Result 클래스 생성으로 해결)
- 결론: API 응답 스펙에 맞추어 별도의 DTO를 반환한다. 


### API 개발 고급 - 지연 로딩과 조회 성능 최적화
- 간단한 주문 조회 V1: 엔티티를 직접 노출
  - 엔티티를 직접 노출하는 것은 좋지 않다.
  - order -> member와 order -> address는 지연 로딩이다. 따라서 실제 엔티티 대신에 프록시 존지
  - jackson 라이브러리는 기본적으로 이 프록시 객체를 json으로 어떻게 생성해야 하는지 모름 -> 예외 발생
  - Hibernate5Module을 스프링 빈으로 등록하면 해결 (스프링 부트 사용중)
  - 주의: 엔티티를 직접 노출할 때는 양방향 연관관계가 걸린 곳은 꼭 한곳을 @JsonIgnore 처리 해야 한다. 안그러면 양쪽에서 서로 호출하면서 무한 루프가 걸린다.
  - 주의: 지연로딩(LAZY)를 피하기 위해 즉시로딩(EAGER)으로 설정하면 안된다. 즉시로딩 때문에 연관관계가 필요 없는 경우에도 데이터를 항상 조회해서 성능 문제가 발생할 수 있다.
- 간단한 주문 조회 V2: 엔티티를 DTO로 변환
  - 엔티티를 DTO로 변환하는 일반적인 방버이다.
  - 쿼리가 총 1 + N + N번 실행된다. (V1과 쿼리수 결과는 같다.)
  - order 조회 1 번 / order -> member 지연로딩 조회 N번 / order -> delivery 지연 로딩 조회 N번
- 간단한 주문조회 v3: 엔티티를 DTO로 변환 - 패치 조인 최적화
  - 엔티티를 패치 조인을 사용해서 쿼리 1번에 조회
  - 패치 조인으로 order -> member, order -> delivery는 이미 조회 된 상태 이므로 지연로딩 X
- 간단한 주문조회 V4: JPA에서 DTO로 바로 조회
  - 일반적인 SQL을 사용할 때 처럼 원하는 값을 선택해서 조회
  - new 명령어를 사용해서 JPQL의 결과를 DTO로 즉시 변환
  - SELECT 절에서 원하는 데이터를 직접 선택하므로 DB -> 애플리케이션 네트웍 용량 최적화(생각보다 미비)
  - 리포지토리 재사용성 떨어짐, API 스펙에 맞춘 코드가 리포지토리에 들어가는 단점
- 정리
  - 엔티티를 DTO로 변환하거나, DTO로 바로 조회하는 두가지 방법은 각각 장단점이 있다. 둘중 상황에 따라서 더 나은 방법을 선택하면 된다. 엔티티로 조회하면 리포지토리 재사용성도 좋고, 개발도 단순해진다.
- 쿼리 방식 선택 권장 순서
  1. 우선 엔티티를 DTO로 변환하는 방법을 선택한다.
  2. 필요하면 페치 조인으로 성능을 최적화 한다. -> 대부분의 성능 이슈가 해결된다.
  3. 그래도 안되면 DTO로 직접 조회하는 방법을 사용한다.
  4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용한다.


### API 개발 고급 - 컬렉션 조회 최적화
- 주문 조회 V1: 엔티티 직접 노출
  - orderItem, item 관계를 직접 초기화하면 Hibernate5Module 설정에 의해 엔티티를 JSON으로 생성한다.
  - 양방향 연관관계면 무한 루프에 걸리지 않게 한곳에 @JsonIgnore를 추가해야 한다.
  - 엔티티를 직접 노출하므로 좋은 방법은 아니다.
- 주문 조회 V2: 엔티티를 DTO로 변환
  - 지연 로딩으로 너무 많은 SQL 실행
  - SQL 실행 수: order 1번 / member, address N번 / orderItem N번 / item N번
  - 참고: 지연 로딩은 영속성 컨텍스트에 있으면 영속성 컨텍스트에 있는 엔티티를 사용하고 없으면 SQL을 실행한다. 따라서 같은 영속성 컨텍스트에서 이미 로딩한 회원 엔티티를 추가로 조회하면 SQL을 실행하지 않는다.
- 주문 조회 V3: 엔티티를 DTO로 변환 - 패치 조인 최적화
  - 패치 조인으로 SQL이 1번만 실행됨
  - distinct를 사용한 이유는 1대다 조인이 있으므로 데이터베이스 row가 증가한다. 그 결과 같은 order 엔테테의 조회 수도 증가하게 된다.
  - JPA의 distinct는 SQL에 distinct를 추가하고, 더해서 같은 엔티티가 조회되면, 애플리케이션에서 중복을 걸러준다. -> order가 컬렉션 패치 조인 때문에 중복 조회 되는 것을 막아준다.
  - 단점: 페이징 불가능. 컬렉션 패치조인은 1개만 사용할 수 있다. 컬렉션 둘 이상에 패치조인을 사용하면 데이터가 부정합하게 조회될 수 있다.
- 주문 조회 V3.1: 엔티티를 DTO로 변환 - 페이징과 한계 돌파
  - 컬렉션을 페치 종니하면 페이징이 불가능하다.
    - 컬렉션을 페치 조인하면 일대다 조인이 발생하므로 데이터가 예측할 수 없이 증가한다.
    - 일대다에서 일(1)을 기준으로 페이징을 하는 것이 목적이다. 그런데 데이터는 다(N)를 기준으로 row가 생성된다.
    - Order를 기준으로 페이징하고 싶은데, 다(N)인 OrderItem을 조인하면 OrderItem이 기준이 되어버린다.
  - 문제점 해결
    - 먼저 ToOne(OneToOne, ManyToOne) 관계를 모두 페치조인한다. ToOne 관계는 row수를 증가시키지 않으므로 페이징 쿼리에 영향을 주지 않는다.
    - 컬렉션은 지연 로딩으로 조회한다.
    - 지연 로딩 성능 최적화를 위해 hiberante.default_batch_fetch_size, @BatchSize를 적용한다.
  - 장점
    - 쿼리 호출 수가 1+N -> 1+1로 최적화 된다
    - 조인보다 DB 데이터 전송량이 최적화 된다. (Order와 OrderItem을 조인하면 Order가 OrderItem만큼 중복해서 조회된다. 이 방법은 각각 조회하므로 전송해야할 중복 데이터가 없다.)
    - 페치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만, DB 데이터 전송량이 감소한다.
    - 컬렉션 페치 조인은 페이징이 불가능 하지만 이 방법은 페이징이 가능하다.
  - 결론
    - ToOne 관계는 페치 조인해도 페이징에 영향을 주지 않는다. 따라서 ToOne관계는 페치조인으로 쿼리 수를 줄이고 해결하고, 나머지는 hiberante.default_batch_fetch_size로 최적화 하자.
  
- 주문 조회 V4: JPA에서 DTO 직접 조회
  - Query: 루트 1번, 컬렉션 N번 실행
  - ToOne(N:1, 1:1) 관계들을 먼저 조회하고, ToMany(1:N) 관계는 각각 별도로 처리한다.
  - row 수가 증가하지 않는 toOne 관계는 조인으로 최적화 하기 쉬우므로 한번에 조회하고, ToMany 관계는 최적화 하기 어려우므로 findOrderItems()같은 별도의 메서드로 조회한다.

- 주문 조회 V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화
  - Query: 루트 1번, 컬렉션 1번
  - ToOne 관계들을 먼저 조회하고, 여기서 얻은 식별자 orderId로 ToMany 관계인 OrderItem을 한꺼번에 조회
  - MAP을 사용해서 매칭 성능 향상 (O(1))

- 주문 조회 V6: JPA에서 DTO로 직접 조회, 플랫 데이터 최적화
  - Query: 1번
  - 단점
    - 쿼리는 한번이지만 조인으로 인해 DB에서 애플리케이션에 전달하는 데이터에 중복 데이터가 추가되므로 상황에 따라 V5 보다 더 느릴 수도 있다.
    - 애플리케이션에서 추가 작업이 크다.
    - 페이징 불가능
- 권장순서
  1. 엔티티 조회 방식으로 우선 접근
     1. 페치조인으로 쿼리 수를 최적화
     2. 컬렉션 최적화
        1. 페이징 필요 hibernate.default_batch_fetch_size, @BatchSize로 최적화
        2. 페이징 필요X -> 페치 조인 사용
  2. 엔티티 조회 방식으로 해결이 안되면 DTO 조회 방식 사용
  3. DTO 조회 방식으로 해결이 안되면 NativeSQL or 스프링 JdbcTemplate
  - 엔티티 조회 방식은 페치 조인이나, hibernate.default_batch_fetch_size, @BatchSize 같이 코드를 거의 수정하지 않고, 옵션만 약간 변경해서, 다야한 성능 최적화를 시도할 수 있다. 반면에 DTO를 직접 조회하는 방식은 성능을 최적화 하거나 성능 최적화 방식을 변경할 때 많은 코드를 변경해야 한다.

### API 개발 고급 - 실무 필수 최적화

- OSIV와 성능 최적화
  - Open Session In View: 하이버네이트
  - Open EntityManager In View: JPA
- OSIV ON
  ![osiv](https://user-images.githubusercontent.com/62706198/149434376-355068d1-6612-4c8d-88ce-66f8ba2b59e1.PNG)
  - spring.jpa.open-in-view: true 기본값
  - OSIV 전략은 트랜잭션 시작처럼 최초 데이터베이스 커넥션 시작 시점부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 데이터베이스 커넥션을 유지한다.
  - 지연 로딩은 영속성 컨텍스트가 살아있어야 가능하고, 영속성 컨텍스트는 기본적으로 데이터베이스 커넥션을 유지한다. 이것 자체가 큰 장점이다.
  - 그런데 이 전략은 넌무 오랜시간동안 데이터베이스 커넥션 리소스를 사용하기 때문에, 실시간 트래픽이 중요한 애플리케이션에서는 커넥션이 모자랄 수 있다. 이것은 결국 장애로 이어진다.

- OSIV OFF
  ![osivoff](https://user-images.githubusercontent.com/62706198/149436296-2aa025e6-3487-40d4-809f-7ead3300002e.PNG)
  - spring.jpa.open.in-view: false OSIV 종료
  - OSIV를 끄면 트랜잭션을 종료할 때 영속성 컨텍스트를 닫고, 데이터베이스 커넥션도 반환한다. 따라서 커넥션 리소스를 낭비하지 않는다.
  - OSIV를 끄면 지연로딩을 트랜잭션 안에서 처리해아 한다. 따라서 지금까지 작성한 많은 지연 로딩 코드를 트랜잭션 안으로 넣어야 하는 단점이 있다.
  - 결론적으로 트랜잭션이 끝나기 전에 지연 로딩을 강제로 호출해 두어야 한다.
  
- 커멘드와 쿼리 분리
  - 실무에서 OSIV를 끈 상태로 복잡성을 관리하는 좋은 방법이 있다. Command와 Query를 분리하는 것이다.
  - 크고 복잡한 애플리케이션을 개발한다면, 이 둘의 관심사를 명확하게 분리하는 선택은 유지보수 관점에서 충분히 의미 있다.
  - OrderService
    - OrderService: 핵심 비즈니스 로직
    - OrderQueryService: 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)
  - 보통 서비스 계층에서 트랜잭션을 유지한다. 두서비스 모두 트랜잭션을 유지하면서 지연 로딩을 사용할 수 있다.