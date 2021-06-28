---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(4)
date: 2021-06-04 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject3/) 에서는 Post 엔티티를 정의하고 이를 등록, 수정, 삭제하는 과정을 진행하였다. 이번 시간에는 저번 시간에 이어 부족한 부분들을 채워보고, 테스트를 진행한뒤, Reply엔티티까지 만들어 보겠다.

---

# Account 정보 수정관련 추가

이전 닉네임을 변경할 때, ProfileFormValidator 라는 클래스의 validate (Overriding)메서드를 이용하였다. 하지만 여기서 문제가 발생하는데, 쓰던 닉네임을 그대로 입력했을 때에도 중복체크가 되어 걸리는 것이다. 따라서 validate 메서드를 다음과 같이 수정하였다.

```java
@Component
@RequiredArgsConstructor
public class ProfileFormValidator implements Validator {

    private final AccountRepository accountRepository;

    @Override
    public boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(ProfileForm.class);
    }

    @Override
    public void validate(Object target, Errors errors) {
        ProfileForm profileForm = (ProfileForm) target;
        Account byNickname = accountRepository.findByNickname(profileForm.getNickname());

        if (changeNickname(profileForm) && byNickname != null) {
            errors.rejectValue("nickname","wrong.value","해당 닉네임을 이미 사용중입니다.");
        }
    }

    private boolean changeNickname(ProfileForm profileForm) {
        UserAccount principal = (UserAccount) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return !profileForm.getNickname().equals(principal.getAccount().getNickname());
    }
}
```
<br>
changeNickname 이라는 메서드를 추가하여 현재 사용중인 닉네임과의 일치를 확인해 보았다. 변경하기 전 상태의 로그인 되어있는 principal의 닉네임을 가져와 매칭을 진행하였다.

---

# 글 작성시 Account 객체에 대한 영속성 문제

이전, 새 글을 작성할때의 핸들러와 서비스단을 살펴보자. 한가지 문제가 존재한다.

```java
// PostController.class

@PostMapping("/new-post")
public String postFormSubmit(@LoggedInUser Account account, Model model, @Valid NewPostForm newPostForm, Errors errors) {
    if (errors.hasErrors()) {
        model.addAttribute(account);
        return "post/form";
    }
    Post post = postService.createNewPost(modelMapper.map(newPostForm, Post.class), account);
    return "redirect:/posts/" + post.getId();
}

//PostService.class

public Post createNewPost(Post newPost, Account account) {
    Post post = postRepository.save(newPost);
    post.setWriter(account);
    return post;
}

```
post가 가지고있는 외래키에 해당하는 account의 매핑을 해주는 과정에 해당한다. 이때의 Account는 어떠한 상태인지 파악이 필요하다.

<br>
늦게 알아차린 이유는 항상 회원가입한 후 바로 글 작성을 하여 테스트를 했기 때문이다. 이 때의 account는 managed 상태에 해당한다. 하지만, __로그인 한 뒤, 글쓰기를 진행할 경우__ 이때의 account는 detached 상태이므로 17번째 라인은 유효하지 않다. 즉, __프록시 객체를 던져준 꼴이된다__. 

<br>
따라서 이 Account 객체를 다시 managed 상태로 만들어 줄 필요가 있다.

```java
//PostService.class 수정
public Post createNewPost(Post newPost, Account account) {

    Account writer = accountRepository.findByEmail(account.getEmail());
    // Detached 상태를 깨워주기

    Post post = postRepository.save(newPost);
    post.setWriter(writer);
    return post;
}
```
> 4번라인처럼 Detached 상태를 벗어나게 끔 해주어야 한다.

---

# 이전 시간 핸들러들에 대한 단위 테스트 진행

## 회원 정보 수정테스트
회원 정보를 수정하는 핸들러들에 대한 테스트를 진행하기 전에, 이곳부터는 Spring Security가 적용되는 구간임을 인지해야 한다. 따라서 이전에 진행했던 (회원가입, 로그인) 테스트와는 달리 `@WithUserDetails` 어노테이션을 적용하여 컨텍스트내 Principal 객체를 주입하여 진행하려한다.

<br>
회원 정보를 변경하는 테스트는 다음과 같다.

```java
@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    /**
    이전 코드들
    */

    @BeforeEach
    void beforeEach() {

        /**
        이전 코드들
        */

        SignUpForm signUpForm2 = new SignUpForm();
        signUpForm2.setEmail("test@naver.com");
        signUpForm2.setNickname("test");
        signUpForm2.setPassword("12345678");
        accountService.createNewAccount(signUpForm2);
    }

    // ------ 설정 관련 ------ //
    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("회원 정보 변경 뷰")
    @Test
    void 회원_정보_변경_뷰() throws Exception {

        mockMvc.perform(get("/settings"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/settings"))
                .andExpect(model().attributeExists("profileForm"))
                .andExpect(model().attributeExists("account"))
                .andExpect(authenticated());
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("회원 정보 변경 - 입력값 정상")
    @Test
    void 회원_정보_변경_입력값_정상() throws Exception {

        mockMvc.perform(post("/settings")
                .param("nickname", "vitriol951")
                .param("bio", "bio")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/settings"))
                .andExpect(view().name("redirect:/settings"))
                .andExpect(flash().attributeExists("message")); 

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");

        assertThat(vitriol.getNickname()).isEqualTo("vitriol951");
        assertThat(vitriol.getBio()).isEqualTo("bio");
        assertThat(vitriol.getModifiedDate()).isAfter(vitriol.getCreatedDate());
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("회원 정보 변경 - 정상 - 닉네임 그대로")
    @Test
    void 회원_정보_변경_닉네임_그대로() throws Exception {

        mockMvc.perform(post("/settings")
                .param("nickname", "vitriol95")
                .param("bio", "bio")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/settings"))
                .andExpect(view().name("redirect:/settings"))
                .andExpect(flash().attributeExists("message"));
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("회원 정보 변경 - 실패 - 중복 닉네임")
    @Test
    void 회원_정보_변경_실패_중복_닉네임() throws Exception {

        mockMvc.perform(post("/settings")
                .param("nickname", "test")
                .param("bio", "bio")
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/settings"));
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("회원 정보 변경 - 입력값 오류")
    @Test
    void 회원_정보_변경_입력값_오류() throws Exception {

        mockMvc.perform(post("/settings")
                .param("nickname", "vitriol95vitriol95vitriol95itriol95itriol95itriol95itriol95itriol95")
                .param("bio", "bio")
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/settings"));
    }

}
```
> @WithUserDetails의 value값은 BeforeEach를 통해 미리 생성한 Account객체를 배정하였다. 두번째 param의 의미는 본 테스트 메서드를 진행할 때만 시큐리티 컨텍스트객체를 이용한다는 뜻이다. (@Before / AfterEach 과정에는 포함되지 않는다.)

게시글에 대한 테스트는 다음과 같다.

```java
@Transactional
@AutoConfigureMockMvc
@SpringBootTest
class PostControllerTest {

    @Autowired
    AccountService accountService;

    @Autowired
    AccountRepository accountRepository;

    @Autowired
    MockMvc mockMvc;

    @Autowired
    PostService postService;

    @Autowired
    PostRepository postRepository;

    @BeforeEach
    void beforeEach() {
        SignUpForm signUpForm = new SignUpForm();
        signUpForm.setEmail("vitriol95@naver.com");
        signUpForm.setNickname("vitriol95");
        signUpForm.setPassword("12345678");
        accountService.createNewAccount(signUpForm);

        SignUpForm signUpForm2 = new SignUpForm();
        signUpForm2.setEmail("vitriol95@gmail.com");
        signUpForm2.setNickname("vitriol951");
        signUpForm2.setPassword("12345678");
        accountService.createNewAccount(signUpForm2);
    }

    @AfterEach
    void afterEach() {
        postRepository.deleteAll();
        accountRepository.deleteAll();
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 작성 뷰")
    @Test
    void 포스트_작성_뷰() throws Exception {

        mockMvc.perform(get("/new-post"))
                .andExpect(view().name("post/form"))
                .andExpect(status().isOk())
                .andExpect(model().attributeExists("account"))
                .andExpect(model().attributeExists("newPostForm"));

    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 작성 - 입력값 정상")
    @Test
    void 포스트_작성_입력값_정상() throws Exception {

        MvcResult mvcResult = mockMvc.perform(post("/new-post")
                .param("title", "TestTitle")
                .param("introduction", "TestIntroduction")
                .param("description", "TestDescription")
                .param("open", String.valueOf(true))
                .with(csrf()))
                .andExpect(status().is3xxRedirection()).andReturn();

        String[] url = mvcResult.getResponse().getRedirectedUrl().split("/");
        Long id = Long.parseLong(url[url.length - 1]); // redirected id 가져오기

        Post post = postRepository.findPostWithAccountById(id);
        assertThat(post).isNotNull();
        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        assertThat(vitriol.getPosts()).contains(post); // account 내에 저장되었는가?
        assertThat(post.isWriter(vitriol)).isTrue(); // 글쓴이가 맞는가?
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 작성 - 입력값 오류")
    @Test
    void 포스트_작성_입력값_오류() throws Exception {

        mockMvc.perform(post("/new-post")
                .param("title", "TestTitle")
                .param("introduction", "TestIntroduction")
                .param("description", "") // 본문이 없음
                .param("open", String.valueOf(true))
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("post/form"));

        Post post = postRepository.findPostWithAccountById(1L);
        assertThat(post).isNull();
        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        assertThat(vitriol.getPosts()).doesNotContain(post);
    }


    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 상세보기 뷰 - 글쓴이가 봤을 때")
    @Test
    void 포스트_상세보기_뷰_글쓴이가_봤을_때() throws Exception {

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Long id = new_post(true, vitriol);

        MvcResult result = mockMvc.perform(get("/posts/"+id))
                .andExpect(status().isOk())
                .andExpect(view().name("post/view"))
                .andExpect(model().attributeExists("account"))
                .andExpect(model().attributeExists("post"))
                .andReturn();
        String body = result.getResponse().getContentAsString();
        assertThat(body).contains("수정");
        assertThat(body).contains("삭제");
    }

    @WithUserDetails(value = "vitriol95@gmail.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 상세보기 뷰 - 글쓴이 아닌 사람이 봤을 때")
    @Test
    void 포스트_상세보기_뷰_글쓴이_아닌_사람이_봤을_때() throws Exception {
        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Long id = new_post(true, vitriol);

        MvcResult result = mockMvc.perform(get("/posts/"+id))
                .andExpect(status().isOk())
                .andExpect(view().name("post/view"))
                .andExpect(model().attributeExists("account"))
                .andExpect(model().attributeExists("post"))
                .andReturn();
        String body = result.getResponse().getContentAsString();
        assertThat(body).doesNotContain("수정");
        assertThat(body).doesNotContain("삭제");
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 업데이트 뷰 - 글쓴이가 봤을 때")
    @Test
    void 포스트_업데이트_뷰_글쓴이() throws Exception {
        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Long id = new_post(true, vitriol);

        mockMvc.perform(get("/posts/" + id + "/update"))
                .andExpect(status().isOk())
                .andExpect(view().name("post/update"))
                .andExpect(model().attributeExists("newPostForm"))
                .andExpect(model().attributeExists("account"));
    }

    @WithUserDetails(value = "vitriol95@gmail.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 상세보기 뷰 - 글쓴이 아닌 사람이 접근불가")
    @Test
    void 포스트_업데이트_뷰_글쓴이_아닌_사람_접근불가() throws Exception {
        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        Long id = new_post(true, vitriol);

        mockMvc.perform(get("/posts/" + id + "/update"))
                .andExpect(status().is4xxClientError());
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 업데이트 - 입력값 정상")
    @Test
    void 포스트_업데이트_입력값_정상() throws Exception {

        Long id = vitriol95Post();
        mockMvc.perform(post("/posts/" + id + "/update")
                .param("title", "newTestTitle")
                .param("introduction", "newIntroduction")
                .param("description", "description")
                .param("open", String.valueOf(true))
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(view().name("redirect:/posts/" + id));

        Post post = postRepository.findPostWithAccountById(id);
        assertThat(post).isNotNull();
        assertThat(post.getAccount().getEmail()).isEqualTo("vitriol95@naver.com");
        assertThat(post.getTitle()).isEqualTo("newTestTitle");
        assertThat(post.getIntroduction()).isEqualTo("newIntroduction");
        assertThat(post.getDescription()).isEqualTo("description");
        assertThat(post.isOpen()).isTrue();
        assertThat(post.getModifiedDate()).isAfter(post.getCreatedDate());
    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 업데이트 - 입력값 오류")
    @Test
    void 포스트_업데이트_입력값_오류() throws Exception {

        Long id = vitriol95Post();

        mockMvc.perform(post("/posts/" + id + "/update")
                .param("title", "50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨50글자넘기면안됨")
                .param("introduction", "intro")
                .param("description", "description")
                .param("open", String.valueOf(true))
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("post/update"));

    }

    @WithUserDetails(value = "vitriol95@naver.com", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @DisplayName("포스트 제거")
    @Test
    void 포스트_제거() throws Exception {

        Long id = vitriol95Post();

        mockMvc.perform(post("/posts/" + id + "/delete")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/"));

        Post post = postRepository.findPostWithAccountById(id);
        assertThat(post).isNull();
    }

    private Long vitriol95Post() {

        Account vitriol = accountRepository.findByEmail("vitriol95@naver.com");
        return new_post(true, vitriol);
    }

    private Long new_post(boolean open, Account account) {
        Post post = new Post();
        post.setWriter(account);
        post.setDescription("description");
        post.setIntroduction("introduction");
        post.setOpen(open);
        post.setTitle("title");
        Post newPost = postService.createNewPost(post, account);
        return newPost.getId();
    }

}
```
> 역시 @WithUserDetails를 이용하여 진행해야한다. 수정과 삭제버튼의 경우는 글쓴이가 아닌 사람들에게는 보이지 않아야한다.

102번째 라인과 121번째 라인에는 실제 response body로 들어온 html파일의 컨텐츠를 검사하고 있다. 이부분을 좀 더 효율적으로 바꾸려면 html내의 요소들의 Xpath를 이용하면 더 효율적일 수 있다.

---

# Reply 객체 만들기

Reply 엔티티의 경우 필요한 프로퍼티는 다음과 같다.
1. 작성자가 누구인지 --> account
2. 어떤 post에 달려있는지 --> post
3. 본문 내용

```java
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

    private String description;

}
```

이들의 확장성을 생각하여 Post 엔티티와 Account 엔티티에도 양방향 매핑을 진행해 주었다.

```java
// Account.class 수정
@Entity
@Table(name = "account")
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(of = "id", callSuper = false)
@Getter
@Setter
public class Account extends LocalDateTimeEntity {

    /**
    이전 코드들
    */
    @OneToMany(mappedBy = "account")
    private List<Reply> replies = new ArrayList<>();

}


// Post.class 수정
@Getter
@Setter
@Table(name = "post")
@NoArgsConstructor
@AllArgsConstructor
@Entity
public class Post extends LocalDateTimeEntity {

    /**
    이전 코드들
    */
    @OneToMany(mappedBy = "post")
    private List<Reply> replies = new ArrayList<>();
}
```

이러한 엔티티 객체가 들어옴으로 인해, PostController에서 보여주던 핸들러와 뷰도 변화가 생길 것이다. 다음시간에는 앞서 말한 변화적용과 함께 Reply의 등록 및 삭제기능을 구현해 보자.