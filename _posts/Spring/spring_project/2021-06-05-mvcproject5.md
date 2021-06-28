---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(5)
date: 2021-06-05 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject4/) 에서는 수정사항과 단위테스트, 마지막으로 Reply 엔티티를 정의하고 양방향 매핑까지 완료하였다. 이번시간에는 Reply객체가 들어옴으로써 발생하는 컨트롤러와 뷰단의 변화를 적용하고 Reply의 등록 및 삭제기능을 구현해보자. 등록과정에서는 비동기 테스트를 위해 ajax를 사용해 보겠다.

---

# 포스트 상세보기 뷰 / 컨트롤러 수정

포스트 상세보기 뷰에 Reply객체를 적용해야한다. html파일은 다음과 같이 정의된다.

```html
<!-- 수정된 post/view.html 파일 -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org" xmlns:sec="http://thymeleaf.org/extras/spring-security">
<head th:replace="fragments.html::head">

</head>
<body>
    <nav th:replace="fragments.html :: main-nav"></nav>

    <main role="main" class="container">
        <div class="d-flex align-items-center p-3 my-3 bg-purple rounded shadow-sm" style="justify-content: space-between">
            <div class="lh-200">
                <h3 class="mb-0 lh-200" th:text="${post.title}">Bootstrap</h3>
                <small th:text="${#temporals.format(post.modifiedDate,'yyyy-MM-dd HH:mm')}">localdate</small>
            </div>

            <div class="lh-200" th:if="${post.isWriter(#authentication.principal)}">

                <div class="form-group mb-0">
                    <a th:href="@{'/posts/'+${post.getId()}+'/update'}"><button name="update" class="btn btn-outline-primary" type="submit">수정</button></a>
                </div>

                <form class="form-inline" th:if="${post.isWriter(#authentication.principal)}" action="#" th:action="@{'/posts/' + ${post.getId()} + '/delete'}" method="post" novalidate>
                    <div class="form-group">
                        <button name="delete" class="btn btn-outline-danger" type="submit">삭제</button>
                    </div>
                </form>
            </div>
        </div>

        <div class="d-flex align-items-center p-3 my-3 bg-purple rounded shadow-sm">
            <div class="lh-100">
                <svg th:if="${#strings.isEmpty(post.account?.profileImage)}" data-jdenticon-value="user111" th:data-jdenticon-value="${post.account.email}"
                     width="32" height="32" class="rounded boarder bg-light"></svg>
                <img th:if="${!#strings.isEmpty(post.account?.profileImage)}" th:src="${post.account.profileImage}"
                     width="32" height="32" class="rounded boarder bg-light">
                <span class="lh-100 h6 mb-0" th:text="${post.account.nickname}"></span>
            </div>
        </div>
        <div class="my-3 p-3 bg-white rounded shadow-sm" th:utext="${post.description}">

        </div>

        <div id="comments" th:fragment="comments" class="my-3 p-3 bg-white rounded shadow-sm">

            <h6 class="border-bottom border-gray pb-2 mb-0"><strong>코멘트</strong></h6>

            <div class="media text-muted pt-3" th:each="reply: ${post.replies}">

                <svg class="bd-placeholder-img mr-2 rounded" th:if="${#strings.isEmpty(reply.account?.profileImage)}" data-jdenticon-value="user111" th:data-jdenticon-value="${reply.account.email}"
                     width="32" height="32"></svg>
                <img th:if="${!#strings.isEmpty(reply.account?.profileImage)}" th:src="${reply.account.profileImage}"
                     width="32" height="32" class="bd-placeholder-img mr-2 rounded">

                <div class="media-body pb-3 mb-0 small lh-125 border-bottom border-gray">
                    <div class="d-flex justify-content-between align-items-center w-100">
                        <strong class="text-gray-dark" th:text="${reply.account.nickname}">Full Name</strong>
                        <span>
                            <form class="form-inline" th:if="${reply.isWriter(#authentication.principal)}" th:action="@{'/posts/' + ${post.getId()} + '/reply/' +${reply.getId()}+'/delete'}" method="post" novalidate>
                                <div class="form-group">
                                    <button name="delete" class="btn btn-link btn-sm" type="submit">삭제</button>
                                </div>
                            </form>
                        </span>
                    </div>
                    <span th:text="${#temporals.format(reply.modifiedDate,'yyyy-MM-dd HH:mm')}">@username</span>
                    <div class="mt-1 text-dark"  th:utext="${reply.description}">
                    </div>
                </div>
            </div>


            <small class="d-block text-right mt-3">
                <a href="#">All suggestions</a>
            </small>

            <div class="needs-validation col-sm-10">
                <div class="form-group">
                    <label for="replyDescription">코멘트 작성</label>
                    <textarea name="replyDescription" id="replyDescription" class="editor form-control" placeholder="본문을 작성해주세요"
                              aria-describedby="replyDescriptionHelp" required minlength="2"></textarea>
                    <small id="replyDescriptionHelp" class="form-text text-muted">
                        매너를 지켜 작성해주세요.
                    </small>
                    <small class="invalid-feedback">2글자 이상이 와야합니다.</small>
                </div>

                <div class="form-group">
                    <button id="replySave" class="btn btn-primary btn-block" type="button">코멘트 달기</button>
                </div>
            </div>
        </div>
    </main>
    <script th:replace="fragments.html :: form-validation"></script>
    <link href="https://cdn.jsdelivr.net/npm/summernote@0.8.18/dist/summernote.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/summernote@0.8.18/dist/summernote.min.js"></script>
    <script th:fragment="summerNote" type="application/javascript">
        $(function(){
            $('.editor').summernote({
                placeholder: '공백만 있는 글은 허용되지 않습니다.',
                tabSize: 2,
                height: 100
            });
        })
    </script>
    <script type="application/javascript" th:inline="javascript">
        $(function() {
            var csrfToken = /*[[${_csrf.token}]]*/ null;
            var csrfHeader = /*[[${_csrf.headerName}]]*/ null;
            $(document).ajaxSend(function (e, xhr, options) {
                xhr.setRequestHeader(csrfHeader, csrfToken);
            });
        });
    </script>
    <script>
        let index = {
            init: function () {
                $("#replySave").on("click", () => {
                    this.replySave();
                });
            },
            replySave: function () {
                let description = $("#replyDescription").val();
                let postId = [[${post.id}]];
                let data = {description: description};
                $.ajax({
                    contentType: "application/json; charset=utf8",
                    method: "POST",
                    url: "/posts/" + String(postId) + "/reply",
                    data: JSON.stringify(data),
                }).done(function (response) {
                    location.reload();
                }).fail(function (a,b,c) {
                    alert(c)
                });
            }
        };
        index.init();
    </script>
</body>
</html>
```

43라인부터 69라인이 현재 포스트에 작성된 댓글들을 보여주는 곳에 해당한다. 또한 76라인부터 90라인까지가 새로운 댓글을 등록하는 창에 해당한다(Summernote를 적용하였다.) 이들의 뷰 로직은 다음과 같다.

<br>
1. 댓글을 남길경우 뷰단에 댓글을 단 사용자의 프로필 이미지, 닉네임과 본문이 나타나게 된다.
2. 댓글 작성자의 경우에만 삭제 버튼에 접근할 수 있다.
> 역시 principal을 매개변수로 한 isWriter 메서드가 필요하다
3. 댓글 작성시, description만 property로 가진 newReplyForm에 정보가 담겨 서버쪽으로 넘어오게 된다.
4. 댓글 작성 작업은 비동기적으로 이루어지게 되며 (114~138) ajax방식으로 전달 된다. 이때 csrf토큰을 넣어주어야 유효하다(105~113).
> ajax 요청이 성공하면 (success) 여기서는 location.reload()로 페이지를 refresh하고 있다. 사실 이렇게 페이지를 갱신한다면 ajax를 이용하는 장점이 없어진다. 해결법으로는 replaceWith 메서드를 이용해 필요한 화면만 갱신할 수 있다. 하지만 이 프로젝트에서는 비동기 테스트를 위한 용도이므로 넘어가도록 하자.
5. 댓글 삭제의 경우 단순한 post요청으로 넘어가게 된다. (59라인)

<br>
이 뷰에 대한 핸들러는 다음과 같이 수정된다.

```java
// PostController.class

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

    @PostMapping(value = "/posts/{id}/reply")
    @ResponseBody
    public ResponseEntity replyFormSubmit(@PathVariable Long id, @LoggedInUser Account account, 
                                            @RequestBody NewReplyForm newReplyForm) {
        Post post = postService.getVanillaPost(id);
        postService.createNewReply(modelMapper.map(newReplyForm, Reply.class), post, account);
        return ResponseEntity.ok().build();
    }

    @PostMapping(value = "/posts/{postId}/reply/{replyId}/delete")
    public String deleteReplySubmit(@PathVariable("postId") Long id, @LoggedInUser Account account, 
                                        @PathVariable("replyId") Long replyId) {
        Post post = postService.getVanillaPost(id);
        Reply reply = replyRepository.findReplyById(replyId);
        postService.deleteReply(reply, post);
        return "redirect:/posts/" + id;
    }
```

변화된 상황을 하나씩 살펴보자.

---

## 포스트 상세보기(/posts/{id}) 컨트롤러

앞선 뷰를 보여주는 핸들러에 해당한다. 위에서 보았듯이, Post객체를 하나 가져와서 연결된 Account와 Reply들을 모두 긁어오고 있다. 이는 `Post post = postRepository.findPostWithUserAndRepliesById(id)` 라는 메서드를 통해 긁어온다. 이는 다음과 같이 구현되어있다.

```java
@Transactional(readOnly = true)
public interface PostRepository extends JpaRepository<Post, Long> {

    @EntityGraph(attributePaths = {"description","account","replies"}, type = EntityGraph.EntityGraphType.LOAD)
    Post findPostWithUserAndRepliesById(Long id);

    /**
    이전 코드들
    */

}

```

위의 @EntityGraph를 이용하여 Lazy로 설정된 attribute들을 모두 긁어오고 있다. 이렇게 되면 뷰에 본문과, 글쓴이에 대한 정보, 댓글들을 모두 가져올 수 있다.

<br>
🤔 하지만 한가지 의문점이 든다. 뷰를 다시보면, 우리는 Reply들의 Account도 필요하다. 즉, 댓글이 보여질 때 댓글쓴이의 프로필과 닉네임이 함께 보여져야 하므로 이것까지 긁어와야 한다는 것이다. 그림으로 나타내면 다음과 같다.

<img src="/assets/img/mvcproject/16.JPG">

Post 엔티티의 본문, Reply, Account(글쓴이)는 @EntityGraph를 통해 한꺼번에 가져와지는 반면 Reply엔티티와 연결된 Account(댓글쓴이)는 LAZY에 해당하므로 프록시 객체가 자리할 것이다. 따라서 뷰단으로 `reply.account.profileImage`와 같은 프로퍼티를 소환할 경우 에러를 뱉어낼 것이다. 

<br>
이를 해결하는 방법에는 2가지가 존재한다.

1. Reply Entity의 Account 프로퍼티 FetchType을 EAGER로 바꾼다.
2. 네이티브 쿼리나 QueryDSL을 이용하여 새로운 쿼리를 정의한다.

필자는 2번방법을 시도하던 중, 잠시 생각이 떠올랐다. 'Account(댓글쓴이)가 없는 상태로 Reply가 보여질 일이 있는가?' 답은 No였다. 따라서 1번 방식으로 우회하여 진행하였다.

```java
@Entity
@Table(name = "reply")
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class Reply extends LocalDateTimeEntity {

    /**
    이전 코드들
    */
    @ManyToOne
    private Account account;

    /**
    이전 코드들
    */
}
```
다음과 같이 진행하면 앞선 `findPostWithUserAndRepliesById`메서드를 사용하더라도 Reply의 Account까지 모두 가져오게 된다.

---

## 댓글 등록 (/posts/{id}/reply) 컨트롤러

```java
// PostController.java
    @PostMapping(value = "/posts/{id}/reply")
    @ResponseBody
    public ResponseEntity replyFormSubmit(@PathVariable Long id, @LoggedInUser Account account, @RequestBody NewReplyForm newReplyForm) {
        Post post = postService.getVanillaPost(id);
        postService.createNewReply(modelMapper.map(newReplyForm, Reply.class), post, account);
        return ResponseEntity.ok().build();
    }
```
Ajax 방식으로 요청을 받아 비동기적으로 진행이 되었기에 @ResponseBody를 이용해 ResponseEntity 객체를 내려주었다. 일종의 api 핸들러역할을 한다고 생각하면 된다.

<br>
이 핸들러는 뷰단에서 비동기적으로 Post요청으로 넘어온 newReplyForm을 받아 Reply객체로 매핑을 진행하고 이를 생성한다. 이때, Reply 엔티티는 Post와 Account객체가 매핑되어야 한다.

<br>
post의 경우 `getVanillaPost`라는 메서드를 통해 가져오고 있다. 이번 Post를 가져올때 우리는 연결된 Account나 description과 같은 Lazy프로퍼티들을 가져올 필요가 없다. 그냥 reply를 넣어주기만 하면 되기 때문이다.

```java
// PostService.java
public Post getVanillaPost(Long id) {
    return postRepository.findPostById(id);
}

// PostRepository.java
@Transactional(readOnly = true)
public interface PostRepository extends JpaRepository<Post, Long> {

    /**
    이전 코드들
    */

    Post findPostById(Long id);
    // 어떠한 연결객체도 깨우지 않은 상태로 가져온다.
    
    /**
    이전 코드들
    */
}
```

다음으로는 postService의 `createNewReply`메서드를 보자. 파라미터로는 Reply와 Post, Account를 받고있다.

```java
// PostService.java

public void createNewReply(Reply reply, Post post, Account account) {
    Account writer = accountRepository.findByEmail(account.getEmail());
    // detached 살려오기

    reply.setAccount(writer);
    post.addReply(reply);
    writer.replyAdd();
    replyRepository.save(reply);
    }
```

우리는 파라미터로 @LoggedInUser 어노테이션이 적용된 Principal객체내의 Account를 받았다. 이 객체는 Detached 상태에 해당하므로 이를 다시 Managed상태로 올려주어야 한다(4 라인). 이 Reply에 대한 작성자와 post 정보를 채워주고 저장하면 된다.
> 여기에 writer.replyAdd()나, post.addReply(reply) 같은 메서드가 존재하는데 나중에 회원 정보를 보여줄 때 댓글수와 게시글 수를 보여주기 위해 새로운 post/replyCount 프로퍼티를 만들어 두었다. 후자의 경우는 양방향 매핑을 하며 Post 내부 프로퍼티인 replyCount를 1 더해준다.

---

## 댓글 삭제 (/posts/{postId}/reply/{replyId}/delete) 컨트롤러

```java
// PostController.java
    @PostMapping(value = "/posts/{postId}/reply/{replyId}/delete")
    public String deleteReplySubmit(@PathVariable("postId") Long id, @LoggedInUser Account account, @PathVariable("replyId") Long replyId) {
        Post post = postService.getVanillaPost(id);
        Reply reply = replyRepository.findReplyById(replyId);
        postService.deleteReply(reply, post);
        return "redirect:/posts/" + id;
    }
```

역시 `getVanillaPost`메서드를 이용하여 게시글을 가져왔으며 reply객체와 함께 `postService.deleteReply`의 파라미터로 넘겨주었다. 이 메서드를 살펴보자.

```java
public void deleteReply(Reply reply, Post post) {
    post.removeReply();
    reply.removeTrace();
    replyRepository.delete(reply);
}
```
post 객체로 들어가 양방향 매핑을 제거하고 removeTrace()메서드를 이용하여 Account와 post객체의 replyCount를 1씩 내려주었다.

<br>

이들이 제대로 동작하는지 한번 살펴보도록 하자.

<img src="/assets/img/mvcproject/17.JPG">

초기의 화면에 해당하고, 댓글을 등록하면 다음과 같이 나타난다.

<img src="/assets/img/mvcproject/18.JPG">

댓글의 글쓴이면서, 게시글의 글쓴이에 해당하므로 수정과 삭제 버튼이 모두 열려있다. 다른 유저로 들어가 댓글을 남겨보자.

<img src="/assets/img/mvcproject/19.JPG">

자신이 쓴 댓글에만 삭제 버튼이 열려있는 것을 확인해 볼 수 있었다.

---

# 버그 수정 - Post 엔티티와 Reply 엔티티의 영속성 전이

필드 테스트중 한가지 오류가 발생하여 수정하려고 한다. (아.. 이걸 왜 생각 안하고 있었지 ㅠㅠ) 게시글에 댓글이 달려있을 경우에 글을 삭제하면 다음과 같은 오류가 발생한다.

<img src="/assets/img/mvcproject/20.JPG">

현재 Post - Reply 관계에서 외래키를 가지고있는 쪽은 Reply에 해당한다. 동시에 연관관계의 주인에 해당한다. 이 때 Post엔티티를 삭제할 경우 RDBMS의 '참조 무결성 원칙'이 깨지게 되어 이러한 오류가 발생하는 것이다.

<br>
이 경우에는, 부모 릴레이션(제공하는 쪽)이 Post엔티티에 해당하며 자식 릴레이션(제공 받는 쪽)이 Reply엔티티에 해당하므로 부모 릴레이션 삭제시에 제약사항이 걸리는 것이다. 따라서 다음과 같은 옵션을 추가해주면 된다.

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

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Reply> replies = new ArrayList<>();

    /**
    이전 코드들
    */
}
```
참조 무결성 제약조건 옵션중 cascade를 이용하며 Reply는 Post와 생명주기를 같이하도록 한다. 그리고 `orphanRemoval = true` 옵션을 주어 참조가 끊어진 Reply들은 자동삭제되게끔 한다(GC 대상)
> 사실 여기서 orphanRemoval 옵션은 안주어도 크게 상관이 없다.

이제 게시글 상세보기 구현 및 Reply에 대한 기능은 모두 구현하였다. 다음에는 게시글을 검색하는 서비스를 만들고, 가입된 유저 목록을 보여주는 기능을 만들어보자.