---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(2)
date: 2021-06-02 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject1/) 에서는 Account 클래스를 만들고 이에 Spring Security를 적용하여 로그인과 회원가입을 진행해보았다. 이번시간에는 Account와 관련된 기능들을 좀 더 다듬고, MockMvc를 이용해 테스트를 진행해 보려한다.


# Account 다듬기
---

## 회원가입 - 추가 1
지난 시간, 회원가입에 추가하지 않은 로직이 존재한다. 바로 '같은 닉네임이나, 이메일이 이미 존재하는 경우 이를 못하게'끔 막아야 하는 것이다. 이는 기존 SignUpForm에서 준 제약조건으로 불가능하므로 Valiator를 이용해야한다.

<br>
@PostMapping("/sign-up")에 @ModelAttribute에 해당하는 SignUpForm객체에 또다른 validation을 진행해야한다. 필자는 여기서 Validator 인터페이스를 이용하려한다. 따라서 다음과 같은 SignUpFormValidator를 정의해 주었다.

```java
package vitriol.mvcservice.modules.account.validator;

@Component
@RequiredArgsConstructor
public class SignUpFormValidator implements Validator {

    private final AccountRepository accountRepository;

    @Override
    public boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(SignUpForm.class);
    }

    @Override
    public void validate(Object target, Errors errors) {
        SignUpForm signUpForm = (SignUpForm) target;
        if (accountRepository.existsByEmail(signUpForm.getEmail())) {
            errors.rejectValue("email", "invalid.email", new Object[]{signUpForm.getEmail()}, "이미 사용중인 이메일입니다.");
        }
        if (accountRepository.existsByNickname(signUpForm.getNickname())) {
            errors.rejectValue("nickname", "invalid.nickname", new Object[]{signUpForm.getNickname()}, "이미 사용중인 닉네임입니다.");
        }
    }
}
```
> Validator를 구현하기 위해서는 두가지 메서드를 오버라이드 해야한다. 첫번째는 어떠한 클래스에 대한 Validation을 진행할 것인지(supports)이며 두번째는 이를 어떻게 validate 할 것인지이다.

validate메서드에는 중복되는 이메일과 닉네임을 체크하여, 이미 존재한다면 error를 던져주도록 하였다. 

<br>
이렇게 구현된 SignUpFormValidator를 컨트롤러에서 Validate하는 과정에 추가해주어야 하므로 AccountController는 다음과 같이 바뀌게 된다.

```java
package vitriol.mvcservice.modules.account;

@Controller
@RequiredArgsConstructor
public class AccountController {

    private final AccountService accountService;
    private final SignUpFormValidator signUpFormValidator;

    @InitBinder("signUpForm")
    public void initBinder(WebDataBinder webDataBinder) {
        webDataBinder.addValidators(signUpFormValidator);
    }

    @GetMapping("/sign-up")
    public String signUpForm(Model model) {
        model.addAttribute("signUpForm", new SignUpForm());
        return "account/sign-up";
    }

    @PostMapping("/sign-up")
    public String signUpFormSubmit(@Valid @ModelAttribute SignUpForm signUpForm, Errors errors) {
        if (errors.hasErrors()) {
            return "account/sign-up";
        }
        Account account = accountService.createNewAccount(signUpForm);
        accountService.login(account);
        return "redirect:/";
    }
}
```
> 10번라인에 해당하는 @InitBinder 어노테이션을 사용하여 signUpFormSubmit 메서드가 진행될 때 이에 대한 Validation을 함께 진행해 주도록 하였다.

이는 @Valid 어노테이션이 수행될 때 같이 수행되게 되므로, 같은 이메일로 가입을 시도하였다면 23 라인의 if문에 걸리게 된다.

---
## 슈퍼클래스

지금까지 구현된 엔티티는 Account하나에 해당한다. 우리는 게시판을 만들기 위해 이후에는 Post, Reply등의 엔티티를 구현해야 할 것이다. 이들은 생성될때 언제 생성되었고, 언제 수정되었는지에 대한 정보를 가지고 있게 되는데, 이를 슈퍼클래스를 이용한다면 불필요한 코드를 줄여볼 수 있다.

<br>
엔티티가 언제 생성되었는지, 또 언제 마지막으로 수정되었는지에 대한 정보를 주기 위한 LocalDateTimeEntity를 다음과 같이 만들어 보았다.

```java
package vitriol.mvcservice.modules.superclass;

@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class LocalDateTimeEntity {

    @CreatedDate
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime modifiedDate;

}
```
> @MappedSuperclass어노테이션을 사용해 이 클래스가 슈퍼클래스임을 명시하고, 이에 대한 @EntityListeners 어노테이션을 적용해 주어야 한다.

그리고 이를 활성화 시키기 위해서는 스프링부트가 실행될 때, 가장 먼저 실행되는 클래스에 해당하는 'MvcServiceApplication'에 @EnableJpaAuditing을 추가해주어야 한다.

```java
package vitriol.mvcservice;

@EnableJpaAuditing
@SpringBootApplication
public class MvcServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(MvcServiceApplication.class, args);
    }
}
```

<br>
마지막으로 엔티티들이 이를 상속하면 된다.

```java
package vitriol.mvcservice.modules.account;

@Entity
@Table(name = "account")
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class Account extends LocalDateTimeEntity {

}
```

<br>
테스트 겸, 회원가입을 진행해보자 이때 만들어진 account 테이블에는 createdDate와 modifiedDate 칼럼이 존재해야한다.

<img src="/assets/img/mvcproject/5.JPG">

잘 들어와 있음을 확인했다.

---
## 프론트 뷰 수정 - 로그인된 사용자를 보여주기
가장 초기의 메인 네비게이션의 경우 로그인, 회원가입, 게시글만 활성화 되어있는 것을 확인할 수 있었다. 이는 아직 인증('authentication')이 되지않은 사용자에게 적합하며, 로그인된 사용자는 인증을 마친것으므로 새로운 뷰를 보여주어야 한다. 이를 구현해보자.

<br>
가장 먼저, 프론트뷰의 main-nav를 다음과 같이 수정해 보았다.

```html
<nav th:fragment="main-nav" class="navbar navbar-expand-lg navbar-light" style="background-color: beige">
    <a class="navbar-brand" href="/">
        <img src="/img/favicon-32x32.png" width="30" height="30" class="d-inline-block align-top" alt="">
        Vitriol
    </a>

    <div class="collapse navbar-collapse" id="navbarSupportedContent">
        <ul class="navbar-nav mr-auto">
            <li class="nav-item" sec:authorize="!isAuthenticated()" th:classappend="${#request.getRequestURI().equals('/login')}? active" >
                <a class="nav-link" href="/login">로그인<span class="sr-only">(current)</span></a>
            </li>
            <li class="nav-item" sec:authorize="!isAuthenticated()" th:classappend="${#request.getRequestURI().equals('/sign-up')}? active">
                <a class="nav-link" href="/sign-up">회원 가입</a>
            </li>
            <li class="nav-item">
                <a class="nav-link" href="#">게시글</a>
            </li>
            <li class="nav-item dropdown" sec:authorize="isAuthenticated()">
                <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
                    내 정보
                </a>
                <div class="dropdown-menu" aria-labelledby="navbarDropdown">
                    <a class="dropdown-item" href="#">글쓰기</a>
                    <a class="dropdown-item" href="#">정보 변경</a>
                    <div class="dropdown-divider"></div>
                    <form class="form-inline my-2 my-lg-0" action="#" th:action="@{/logout}" method="post">
                        <button class="dropdown-item" type="submit">로그아웃</button>
                    </form>
                </div>
            </li>
            &nbsp;&nbsp;&nbsp;
            <li class="nav-item" sec:authorize="isAuthenticated()">
                <a class="nav-link">
                    <svg th:if="${#strings.isEmpty(account?.profileImage)}" data-jdenticon-value="user111" th:data-jdenticon-value="${#authentication.name}"
                         width="30" height="30" class="rounded boarder bg-light"></svg>
                    <img th:if="${!#strings.isEmpty(account?.profileImage)}" th:src="${account.profileImage}"
                         width="30" height="30" class="rounded boarder bg-light">
                </a>
            </li>

            <li class="nav-item" sec:authorize="isAuthenticated()">
                <a class="nav-link"><span th:text="${#authentication.getPrincipal().getAccount().getNickname()}" style="color: blue">UserEmail</span>님 환영합니다.</a>
            </li>
        </ul>
        <form class="form-inline my-2 my-lg-0" th:if="${#request.getRequestURI().equals('/')}">
            <input class="form-control mr-sm-2" type="search" placeholder="게시글 검색" aria-label="Search">
            <button class="btn btn-outline-success my-2 my-sm-0" type="submit">Search</button>
        </form>
    </div>
</nav>
```
코드가 많이 길지만, 9, 12, 18, 32, 41라인만 보고 지나가면 된다.
가장먼저 스프링 시큐리티 - 타임리프의 기능인 sec:authorize 프로퍼티를 이용한 것이다. 

<br>
`sec:authorize="!isAuthenticated()"`인 경우에는 아직 사용자 인증이 되지 않았다는 것에 해당하며 이때 로그인과 회원가입이 각각 보여지게 된다. (9, 12)
<br>
이의 반대인 `sec:authorize="isAuthenticated()"`인 경우엔 인증이 된 사용자만 이용할 수 있다는 뜻으로 이때 글쓰기, 정보변경, 로그아웃이 담긴 dropdown(18라인)과 사용자 프로필 이미지 및 닉네임(32,41)에 접근할 수 있게된다.

<br>
아직 사용자가 설정한 프로필 이미지가 없다면 jdenticon으로 랜덤한 default 이미지가 부여된다.

<br>
이를 구현함에 따라 메인화면을 보여주는 컨트롤러 또한 변경해 주어야한다.

```java
package vitriol.mvcservice;

@Controller
public class MainController {

    @GetMapping("/")
    public String main(@LoggedInUser Account account, Model model) {
        if (account != null) {
            model.addAttribute(account);
        }
        return "index";
    }

    @GetMapping("/login")
    public String login() {
        return "login";
    }
}
```

여기서 @LoggedInUser라는 새로운 어노테이션을 정의하여 사용하였다. 이로부터 현재 SecurityContext에 있는 Account를 가져와서 사용할 수 있게 된다. 일단 이는 다음과 같이 구현되어 있다. 

```java
package vitriol.mvcservice.modules.account;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@AuthenticationPrincipal
public @interface LoggedInUser {

}
```
> 핵심은 @AuthenticationPrincipal 어노테이션에 해당한다. 이는 직접 SecurityContext에 있는 Principal 객체를 가져오는 것이다. 따라서 UserAccount 클래스의 인스턴스를 던져주게 될 것이다.


<br>
로그인 전후의 화면을 비교해보자.

<img src="/assets/img/mvcproject/6.JPG">

<img src="/assets/img/mvcproject/7.JPG">


# 테스트 코드 작성
---

## 메인 화면 컨트롤러 테스트 - 로그인

```java
package vitriol.mvcservice;

@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class MainControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private AccountService accountService;

    @Autowired
    private AccountRepository accountRepository;

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
        accountRepository.deleteAll();
    }

    @DisplayName("메인 화면")
    @Test
    void 메인화면() throws Exception {
        mockMvc.perform(get("/"))
                .andExpect(status().isOk())
                .andExpect(view().name("index"))
                .andExpect(unauthenticated());
    }

    @DisplayName("로그인 성공")
    @Test
    void 로그인_성공() throws Exception {
        mockMvc.perform(post("/login")
                .param("username", "vitriol95@naver.com")
                .param("password", "12345678")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/"))
                .andExpect(authenticated().withUsername("vitriol95@naver.com"));
    }

    @DisplayName("로그인 실패 - 없는 이메일")
    @Test
    void 로그인_실패_없는_이메일() throws Exception {
        mockMvc.perform(post("/login")
                .param("username", "vitriol9@naver.com")
                .param("password", "12345678")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/login?error"))
                .andExpect(unauthenticated());
    }

    @DisplayName("로그인 실패 - 비밀번호 오류")
    @Test
    void 로그인_실패_비밀번호_오류() throws Exception {
        mockMvc.perform(post("/login")
                .param("username", "vitriol95@naver.com")
                .param("password", "123456789")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/login?error"))
                .andExpect(unauthenticated());
    }
}
``` 
> SpringBootTest와 같은 Context를 공유할 수 있게끔 @AutoConfigureMockMvc 어노테이션을 적용해 주었고, 이들에 대한 테스트는 모두 mockMvc를 통해 진행해 주었다.

> 로그인이 실패한 케이스의 경우 "/login?error"페이지로 리다이렉팅이 되는데 이 과정은 스프링 시큐리티에서 제공하는 기능에 해당한다.

이어서 회원가입에 대한 컨트롤러 테스트를 진행해 보았다.

## 회원가입 컨트롤러 테스트
---
```java
package vitriol.mvcservice.modules.account;

@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;
    @Autowired
    private AccountService accountService;
    @Autowired
    private AccountRepository accountRepository;

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
        accountRepository.deleteAll();
    }

    @DisplayName("회원가입 뷰")
    @Test
    void 회원가입_뷰() throws Exception {
        mockMvc.perform(get("/sign-up"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(model().attributeExists("signUpForm"))
                .andExpect(unauthenticated());
    }

    @DisplayName("회원가입 성공")
    @Test
    void 회원가입_성공() throws Exception {
        mockMvc.perform(post("/sign-up")
                .param("email", "vitriol951@naver.com")
                .param("nickname", "vitriol951")
                .param("password", "12345678")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(view().name("redirect:/"))
                .andExpect(authenticated().withUsername("vitriol951@naver.com"));

        assertThat(accountRepository.existsByEmail("vitriol951@naver.com")).isTrue();
        Account vitriol = accountRepository.findByEmail("vitriol951@naver.com");
        assertThat(vitriol).isNotNull();
        assertThat(vitriol.getPassword()).isNotEqualTo("12345678");
        assertThat(vitriol.getCreatedDate()).isBefore(LocalDateTime.now()); // mapped superclass
    }

    @DisplayName("회원가입 실패 - 이미 존재하는 이메일")
    @Test
    void 회원가입_실패_이미_존재하는_이메일() throws Exception {
        mockMvc.perform(post("/sign-up")
                .param("email", "vitriol95@naver.com")
                .param("nickname", "vitriol123")
                .param("password", "12345678")
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(unauthenticated());
    }

    @DisplayName("회원가입 실패 - 이미 존재하는 닉네임")
    @Test
    void 회원가입_실패_이미_존재하는_닉네임() throws Exception {
        mockMvc.perform(post("/sign-up")
                .param("email", "vitriol951@naver.com")
                .param("nickname", "vitriol95")
                .param("password", "12345678")
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(unauthenticated());
    }

    @DisplayName("회원가입 실패 - 잘못된 비밀번호")
    @Test
    void 회원가입_실패_잘못된_비밀번호() throws Exception {
        mockMvc.perform(post("/sign-up")
                .param("email", "vitriol951@naver.com")
                .param("nickname", "vitriol")
                .param("password", "12")
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(unauthenticated());
    }
}
```

<br>
Account에 대한 대략적인 기능들은 모두 마무리 지었고 이에 대한 테스트 까지 완료하였다. 다음시간에는 Account에 대한 정보 수정을 가능하게끔 할 예정이며 Post라는 새로운 엔티티를 만들어 보겠다.