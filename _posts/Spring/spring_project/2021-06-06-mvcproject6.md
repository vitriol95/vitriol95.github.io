---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(6)
date: 2021-06-06 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject5/) 에서는  Reply객체가 들어옴으로써 발생하는 컨트롤러와 뷰단의 변화를 적용하고 Reply의 등록 및 삭제기능을 구현해 보았다. 이번시간에는 게시글에 대한 검색기능과, 유저 목록을 보여주는 기능을 구현해보자.

---

# 게시글 검색하기

게시글 검색의 경우 메인 nav-bar의 오른쪽에 구현이 되어 있다. 기능은 다음과 같다.

- 검색창에 키워드를 입력하고 검색하면 본 키워드가 제목에 포함된 게시글만을 보여주게 된다. 
- 이때 카드형식으로 게시글이 노출되며 제목, 간단한 설명(introduction), 작성자, 댓글 수, 등록일이 보이게 된다.
- 페이징을 적용한다. 총 12개가 한 화면에 보이게끔 설정해 두었다.
- 보여지는 게시글은 정렬방식을 달리할 수 있다. 최신순으로 부터 볼 수 있으며, 댓글이 많은 순으로 볼 수도 있다.

<img src="/assets/img/mvcproject/21.JPG">

가장 먼저, main-nav의 게시글 검색에 해당하는 인라인 폼을 다음과 같이 짜두었다.

```html
<form class="form-inline my-2 my-lg-0" th:action="@{/search/post}" method="get">
    <input class="form-control mr-sm-2" name="keyword" type="search" placeholder="게시글 검색">
    <button class="btn btn-outline-success my-2 my-sm-0" type="submit">Search</button>
</form>
```

get 방식을 통해 /search/post로 접근하게 되며, 이때 input 태그에 걸려있는 `name="keyword"`이므로 쿼리스트링으로 ?keyword= 가 붙게될 것이다. 즉 URL은 다음과 같다. `/search/post?keyword= `

<br>
이에 대한 컨트롤러를 구현해보면 다음과 같다. (MainController.java 에 구현하였다.)

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
```
파라미터를 살펴보자. 페이징을 적용할 것이기 때문에 @PagebleDefault 어노테이션을 이용하여 Pageable 객체를 전달하였다. 기본 정렬은 등록 날짜 최신순(DESC)으로 하였다. 그리고 @ModelAttribute에 해당하는 `String Keyword`가 들어와있다.

<br>
뷰 단으로, 쿼리를 진행한 Page 객체인 postPage를 넘겨주어야 하며, 검색했던 keyword와 정렬방식에 해당하는 sortProperty를 넘겨주었다.

<br>
여기서 가장 핵심은 `postRepository.findByKeyword(keyword, pageable)` 이다. 키워드와 pageable을 파라미터로 받아 실제 그에 해당하는 post를 가져오는 메서드에 해당한다. 필자는 이를 QueryDsl을 이용하여 간소화 시켜보고자 한다.
<br>

---

## gradle에 QueryDsl 의존성 추가하기
가장 먼저, build.gradle에 QueryDsl의 의존성을 추가해주어야 한다. 필자는 gradle7.0을 사용하여 프로그램을 빌드중이므로 다음과 같이 추가해 주었다.

```groovy
// build.gradle

dependencies {
    /**
    이전코드
    */

    /** query DSL*/
    compile "com.querydsl:querydsl-jpa"
    compile "com.querydsl:querydsl-core"
    annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa" // querydsl JPAAnnotationProcessor 사용 지정
    annotationProcessor "jakarta.persistence:jakarta.persistence-api:2.2.3"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api:1.3.5"
}

// querydsl 의 Q파일의 위치 및 task 적용
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

```
최신 gradle 버전에서 권장하는 방식인 annotationProcessor를 이용하여 dependencies를 추가해주었다.

<br>
이후 17 라인부터에 대한 간략한 설명은 다음과 같다.

1. 가장 먼저, Q파일들이 생성될 위치를 generated에 정의해준다. 
2. 이후, main.java.srcDirs 에 앞서 정의한 generated를 추가해준다.
> gradle에서 기본 제공되는 main.java.srcDirs의 경우는 'src/main/java'에만 해당하지만, 이 sourceSets에 'src/main/generated'를 포함시켜주는 것이다. 이렇게 되면 'src/main/generated'역시 gradle build의 대상이 된다.
3. 22번째 라인의 경우 gradle이 build될때 수행하는 JavaCompile이라는 태스크에 옵션을 추가해주는 것에 해당한다.
>  우리는 annotaionProcessor로 queryDsl을 등록해 두었으므로 JavaCompile이라는 태스크가 진행될 때 generated 파일을 생성하게 된다.
4. 마지막으로 gradle의 clean 태스크에 마지막 명령을 추가해준다. 
> gradle clean을 진행할 때, 이 generated 디렉토리를 함께 지워버리는 것이다. 만약 Q파일들만 따로 지우고 싶다면 새로운 task를 정의하는 방법도 있다.

<br>
이로써 gradle 설정은 모두 마무리 되었다. 다시 `postRepository.findByKeyword` 로 돌아가자.

---

## 검색 쿼리 구현하기(findByKeyword)

queryDsl을 사용하기 위해 `PostRepositoryEx`라는 새로운 repository를 생성하고 이를 JPA Repository인 `PostRepository`가 상속하게 한다. 그리고 이 `PostRepositoryEx`는 findByKeyword라는 메서드를 가지게 된다.

```java
// PostRepositoryEx.java
@Transactional(readOnly = true)
public interface PostRepositoryEx {

    Page<Post> findByKeyword(String keyword, Pageable pageable);

}

// PostRepository.java
@Transactional(readOnly = true)
public interface PostRepository extends JpaRepository<Post, Long>, PostRepositoryEx {

    /**
    이전 코드들
    */
}
```

이후 PostRepositoryEx에 구현체에 해당하는 PostRepositoryExImpl이라는 클래스를 생성하여 이 메서드를 구현해야한다.

```java
public class PostRepositoryExImpl extends QuerydslRepositorySupport implements PostRepositoryEx {

    public PostRepositoryExImpl() {
        super(Post.class);
    }

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
}
```

다음과 같이 QueryDsl을 이용하여 findByKeyword를 구현하였다. 게시글이 열려있으며(open) 대소문자 구분없이 타이틀을 포함하고 있다면 이 값을 가져오게 된다.<br> 글쓴이에 해당하는 post.account들은 프로필 이미지나 닉네임을 보여주어야 하기에 fetchJoin으로 모두 managed 상태로 만들어주었다.
> 아.. queryDsl은 정말 너무 편하다.

<br>
그리고 마지막으로 paging을 적용한 결과와, pageable, 전체 카운트를 PageImpl(구현체)에 담아 전달한다.

<br>
🤔 하지만 다음과 같은 의구심이 발생할 수 있다. __'게시글 목록을 보여줄 때, 댓글의 개수도 함께 보여주어야 하는데, 이러면 post.reply도 함께 조인해서 가져와야 하는 것이 아닌가?'__

<br>
이에 대한 비용을 조금이나마 줄여보기 위해서 필자는 Post엔티티에 replyCount라는 프로퍼티를 정의해 두었다. 댓글이 등록될 경우 카운트가 1증가하고, 삭제될 경우 1내려가게 된다. 하지만 이 방식도 장단점이 존재한다.
- __기존 댓글이 등록될 경우__) Reply 엔티티에 대한 Insert 쿼리가 한방이 나가고 종료가 된다. => 총 1개의 쿼리
- __위의 방식으로 댓글 등록을 진행할 경우__) InsertQuery와 함께 post에 대한 update 쿼리가 함께 나가게 된다. => 총 2개의 쿼리 

---
## 검색 뷰 구현
마지막으로 검색 결과를 보여주는 뷰는 다음과 같이 구현하였다.

```html
<!-- search.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org" xmlns:sec="http://thymeleaf.org/extras/spring-security">
<head th:replace="fragments.html :: head"></head>
<body>
<nav th:replace="fragments.html :: main-nav"></nav>

<div class="container">
    <div class="py-5 text-center">
        <p class="lead" th:if="${postPage.getTotalElements() == 0}">
            <strong th:text="${keyword}" id="keyword" class="context"></strong>에 해당하는 게시글이 없습니다.
        </p>
        <p class="lead" th:if="${postPage.getTotalElements() > 0}">
            '<strong th:text="${keyword}" id="keyword" class="context"></strong>'에 해당하는 게시글을
            <span th:text="${postPage.getTotalElements()}"></span>개
            찾았습니다.
        </p>
        <div class="dropdown" th:if="${postPage.getTotalElements() > 0}">
            <button class="btn btn-light dropdown-toggle" type="button" id="dropdownMenuButton" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
                검색 결과 정렬 방식
            </button>
            <div class="dropdown-menu" aria-labelledby="dropdownMenuButton">
                <a class="dropdown-item" th:classappend="${#strings.equals(sortProperty, 'createdDate')}? active"
                   th:href="@{'/search/post?sort=createdDate,desc&keyword=' + ${keyword}}">
                    게시글 최신순
                </a>
                <a class="dropdown-item" th:classappend="${#strings.equals(sortProperty, 'replyCount')}? active"
                   th:href="@{'/search/post?sort=replyCount,desc&keyword=' + ${keyword}}">
                    댓글 수
                </a>
            </div>
        </div>
    </div>
    <div class="row justify-content-center">
        <div class="col-sm-10">
            <div class="row">
                <div class="col-md-4" th:each="post: ${postPage.getContent()}">
                    <div class="card mb-4 shadow-sm">
                        <img src="/img/ffff0.JPG" class="context card-img-top" th:alt="${post.title}" >
                        <div class="card-body">
                            <a th:href="@{'/posts/' + ${post.id}}" class="text-decoration-none">
                                <h5 class="card-title context" th:text="${post.title} + '('+${post.replyCount}+')'"></h5>
                            </a>
                            <p class="card-text" th:text="${post.introduction}">Introduction</p>

                            <div class="d-flex justify-content-between align-items-center">
                                <span>
                                    <svg th:if="${#strings.isEmpty(post.account?.profileImage)}" data-jdenticon-value="user111" th:data-jdenticon-value="${post.account.email}"
                                         width="20" height="20" class="rounded boarder bg-light"></svg>
                                    <img th:if="${!#strings.isEmpty(post.account?.profileImage)}" th:src="${post.account.profileImage}"
                                         width="20" height="20" class="rounded boarder bg-light">
                                    <span class="lh-100 h6 mb-0" th:text="${post.account.nickname}"></span>
                                </span>
                                <small class="text-muted date" th:text="${#temporals.format(post.modifiedDate,'yyyy-MM-dd HH:mm')}"></small>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    <div class="row justify-content-center">
        <div class="col-sm-10">
            <nav>
                <ul class="pagination justify-content-center">
                    <li class="page-item" th:classappend="${!postPage.hasPrevious()}? disabled">
                        <a th:href="@{'/search/post?keyword=' + ${keyword} + '&sort=' + ${sortProperty} + ',desc&page=' + ${postPage.getNumber() - 1}}"
                           class="page-link" tabindex="-1" aria-disabled="true">
                            Previous
                        </a>
                    </li>
                    <li class="page-item" th:classappend="${i == postPage.getNumber()}? active" th:each="i: ${#numbers.sequence(0, postPage.getTotalPages() - 1)}">
                        <a th:href="@{'/search/post?keyword=' + ${keyword} + '&sort=' + ${sortProperty} + ',desc&page=' + ${i}}"
                           class="page-link" href="#" th:text="${i + 1}">1</a>
                    </li>
                    <li class="page-item" th:classappend="${!postPage.hasNext()}? disabled">
                        <a th:href="@{'/search/study?keyword=' + ${keyword} + '&sort=' + ${sortProperty} + ',desc&page=' + ${postPage.getNumber() + 1}}"
                           class="page-link">
                            Next
                        </a>
                    </li>
                </ul>
            </nav>
        </div>
    </div>
</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/mark.js/8.11.1/mark.min.js" integrity="sha512-5CYOlHXGh6QpOFA/TeTylKLWfB3ftPsde7AnmhuitiTX4K5SqCLBeKro6sPS8ilsz1Q4NRx3v8Ko2IBiszzdww==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/mark.js/8.11.1/jquery.mark.min.js" integrity="sha512-mhbv5DqBMgrWL+32MmsDOt/OAvqr/cHimk6B8y/bx/xS88MVkYGPiVv2ixKVrkywF2qHplNRUvFsAHUdxZ3Krg==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
<script type="application/javascript">
    $(function(){
        var mark = function() {
            // Read the keyword
            var keyword = $("#keyword").text();

            // Determine selected options
            var options = {
                "each": function(element) {
                    setTimeout(function() {
                        $(element).addClass("animate");
                    }, 150);
                }
            };

            // Mark the keyword inside the context
            $(".context").unmark({
                done: function() {
                    $(".context").mark(keyword, options);
                }
            });
        };

        mark();
    });
</script>
</body>
</html>
```
> 87~114 라인에 해당하는 js는 mark.js 라는 오픈소스를 사용하였다. 이를 통해 검색한 keyword가 검색 결과에서 하이라이팅이 된다.

<br>
이제 실제로 잘 진행되는지 확인해 보자.

<img src="/assets/img/mvcproject/22.png">

페이징 또한 잘 되어 있었고, 최신순으로 정렬되어 있었다.

<img src="/assets/img/mvcproject/23.png">

검색 조건을 댓글수로 바꾼 결과에 해당한다. 역시 잘 정렬이 되어 있었다.

---

# 유저 목록 보여주기

유저 목록의 경우 로그인을 하였을때 내정보 바로 옆에 위치하고 있으며 위 버튼을 클릭할 시 '/accounts'페이지를 Get방식으로 이동하게 된다.

<img src="/assets/img/mvcproject/24.png">

이 페이지에서는 다음과 같이 유저 정보를 보여줄 예정이다.
1. 총 몇명의 유저가 있는지 알려준다.
2. 페이징을 적용한다. 한 페이지당 20건의 유저가 노출된다.
3. 유저들은 테이블의 형태로 보여지며 아이콘, 닉네임, 게시글 수, 댓글 수, 가입일자가 나타난다.
4. 게시글이 많은 순으로 보여지게 되며 다른 검색 조건은 구현하지 않았다.

<br>
이에 대한 컨트롤러는 다음과 같다.

```java
// AccountController.java
    @GetMapping("/accounts")
    public String allAccountView(@PageableDefault(size = 20, sort = "postCount", direction = Sort.Direction.DESC) Pageable pageable, @LoggedInUser Account account, Model model) {
        Page<Account> accountPage = accountRepository.findAll(pageable);
        model.addAttribute("account", account);
        model.addAttribute("accountPage", accountPage);
        return "account/all";
    }
```
Spring Data Jpa가 지원하는 findAll(Pageable) 메서드를 이용하였고 이를 바탕으로 accountPage를 가져왔다. 이를 모델에 넣어 뷰에 전달하기만 하면 된다. 뷰는 다음과 같이 구현하였다.

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://thymeleaf.org/extras/spring-security">
<head th:replace="fragments.html :: head"></head>
<body>
    <nav th:replace="fragments.html :: main-nav"></nav>

    <div class="py-5 text-center">
        <p class="lead">Vitriol's Account</p>
        <h2>
            전체 유저보기 - <span>총 <span th:text="${accountPage.getTotalElements()}"></span>명의 유저가 활동중입니다.</span>
        </h2>
        <small class="text-info">현재는 검색으로 유저를 찾으실 수 없습니다.</small>
    </div>
    <div class="container">
        <table class="table">
            <thead>
            <tr>
                <th scope="col">Icon</th>
                <th scope="col">닉네임</th>
                <th scope="col">게시글 수</th>
                <th scope="col">댓글 수</th>
                <th scope="col">가입 날짜</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="all : ${accountPage.getContent()}">
                <td><svg th:if="${#strings.isEmpty(all?.profileImage)}" th:data-jdenticon-value="${all.email}"
                         width="32" height="32" class="rounded boarder bg-light"></svg>
                    <img th:if="${!#strings.isEmpty(all?.profileImage)}" th:src="${all.profileImage}"
                         width="32" height="32" class="rounded boarder bg-light"></td>
                <td th:text="${all.nickname}"></td>
                <td th:text="${all.postCount}"></td>
                <td th:text="${all.replyCount}"></td>
                <td th:text="${#temporals.format(all.createdDate,'yyyy-MM-dd HH:mm')}"></td>
            </tr>
            </tbody>
        </table>
        <div class="row justify-content-center">
            <div class="col-sm-10">
                <nav>
                    <ul class="pagination justify-content-center">
                        <li class="page-item" th:classappend="${!accountPage.hasPrevious()}? disabled">
                            <a th:href="@{'/accounts&page=' + ${accountPage.getNumber() - 1}}"
                               class="page-link" tabindex="-1" aria-disabled="true">
                                Previous
                            </a>
                        </li>
                        <li class="page-item" th:classappend="${i == accountPage.getNumber()}? active" th:each="i: ${#numbers.sequence(0, accountPage.getTotalPages() - 1)}">
                            <a th:href="@{'/accounts&page=' + ${i}}" class="page-link" href="#" th:text="${i + 1}">1</a>
                        </li>
                        <li class="page-item" th:classappend="${!accountPage.hasNext()}? disabled">
                            <a th:href="@{'/accounts&page=' + ${accountPage.getNumber() + 1}}" class="page-link">
                                Next
                            </a>
                        </li>
                    </ul>
                </nav>
            </div>
        </div>
    </div>

</body>
</html>
```
실제 페이지에 접속하여 테스트해보자.

<img src="/assets/img/mvcproject/25.png">

페이징이 잘 적용되어 있었으며, 게시글수로 정렬이 되어있었다.

<br>
다음 시간에는 구현한 기능들에 대해 단위 테스트를 진행해보고, 이 과정에서 나타나는 버그들을 수정해보는 시간을 가질 예정이다. 또한 DB를 h2가 아닌 MySQL로 옮기는 작업도 필요하다. 배포이전의 마지막 시간이 될 예정이다.