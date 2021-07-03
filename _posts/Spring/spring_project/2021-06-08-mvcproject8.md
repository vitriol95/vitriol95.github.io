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
@@ -352,4 +353,466 @@
@Query("delete from Reply r where r.id in (:ids)")
int bulkDeleteByRemovePost(@Param("ids") Set<Long> ids);
```
```
따라서 delete 문을 where ~ in 조건을 이용해 bulkDelete 할 수 있다. 이때 in안에 들어가는 객체 사이즈의 최대값 또한 조절해 줄 수 있다. 필자는 아래와 같이 설정해 주었다.

```yaml
  jpa:
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 1000
```

<br>

그리고 이후에 @Modifying 어노테이션에 주목해보자. 이 쿼리는 조회쿼리를 제외한 변경, 삭제, 삽입등의 작업이 일어날 때 작성해주어야 하는 어노테이션에 해당한다. 옵션은 두가지가 존재한다. clear / flushAutomatically가 그것이다.
> 둘다 default는 false이다.
<br>
이 두가지는 '동기화'를 위해 존재한다고 봐도 무방하다. 많은 경우 두가지를 true로 놓는 경우가 안전하지만, 필자는 여기서 clearAutomatically는 false로 설정하였다. 이를 true로 놓게되면 영속성 컨텍스트를 모두 비워두게 된다. 

<br>
비워 놓게되면 어떤 일이 발생할까. 우리는 이 작업이후, post의 delete를 수행해야 한다는 점을 생각해보자. 영속성 컨텍스트가 비어있으므로 다시 post를 id를 통해 검색해오는 쿼리가 발생하게 될 것이다. 즉 다음의 쿼리가 Delete 쿼리 이전에 나갈 것이다.

```text
Hibernate: 
    select
        post0_.id as id1_1_0_,
        post0_.created_date as created_2_1_0_,
        post0_.modified_date as modified3_1_0_,
        post0_.account_id as account_9_1_0_,
        post0_.description as descript4_1_0_,
        post0_.introduction as introduc5_1_0_,
        post0_.open as open6_1_0_,
        post0_.reply_count as reply_co7_1_0_,
        post0_.title as title8_1_0_,
    from
        post post0_ 
    where
        post0_.id=?
```

## 댓글 등록

댓글 등록의 경우 Ajax방식으로 Json 객체를 Post요청으로 보내며 로직은 다음을 따른다.

```java
// PostController.java
@PostMapping(value = "/posts/{id}/reply")
@ResponseBody
public ResponseEntity replyFormSubmit(@PathVariable Long id, @LoggedInUser Account account, @RequestBody NewReplyForm newReplyForm) {
    Post post = postService.getVanillaPost(id);
    postService.createNewReply(modelMapper.map(newReplyForm, Reply.class), post, account);
    return ResponseEntity.ok().build();
}
// PostService.java
public Post getVanillaPost(Long id) {
    return postRepository.findPostById(id);
}
public void createNewReply(Reply reply, Post post, Account account) {
    Account writer = accountRepository.findByEmail(account.getEmail());
    reply.setWriter(writer);
    reply.postedOn(post);
    replyRepository.save(reply);
}
// PostRepository.java
Post findPostById(Long id);
// Reply.java
/** 양방향 매핑 메서드 With Post */
public void postedOn(Post post) {
    this.post = post;
    post.replyAdd(this);
}
public void setWriter(Account account) {
    this.account = account;
}
```

바로 실제 쿼리를 살펴보도록 하자.

```text
Hibernate: 
    select
        post0_.id as id1_1_,
        post0_.created_date as created_2_1_,
        post0_.modified_date as modified3_1_,
        post0_.account_id as account_9_1_,
        post0_.description as descript4_1_,
        post0_.introduction as introduc5_1_,
        post0_.open as open6_1_,
        post0_.reply_count as reply_co7_1_,
        post0_.title as title8_1_ 
    from
        post post0_ 
    where
        post0_.id=?
Hibernate: 
    select
        account0_.id as id1_0_,
        account0_.created_date as created_2_0_,
        account0_.modified_date as modified3_0_,
        account0_.bio as bio4_0_,
        account0_.email as email5_0_,
        account0_.nickname as nickname6_0_,
        account0_.password as password7_0_,
        account0_.post_count as post_cou8_0_,
        account0_.profile_image as profile_9_0_ 
    from
        account account0_ 
    where
        account0_.email=?
Hibernate: 
    insert 
    into
        reply
        (id, created_date, modified_date, account_id, description, post_id) 
    values
        (null, ?, ?, ?, ?, ?)
Hibernate: 
    update
        post 
    set
        created_date=?,
        modified_date=?,
        account_id=?,
        description=?,
        introduction=?,
        open=?,
        reply_count=?,
        title=? 
    where
        id=?
```

1. 가장 먼저, `postService.getVanillaPost` 메서드로 연관관계 매핑이 없는 순수 post를 가져온다. 따라서 __SELECT__ 문이 나가게 된다.
2. 이후 `postService.createNewReply`를 호출한다. 이때 댓글의 작성자는 시큐리티 컨텍스트에 담겨있는 Account에 해당하며 이는 Detached상태에 해당한다. 따라서 이를 Managed 상태로 만들어주는 `accountRepository.findByEmail` __SELECT__ 쿼리가 나가게 된다.
3. Reply의 __INSERT__ 쿼리가 실행된다. 이때 account(댓글쓴이)와 post는 매핑이 되어있다.
4. 마지막으로 post의 replyCount를 1 증가시켜주고 있으므로 이에 대한 __UPDATE__ 쿼리가 나간다. 또한 양방향 매핑을 해놓는다.

---

## 댓글 삭제

댓글 삭제의 경우는 다음과 같은 로직을 따르게 된다.

```java
// PostController.java
@PostMapping(value = "/posts/{postId}/reply/{replyId}/delete")
public String deleteReplySubmit(@PathVariable("postId") Long id, @PathVariable("replyId") Long replyId) {
    Post post = postService.getVanillaPost(id);
    Reply reply = replyRepository.findReplyById(replyId);
    postService.deleteReply(reply, post);
    return "redirect:/posts/" + id;
}
// PostService.java
public Post getVanillaPost(Long id) {
    return postRepository.findPostById(id);
}
public void deleteReply(Reply reply, Post post) {
    reply.depostedOn(post);
    replyRepository.delete(reply);
}
// PostRepository.java
Reply findReplyById(Long id);
// Reply.java
public void depostedOn(Post post) {
    post.replyRemove(this);
}
// Post.java
public void replyRemove(Reply reply) {
    this.getReplies().remove(reply);
    this.replyCount--;
}
```

이의 쿼리는 다음과 같았다.

```text
Hibernate: 
    select
        post0_.id as id1_1_,
        post0_.created_date as created_2_1_,
        post0_.modified_date as modified3_1_,
        post0_.account_id as account_9_1_,
        post0_.description as descript4_1_,
        post0_.introduction as introduc5_1_,
        post0_.open as open6_1_,
        post0_.reply_count as reply_co7_1_,
        post0_.title as title8_1_ 
    from
        post post0_ 
    where
        post0_.id=?
Hibernate: 
    select
        reply0_.id as id1_2_,
        reply0_.created_date as created_2_2_,
        reply0_.modified_date as modified3_2_,
        reply0_.account_id as account_5_2_,
        reply0_.description as descript4_2_,
        reply0_.post_id as post_id6_2_ 
    from
        reply reply0_ 
    where
        reply0_.id=?
Hibernate: ---------------------(*)
    select
        replies0_.post_id as post_id6_2_1_,
        replies0_.id as id1_2_1_,
        replies0_.id as id1_2_0_,
        replies0_.created_date as created_2_2_0_,
        replies0_.modified_date as modified3_2_0_,
        replies0_.account_id as account_5_2_0_,
        replies0_.description as descript4_2_0_,
        replies0_.post_id as post_id6_2_0_ 
    from
        reply replies0_ 
    where
        replies0_.post_id=?
Hibernate: 
    update
        post 
    set
        created_date=?,
        modified_date=?,
        account_id=?,
        description=?,
        introduction=?,
        open=?,
        reply_count=?,
        title=? 
    where
        id=?
Hibernate: 
    delete 
    from
        reply 
    where
        id=?
```

쿼리는 다음과 같은 순서로 나가게 된다.
1. 앞서 댓글등록과 같은 `postService.getVanillaPost`에 의해 댓글이 속한 Post에 대한 __SELECT__ 쿼리가 하나 나가게 된다.
2. 이후 삭제할 Reply를 `replyRepository.findReplyById`를 통해 가져오게 된다.
Reply에 대한 __SELECT__ 쿼리가 나간다.
3. 세번째 쿼리는 (*)처리가 되어있다. 이는 `reply.depostedOn` 메서드를 따라가면 알 수 있다. 이는 양방향 매핑을 해제하기 위해서 존재하며, post가 가지고있는 replies 목록중에 지울 reply를 제거한다. 이 과정에서 3번째 SELECT문이 나가게 된다. Post와 reply는 fetchType이 Lazy이므로 발생하는 현상이다.
> 이는 더 좋은 방법으로 수정할 수 있을 것 같다.
4. 이후, post의 replyCount를 1내려주는 __UPDATE__ 쿼리가 나가며 댓글을 삭제하는 __DELETE__ 쿼리가 나가게 된다.

### 더 효율적인 방법으로 바꾸어보자.
(*)표시되어있는 쿼리이자, 3번 상황을 좀 더 효율적으로 바꿀 수 있을 것 같다. 처음 Post객체를 가져올때 가지고 있는 reply를 모두 페치조인하여 가져오면 저 불필요한 SELECT쿼리를 없앨 수 있다. 아래와 같이 Post를 가져오는 쿼리를 수정해 보았다.

```java
// PostRepository.java
@Query(value = "select p from Post p left join fetch p.replies where p.id = :id")
Post findPostToDeleteReplyById(@Param("id") Long id);
```
변경 이후 쿼리는 다음과 같았다.

```text
Hibernate: 
    select
        post0_.id as id1_1_0_,
        replies1_.id as id1_2_1_,
        post0_.created_date as created_2_1_0_,
        post0_.modified_date as modified3_1_0_,
        post0_.account_id as account_9_1_0_,
        post0_.description as descript4_1_0_,
        post0_.introduction as introduc5_1_0_,
        post0_.open as open6_1_0_,
        post0_.reply_count as reply_co7_1_0_,
        post0_.title as title8_1_0_,
        replies1_.created_date as created_2_2_1_,
        replies1_.modified_date as modified3_2_1_,
        replies1_.account_id as account_5_2_1_,
        replies1_.description as descript4_2_1_,
        replies1_.post_id as post_id6_2_1_,
        replies1_.post_id as post_id6_2_0__,
        replies1_.id as id1_2_0__ 
    from
        post post0_ 
    left outer join
        reply replies1_ 
            on post0_.id=replies1_.post_id 
    where
        post0_.id=?
Hibernate: 
    select
        reply0_.id as id1_2_,
        reply0_.created_date as created_2_2_,
        reply0_.modified_date as modified3_2_,
        reply0_.account_id as account_5_2_,
        reply0_.description as descript4_2_,
        reply0_.post_id as post_id6_2_ 
    from
        reply reply0_ 
    where
        reply0_.id=?
Hibernate: 
    update
        post 
    set
        created_date=?,
        modified_date=?,
        account_id=?,
        description=?,
        introduction=?,
        open=?,
        reply_count=?,
        title=? 
    where
        id=?
Hibernate: 
    delete 
    from
        reply 
    where
        id=?
```
불필요한 쿼리 하나를 줄여보았다.

---

## 유저 목록 보기

<img src="/assets/img/mvcproject/35.JPG">

로직은 다음과 같다.

```java
@GetMapping("/accounts")
public String allAccountView(@PageableDefault(size = 20, sort = "postCount", direction = Sort.Direction.DESC) Pageable pageable, @LoggedInUser Account account, Model model) {
    Page<Account> accountPage = accountRepository.findAll(pageable);
    model.addAttribute("account", account);
    model.addAttribute("accountPage", accountPage);
    return "account/all";
}
```
단순하다. findAll메서드는 JpaRepository를 그대로 사용하였다. 쿼리는 카운트를 포함한 2개가 나가게 된다.

```text
Hibernate: 
    select
        count(*) as col_0_0_ 
    from
        account account0_
Hibernate: 
    select
        account0_.id as id1_0_,
        account0_.created_date as created_2_0_,
        account0_.modified_date as modified3_0_,
        account0_.bio as bio4_0_,
        account0_.email as email5_0_,
        account0_.nickname as nickname6_0_,
        account0_.password as password7_0_,
        account0_.post_count as post_cou8_0_,
        account0_.profile_image as profile_9_0_ 
    from
        account account0_ 
    order by
        account0_.post_count desc limit ?
```
정렬을 게시글 개수의 내림차순으로 진행한것도 잘 적용이 되어있다.

---

## 게시글 검색

<img src="/assets/img/mvcproject/36.JPG">

게시글 검색의 경우 다음의 로직을 따른다.

```java
// MainController.java
@GetMapping("/search/post")
public String searchPost(@PageableDefault(size = 12, sort = "createdDate", direction = Sort.Direction.DESC) Pageable pageable, @LoggedInUser Account account, String keyword, Model model) {
    Page<Post> postPage = postRepository.findByKeyword(keyword, pageable);
    model.addAttribute(account);
    model.addAttribute("postPage", postPage);
    model.addAttribute("keyword", keyword);
    model.addAttribute("sortProperty", pageable.getSort().toString().contains("createdDate") ? "createdDate" : "replyCount");
    return "search";
}
// PostRepositoryExImpl.java
@Override
public Page<Post> findByKeyword(String keyword, Pageable pageable) {
    QPost post = QPost.post;
    JPQLQuery<Post> query = from(post).where(post.open.isTrue()
            .and(post.title.containsIgnoreCase(keyword)))
            .leftJoin(post.account, QAccount.account).fetchJoin();
    JPQLQuery<Post> pageableQuery = getQuerydsl().applyPagination(pageable, query);
    QueryResults<Post> fetchResults = pageableQuery.fetchResults();
    return new PageImpl<>(fetchResults.getResults(), pageable, fetchResults.getTotal());
}
```
QueryDsl을 사용하여 메서드를 구현했었다. 이는 페이지에 대한 쿼리와 카운트 쿼리를 가져오게 될 것이다.

```text
Hibernate: 
    select
        count(post0_.id) as col_0_0_ 
    from
        post post0_ 
    left outer join
        account account1_ 
            on post0_.account_id=account1_.id 
    where
        post0_.open=? 
        and (
            lower(post0_.title) like ? escape '!'
        )
Hibernate: 
    select
        post0_.id as id1_1_0_,
        account1_.id as id1_0_1_,
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
        account1_.profile_image as profile_9_0_1_ 
    from
        post post0_ 
    left outer join
        account account1_ 
            on post0_.account_id=account1_.id 
    where
        post0_.open=? 
        and (
            lower(post0_.title) like ? escape '!'
        ) 
    order by
        post0_.created_date desc limit ?
```

# DB를 MySQL로 교체하기

가장 먼저, build.gradle을 다음처럼 수정하였다.

```groovy
plugins {
    id 'org.springframework.boot' version '2.4.7'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'vitriol'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
    jcenter()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity5'
    implementation 'org.springframework.boot:spring-boot-starter-validation:2.4.0'
    implementation 'org.modelmapper:modelmapper:2.3.7'
    compileOnly 'org.projectlombok:lombok'

    // Lombok for Test
    testCompileOnly 'org.projectlombok:lombok'
    testAnnotationProcessor 'org.projectlombok:lombok'

    // MySQL
    runtimeOnly 'mysql:mysql-connector-java'

    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'

    /** query DSL*/
    compile "com.querydsl:querydsl-jpa"
    compile "com.querydsl:querydsl-core"
    annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api:2.2.3"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api:1.3.5"
}

def generated='src/main/generated'
sourceSets {
    main.java.srcDirs += [ generated ]
}

tasks.withType(JavaCompile) {
    options.annotationProcessorGeneratedSourcesDirectory = file(generated)
}

clean.doLast {
    file(generated).deleteDir()
}

test {
    useJUnitPlatform()
}

```
기존의 runtimeOnly였던 h2설정을 지우고, mysql을 넣어주었다.

<br>
이후 application.yml을 다음과 같이 수정해 주었다.

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mvcservice?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&useSSL=false&allowPubilcKeyRetrieval=true
    username: 계정 이름
    password: 비밀번호
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 1000
    database: mysql
    database-platform: org.hibernate.dialect.MySQL8Dialect

server:
  tomcat:
    max-http-form-post-size: 5MB
```

이제 모든 로컬환경에서의 준비를 끝냈다. 다음시간 부터는 직접 AWS에 빌드해보자!