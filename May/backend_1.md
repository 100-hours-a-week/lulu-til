# [Project] 커뮤니티 백엔드 개발기 — ERD 설계부터 JWT 인증까지

> 회원, 게시글, 댓글, 좋아요 CRUD와 JWT 기반 인증/인가를 직접 설계하고 구현한 과정을 정리했다.
> 단순히 "작동하는 코드"가 아니라 **왜 이 선택을 했는가**에 집중했다.

- 🔗 GitHub: https://github.com/nominsol/Community

---

## 1. ERD 설계

### ERD 전체 구조

까마귀발 표기법(Crow's Foot Notation)으로 작성했다.
테이블은 총 5개 — `users`, `posts`, `comments`, `likes`, `refresh_tokens`.

---

### 자료형 선택 이유

#### ID — `BIGINT`

`INT`와 `BIGINT` 중 고민했다.

| 항목 | INT (UNSIGNED) | BIGINT (UNSIGNED) |
|------|---------------|------------------|
| 크기 | 4 byte | 8 byte |
| 최대 범위 | 약 42억 | 약 184경 |

**BIGINT를 쓰면 생기는 트레이드오프:**
- 디스크에서 더 많은 페이지를 사용 → 데이터가 RAM에 없을 때 읽기 속도에 영향
- 유지관리 작업(백업, 인덱스 재구축)이 더 오래 걸릴 수 있음
- RAM에서 더 많은 공간을 차지

그럼에도 **BIGINT를 선택한 이유:**

> 현재는 개인 프로젝트이지만, 실무에서 확장을 고려하거나 데이터가 지속적으로 쌓이는 서비스라면 BIGINT가 일반적이다. INT는 약 42억 건에서 오버플로우가 발생한다. 미래의 확장성을 고려해 처음부터 BIGINT를 채택했다.

---

#### 비밀번호 — `VARCHAR(255)`

비밀번호는 암호화하여 저장해야 하며, 알고리즘에 따라 결과 문자열의 길이가 달라진다.

- BCrypt: 60자
- SHA-256: 64자
- Argon2: 알고리즘 파라미터에 따라 가변

`VARCHAR`는 지정한 크기만큼 공간을 다 쓰는 게 아니라, **실제 저장된 문자 수만큼만 용량을 차지**한다. 따라서 `VARCHAR(255)`로 넉넉하게 설정해도 성능이나 용량에 불이익이 없다.

---

#### 프로필 이미지 — `VARCHAR` (URL 경로 저장)

이미지 저장 방식은 두 가지가 있다.

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **경로 저장 (URL)** | S3 등 스토리지에 업로드 후 DB에 경로만 저장 | DB 용량 부하 적음, 빠른 로딩 | 스토리지 별도 관리 필요 |
| **직접 저장 (BLOB)** | 이미지 자체를 DB에 저장 | 데이터 일원화, 백업 편의성 | DB 용량 급증, 서버 부하 증가 |

보안이 매우 중요한 이미지가 아니고, DB 서버의 부하를 줄이기 위해 **URL 경로 저장 방식**을 채택했다.

---

#### 게시글 내용 — `TEXT`

| 항목 | VARCHAR | TEXT |
|------|---------|------|
| 최대 크기 | 65,535 byte | 제한 없음 (최대 65,535자) |
| 글자 수 지정 | 필요 | 불필요 |
| 저장 위치 | 행과 함께 저장 | 길어질 경우 off-page 저장 |
| 조회 성능 | 빠름 (단일 디스크 읽기) | 상대적으로 느릴 수 있음 |

게시글 본문의 길이 제한이 없도록 기획되어 있어 `TEXT` 타입을 선택했다.

---

### 테이블 간 관계 설계

#### 관계 요약

| 관계 | 종류 | 카디널리티 |
|------|------|-----------|
| 회원 → 게시글 | Non-Identifying | Zero or Many |
| 회원 → 댓글 | Non-Identifying | Zero or Many |
| 회원 → 좋아요 | Identifying | Zero or Many |
| 게시글 → 댓글 | Non-Identifying | Zero or Many |
| 게시글 → 좋아요 | Identifying | Zero or Many |
| 회원 → 리프레쉬 토큰 | Non-Identifying | Zero or One |

#### 관계별 설계 근거

**회원 — 게시글 `Non-Identifying | Zero or Many`**
- 회원이 없으면 게시글은 존재할 수 없다.
- 회원 ID는 게시글 테이블의 PK가 아니다(FK로만 사용).
- 한 회원은 게시글을 0개 이상 가질 수 있고, 하나의 게시글은 반드시 하나의 회원에 속한다.

**회원 — 좋아요 `Identifying | Zero or Many`**
- 좋아요 테이블의 PK는 `(user_id, post_id)` 복합키다.
- 회원 ID가 좋아요 테이블의 PK 구성 요소이므로 **식별 관계**다.
- 한 회원은 같은 게시글에 중복 좋아요를 남길 수 없다는 유니크 제약도 복합 PK로 자동 보장된다.

**회원 — 댓글 `Non-Identifying | Zero or Many`**
- 회원 ID는 댓글 테이블의 PK가 아니다(FK로만 사용).
- 한 회원은 댓글을 0개 이상 가질 수 있고, 하나의 댓글은 반드시 하나의 회원에 속한다.

**게시글 — 좋아요 `Identifying | Zero or Many`**
- 게시글 ID도 좋아요 테이블의 PK 구성 요소다.
- 게시글이 삭제되면 관련 좋아요도 CASCADE로 삭제된다.

**게시글 — 댓글 `Non-Identifying | Zero or Many`**
- 게시글 ID는 댓글 테이블의 PK가 아니다(FK로만 사용).
- 하나의 댓글은 반드시 하나의 게시글에만 속한다.

**회원 — 리프레쉬 토큰 `Non-Identifying | Zero or One`**
- 한 회원은 Refresh Token을 0개 또는 최대 1개만 가진다.
- RTR(Refresh Token Rotation) 전략을 적용하므로 토큰 발급 시 기존 토큰을 교체한다.

---

## 2. DDL — 테이블 생성

```sql
-- 1. 회원 테이블 (users)
CREATE TABLE users (
    user_id      BIGINT       AUTO_INCREMENT PRIMARY KEY COMMENT '회원 ID',
    user_email   VARCHAR(100) NOT NULL                   COMMENT '이메일',
    user_pwd     VARCHAR(255) NOT NULL                   COMMENT '비밀번호',
    user_name    VARCHAR(20)  NOT NULL                   COMMENT '회원명',
    profile_image VARCHAR(255) NULL                      COMMENT '프로필사진',
    reg_dat      DATETIME     NOT NULL                   COMMENT '등록일자',
    upd_dat      DATETIME     NOT NULL                   COMMENT '수정일자'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='회원';

-- 2. 게시글 테이블 (posts)
CREATE TABLE posts (
    post_id    BIGINT        AUTO_INCREMENT PRIMARY KEY COMMENT '게시글 ID',
    user_id    BIGINT        NOT NULL                   COMMENT '작성자 회원 ID',
    title      VARCHAR(100)  NOT NULL                   COMMENT '제목',
    content    TEXT          NOT NULL                   COMMENT '내용',
    image      VARCHAR(255)  NULL                       COMMENT '이미지',
    view_count INT           NOT NULL DEFAULT 0         COMMENT '조회수',
    reg_dat    DATETIME      NOT NULL                   COMMENT '등록일자',
    upd_dat    DATETIME      NOT NULL                   COMMENT '수정일자',
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='게시글';

-- 3. 댓글 테이블 (comments)
CREATE TABLE comments (
    comment_id BIGINT   AUTO_INCREMENT PRIMARY KEY COMMENT '댓글 ID',
    user_id    BIGINT   NOT NULL                   COMMENT '작성자 회원 ID',
    post_id    BIGINT   NOT NULL                   COMMENT '대상 게시글 ID',
    content    TEXT     NOT NULL                   COMMENT '내용',
    reg_dat    DATETIME NOT NULL                   COMMENT '등록일자',
    upd_dat    DATETIME NOT NULL                   COMMENT '수정일자',
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='댓글';

-- 4. 좋아요 테이블 (likes)
CREATE TABLE likes (
    user_id BIGINT NOT NULL COMMENT '회원 ID',
    post_id BIGINT NOT NULL COMMENT '게시글 ID',
    PRIMARY KEY (user_id, post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='좋아요';

-- 5. 리프레쉬 토큰 테이블 (refresh_tokens)
CREATE TABLE refresh_tokens (
    token_id   BIGINT       AUTO_INCREMENT PRIMARY KEY COMMENT '토큰 ID',
    token      VARCHAR(500) NOT NULL UNIQUE            COMMENT '토큰',
    user_id    BIGINT       NOT NULL                   COMMENT '회원 ID',
    expires_at DATETIME     NOT NULL                   COMMENT '만료예정',
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='리프시 토큰';
```

### DDL 설계 근거

**AUTO_INCREMENT**

데이터를 INSERT할 때마다 PK 값을 1씩 자동 증가시킨다. 직접 ID를 관리하다 발생하는 중복 문제를 원천 차단하고, INSERT 시 ID를 별도로 지정하지 않아도 된다.

**ENGINE=InnoDB**

MySQL의 기본 엔진이 InnoDB이기 때문에 명시하지 않아도 적용된다. 하지만 DBA 설정이나 환경에 따라 기본값이 달라질 수 있으므로, **항상 명시적으로 선언**하는 것이 안전하다.

InnoDB를 선택한 이유는 트랜잭션 지원, 외래키 제약, 행 단위 잠금 등 RDBMS의 핵심 기능을 모두 제공하기 때문이다. MyISAM은 이 기능들이 없다.

**utf8mb4**

`utf8` 설정은 최대 3바이트까지만 지원한다. 이모지(😊, 🚀)나 일부 희귀 한자, 특수 수학 기호 등은 4바이트를 요구한다. `utf8`을 사용하면 이모지가 포함된 텍스트를 저장할 때 에러가 발생하기 때문에 `utf8mb4`를 사용했다.

**ON DELETE CASCADE**

부모 레코드가 삭제되면 자식 레코드도 함께 삭제된다. 회원이 탈퇴하면 그 회원의 게시글, 댓글, 좋아요, 리프레쉬 토큰이 모두 자동으로 정리된다. 고아 데이터가 DB에 남지 않도록 보장한다.

---

## 3. 인증·인가 구현 (Spring)

### 기술 선택 이유

#### 세션 vs JWT

| 방식 | 특징 | 선택 여부 |
|------|------|----------|
| 세션(Session) | 서버에 상태 저장 필요, 서버 수평 확장 시 세션 공유 문제 발생 | ❌ 미선택 |
| **JWT** | Stateless, 확장성 우수, 모바일/SPA 클라이언트에서도 쉽게 사용 가능 | ✅ 선택 |

세션 방식은 서버가 여러 대로 늘어날 때(스케일 아웃) Redis 같은 별도의 세션 공유 스토리지가 필요하다. JWT는 토큰 자체에 사용자 정보를 담기 때문에 어느 서버가 요청을 받아도 동일하게 처리할 수 있다.

#### Access Token + Refresh Token 구조

처음에는 Access Token 하나만 쓰는 구조를 생각했다. 하지만 유효기간 설정에서 딜레마가 생겼다.

- 짧게 설정 → 사용자가 자주 로그인해야 해서 불편
- 길게 설정 → 토큰 탈취 시 대응 불가

그래서 Access Token은 짧게 유지하고, Refresh Token을 DB에 저장해 갱신하는 **RTR(Refresh Token Rotation)** 구조를 적용했다.

| 토큰 | 역할 | 유효기간 | 저장 위치 |
|------|------|---------|----------|
| Access Token | 매 API 요청에 사용하는 출입증 | 짧게 (30분 내외) | 클라이언트 메모리 |
| Refresh Token | Access Token 재발급용 인증서 | 길게 (2주 내외) | DB + HttpOnly 쿠키 |

Refresh Token을 **HttpOnly 쿠키**로 전달한 이유는 JavaScript에서 접근이 불가능하여 XSS 공격으로 탈취되는 것을 방어할 수 있기 때문이다.

#### JPA Auditing 사용 이유

모든 테이블에 `reg_dat`(등록일자), `upd_dat`(수정일자) 컬럼이 필요했다. 처음에는 각 저장 로직마다 `LocalDateTime.now()`를 직접 넣으려 했는데 두 가지 문제가 있었다.

1. 개발자가 실수로 누락할 수 있다.
2. 모든 INSERT/UPDATE 로직에 중복 코드가 생긴다.

Spring Data JPA의 `@EnableJpaAuditing`과 `@CreatedDate`, `@LastModifiedDate` 어노테이션을 사용하면 프레임워크가 자동으로 처리해준다.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime regDat;

    @LastModifiedDate
    private LocalDateTime updDat;
}
```

모든 엔티티가 `BaseEntity`를 상속하면 등록/수정 일자를 신경 쓰지 않아도 된다.

#### 예외 처리 구조

처음에는 각 Service 메서드에서 `try-catch`를 직접 쓰거나 `RuntimeException`을 던지는 방식을 생각했다. 하지만 이렇게 하면 HTTP 상태코드를 어디서 결정해야 하는지 불명확해지고, 중복 코드가 늘어난다.

아래 구조로 개선했다.

```
BusinessException
    └── AuthorizedException (401 고정)

@RestControllerAdvice
    └── GlobalExceptionHandler (예외를 한 곳에서 처리)
```

`BusinessException`에 에러 메시지(code)와 `HttpStatus`를 함께 보유하도록 설계했다. 덕분에 Service 계층에서 예외를 던지면 `GlobalExceptionHandler`가 잡아서 일관된 응답 형식으로 반환한다.

---

### 전체 패키지 구조

```
com.example.Community
├── auth/          JWT 관련 (JwtProvider, JwtAuthenticationFilter, JwtProperties)
├── config/        시드 데이터 설정 (SeedConfig)
├── controller/    API 엔드포인트 (Auth, Post, User)
├── service/       비즈니스 로직 (Auth, Post, User)
├── domain/
│   ├── entity/    JPA 엔티티 (User, Post, RefreshToken)
│   └── repository/ Spring Data JPA 인터페이스
├── dto/           요청/응답 DTO
└── exception/     예외 처리 (BusinessException, GlobalExceptionHandler)
```

레이어드 아키텍처를 기반으로 패키지를 구성했다. `auth` 패키지는 JWT 관련 코드만 모아두어 인증 로직이 흩어지지 않도록 했다.

---

### 주요 구현 설명

#### JWT 인증 필터 (JwtAuthenticationFilter)

Spring의 `OncePerRequestFilter`를 상속하여 모든 요청에 JWT 검증을 수행한다. 회원가입, 로그인, 토큰 갱신은 인증이 필요 없는 경로이므로 WHITE_LIST로 필터를 우회하도록 처리했다.

```java
// WHITE_LIST에 있는 경로는 필터를 건너뜀
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    String path = request.getRequestURI();
    return WHITE_LIST.stream().anyMatch(path::startsWith);
}

// Authorization 헤더에서 Bearer 토큰 추출 후 검증
@Override
protected void doFilterInternal(HttpServletRequest request, ...) {
    String token = resolveToken(request);
    if (jwtProvider.validate(token)) {
        Long userId = jwtProvider.getUserId(token);
        request.setAttribute("userId", userId);  // 컨트롤러에서 사용
    }
    filterChain.doFilter(request, response);
}
```

검증 성공 시 `userId`를 `request.setAttribute`에 저장하여 컨트롤러에서 꺼내 쓸 수 있게 했다. `SecurityContext`를 사용하지 않아 구조가 단순하다.

#### 게시글 권한 처리

게시글 수정/삭제 시, JWT에서 추출한 `userId`와 게시글 작성자의 ID를 비교한다.

```java
if (!post.getAuthor().getId().equals(userId)) {
    throw new AuthorizedException("forbidden");
}
```

본인 게시글이 아니면 `AuthorizedException`을 던지고, `GlobalExceptionHandler`가 `401 Unauthorized`로 응답한다.

---

## 4. 회고

이번 프로젝트를 진행하면서 가장 많이 고민한 부분은 크게 세 가지였다.

**첫 번째는 자료형 선택이다.** `INT`냐 `BIGINT`냐, `VARCHAR`냐 `TEXT`냐 같은 선택들이 처음에는 별것 아닌 것처럼 보였다. 하지만 직접 트레이드오프를 정리해보니, 이런 결정들이 쌓여서 서비스의 성능과 확장성을 결정한다는 걸 체감했다.

**두 번째는 인증 구조다.** Access Token 하나만 쓰다가 RTR까지 도입하는 과정이 번거롭게 느껴지기도 했다. 하지만 보안 취약점을 직접 따라가다 보니 각 구조가 왜 필요한지 납득하면서 구현할 수 있었다.

**세 번째는 예외 처리다.** 처음에는 `try-catch`를 곳곳에 뿌리다가 `GlobalExceptionHandler`로 일원화하는 과정에서, 코드가 얼마나 깔끔해지는지를 직접 경험했다. 구조를 잡는 데 시간이 걸리더라도 미리 설계하는 게 결국 빠르다는 것도 배웠다.
