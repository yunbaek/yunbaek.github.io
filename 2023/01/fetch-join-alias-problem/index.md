# Fetch join 시에 별칭을 쓰지 않는 이유


## Intro
Fetch join 시에 별칭 사용을 권장하지 않는 이유를 알아본다.
<br>
## Fetch Join
> `fetch join` 은 `SQL` 이 아닌 `JPQL` 에서 성능 최적화를 위해 연관되어 있는 모든 엔티티 혹은 컬렉션을 함께 조회하는 기능
<br>
### Fetch Join 한계
`fetch join` 이 성능 최적화에 도움을 주지만, 몇 가지 한계점이 있기 때문에 사용에 주의해야 한다.

- 별칭 사용 불가
- 카테시안 곱(Cartecian Product) 연산으로 인한 중복 발생
- 일대다 조인 2개 이상 불가
- 페이징 처리 불가

이 포스팅에서는 왜 `fetch join` 에 별칭 사용이 불가한지 알아본다.
<br>
### 별칭 사용 불가 이유
결론부터 말하면 `JPA fetch join` 에서 별칭 사용이 불가한 이유는 `데이터 일관성이 깨질 수 있기 때문`이다.<br>
- 기본적으로 JPA 에서 객체 그래프 탐색은 연관되어 있는 모든 엔티티를 가지고 온다고 가정하고 사용해야 한다.
- `fetch join` 대상에 별칭을 붙이고 필터링을 하면 연관되어 있는 모든 엔티티를 가지고 오지 못하여 데이터 정합성에 문제가 발생한다.

<br>간단한 예제를 만들어 보자

- Member.class
  ```java
  @Entity
  @Getter
  @Setter
  public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) 
    private long id;

    private String username;

    @ManyToOne
    @JoinColumn(name = "team_id")
    private Team team;
  }
  ```

- Team.class
  ```java
  @Entity
  @Getter
  @Setter
  public class Team {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
  }
  ```

- FetchJoiAliasTest.class
    ```java
    @DisplayName("fetch join 시에 fetch 대상에 조건을 걸면 의도하지 않은 결과 반환")
    @Test
    void fetchJoinTest() {
      Team team = new Team();
      team.setName("teamA");
      em.persist(team);
  
      Member member1 = new Member();
      member1.setUsername("user1");
      member1.setTeam(team);
      em.persist(member1);

      Member member2 = new Member();
      member2.setUsername("user2");
      member2.setTeam(team);
      em.persist(member2);

      em.flush();
      em.clear();

      List<Team> resultUsingAlias = em.createQuery(
              "select t from Team t join fetch t.members m where m.username = 'user1'", Team.class)
          .getResultList(); // alias 이용

      assertThat(resultUsingAlias.get(0).getMembers())
          .hasSize(1)
          .extracting("username").containsExactly("user1"); // (1)

      List<Team> result = em.createQuery(
              "select distinct t from Team t join fetch t.members", Team.class)
          .getResultList(); // alias 이용 x (중복 제거를 위해 distinct)

      assertThat(result.get(0).getMembers())
          .hasSize(1)
          .extracting("username").containsExactly("user1"); // (2)
    }
    ```
    
- (1): 위에서 말한 것과 같이 `fetch join` 은 연관된 엔티티나 컬렉션을 모두 가지고 온다는 가정 하에 사용해야 하기 때문에, `resultUsingAlias` 는  `member1, member2` 가 모두 있어야 하지만, 필터링 된 `member1` 만 조회되어 DB 와 객체의 상태의 일관성이 깨지게 된다. 
<br>
- (2): `result` 값에 대한 테스트 결과를 보면 이러한 결과를 통해 어떤 문제가 발생할 수 있는지 알 수 있다. 위의 `resultUsingAlias` 와 테스트 결과가 같은 이유는 `Team`의 식별자(id)가 같기 때문에 다시 DB에서 조회 하지 않고, 기존에 영속성 컨텍스트에 캐싱되어 있는 엔티티(`resultUsingAlias`)를 가지고 오기 때문이다.<br>
이처럼 불안정한 엔티티를 불러와 작업하게 된다면 의도치 않게 깨진 데이터가 DB에 반영되게 되어 최악의 경우 `member2` 가 DB 에서 삭제될 수도 있다.
<br>
`fetch join` 에 별칭을 사용할 수 없다고 했는데 , 위 예제에서는 사용이 가능했다.
이는 JPA 에서는 `fetch join` 의 별칭 사용을 금지했지만, 그 구현체인 `Hibernate` 에서는 별칭 사용이 가능하기 때문이다.

### 어떻게 사용해야 할까?
조회 성능 최적화를 위해 `fetch join` 에 별칭을 사용할 때에는 일관성을 해치지 않는 범위 내에서 조심스럽게 사용해야 한다. 
<br>

#### 1. Stateless Session 사용
[Stateless Session(무상태 세션)](https://docs.jboss.org/hibernate/orm/5.2/javadocs/org/hibernate/StatelessSession.html)은 데이터베이스에 대해 대량 작업을 수행하기 위해 주로 사용되는 세션으로, 1차 캐싱을 구현하지 않고, 2차 캐시와 상호작용하지 않는 특징이 있다.
```java
	@DisplayName("무상태 세션을 활용하여 캐싱 문제 해결")
	@Test
	void fetchJoinUsingStatelessSessionTest() {
		List<Team> resultUsingAlias = em.createQuery(
				"select t from Team t join fetch t.members m where m.username = 'user1'", Team.class)
			.getResultList();

		Session session = em.unwrap(Session.class);
		SessionFactory sessionFactory = session.getSessionFactory();
		List<Team> result = session.doReturningWork(connection -> {
			StatelessSession statelessSession = sessionFactory.openStatelessSession(connection);
			return statelessSession.createQuery("select distinct t from Team t join fetch t.members", Team.class)
				.getResultList();
		});
		assertThat(result.get(0).getMembers())
			.hasSize(2)
			.extracting("username").containsExactly("user1", "user2");
	}
```

#### 2. DTO 를 통한 조회
엔티티는 객체 그래프를 유지하고 DB 와 데이터 일관성을 유지해야 하는 반면, 엔티티가 아닌 `값` 으로 조회 하면 영속성 컨텍스트와 관련이 없기 때문에 안전한 사용이 가능하다.
```java
  @DisplayName("DTO를 통한 조회")
  @Test
  void joinUsingAliasWithDtoTest() {
    List<TeamInquiryDTO> resultList = em.createQuery(
        "select new com.yunbaek.springdatajpa.team.dto.TeamInquiryDTO(t.id, t.name, m.id, m.username) "
          + "from Team t join t.members m", TeamInquiryDTO.class)
      .getResultList();

    for (TeamInquiryDTO teamInquiryDTO : resultList) {
      System.out.println("teamInquiryDTO = " + teamInquiryDTO);
    }

    assertThat(resultList).hasSize(2);
  }
```
```shell
teamInquiryDTO = TeamInquiryDTO(teamId=1, teamName=teamA, memberId=1, userName=user1)
teamInquiryDTO = TeamInquiryDTO(teamId=1, teamName=teamA, memberId=2, userName=user2)
```

