---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(1)
date: 2021-06-01 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 프로젝트 목표
---
스프링 부트를 이용하여 간단한 게시판 페이지를 만들어 보려한다. 그리고 이를 AWS의 EC2를 이용하여 직접 라이브 서버에 띄워보는 것을 목표로 한다.

# 초기 설정
---
JDK 버전은 11에 해당하며, 스프링 부트의 버전은 2.4.7을 사용했다. 불러온 라이브러리들은 __lombok, devtool, web, dataJpa__ 마지막으로 템플릿으로 사용할 __thymeleaf__ 까지 가져오도록 하였다.
> 개발단계에서는 데이터베이스로 h2를 사용하고, 이를 실제로 띄울 때에는 MySQL이나 MariaDB를 사용할 예정이다.

```groovy
// build.gradle
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
    compileOnly 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()
}
```
> Repository에 jcenter()또한 추가해 주었다. Intellij 사용시 Enable Annotation Processor를 체크해주는 것 또한 잊지 말자.

## index 확인
이후, 컨트롤러 체크를 위해 index페이지를 만들어 보았고(resources -> templates -> index.html) 간단한 컨트롤러를 만들어 실행시켜 보았다.

```java
package vitriol.mvcservice;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class MainController {

    @GetMapping("/")
    public String main() {
        return "index";
    }

}
```

```html
<!-- index.html 파일에 해당한다. 자주쓸 head나 네비게이션들은 fragment처리를 해두었다. -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://thymeleaf.org/extras/spring-security">
<head th:replace="fragments.html :: head"></head>
<body>
    <nav th:replace="fragments.html :: main-nav"></nav>
</body>
</html>
```
> 프론트에 관련된 프로젝트가 아니기에, 이에 대한 설명은 최소화 하고 넘어가도록 하겠다. 나중에 spring Security적용을 할것이므로 이에 대한 프론트 태그(4라인)도 미리 추가해 두었다.

<br>
어플리케이션을 시작하고, 메인화면("/")이 잘 실행되는지 확인해보자.

<img src="/assets/img/mvcproject/1.JPG">

> 순조롭게 진행됨을 확인할 수 있다. 아직은 main-nav만 보인다.

---
# 엔티티 만들기 - 사용자
'사용자' 즉, Account 엔티티는 다음과 같은 역할을 한다.

1. 로그인된 사용자만 글을 작성하고, 볼 수 있다. 댓글도 마찬가지이다.
2. 가입할때는 이메일 주소와 비밀번호, 사용할 닉네임을 작성해야한다.
3. 다른 프로퍼티로는 ,프로필 이미지를 지정할 수 있으며 짧은 소개글 또한 적을 수 있다.

> 사용자의 'Authentication'과정이 필요하게 되므로 Spring security가 필요하게 된다. 또한 이를 저장해둘 h2의 플러그인도 필요하다.

Account 객체를 만들기 전 다음과 같이 build.gradle 설정을 추가해준다.
```groovy
dependencies {
    // 이전 build.gradle에 추가
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity5'
    implementation 'org.springframework.boot:spring-boot-starter-validation:2.4.0'
    implementation 'org.modelmapper:modelmapper:2.3.7'
    runtimeOnly 'com.h2database:h2:1.3.176' // h2
    testImplementation 'org.springframework.security:spring-security-test'
}
```
> 클래스간의 변환을 자유롭게 해주는 model mapper또한 가져왔다.

<br>
그리고 h2데이터베이스에 관한 설정을 해줄 application.yml파일도 다음과 같이 맞추어준다.
```yaml
# application.yml
spring:
  h2:
    console:
      enabled: true
      settings:
        web-allow-others: true
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: create
    show-sql: true
  datasource:
    driver-class-name: org.h2.Driver
    username: sa
    url: jdbc:h2:mem:testdb
    password:
```

<br> 
준비는 모두 끝났으므로, 앞서 이야기한 사항들을 적용해 Account 객체를 만들어보자.

```java
package vitriol.mvcservice.modules.account;

@Entity
@Table(name = "account")
@EqualsAndHashCode(of = "id")
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class Account {

    @Id
    @GeneratedValue
    private Long id;

    @Column(unique = true)
    private String email;

    @Column(unique = true)
    private String nickname;
    private String password;
    private String bio;

    @Lob
    @Basic(fetch = FetchType.EAGER)
    private String profileImage;
    
}
```
특별한 점이 있다면 31번째 라인인 profileImage 프로퍼티에 대한 설정이다. 프로필이미지는 대용량값에 해당하기에 @Lob 을 붙혀주었으며, 이는 항상 사용자를 보일때 같이 보여줄 예정이므로 Eager로 Fetch하도록 하였다.
> 그리고, 중복된 이메일이나 닉네임으로 가입할 수 없도록 unique옵션을 주었다.

---
## 회원가입 기능 구현
가장 먼저, 회원가입에 해당하는 뷰는 다음과 같다. URI는 /sign-up에 해당한다.

<img src="/assets/img/mvcproject/2.JPG">

여기에서는 사용자의 이메일, 닉네임 패스워드를 입력받아야한다. 이메일과 패스워드는 로그인을 할 때에 사용되는 정보에 해당하며, 중복된 이메일과 닉네임이 있을시에는 가입을 못하게 막아야한다.

<br>
뷰에서 보여지는 폼으로부터 전달될 정보를 담을 객체가 필요하게 된다. 즉 Dto가 필요하다. 필자는 이를 'SignUpForm'이라는 클래스로 만들어보았다.

```java
package vitriol.mvcservice.modules.account.form;

@Data
public class SignUpForm {

    @Email
    @NotBlank
    private String email;

    @NotEmpty
    @Length(min = 3, max = 20)
    @Pattern(regexp = "^[ㄱ-ㅎ가-힣a-z0-9_-]{3,20}$")
    private String nickname;

    @NotBlank
    @Length(min = 8, max = 50)
    private String password;
}
```
> javax.validation 의 기능들을 이용하여 제약을 두고있다. 각각의 형식이나 길이를 제한하고 있으며, 닉네임의 경우엔 정규표현식을 이용하여 '한글,영어,숫자,언더바 및 스코어'까지 3~20글자 허용한다.

<br>
이제 실제 /sign-up 으로 Post요청이 들어올 시 컨트롤러를 구현해보자. 

```java
package vitriol.mvcservice.modules.account;

@Controller
@RequiredArgsConstructor
public class AccountController {

    private final AccountService accountService;

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
> @PostMapping으로 받아와야하는 인자들은 www-form-urlencoded의 형태로 들어오는 것에 해당하기에 @ModelAttribute를 사용해주었고, 이 과정에서 입력값으로 인해 발생하는 오류를 바인딩 해주는 validation의 Error를 함께 가져와주었다.

<br>
에러가 없다면 폼을 서비스단으로 넘겨 회원가입을 진행시키고, 이후에 자동으로 로그인 되도록 하였다. (32~33라인) 이제 서비스단의 메서드를 확인해보자.

```java
package vitriol.mvcservice.modules.account;

@Service
@RequiredArgsConstructor
@Transactional
public class AccountService {

    private final AccountRepository accountRepository;
    private final BCryptPasswordEncoder passwordEncoder;
    private final ModelMapper modelMapper;

    public Account createNewAccount(SignUpForm signUpForm) {
        signUpForm.setPassword(passwordEncoder.encode(signUpForm.getPassword()));
        Account account = modelMapper.map(signUpForm, Account.class);
        return accountRepository.save(account);
    }
}
```
> 역시 AccountRepository또한 필요하다. 이는 JpaRepository를 extend한 인터페이스로 정의해 주었기에 기본 기능으로 save를 가지고 있다.

<br>
createNewMethod의 경우는 회원가입 폼을 전달받아 패스워드를 인코딩해주고, 이를 실제 Account클래스로 매핑시켜준뒤, 저장소에 저장하는 기능만 가지고 있다. 

<br>
이때 BCrpytPasswordEncoder와 ModelMapper를 주입받고 있는데, 이 주입받는 객체 역시 스프링 빈으로 등록 되어 있어야 한다. 이는 AppConfig라는 클래스에 정의해 두었다.

```java
package vitriol.mvcservice.config;

@Configuration
public class AppConfig {

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public ModelMapper modelMapper() {
        ModelMapper modelMapper = new ModelMapper();
        modelMapper.getConfiguration()
                .setDestinationNameTokenizer(NameTokenizers.UNDERSCORE)
                .setSourceNameTokenizer(NameTokenizers.UNDERSCORE);
        return modelMapper;
    }

}
```
> modelMapper의 경우를 보면 Tokenizer 단위를 "_" 단위로 끊고있다. 이는 데이터베이스의 네이밍 방식때문에 그렇다. 자바의 경우는 CamelCase와 다른방식에 해당한다.

<br>
다시 서비스로 돌아와서 이번엔 로그인 시켜주는 메서드를 만들어보자.

---
## 로그인 기능 구현
로그인 기능을 구현할 때, 가장 먼저 스프링 시큐리티 설정을 진행해주어야 한다. 이에 대한 Configuration을 해주는 클래스인 SecurityConfig라는 클래스를 다음과 같이 생성하였다.

```java
package vitriol.mvcservice.config;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(WebSecurity web) throws Exception {

        web
                .ignoring()
                .mvcMatchers("/img/**","/h2-console/**")
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations());

    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .authorizeRequests()
                .mvcMatchers("/", "/login", "/sign-up").permitAll()
                .anyRequest()
                .authenticated();

        http
                .formLogin()
                .loginPage("/login").permitAll();

        http
                .logout()
                .logoutSuccessUrl("/");

        http
                .headers().frameOptions().disable();
    }
}
```
> 가장먼저 메인페이지("/")와 로그인, 회원가입 기능은 모두 접근가능하도록 열어두었다. 이외에는 모두 인증과정(로그인)이 필요하다.

<br>
필자의 경우는 시큐리티가 제공하는 로그인 페이지말고, 직접 커스텀한 로그인페이지를 사용할 것이기에 26 ~ 28 라인에서 이를 직접 정의해주었다.

> 추가로, 34~35라인은 /h2-console에 접근하기 위해 프레임옵션을 꺼둔것에 해당한다. 

<br>
또한 로그인 뷰 페이지는 다음과 같다. URI는 /login에 해당한다.

<img src="/assets/img/mvcproject/3.JPG">

이때 전달받은 두 값으로 부터 실제 Account를 판별하는 과정이 필요하다. 우리는 Authentication 진행할 객체가 Account에 해당하므로 security.core.userdetails의 User를 상속받은 인증객체가 필요하다. 필자는 이를 'UserAccount'라는 클래스로 만들어 두었다.

```java
package vitriol.mvcservice.modules.account;

@Getter
public class UserAccount extends User {

    private Account account;

    public UserAccount(Account account) {
        super(account.getEmail(), account.getPassword(), List.of(new SimpleGrantedAuthority("ROLE_USER")));
        this.account = account;
    }
}
```
> User 클래스를 상속받았으므로, 이것이 Principal 객체에 해당하게 된다.

<br>
이후, 실제 Authentication을 진행할 때 이러한 principal을 가져오기 위한 메서드가 필요하다. 우리는 로그인시 입력한 email값을 기준으로 이러한 principal들을 불러와야 한다. 이 기능은 UserDetailService의 loadByUsername에 해당하며, 필자는 이를 AccountService에 오버라이딩 하여 진행하였다.

```java
@Service
@RequiredArgsConstructor
@Transactional
public class AccountService implements UserDetailsService {

    private final AccountRepository accountRepository;
    private final BCryptPasswordEncoder passwordEncoder;
    private final ModelMapper modelMapper;

    public Account createNewAccount(SignUpForm signUpForm) {
        signUpForm.setPassword(passwordEncoder.encode(signUpForm.getPassword()));
        Account account = modelMapper.map(signUpForm, Account.class);
        return accountRepository.save(account);
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Account account = accountRepository.findByEmail(username);
        if (account == null) {
            throw new UsernameNotFoundException(username);
        }
        return new UserAccount(account); // principal
    }

    public void login(Account account) {
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(new UserAccount(account), account.getPassword(), List.of(new SimpleGrantedAuthority("ROLE_USER")));
        SecurityContext context = SecurityContextHolder.getContext();
        context.setAuthentication(token);
    }
}
```
리포지토리는 다음과 같은 메서드들을 가지고 있다.
```java
package vitriol.mvcservice.modules.account;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.transaction.annotation.Transactional;

@Transactional(readOnly = true)
public interface AccountRepository extends JpaRepository<Account, Long> {

    boolean existsByEmail(String email);
    boolean existsByNickname(String nickname);
    Account findByEmail(String email);
}

```
<br> 
17라인부터 시작하는 loadByUsername 메서드를 한번 살펴보자. 패스워드가 맞는지에 대한 여부는 스프링 시큐리티가 진행해주고 있으므로 우리는 그에 맞는 principal 객체를 전달해주어야 한다.

1. 로그인 폼으로부터 입력받은 이메일 정보를 accountRepository에서 찾게 된다.
2. 만약 그러한 이메일이 존재하지 않을시, 예외를 던져준다.
3. 그러한 이메일이 있고, 패스워드가 맞다면 Principal 객체를 반환한다. 이는 앞서 우리가 만들어둔 UserAccount 클래스에 해당한다.
> 이 Principal 객체는 내부에 Account를 가지고있다. 따라서 우리는 SecurityContextHolder를 통해 로그인된 유저의 이메일, 닉네임등을 가져올 수 있다.

<br>
이후 25라인부터 시작되는 로그인을 살펴보자. 

1. SecurityContext에 넣을 Token 객체를 생성해 주어야 한다. 이때의 파라미터는 Principal과 패스워드, Role(Authorization)에 해당한다.
2. SecurityContextHolder에 이 Token을 넣어주어 Authentication 과정을 끝낸다.

<br>
🤥 이때, 이러한 생각이 들 수 있다. 지금의 과정은 회원가입 이후 자동으로 로그인 되는 로직에 해당한다. 따라서 login메서드로 오기전에는 이미 '회원가입을 할 수 있는 객체'라는 인증이 되어있는 상태에 해당한다. 그런데, 우리가 만든뷰('/login')으로 Post요청이 들어올 땐 어떻게 될까?

<br>
결론적으로 말하자면, 이는 __스프링 시큐리티가 알아서 처리를 해준다__ 이다. 우리가 직접 구현해야할 코드를 줄여주는 매우 훌륭한 기능을 제공한다. 로그인폼으로 이메일과 패스워드를 입력받으면 오버라이딩한 loadUserByUsername으로 해당 이메일에 해당하는 Account가 있는지 탐색하고, 이에 대한 Password Matching까지 시켜주는 것이다.

<br>
하지만, 아주 주의해야할 점이 존재한다. 우리는 시큐리티가 제공하는 login 폼이 아닌 새롭게 정의한 login폼을 사용하고 있다. 이때, 폼을 다음과 같이 주어야한다.

```html
<!-- login.html 중 일부-->

<form class="needs-validation col-sm-6" action="#" th:action="@{/login}" method="post" novalidate>
    <div class="form-group">
        <label for="username">이메일</label>
        <input id="username" type="text" name="username" class="form-control">
    </div>
    <div class="form-group">
        <label for="password">패스워드</label>
        <input id="password" type="password" name="password" class="form-control">
    </div>

    <div class="form-group">
        <button class="btn btn-success btn-block" type="submit">
        로그인
        </button>
    </div>
</form>
```
바로 form의 input자식들에 대한 id 값인데 반드시 username과 password로 설정해주어야 한다(7, 11라인). 스프링 시큐리티는 이 두가지값을 파싱하여 앞선 절차를 진행해준다.

<br>
여기까지 진행되었다면 간단히 확인만 해보자. 이메일을
vitriol95@snu.ac.kr, 닉네임을 vitriol95로 설정하고 회원가입을 진행했다. 이후 h2-console로 들어가서 이 값이 제대로 있는지 확인해보자.

<img src="/assets/img/mvcproject/4.JPG">

잘 전달되었다. 비밀번호 또한 해싱되어 저장되있는 것을 확인할 수 있다.

<br>
다음편에서는 Account와 관련된 클래스들을 좀 더 다듬고, 이들의 테스트를 진행해 볼 예정이다.