---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(3)
date: 2021-06-03 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject2/) 에서는 Account와 관련된 기능들을 좀 더 다듬고, MockMvc를 이용해 테스트를 진행해 보았다. 이번 시간에는 Account의 정보 수정을 구현하고 Post라는 엔티티를 정의해보려한다.

---

# Account 정보 수정

정보변경에 대한 기능은 프론트뷰에선 다음과 같이 위치해 있으며 클릭시 URI는 "/settings"에 해당한다.

<img src="/assets/img/mvcproject/8.JPG">

클릭하여 들어올 시, 뷰는 다음과 같이 구성된다.

<img src="/assets/img/mvcproject/9.JPG">

닉네임과 소개글, 프로필 이미지를 변경할 수 있다. 프로필 이미지 변경에 대한 프로그램은 [Cropper](https://fengyuanchen.github.io/cropperjs/)를 이용하여 구현하였다.

<br>
가장 먼저, "/settings" 뷰에는 회원가입과 마찬가지로 폼에 입력될 정보를 받을 객체가 필요하다. 이에 필자는 'ProfileForm'이라는 새로운 Dto 클래스를 정의하였다.

```java
package vitriol.mvcservice.modules.account.form;

@Data
public class ProfileForm {

    @NotBlank
    @Length(min = 3, max = 20)
    @Pattern(regexp = "^[ㄱ-ㅎ가-힣a-z0-9_-]{3,20}$")
    private String nickname;

    @Length(max = 35)
    private String bio;

    private String profileImage;
}
```
> 이 역시 마찬가지로 javax.validation의 제약사항들을 적용해 주었다.

<br>
앞서 회원가입의 경우, @InitBinder에 Validator를 추가해 준 것처럼 이 ProfileForm 또한 이미 존재하는 닉네임을 받으면 안되므로 이를 구현해주어야 한다. 

```java
package vitriol.mvcservice.modules.account.validator;

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
        if (byNickname != null) {
            errors.rejectValue("nickname","wrong.value","해당 닉네임을 이미 사용중입니다.");
        }

    }
}
```
> 여기서는 SignUpForm Validator와 달리, 닉네임만 체크해주면 된다.

<br>
이후, AccountController에 다음과 같은 핸들러들을 추가해 주어야한다.

```java
package vitriol.mvcservice.modules.account;

@Controller
@RequiredArgsConstructor
public class AccountController {

    /**
    이전 코드들
    */

    private final ProfileFormValidator profileFormValidator;;

    @InitBinder("profileForm")
    public void initBinder2(WebDataBinder webDataBinder) {
        webDataBinder.addValidators(profileFormValidator);
    }

    @GetMapping("/settings")
    public String updateProfileView(@LoggedInUser Account account, Model model) {
        model.addAttribute(account);
        model.addAttribute(modelMapper.map(account, ProfileForm.class));
        return "account/settings";
    }

    @PostMapping("/settings")
    public String updateProfile(@LoggedInUser Account account, @Valid @ModelAttribute ProfileForm profileForm, Errors errors,
                                Model model, RedirectAttributes redirectAttributes) {
        if (errors.hasErrors()) {
            model.addAttribute(account);
            return "account/settings";
        }
        accountService.updateProfile(account, profileForm);
        redirectAttributes.addFlashAttribute("message", "수정을 완료하였습니다");
        return "redirect:" + "/settings";
    }
}

```
> @GetMapping("/settings")에 대해서는 이미 적어둔 짧은 소개나 프로필 이미지가 존재할 수 있으므로 뷰에는 modelmapper를 통해 뿌려주었다.

<br>
post요청에 대해서는 회원가입을 처리하는 컨트롤러와 매우 유사하며 accountService로 들어가서 updateProfile이라는 메서드를 호출하게 된다.

```java
package vitriol.mvcservice.modules.account;

@Service
@RequiredArgsConstructor
@Transactional
public class AccountService implements UserDetailsService {

   /**
   이전 코드들
   */
    public void updateProfile(Account account, ProfileForm profileForm) {
        modelMapper.map(profileForm, account);
        accountRepository.save(account);
    }
}
```
> 새롭게 modelmapper를 이용해 매핑을 한뒤, 얻어진 어카운트를 다시 save하면 된다. 이미 존재한 데이터이므로 merge에 해당하겠다.

<img src="/assets/img/mvcproject/10.JPG">

기존 정보에서 닉네임과 프로필 이미지를 변경해 보았다.

<img src="/assets/img/mvcproject/11.JPG">

---

# Post 엔티티 

post의 경우 간단한 정보는 다음과 같다.

1. 제목, 짧은 요약, 본문, 공개여부(비밀글 여부)를 프로퍼티로 가진다.
2. 글쓴이에 대한 정보가 필요하다. 이는 Account와의 관계성을 가진다.
> Account와 Post는 일대다 관계를 맺게 된다.

3. 댓글(Reply)과의 일대다 관계또한 존재한다.

<br>

이를 바탕으로 구현해본 Post객체는 다음과 같다.

```java
package vitriol.mvcservice.modules.post;

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

    @Lob
    @Basic(fetch = FetchType.LAZY)
    private String description;

    private boolean open;

    public void setWriter(Account account) {
        this.account = account;
        account.getPosts().add(this);
    }
}
```
> 역시 슈퍼클래스인 LocalDateTimeEntity를 가지고있다.
특별한점은 22번째 라인인 본문의 fetch타입에 해당한다. 필자의 경우는 Lazy로 가져오게끔 설정을 해두었다. 게시글을 하나하나 볼때는 이를 가져와야 하지만, 나중에는 메인화면에 페이징을 하여 뿌릴 예정이므로 이때는 본문이 필요가 없다.

> Account역시 일단은 Fetch로 잡아두었다. 필자는 관계성 필드의 경우 효율적인 쿼리를 위해서 일단은 모두 Lazy로 잡아두는 것을 선호한다.

<br>
이후 Account와 Post의 매핑에 대해 생각해 보았다. Post의 경우에는 Account가 무조건 필요하지만, Account가 이러한 Post를 무조건 가져야 하는 이유는 없다. 하지만, 필자의 경우는 Account가 이러한 게시글 목록을 볼 수있게끔 하고싶기에 양방향 매핑을 진행하기로 하였다.
> 이로인해 setWriter라는 양방향 매핑 메서드가 추가된 것이다.

<br>
따라서 Account 클래스에는 다음이 추가된다.

```java
package vitriol.mvcservice.modules.account;

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
    private Set<Post> posts = new HashSet<>();
}
```
> 양방향 매핑에 해당하므로 반드시 mappedBy를 설정해 주어야한다. 연관관계의 주인을 '다' 쪽인 Post로 설정해 주었다.

## Post 작성하기
이제 실제로 post를 작성해보자.

<img src="/assets/img/mvcproject/12.JPG">

이는 다음의 위치에 있으며, URI는 "/new-post"에 해당한다. 들어오게 되면 다음과 같은 뷰가 보이도록 구현해 두었다. 본문의 경우에는 [summerNote](https://summernote.org/)를 이용하였다.

<img src="/assets/img/mvcproject/13.JPG">

<br>
이에 들어올 내용을 받아줄 Dto 객체가 필요하다. 필자는 이를 "NewPostForm"이라는 클래스로 구현해 보았다.

```java
package vitriol.mvcservice.modules.post.form;
@Data
public class NewPostForm {

    private Long id;

    @NotBlank
    @Length(max = 50)
    private String title;

    @NotBlank
    @Length(max = 100)
    private String introduction;

    @NotBlank
    private String description;

    private boolean open;
}
```
> id 프로퍼티를 가져온 이유는 나중에 글을 수정하려 할 때 용이하기 위해 같이 적어두었다.

<br>
실제 폼이 가지고 있는 프로퍼티들과 대응되게 끔 구현을 해두었다. 체크박스가 클릭이 되면 `open = true`가 된다. 이제 실제 컨트롤러를 구현해보자.


```java
package vitriol.mvcservice.modules.post;

@Controller
@RequiredArgsConstructor
public class PostController {

    private final PostService postService;
    private final ModelMapper modelMapper;

    @GetMapping("/new-post")
    public String newPostForm(@LoggedInUser Account account, Model model) {

        model.addAttribute(account);
        model.addAttribute(new NewPostForm());
        return "post/form";
    }

    @PostMapping("/new-post")
    public String postFormSubmit(@LoggedInUser Account account, Model model, @Valid NewPostForm newPostForm, Errors errors) {
        if (errors.hasErrors()) {
            model.addAttribute(account);
            return "post/form";
        }

        Post post = postService.createNewPost(modelMapper.map(newPostForm, Post.class), account);
        return "redirect:/posts/" + post.getId();
    }
```
> 앞선 AccountController의 회원가입과 굉장히 유사하다.

@PostMapping의 경우, 입력값이 정상이라면 postService의 createNewPost를 호출하게 된다. 서비스단을 살펴보자.


```java
package vitriol.mvcservice.modules.post;

@Service
@Transactional
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepository;
    private final ModelMapper modelMapper;

    public Post createNewPost(Post newPost, Account account) {

        Post post = postRepository.save(newPost);
        post.setWriter(account);
        return post;
    }
}
```
> 기능은 정말 단순하다. 앞서 post엔티티에 정의해둔 setWriter라는 메서드를 사용하여 양방향 매핑또한 진행해주면 된다.

<br>
이후, 컨트롤러에서는 글이 성공적으로 생성되었으면 /posts/id 로 리다이렉팅 시켜주고 있다. 이에 대한 뷰는 다음과 같다.
> 사실 이렇게 id로 접근하는 방식은 좋지않다.

<img src="/assets/img/mvcproject/14.JPG">

네가지의 블록으로 이루어져 있다. 
1. 맨 위 블록에 제목과 생성 / 수정날짜가 있으며 만약 작성자일 경우 수정과 삭제버튼이 노출된다.
2. 두번째 블록에는 작성자 항목에 해당한다.
3. 세번째 블록이 본문에 해당한다.
4. 네번째 블록은 Reply들이 위치할 곳에 해당한다.

<br>
이들의 뷰 구현부는 다음과 같다.

```html

<main role="main" class="container">
    <div class="d-flex align-items-center p-3 my-3 bg-purple rounded shadow-sm" 
    style="justify-content: space-between">
        <div class="lh-200">
            <h3 class="mb-0 lh-200" th:text="${post.title}">Bootstrap</h3>
            <small th:text="${#temporals.format(post.modifiedDate,'yyyy-MM-dd HH:mm')}">localdate</small>
        </div>

        <div class="lh-200" th:if="${post.isWriter(#authentication.principal)}">
            <form class="form-inline" th:if="${post.isWriter(#authentication.principal)}" action="#" 
            th:action="@{'/posts/' + ${post.getId()} + '/update'}" method="get" novalidate>
                <div class="form-group">
                    <button class="btn btn-outline-primary" type="submit">수정</button>
                </div>
            </form>
            <form class="form-inline" th:if="${post.isWriter(#authentication.principal)}" action="#" 
            th:action="@{'/posts/' + ${post.getId()} + '/delete'}" method="post" novalidate>
                <div class="form-group">
                    <button class="btn btn-outline-danger" type="submit">삭제</button>
                </div>
            </form>
        </div>
    </div>

    <div class="d-flex align-items-center p-3 my-3 bg-purple rounded shadow-sm">
        <div class="lh-100">
            <svg th:if="${#strings.isEmpty(post.account?.profileImage)}" data-jdenticon-value="user111" 
            th:data-jdenticon-value="${post.account.email}" width="32" height="32" class="rounded boarder bg-light"></svg>
            <img th:if="${!#strings.isEmpty(post.account?.profileImage)}" th:src="${post.account.profileImage}"
                    width="32" height="32" class="rounded boarder bg-light">
            <span class="lh-100 h6 mb-0" th:text="${post.account.nickname}"></span>
        </div>
    </div>
    <div class="my-3 p-3 bg-white rounded shadow-sm" th:utext="${post.description}">

    </div>

    <div class="my-3 p-3 bg-white rounded shadow-sm">
        <h6 class="border-bottom border-gray pb-2 mb-0"><strong>코멘트</strong></h6>
        <div class="media text-muted pt-3">
            <svg class="bd-placeholder-img mr-2 rounded" width="32" height="32" xmlns="http://www.w3.org/2000/svg" 
            role="img" aria-label="Placeholder: 32x32" preserveAspectRatio="xMidYMid slice" focusable="false"><title>Placeholder</title>
            <rect width="100%" height="100%" fill="#007bff"></rect><
                text x="50%" y="50%" fill="#007bff" dy=".3em">32x32</text></svg>

            <div class="media-body pb-3 mb-0 small lh-125 border-bottom border-gray">
                <div class="d-flex justify-content-between align-items-center w-100">
                    <strong class="text-gray-dark">Full Name</strong>
                    <a href="#">Follow</a>
                </div>
                <span class="d-block">@username</span>
            </div>
        </div>

        <small class="d-block text-right mt-3">
            <a href="#">All suggestions</a>
        </small>
    </div>
</main>
```

가장먼저 9번째 줄부터 시작하는 div 컴포넌트를 보자. 이 컴포넌트는 수정과 삭제 버튼을 담고 있는 것으로 글을 보고 있는 사용자가 실제 글쓴이인지 프론트 단에서 확인하는 과정이 필요하다. 따라서 이는 post내의 isWriter라는 함수를 사용하여 판별하려 하였다. 이 함수의 인자로는 principal에 해당하는 UserAccount가 들어오게 될 것이다.

<br>
따라서 Post 클래스의 다음의 메서드를 추가해주어야 한다.

```java
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

    public boolean isWriter(UserAccount userAccount) {
        return this.account.equals(userAccount.getAccount());
    }

}
```

이렇게 되면, thymealeaf 템플릿을 거쳐 이 메서드를 프론트단에서 이용할 수 있게 된다.


## Post의 수정 및 삭제 기능