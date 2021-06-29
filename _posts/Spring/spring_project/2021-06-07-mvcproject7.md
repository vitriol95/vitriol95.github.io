---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(7)
date: 2021-06-07 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject6/) 에서는 게시글에 대한 검색기능과, 유저 목록을 보여주는 기능을 구현해 보았다. 이번 시간에는 배포이전 마지막단계로 단위테스트와 함께 쿼리 효율을 분석해보고, 필요할 경우 리팩토링을 진행할 예정이다. 또한 현재 사용중인 DB를 H2가 아닌 MySQL로 이전시킬 것이다.

---

# 단위 테스트 구현

가장 먼저, 테스트 메서드의 편의성을 위해 Account, Post, Reply를 만들어내는 AccountFactory, PostFactory, ReplyFactory 클래스를 테스트 폴더안에 만들어 두었다.

```java
// AccountFactory.java
@Component
@RequiredArgsConstructor
public class AccountFactory {

    @Autowired
    AccountRepository accountRepository;

    public Account newAccount(String nickname) {
        Account account = new Account();
        account.setEmail(nickname + "@naver.com");
        account.setNickname(nickname);
        account.setPassword("123123123");
        accountRepository.save(account);
        return account;
    }
}

// PostFactory.java
@Component
@RequiredArgsConstructor
public class PostFactory {

    @Autowired
    PostService postService;

    public Post newPost(String title, Account writer) {


        Post post = new Post();
        post.setTitle(title);
        post.setIntroduction("test");
        post.setDescription("description");
        post.setOpen(true);
        postService.createNewPost(post, writer);
        return post;
    }
}

// ReplyFactory.java
@Component
@RequiredArgsConstructor
public class ReplyFactory {

    @Autowired
    PostService postService;

    public Reply newReply(Post post, Account account) {
        Reply reply = new Reply();
        String randomString = RandomString.make(10);
        reply.setDescription(randomString);
        postService.createNewReply(reply, post, account);
        return reply;
    }
}
```

이전 테스트들은 회원 가입 및 로그인, 게시글 작성 및 수정 / 삭제에 대한 테스트를 모두 마친 상태이므로, 댓글과 관련된 컨트롤러 테스트를 진행해야한다. 이는 PostController내에 구현되어 있으므로 새로운 테스트 클래스인 PostWithReplyControllerTest 파일을 생성해 테스트를 진행하겠다.

```java
// PostWithReplyControllerTest.java
@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class PostWithReplyControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    @Autowired
    AccountService accountService;

    @Autowired
    AccountRepository accountRepository;

    @Autowired
    PostService postService;

    @Autowired
    PostRepository postRepository;

    @Autowired
    AccountFactory accountFactory;

    @Autowired
    PostFactory postFactory;

    @Autowired
    ReplyFactory replyFactory;

    @Autowired
    ReplyRepository replyRepository;

    @BeforeEach
    void beforeEach() {
        SignUpForm signUpForm = new SignUpForm();
        signUpForm.setEmail("vitriol95@naver.com");
        signUpForm.setNickname("vitriol95");
        signUpForm.setPassword("12345678");
        accountService.createNewAccount(signUpForm);
    }

    @AfterEach
    void afterEach() {
        postRepository.deleteAll();
        accountRepository.deleteAll();
    }

    @DisplayName("댓글 입력 폼")
    @Test
    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    void 댓글_입력_폼() throws Exception {

        Account account = accountFactory.newAccount("test1");
        Post post = postFactory.newPost("title", account);
        Long postId = post.getId();

        mockMvc.perform(get("/posts/" + postId))
                .andExpect(view().name("post/view"))
                .andExpect(model().attributeExists("newReplyForm"))
                .andExpect(status().isOk());
    }

    @DisplayName("댓글 작성")
    @Test
    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    void 댓글_작성() throws Exception {

        Account account = accountFactory.newAccount("test1");
        Post post2 = postFactory.newPost("title", account);
        Long postId = post2.getId();

        NewReplyForm replyForm = new NewReplyForm();
        replyForm.setDescription("aaa");

        // Ajax
        mockMvc.perform(post("/posts/" + postId + "/reply")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(replyForm))
                .with(csrf()))
                .andExpect(status().isOk());

        Reply reply = replyRepository.findByPost(post2).get(0);

        assertThat(reply).isNotNull();
        assertThat(reply.getAccount().getEmail()).isEqualTo("vitriol95@naver.com");
        assertThat(reply.getDescription()).isEqualTo("aaa");
        assertThat(reply.getPost()).isEqualTo(post2);
        assertThat(post2.getReplies()).contains(reply);
        assertThat(post2.getReplyCount()).isEqualTo(1L);

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        assertThat(vitriol.getReplyCount()).isEqualTo(1L);
    }

    @DisplayName("입력된 댓글 뷰")
    @Test
    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    void 입력된_댓글_뷰() throws Exception {

        Account test1 = accountFactory.newAccount("test1");
        Account test2 = accountFactory.newAccount("test2");
        Post post1 = postFactory.newPost("test1's post", test1);
        Reply reply1 = replyFactory.newReply(post1, test1);
        Reply reply2 = replyFactory.newReply(post1, test2);

        MvcResult mvcResult = mockMvc.perform(get("/posts/" + post1.getId()))
                .andExpect(view().name("post/view"))
                .andExpect(model().attributeExists("newReplyForm"))
                .andExpect(status().isOk()).andReturn();

        String content = mvcResult.getResponse().getContentAsString();

        assertThat(content).contains(reply1.getDescription());
        assertThat(content).contains(reply1.getAccount().getNickname());
        assertThat(content).contains(reply1.getCreatedDate().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")));
        assertThat(content).contains(reply2.getDescription());
        assertThat(content).contains(reply2.getAccount().getNickname());
        assertThat(content).contains(reply2.getCreatedDate().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")));
        assertThat(content).doesNotContain("삭제");
    }

    @DisplayName("댓글 삭제")
    @Test
    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    void 댓글_삭제() throws Exception {
        Account test1 = accountFactory.newAccount("test1");
        Post post1 = postFactory.newPost("title112", test1);

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Reply reply1 = replyFactory.newReply(post1, vitriol);

        mockMvc.perform(post("/posts/" + post1.getId() + "/reply/" + reply1.getId() + "/delete")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/posts/" + post1.getId()));

        Reply replyById = replyRepository.findReplyById(reply1.getId());
        assertThat(replyById).isNull();
        assertThat(post1.getReplies()).doesNotContain(reply1);
        assertThat(post1.getReplyCount()).isEqualTo(0L);
        assertThat(vitriol.getReplies()).doesNotContain(reply1);
        assertThat(vitriol.getReplyCount()).isEqualTo(0L);
    }
}
```

70번째 댓글작성 메서드는 Ajax요청으로 데이터를 JSON형태로 Request Body에 넣어 전달하기에 코드가 살짝 다르다.

<br>
이어서 회원 목록에 대한 테스트 코드와, 게시글 검색에 대한 테스트 코드를 작성해 보았다. 전자의 경우는 AccountControllerTest 안에, 후자의 경우는 MainControllerTest안에 구현되었다.

```java
// AccountControllerTest.java
@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    /**
    이전 코드들
    */
    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("회원 전체 검색")
    @Test
    void 회원_전체_검색() throws Exception {

        for (int i = 1; i <= 41; i++) {
            accountFactory.newAccount("testAccount" + i);
        } // 총 41 + 2(BeforeEach) 유저가 존재. 3페이지 분량 (20, 20, 3)

        mockMvc.perform(get("/accounts"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/all"))
                .andExpect(model().attributeExists("accountPage"))
                .andExpect(model().attribute("accountPage", IsIterableWithSize.iterableWithSize(20)));

        mockMvc.perform(get("/accounts?page=1"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/all"))
                .andExpect(model().attributeExists("accountPage"))
                .andExpect(model().attribute("accountPage", IsIterableWithSize.iterableWithSize(20)));

        mockMvc.perform(get("/accounts?page=2"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/all"))
                .andExpect(model().attributeExists("accountPage"))
                .andExpect(model().attribute("accountPage", IsIterableWithSize.iterableWithSize(3)));

        assertThat(accountRepository.count()).isEqualTo(43L);
    }
}

// MainControllerTest.java

@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class MainControllerTest {

    /**
    이전 코드들
    */

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("게시글 검색 - 전체 검색")
    @Test
    void 게시글_검색_전체_검색() throws Exception {

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Account creator = accountFactory.newAccount("creator");
        for (int i = 1; i <= 14; i++) {
            Post withR = postFactory.newPost("testTitle" + i, creator);
            if (i == 13) {
                replyFactory.newReply(withR, vitriol);
            }
        }

        mockMvc.perform(get("/search/post")
                .param("keyword", ""))
                .andExpect(status().isOk())
                .andExpect(view().name("search"))
                .andExpect(model().attributeExists("postPage"))
                .andExpect(model().attributeExists("keyword"))
                .andExpect(model().attributeExists("sortProperty"))
                .andExpect(model().attribute("keyword", ""))
                .andExpect(model().attribute("postPage", IsIterableWithSize.iterableWithSize(12)));

        mockMvc.perform(get("/search/post")
                .param("keyword", "")
                .param("page", "1"))
                .andExpect(status().isOk())
                .andExpect(view().name("search"))
                .andExpect(model().attributeExists("postPage"))
                .andExpect(model().attributeExists("keyword"))
                .andExpect(model().attributeExists("sortProperty"))
                .andExpect(model().attribute("keyword", ""))
                .andExpect(model().attribute("postPage", IsIterableWithSize.iterableWithSize(2)));

        // 정렬 방식
        mockMvc.perform(get("/search/post")
                .param("sort", "createdDate,desc")
                .param("keyword", ""))
                .andExpect(status().isOk())
                .andExpect(view().name("search"))
                .andExpect(model().attributeExists("postPage"))
                .andExpect(model().attributeExists("keyword"))
                .andExpect(model().attributeExists("sortProperty"))
                .andExpect(model().attribute("keyword", ""))
                .andExpect(model().attribute("postPage", IsIterableWithSize.iterableWithSize(12)))
                .andExpect(model().attribute("sortProperty", "createdDate"));

        mockMvc.perform(get("/search/post")
                .param("sort", "replyCount,desc")
                .param("keyword", ""))
                .andExpect(status().isOk())
                .andExpect(view().name("search"))
                .andExpect(model().attributeExists("postPage"))
                .andExpect(model().attributeExists("keyword"))
                .andExpect(model().attributeExists("sortProperty"))
                .andExpect(model().attribute("keyword", ""))
                .andExpect(model().attribute("postPage", IsIterableWithSize.iterableWithSize(12)))
                .andExpect(model().attribute("sortProperty", "replyCount"));

        assertThat(postRepository.count()).isEqualTo(14L);
        assertThat(replyRepository.count()).isEqualTo(1L);
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("게시글 검색 - 특정 검색어")
    @Test
    void 게시글_검색_특정_검색어() throws Exception {

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Account creator = accountFactory.newAccount("creator");

        for (int i = 1; i <= 10; i++) {
            postFactory.newPost("testTitle" + i, creator);
        }
        for (int i = 1; i <= 10; i++) {
            postFactory.newPost("vitriol", vitriol);
        }


        mockMvc.perform(get("/search/post")
                .param("keyword", "vitriol"))
                .andExpect(status().isOk())
                .andExpect(view().name("search"))
                .andExpect(model().attributeExists("postPage"))
                .andExpect(model().attributeExists("keyword"))
                .andExpect(model().attributeExists("sortProperty"))
                .andExpect(model().attribute("keyword", "vitriol"))
                .andExpect(model().attribute("postPage", IsIterableWithSize.iterableWithSize(10)));

    }
}
```

게시글 검색의 경우에는 정렬 방식에 대한 테스트 또한 진행해 보았다.

---

# 쿼리 분석하기
<br>

## 회원가입 및 로그인

이 두가지 기능의 경우 매우 단순하므로, 따로 코드를 보며 예상해보지 않고 바로 서버를 구동하며 쿼리를 살펴보겠다.
<br>

<img src="/assets/img/mvcproject/26.JPG">

가장 먼저 회원가입을 진행한 경우 쿼리는 다음과 같다.

<img src="/assets/img/mvcproject/27.JPG">

총 3개의 쿼리가 나가고 있다. 이들을 순번대로 살펴보자.

1. 회원가입 시, 중복 이메일의 가입을 막고있기에 같은 이메일이 있는지를 검사하는 SELECT 문이 나간다.
2. 또한 중복 닉네임의 가입을 막고있기에 같은 닉네임이 있는지 검사하는 SELECT 문이 나가게 된다.
> 위 두가지 제약사항은 Validator를 상속한 SignUpFormValidator에서 구현하였다.
3. 마지막으로 두 조건을 모두 만족시킬 때, Account 객체의 Insert문이 나가게 된다.
> 직관대로 쿼리가 나가고 있음을 확인해 보았다.

<br>
다음으로는 로그인의 쿼리를 분석해보자.

<img src="/assets/img/mvcproject/28.JPG">

SELECT 쿼리가 딱 1개만 나가고 있다. 살짝 의구심이 들 수 있지만 이는 SpringSecurity의 기능이 많이 포함되어있기 때문이다. 우리가 직접 구현한 기능은 `UserDetailService`의 `loadUserByUsername`메서드에 해당한다. 우리는 로그인시 Account의 email 프로퍼티를 username으로 사용하기에 이를 repository에서 불러와주는 과정만 구현하였다. 따라서 이 쿼리만 나갈 뿐, 나머지 인증과 관련된 절차는 스프링 시큐리티가 진행하게 된다.

---

## 회원 정보 수정

<img src="/assets/img/mvcproject/29.JPG">

이어서 회원 정보를 수정해보자. 기존 공백이었던 소개글만 추가하고 수정하기 버튼을 눌러보겠다.

<img src="/assets/img/mvcproject/30.JPG">

우리가 원하는 결과가 나왔다. 우리는 프로필 수정 폼에 대한 Validator인 ProfileFormValidator의 validate 메서드를 통해 중복되는 닉네임으로의 수정을 막고있다. 따라서 닉네임을 통해 유저를 검색한다.

<br>
여기서는 닉네임이 변경되지 않았으므로 SELECT 문을 통한 객체가 바로 수정해야할 객체에 해당한다. 따라서 이에 대해 곧바로 UPDATE 쿼리를 보내주고 있다.

<br>
그렇다면 닉네임을 변경한다면 어떻게 될까?

```text
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
        account0_.profile_image as profile_9_0_,
        account0_.reply_count as reply_c10_0_ 
    from
        account account0_ 
    where
        account0_.nickname=?
Hibernate: 
    select
        account0_.id as id1_0_0_,
        account0_.created_date as created_2_0_0_,
        account0_.modified_date as modified3_0_0_,
        account0_.bio as bio4_0_0_,
        account0_.email as email5_0_0_,
        account0_.nickname as nickname6_0_0_,
        account0_.password as password7_0_0_,
        account0_.post_count as post_cou8_0_0_,
        account0_.profile_image as profile_9_0_0_,
        account0_.reply_count as reply_c10_0_0_ 
    from
        account account0_ 
    where
        account0_.id=?
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
        profile_image=?,
        reply_count=? 
    where
        id=?

```

닉네임을 통해 Account를 검색하는 것은 확인했는데, 그 아래에 새롭게 id를 통해 account를 검색하고 있다. 이 이유는 다음과 같다.

1. nickname을 통해 검색한 Account는 null에 해당한다. (그래서 닉네임을 변경할 수 있는 것이다.)
2. 따라서 아직 수정할 객체가 SELECT 되지 않았으므로 이 객체를 가져와야한다.
3. 이후 업데이트를 진행한다.

---

## 글 작성

<img src="/assets/img/mvcproject/31.JPG">

가장 먼저 다음과 같이 글을 작성해 보았다. 쿼리를 직접 보기전에 어떠한 쿼리가 나가게 될지 확인해보자. 글 작성에 대한 로직은 다음과 같다.

```java
// Handler(Controller)
@PostMapping("/new-post")
public String postFormSubmit(@LoggedInUser Account account, Model model, @Valid NewPostForm newPostForm, Errors errors) {
    if (errors.hasErrors()) {
        model.addAttribute(account);
        return "post/form";
    }
    Post post = postService.createNewPost(modelMapper.map(newPostForm, Post.class), account);
    return "redirect:/posts/" + post.getId();
}

// Service
public Post createNewPost(Post newPost, Account account) {

        Account writer = accountRepository.findByEmail(account.getEmail());
//         Detached 상태인 애를 데려오기

        newPost.setWriter(writer);
        return postRepository.save(newPost);
}

// post.setWriter
public void setWriter(Account account) {
    this.account = account;
    account.postAdd(this);
}

// account.postAdd
public void postAdd(Post post) {
    this.getPosts().add(post);
    this.postCount++;
}
```
여기서 findAccountByEmail은 LAZY로 설정된 연결 객체(Post,Reply)등을 가져오지 않고 순수 Account의 프로퍼티만 가져오는 것에 해당한다. 이 로직을 통해 예상해본 쿼리는 다음과 같다.

1. 가장 먼저, Detached상태인 Account를 깨워주기 위한 __SELECT__ 문이 나갈것이다. (email을 통해서)
2. 이후 Post(게시글)에 대한 __Insert__ 문을 보내게 된다. 여기에는 연결된 account의 정보가 이미 포함되어있다.
3. 양방향 매핑 매서드인 `setWriter()`를 통해 account.postAdd()를 호출하게 되고 postCount를 1증가시켜주는 __Update__ 문이 나가게 된다.

<br>
실제 쿼리는 다음과 같았다.

```text
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
        account0_.profile_image as profile_9_0_,
        account0_.reply_count as reply_c10_0_ 
    from
        account account0_ 
    where
        account0_.email=?
Hibernate: 
    select
        posts0_.account_id as account_9_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.created_date as created_2_1_1_,
        posts0_.modified_date as modified3_1_1_,
        posts0_.account_id as account_9_1_1_,
        posts0_.description as descript4_1_1_,
        posts0_.introduction as introduc5_1_1_,
        posts0_.open as open6_1_1_,
        posts0_.reply_count as reply_co7_1_1_,
        posts0_.title as title8_1_1_ 
    from
        post posts0_ 
    where
        posts0_.account_id=?
Hibernate: 
    insert 
    into
        post
        (id, created_date, modified_date, account_id, description, introduction, open, reply_count, title) 
    values
        (null, ?, ?, ?, ?, ?, ?, ?, ?)
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
        profile_image=?,
        reply_count=? 
    where
        id=?
```
> 엇.. 내가 예상했던 쿼리보다 1개가 더 많다.

예상대로 첫번째 쿼리는 Detached 상태를 깨워주는 쿼리이며, 세번째는 Post에 대한 Insert 쿼리, 그리고 마지막은 Account에 대한 Update 쿼리가 나갔다. 두번째 쿼리는 정체가 뭘까?

<br> 
두번째 쿼리는 바로 `account.postAdd()` 메서드 떄문에 생겨난 것이다. 이는 account 객체의 getPosts()에 새로운 게시글을 넣어주는 __양방향 매핑 메서드__ 에 해당한다. 이때 account가 가지고 있는 post들을 모두 긁어와야 하므로 두번째와 같은 쿼리가 나타나게 되는 것이다.
> 이를 좀 더 효율적으로 바꿀 순 없을까?

<br>

### 좀 더 효율적인 방식으로 바꾸어 보기(글 작성)

필자는 이 두번째 쿼리를 없애버릴 수 있겠다는 생각을 했다. 처음 Detached 상태인 account를 가져올 때, 관련된 post 까지 모두 가져오면 되지 않겠는가? 따라서 서비스단에서 Detached 상태인 Account를 가져오는 쿼리를 다음과 같이 수정했다.

```java
// Service

public Post createNewPost(Post newPost, Account account) {

//        Account writer = accountRepository.findByEmail(account.getEmail());
// 기존방식
    Account writer = accountRepository.findAccountWithPostsByEmail(account.getEmail());
//  새로운 방식
    newPost.setWriter(writer);
    return postRepository.save(newPost);
}

// AccountRepository.java
@Transactional(readOnly = true)
public interface AccountRepository extends JpaRepository<Account, Long> {

    /**
    이전 코드들
    */

    @Query("select a from Account a left join fetch a.posts where a.email = :email")
    Account findAccountWithPostsByEmail(@Param("email") String email);
}
```
여러가지 방법이 있지만, 필자는 @Query를 이용해 fetch join한 결과를 가져왔다. 이렇게 되면 account 객체를 가져올때 연관된 post들이 한꺼번에 검색되어 딸려오게 된다.

<br>
이렇게 적용한 결과 쿼리는 어떻게 바뀔까?
```text
Hibernate: 
    select
        account0_.id as id1_0_0_,
        posts1_.id as id1_1_1_,
        account0_.created_date as created_2_0_0_,
        account0_.modified_date as modified3_0_0_,
        account0_.bio as bio4_0_0_,
        account0_.email as email5_0_0_,
        account0_.nickname as nickname6_0_0_,
        account0_.password as password7_0_0_,
        account0_.post_count as post_cou8_0_0_,
        account0_.profile_image as profile_9_0_0_,
        account0_.reply_count as reply_c10_0_0_,
        posts1_.created_date as created_2_1_1_,
        posts1_.modified_date as modified3_1_1_,
        posts1_.account_id as account_9_1_1_,
        posts1_.description as descript4_1_1_,
        posts1_.introduction as introduc5_1_1_,
        posts1_.open as open6_1_1_,
        posts1_.reply_count as reply_co7_1_1_,
        posts1_.title as title8_1_1_,
        posts1_.account_id as account_9_1_0__,
        posts1_.id as id1_1_0__ 
    from
        account account0_ 
    left outer join
        post posts1_ 
            on account0_.id=posts1_.account_id 
    where
        account0_.email=?
Hibernate: 
    insert 
    into
        post
        (id, created_date, modified_date, account_id, description, introduction, open, reply_count, title) 
    values
        (null, ?, ?, ?, ?, ?, ?, ?, ?)
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
        profile_image=?,
        reply_count=? 
    where
        id=?
```
쿼리 3개로 마무리 되었다. 제일 처음의 select문은 Account와 함께 연관된 post들을 모두 가져온다.
> 사실 쿼리 개수가 적어진다고 무조건 성능이 좋아지는 것은 아니다. Grafana와 같은 모니터링 툴이 있다면 효율을 더 잘 따져볼 수 있다.

<br>
🤔 그런데 '왜 `account.getPosts()` 에 새로운 객체를 추가했는데 Insert문이나 Update문이 안나가지?' 라고 생각할 수 있다. 이는 DB와 객체지향언어의 패러다임 차이를 생각해 보면 금방 알 수 있다. 이 정보는 데이터베이스가 가지고있어야할 정보가 아니기 때문이다.

---

## 글 상세보기 뷰
글을 쓰고 나면 다음과 같은 페이지로 리다이렉팅이 되는데 이때의 뷰가 글 상세보기 뷰에 해당한다.

<img src="/assets/img/mvcproject/32.JPG">

이때의 쿼리는 다음의 로직을 따라 생성된다.

```java
// Controller
@GetMapping("/posts/{id}")
public String postView(@LoggedInUser Account account, @PathVariable("id") Long id, Model model) {
    Post post = postRepository.findPostWithUserAndRepliesById(id);
    if (!post.isOpen()) {
        model.addAttribute(account);
        return "redirect:/";
    }
    model.addAttribute(account);
    model.addAttribute("post", post);
    model.addAttribute(new NewReplyForm());
    return "post/view";
}

// PostRepository.java
@Transactional(readOnly = true)
public interface PostRepository extends JpaRepository<Post, Long>, PostRepositoryEx {

    @EntityGraph(attributePaths = {"description","account","replies"}, type = EntityGraph.EntityGraphType.LOAD)
    Post findPostWithUserAndRepliesById(Long id);

    /**
    이전 코드들
    */

}
```

사실 이 쿼리에 대해서는 지난시간에 이미 진행한적이 있다. [이곳](https://vitriol95.github.io/posts/mvcproject5/)에서 상세히 다루어두었다. 따라서 쿼리는 다음과 같이 SELECT문 하나로 정리가 된다.

```text
Hibernate: 
    select
        post0_.id as id1_1_0_,
        replies1_.id as id1_2_1_,
        account2_.id as id1_0_2_,
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
        replies1_.id as id1_2_0__,
        account2_.created_date as created_2_0_2_,
        account2_.modified_date as modified3_0_2_,
        account2_.bio as bio4_0_2_,
        account2_.email as email5_0_2_,
        account2_.nickname as nickname6_0_2_,
        account2_.password as password7_0_2_,
        account2_.post_count as post_cou8_0_2_,
        account2_.profile_image as profile_9_0_2_,
        account2_.reply_count as reply_c10_0_2_ 
    from
        post post0_ 
    left outer join
        reply replies1_ 
            on post0_.id=replies1_.post_id 
    left outer join
        account account2_ 
            on post0_.account_id=account2_.id 
    where
        post0_.id=?
```

다음과 같이 Post의 글쓴이, 댓글들과 함께 댓글들의 글쓴이들이 모두 outerjoin되어 딸려오고 있다.

### 좀 더 효율적으로 바꾸어보기 (글 상세보기 뷰)

좀 더 효율적인 방법이라기 보다는 JPQL의 @Query를 이용해 좀 더 가독성 있게 바꾸어 보았다.
```java
// PostRepository.java

@Transactional(readOnly = true)
public interface PostRepository extends JpaRepository<Post, Long>, PostRepositoryEx {

    
    // @EntityGraph(attributePaths = {"description","account","replies"}, type = EntityGraph.EntityGraphType.LOAD)
    // Post findPostWithUserAndRepliesById(Long id); 

    // 대체
    @Query(value = "select distinct p from Post p join fetch p.account left join fetch p.replies where p.id = :id")
    Post findPostWithAccountAndRepliesById(@Param("id") Long id);
    
}
```
그리고 댓글이 4개 달렸을 시 위 쿼리에 의한 DB상태는 다음과 같다.
<img src="/assets/img/mvcproject/33.JPG">
> 1:N 상황에 해당하므로 댓글이 4개 달렸을시 다음과 같은 결과가 나오게 된다.

---

## 글 수정 및 삭제

### 글 수정 뷰

글을 수정하는 경우 뷰는 아래와 같다. 

<img src="/assets/img/mvcproject/34.JPG">

이때 발생하는 쿼리는 다음과 같은 로직을 따른다.
```java
// Controller
@GetMapping("/posts/{id}/update")
public String updatePostFormView(@LoggedInUser Account account, @PathVariable Long id, Model model) {
    Post post = postService.getPostToUpdate(id, account);
    model.addAttribute(account);
    NewPostForm map = modelMapper.map(post, NewPostForm.class);
    model.addAttribute("newPostForm", map);
    return "post/update";
}

// Service
public Post getPostToUpdate(Long id, Account account) {
    Post post = postRepository.findPostWithAccountById(id);
    validateWriter(account, post);
    return post;
}

private void validateWriter(Account account, Post post) {
    if (!post.isWriter(account)) {
        throw new AccessDeniedException("작성자가 아닙니다.");
    }
}

// Repo
@EntityGraph(attributePaths = "account",type = EntityGraph.EntityGraphType.LOAD)
Post findPostWithAccountById(Long id);
```

실질적으로 쿼리는 `postService.getPostToUpdate`에 의해 하나가 발생할 것이다. 이는 `postRepository.findPostWithAccountById`를 호출하여 Post객체와 연관된 Account(글쓴이)를 함께 불러오게 된다.
> 여기서 댓글까지 불러올 필요가 없다.

<br>
이후 `validateWriter()` 메서드를 통해 현재 스프링 시큐리티 컨텍스트에 담긴 Account가 글쓴이와 맞는지 확인하는 과정을 거친다. 이때 발생하는 쿼리는 없을 것이다.

<br>
실제 쿼리는 아래와 같았다.

```text
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
        account1_.profile_image as profile_9_0_1_,
        account1_.reply_count as reply_c10_0_1_ 
    from
        post post0_ 
    left outer join
        account account1_ 
            on post0_.account_id=account1_.id 
    where
        post0_.id=?
```
이 SELECT문 하나만 나가게 되고, modelMapper에 의해 DTO로 변경되어 뷰에 그려지게 된다.

### 글 수정 POST 요청

가장 먼저 쿼리는 다음과 같은 로직을 따라 생성될 것이다.

```java
// Controller
@PostMapping("/posts/{id}/update")
public String updatePostFormSubmit(@LoggedInUser Account account, @PathVariable Long id, Model model, @Valid NewPostForm newPostForm, Errors errors) {

    Post post = postService.getPostToUpdate(id, account);

    if (errors.hasErrors()) {
        model.addAttribute(account);
        return "post/update";
    }

    model.addAttribute(account);
    postService.updatePost(post, newPostForm);
    return "redirect:/posts/" + id;
}

// Service 
public Post getPostToUpdate(Long id, Account account) {
    Post post = postRepository.findPostWithAccountById(id);
    validateWriter(account, post);
    return post;
}

public void updatePost(Post post, NewPostForm newPostForm) {
    modelMapper.map(newPostForm, post);
}
```


