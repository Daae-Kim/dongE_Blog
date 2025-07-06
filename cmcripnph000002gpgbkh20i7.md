---
title: "Jpa 영속성 컨텍스트 및 이점"
datePublished: Sun Jul 06 2025 10:16:55 GMT+0000 (Coordinated Universal Time)
cuid: cmcripnph000002gpgbkh20i7
slug: jpa
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1751796895847/c044b5e1-a6ff-48e3-bfb7-3aeb49924d02.png
tags: orm, jpa, spring-data-jpa

---

## 영속성 컨텍스트란?

영속성 컨텍스트란 엔티티를 영구 저장하는 환경으로 JPA가 엔티티를 메모리에서 관리하는 공간이다. 엔티티 매니저 팩토리를 통해서 고객의 요청이 올때마다 엔티티 매니저를 생성하고, 이때 엔티티 매니저와 1:1로 영속성 컨텍스트가 생긴다.

```java
EntityManaver.persist(entity);
```

위의 코드에서 entity는 persist 됨과 동시에 영속성 컨텍스트에서 관리하게된다.

영속성 컨텍스트에서 관리한다는것은 DB 에 저장한다는 말은 아니다.

entity를 DB에 바로 저장하지 않고, 영속성 컨텍스트를 DB와 애플리케이션 중간에 두고 사용하는 이유는 뭘까?

## 영속성 컨텍스트의 이점 5가지

JPA는 ORM으로써 객체지향 프로그래밍과 관계형 DB 사이의 간극을 줄이기 위해 중간다리로서 등장하였다. 영속성 컨텍스트를 사용하게 되면 다음과 같은 5가지 이점을 얻을 수 있다.

1. **1차 캐시**
    
    반복적인 엔티티 조회시 1차 캐시에 저장하여 DB 로의 반복적인 접근을 최소화 한다. 만약 DB에는 조회하려는 데이터가 있고, 1차 캐시에는 없는 상태라면 조회 쿼리를 날려 조회한 결과를 1차 캐시에 저장해둔다. 만약 조회하려는 데이터가 1차 캐시에 존재한다면 DB 조회 쿼리를 날리지 않고, 1차 캐시에서 데이터를 가져온다.
    
    ![코드와 실행 결과 (persist 이후 조회시 DB 조회쿼리를 날리지 않고, 1차 캐시(영속성 컨텍스트)에서 데이터를 가져온다.)](https://cdn.hashnode.com/res/hashnode/image/upload/v1751794914131/59ea6994-893d-4ee7-b4e0-c34608d8aab0.png align="left")
    
    id 가 101L인 member 를 persist 하고 find 했을때, 조회 쿼리가 나가지 않고, 1차 캐시에 저장해둔 findMemberId와 findMemberName을 가져온것을 볼 수 있다.
    
2. **동일성 보장**
    
    같은 트랜잭션 안에서 같은것을 비교하면 true 가 나온다. 마치 컬렉션에서 객체를 비교하는것과 같이 사용할 수 있다.
    
    ```java
    Member a = em.find(Member.class, "member1");
    Member b = em.find(Member.class, "member1");
    System.out.println(a == b); // 동일성 비교 true
    ```
    
3. **트랜잭션을 지원하는 쓰기 지연**
    
    버퍼링을 모아서 write하는 이점을 얻을 수 있다.
    
    ```java
    EntityManager em = emf.createEntityManager();
    EntityTransaction transaction = em.getTransaction(); //엔티티 매니저는 데이터 변경시 트랜잭션을 시작해야 한다.
    transaction.begin(); // [트랜잭션] 시작 
    em.persist(memberA);
    em.persist(memberB); //여기까지 INSERT SQL을 데이터베이스에 보내지 않는다.
    transaction.commit(); // [트랜잭션] 커밋 : 커밋하는 순간 데이터베이스에 INSERT SQL을 보낸다.
    ```
    
    em.persist(memberA) 와 em.persist(memberB) 2개의 persist를 쓰기 지연 SQL 저장소에 저장해두었다가, commit하는 순간 한꺼번에 DB에 insert 쿼리를 보낸다.
    
4. **Dirty Checking, 변경 감지**
    
    JPA를 사용하면 객체 변경시 update 명령만을 실행하고 저장하는 명령을 실행하지 않는다. 그 이유는 영속성 컨텍스트에서 commit 시의 엔티티와 기존에 있었던 스냅샷을 비교해 변경된 사항이 있으면 update 쿼리를 쓰기 지연 SQL 저장소에 저장해두고, DB에 저장하기 때문이다.
    
    스냅샷은 엔티티를 영속성 컨텍스트에 처음 저장해두었을때 상태를 스냅샷으로 남겨둔다. 트랜잭션이 커밋되는 시점에 flush()가 호출되면서 이 스냅샷과 엔티티를 비교하여 변경사항을 DB에 저장한다.
    
5. **지연로딩**
    
    지연로딩은 필요한 시점에 연관된 객체의 데이터를 한꺼번에 불러오는것이다.
    

## 영속성 컨텍스트 제어하기 - flush, clear, detach

### flush() : 물을 내리다, 씻어 내리다 → 데이터를 일괄적으로 처리하는 작업

변경 감지에서 트랜잭션이 커밋되는 시점에 flush()가 호출된다고 했다. flush는 영속성 컨텍스트의 변경내용을 데이터 베이스에 반영하는 역할을 한다. 또한 영속성 컨텍스트의 변경 내용을 데이터 베이스에 동기화 하는 과정이다. JPQL 쿼리를 실행하면 DB와 동기화를 위해 항상 flush 가 자동으로 호출된다.

### em.detach(entity)

특정 엔티티를 준영속 상태로 전환한다. 준영속 상태란 영속 상태의 엔티티가 영속성 컨텍스트에서 분리된 상태를 말한다. 영속성 컨텍스트에서 분리되면 영속성 컨텍스트가 제공하는 기능(1차 캐시, 동일성 보장, 변경 감지, 쓰기지연등)을 사용하지 못한다.

### em.clear()

영속성 컨텍스트를 완전히 초기화 하는 명령어이다. 영속성 컨텍스트를 완전히 초기화 한다는것은 1차 캐시에 저장되어있는 데이터를 모조리 초기화 한다는 말이다.