---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(8) / 요약 및 최종코드
date: 2021-06-08 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject8/) 에서 게시글 삭제 쿼리 분석을 하던중, 관계가 잘못되었음 (엄밀히 따지면 불필요)을 깨닫고 이를 수정하려 한다.. Account의 양방향 매핑을 모두 끊고, count또한 세지 않는다. 각각 엔티티들의 최종 코드를 보면 다음과 같다.

---

# 엔티티들 최종 코드

```java
// Account.java
@Entity
@Table(name = "account")
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(of = "id", callSuper = false)
@Getter
@Setter
public class Account extends LocalDateTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String email;

    @Column(unique = true)
    private String nickname;
    private String password;
    private String bio;

    @Lob
    private String profileImage;

    private Long postCount = 0L;

    public void plusPostCount() {
        this.postCount++;
    }

    public void minusPostCount() {
        this.postCount--;
    }
}

// Post.java
@Getter
@Setter
@Table(name = "post")
@NoArgsConstructor
@AllArgsConstructor
@Entity
public class Post extends LocalDateTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Account account;

    private String title;
    private String introduction;

    @OneToMany(mappedBy = "post")
    private List<Reply> replies = new ArrayList<>();

    private Long replyCount = 0L;

    @Lob
    @Basic(fetch = FetchType.EAGER)
    private String description;

    private boolean open;

    public void setWriter(Account account) {
        this.account = account;
        account.plusPostCount();
    }

    public void unsetWriter(Account account) {
        account.minusPostCount();
    }


    public boolean isWriter(UserAccount userAccount) {
        return this.account.equals(userAccount.getAccount());
    }

    public boolean isWriter(Account account) {
        return this.account.equals(account);
    }

    /** 양방향 매핑 with Reply*/
    public void replyAdd(Reply reply) {
        this.getReplies().add(reply);
        this.replyCount++;
    }

    public void replyRemove(Reply reply) {
        this.getReplies().remove(reply);
        this.replyCount--;
    }

    public void makeRepliesEmpty() {
        this.replies = new ArrayList<>();
    }
}

// Reply.java
@Entity
@Table(name = "reply")
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class Reply extends LocalDateTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Account account;

    @ManyToOne(fetch = FetchType.LAZY)
    private Post post;

    @Lob
    private String description;

    public boolean isWriter(UserAccount userAccount) {
        return this.account.equals(userAccount.getAccount());
    }


    /** 양방향 매핑 메서드 With Post */
    public void postedOn(Post post) {
        this.post = post;
        post.replyAdd(this);
    }
    
    public void depostedOn(Post post) {
        post.replyRemove(this);
    }

    public void setWriter(Account account) {
        this.account = account;
    }
}
```

변화된 점은 Account는 모두 단방향 매핑을 이어가며 Post와 Reply만 양방향 매핑을 유지한다는 점이다.

---

# 최종 쿼리 효율 확인 (변경 부분만)

## 게시글 삭제

게시글 삭제의 경우 최종 로직은 다음과 같다.

```java
// PostController.java
@PostMapping("/posts/{id}/delete")
public String deletePostSubmit(@LoggedInUser Account account, @PathVariable Long id) {
    Post post = postService.getPostToDelete(id, account);
    postService.deletePost(post);
    return "redirect:/";
}

// PostService.java
public Post getPostToDelete(Long id, Account account) {
    Post post = postRepository.findDeletePostWithAccountAndRepliesById(id);
    validateWriter(account, post);
    return post;
}

public void deletePost(Post post) {
    post.unsetWriter(post.getAccount());
    List<Reply> targetReplies = post.getReplies();

    replyRepository.bulkDeleteByRemovePost(targetReplies.stream().map(Reply::getId).collect(Collectors.toSet()));
    post.makeRepliesEmpty();
    postRepository.delete(post);
}

// PostRepository.java
@Query(value = "select distinct p from Post p join fetch p.account left join fetch p.replies where p.id = :id")
Post findDeletePostWithAccountAndRepliesById(@Param("id") Long id);

// ReplyRepository.java
@Transactional
@Modifying(flushAutomatically = true)
@Query("delete from Reply r where r.id in (:ids)")
int bulkDeleteByRemovePost(@Param("ids") Set<Long> ids);

// Post.java
public void unsetWriter(Account account) {
    account.minusPostCount();
}

public void makeRepliesEmpty() {
    this.replies = new ArrayList<>();
}

// Account.java
public void minusPostCount() {
    this.postCount--;
}
```
이 흐름대로 어플리케이션이 진행될때 나가는 쿼리 흐름을 한번 살펴보자.
<br>

1. 가장 먼저 `postService.getPostToDelete`를 호출한다. 이는 `postRepository.findDeletePostWithAccountAndRepliesById`메서드를 호출하게 되는데, 이를 JPQL쿼리로 구현하였다.
2. 이 쿼리는 게시글과 함께 글쓴이, 댓글(들)을 페치조인하여 가져온다. 이때 __SELECT__ 쿼리가 한개 나갈것이다.
> 댓글쓴이는 정보가 필요없다. 왜냐하면 댓글(Reply)과 댓글쓴이(Account)는 더이상 양방향 매핑 관계가 아니기 때문이다.
3. 이후, 글쓴이가 맞는지에 대한 확인을 위한 `validateWriter`메서드를 부른다. 이 메서드는 시큐리티 컨텍스트에 있는 principal 내 account와 Post의 account를 비교한다. 우리는 이미 SELECT쿼리에 의해 'post.account'를 가져왔으므로 추가쿼리는 없다.
4. 이후 `postService.deletePost`로 들어가게 된다. 가장 먼저 `post.unsetwriter`를 호출하고 이는 `account.minusPostCount`로 인해 글쓴이의 postCount를 1 내린다. 이때 __UPDATE__ 쿼리가 발생한다.
> 이미 페치조인하여 가져왔으므로 SELECT 쿼리는 당연히 필요없다. 
5. 다음으로는 `replyRepository.bulkDeleteByRemovePost`를 호출한다. 이는 JPQL의 @Query로 작성된 Bulk Delete문에 해당한다. 이때 Where절이 id in (?) 형태인 __DELETE__ 쿼리가 발생하게 된다.
> ReplyRepository.java는 @Transactional(readOnly=true)이므로 @Transactional 어노테이션을 써주어야 하며, 삭제가 이루어지므로 @Modifying 어노테이션도 필요하다.
6. 마지막으로 `postRepository.delete`라는 SpringDataJpa 메서드에 의해 실제 post가 삭제된다. 이때 __DELETE__ 쿼리가 발생한다.

<br>
즉, 총 1개의 SELECT문과 1개의 UPDATE, 2개의 DELETE문이 발생하게 된다. 실제 쿼리도 아래와 같았다.

```text
Hibernate: 
    select
        distinct post0_.id as id1_1_0_,
        account1_.id as id1_0_1_,
        replies2_.id as id1_2_2_,
        post0_.created_date as created_2_1_0_,
        post0_.modified_date as modified3_1_0_,
        post0_.account_id as account_9_1_0_,
        post0_.description as descript4_1_0_,
        post0_.introduction as introduc5_1_0_,
        post0_.open as open6_1_0_,
        post0_.reply_count as reply_co7_1_0_,
        post0_.title as title8_1_0_,
        account1_.created_date as created_2_0_1_,
        account1_.modified_date as modified3_0_1_,
        account1_.bio as bio4_0_1_,
        account1_.email as email5_0_1_,
        account1_.nickname as nickname6_0_1_,
        account1_.password as password7_0_1_,
        account1_.post_count as post_cou8_0_1_,
        account1_.profile_image as profile_9_0_1_,
        replies2_.created_date as created_2_2_2_,
        replies2_.modified_date as modified3_2_2_,
        replies2_.account_id as account_5_2_2_,
        replies2_.description as descript4_2_2_,
        replies2_.post_id as post_id6_2_2_,
        replies2_.post_id as post_id6_2_0__,
        replies2_.id as id1_2_0__ 
    from
        post post0_ 
    inner join
        account account1_ 
            on post0_.account_id=account1_.id 
    left outer join
        reply replies2_ 
            on post0_.id=replies2_.post_id 
    where
        post0_.id=?
Hibernate: 
    update
        account 
    set
        created_date=?,
        modified_date=?,
        bio=?,
        email=?,
        nickname=?,
        password=?,
        post_count=?,
        profile_image=? 
    where
        id=?
Hibernate:
    delete 
    from
        reply 
    where
        id in (
            ? , ? , ?
        )
Hibernate:
    delete 
    from
        post 
    where
        id=?

```
이 로직을 어떻게 구현할것인가 정말 고민이 많았다. 이 부분은 프로젝트 진행중에 가장 많이 고민을 한 부분이므로 수정과정을 잠시만 적어보려한다.

### Bulk Delete문을 작성한 이유
가장 먼저 이렇게 시도해보았다. post와 reply는 양방향 매핑이 되어있으며 이때, 연관관계의 주인은 reply이다. 현재와 같은 상황에서 post객체를 지우게 되면 DB의 __참조 무결성 제약조건__ 에 어긋나게 되어 오류를 뱉어내게 된다.

<br>
따라서 Post.java를 다음과 같이 수정해 delete를 진행하려고 했었다.

```java
// Post.java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Reply> replies = new ArrayList<>();
```
참조 무결성 제약조건을 cascade로 설정하여 부모 릴레이션인 post가 삭제되면 replies가 따라서 삭제되게 된다. 또한 orphanRemoval을 true로 주어 참조가 끊어진 자식객체를 삭제하도록한다. 이렇게되면 post와 거기에 연관된 reply들은 생명주기가 같아진다.

<br>
이렇게 설정해주면 reply를 delete할 필요없이 post만 delete해주면 된다. 하지만 __이때는 딸려있는 reply각각을 delete하게 된다__ 즉, 댓글이 3개인경우 delete문이 3개가 나가게 되는 것이다.
> 일종의 1+N 문제라고 봐도 되려나..

```text
Hibernate: 
    delete 
    from
        reply 
    where
        id=?
Hibernate: 
    delete 
    from
        reply 
    where
        id=?
Hibernate: 
    delete 
    from
        reply 
    where
        id=?
```
> cascade로 제약조건을 주고 post를 지웠을 시, 댓글이 3개 달려있었다면 다음과 같이 쿼리가 나가게 된다.

<br>
필자는 이를 하나로 줄여보기 위해 bulkDelete문을 새로 정의했고 이를 사용하여 하나의 쿼리로 해결해 보았다.

```java
// ReplyRepository.java
@Transactional
@Modifying(flushAutomatically = true)
@Query("delete from Reply r where r.id in (:ids)")
int bulkDeleteByRemovePost(@Param("ids") Set<Long> ids);

```