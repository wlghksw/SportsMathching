# KDT 스포츠 커뮤니티 플랫폼

![A-ezgif com-video-to-gif-converter](https://github.com/user-attachments/assets/936035cd-77ce-4479-b3e6-c41e12f21812)


## 1. 프로젝트 개요

- 천안 지역 생활 체육 동호인을 위한 Spring Boot 3.5 기반 웹 애플리케이션입니다.
- 사용자 가입, 팀 구성, 매치 일정 관리, 자기소개 게시판, 커뮤니티 게시판을 하나의 서비스에서 제공합니다.
- Mustache 서버 렌더링 + MariaDB 영속 계층 + Flask 욕설 필터 서비스를 연동해 운영합니다.

## 2. 핵심 기능 요약

- **회원 관리**: 커스텀 회원가입/로그인, 세션 기반 인증, 닉네임·전화번호 중복 검증, 관리자에 의한 정지/복구.
- **팀 허브**: 팀 생성·로고 업로드·정원 관리, 신청 상태(pending/active/inactive) 흐름, 선호 요일/지역/종목 필터링.
- **매치 센터**: 관리자 등록 매치, 종목별 포지션/요금 책정, 정원 자동 갱신, 신청 승인/거절, 개인 달력 데이터 노출.
- **커뮤니티**: 공지/자유 게시판, 키워드 검색, 수동 페이지네이션, 작성자·관리자 전용 수정/삭제.
- **MyPost(자기소개)**: 스포츠별 자기소개, 이미지/영상 업로드, 좋아요 & 대댓글, Flask 기반 욕설 차단.
- **관리자 콘솔**: 사용자/게시글/매치 신청 일괄 관리, 강제 탈퇴와 역할 변경.
- **파일 서비스**: `/uploads/**` 경로로 정적 제공, 팀 로고 및 게시물 미디어 보관.

## 3. 코드 구조 및 주요 컴포넌트 분석

```
src/main/java/com/example/KDT/
├─ config/        # 공통 Bean, Spring Security, Flask 연동, 정적 리소스 설정
├─ controller/    # MVC 엔드포인트 (Team, Match, MyPost, Community, Admin 등)
├─ dto/           # 뷰/폼 전송 객체 (TeamDTO, UserDTO 등)
├─ entity/        # JPA 엔티티 (User, Team, Match, Region, MyPost 등)
├─ integration/   # 외부 서비스 클라이언트 (AbuseFilterClient)
├─ repository/    # Spring Data JPA 인터페이스
└─ service/       # 비즈니스 로직 + 트랜잭션 경계
```

- **`SecurityConfig`**: 세션 인증, `UserDetailsService`에서 MariaDB 사용자 조회, `AuthenticationSuccessHandler`로 세션에 `loggedInUser` 저장. `/match/**` GET 전면 허용, 수정/삭제는 관리자 제한.
- **`FlaskServerStarter` / `SimpleFlaskStarter`**: 앱 부팅 시 Flask 욕설 필터를 자동 실행하거나, 수동 실행 안내 로그를 출력.
- **`MatchService`**: 스포츠별 포지션/정원/가격 상수 관리, 신청 승인 시 `MatchApplication` 상태와 `Match.currentPlayers`를 동시 갱신.
- **`TeamService`**: 팀장 확인 후 수정/삭제 허용, 신청 수락 시 `team.currentMembers` 증가, `TeamMember` 상태 머신(pending → active/inactive) 구현.
- **`MyPostController` & `MyPostCommentController`**: 세션 기반 인증, 조회수/좋아요 증가 로직, Flask 욕설 필터 호출 후 댓글 차단 분기 처리.
- **`CommunityController`**: Enum 카테고리 매핑, in-memory 페이지네이션 및 키워드 필터, 관리자/작성자 권한 체크.
- **`AdminController`**: 관리용 대시보드 + 사용자/게시글/매치 신청에 대한 CRUD 및 상태 변경 라우팅.

## 4. 기술 스택

- **백엔드**: Java 17, Spring Boot 3.5, Spring MVC, Spring Security, Spring Data JPA
- **프런트**: Mustache 템플릿, Bootstrap 기반 정적 자원(템플릿 내 포함)
- **DB**: MariaDB (기본), H2 (런타임 의존성)
- **빌드/유틸**: Gradle, Lombok, JUnit 5
- **외부 연동**: Reactor WebClient → Flask 욕설 필터(`forFlask.py`), JSON DB(`final_abuse_db.json`)

## 5. 개발 환경 & 설치 방법

1. **Java & Gradle**
   ```bash
   # macOS 예시
   brew install openjdk@17
   ./gradlew --version
   ```
2. **데이터베이스 준비**
   ```sql
   CREATE DATABASE sports_community CHARACTER SET utf8mb4;
   GRANT ALL PRIVILEGES ON sports_community.* TO 'cjh'@'localhost' IDENTIFIED BY '0829';
   FLUSH PRIVILEGES;
   ```
   - `src/main/resources/application.properties`에서 URL/계정을 환경에 맞게 변경합니다.
   - 참고: `data.sql`은 전체 테이블을 **TRUNCATE 후 샘플 데이터를 삽입**합니다. `spring.sql.init.mode` 값을 `always`로 바꾸면 애플리케이션 기동 시 자동 실행합니다.
3. **Flask 욕설 필터 (선택)**
   ```bash
   conda create -n myflask python=3.11
   conda activate myflask
   pip install flask
   python forFlask.py
   ```
   - 자동 실행을 원하면 macOS 기준 `/opt/anaconda3`에 conda가 있어야 합니다 (`FlaskServerStarter`).
   - 헬스체크: `curl http://127.0.0.1:5000/health`
   - 환경 변수: `REQUIRE_API_KEY`, `API_KEY`, `THRESHOLD`
4. **Spring Boot 실행**
   ```bash
   ./gradlew bootRun
   ```
   - 기본 포트: `http://localhost:8080`
   - 초기 관리자 계정: `admin / 1234` (`data.sql` 기준, 암호화 X)
5. **테스트**
   ```bash
   ./gradlew test
   ```
   - 현재는 컨텍스트 로드 테스트만 존재합니다. 서비스별 단위/통합 테스트 추가를 권장합니다.

## 6. 주요 API 문서 (요약)

> 인증: 별도 명시가 없으면 GET은 익명 허용, 작성/수정/삭제는 로그인 또는 관리자 권한 필요. `SecurityConfig` 참고.

### 6.1 인증/사용자

| 메서드   | 경로        | 설명             | 인증   | 비고                                       |
| -------- | ----------- | ---------------- | ------ | ------------------------------------------ |
| GET      | `/login`    | 로그인 페이지    | 익명   | `error`, `logout` 파라미터 처리            |
| POST     | `/logout`   | 로그아웃         | 로그인 | 세션 종료 후 `/main` 리다이렉트            |
| GET      | `/register` | 회원가입 폼      | 익명   | 초기 값 세팅                               |
| POST     | `/register` | 회원가입 처리    | 익명   | `UserService.signup`, 검증 실패 시 폼 유지 |
| GET      | `/check-id` | 아이디 중복 검사 | 익명   | 결과 메시지를 register.mustache에 바인딩   |
| GET/POST | `/find-id`  | 아이디 찾기      | 익명   | 이름/전화/출생년도 검증                    |

### 6.2 메인/홈

| 메서드 | 경로    | 설명     | 인증 | 비고                                  |
| ------ | ------- | -------- | ---- | ------------------------------------- |
| GET    | `/main` | 대시보드 | 선택 | 공지/자유 게시판 Top5, 팀/매치 데이터 |

### 6.3 팀(teams)

| 메서드 | 경로                                                   | 설명                | 인증   | 비고                                       |
| ------ | ------------------------------------------------------ | ------------------- | ------ | ------------------------------------------ |
| GET    | `/teams/home`                                          | 팀 목록 & 신청 현황 | 선택   | 키워드 검색, 로그인 시 myTeams 블록        |
| GET    | `/teams/new`                                           | 팀 등록 폼          | 로그인 | 팀장 후보만 접근                           |
| POST   | `/teams/save`                                          | 팀 생성             | 로그인 | 파일 업로드 포함, 최대 인원/현재 인원 입력 |
| GET    | `/teams/{id}`                                          | 팀 상세             | 선택   | 팀장/멤버 여부 판단                        |
| GET    | `/teams/{id}/edit`                                     | 팀 수정 폼          | 팀장   | 팀장 여부 확인 실패 시 RuntimeException    |
| POST   | `/teams/update`                                        | 팀 수정 저장        | 팀장   | DTO 검증 후 업데이트                       |
| POST   | `/teams/{id}/delete`                                   | 팀 삭제             | 팀장   | 관련 `TeamMember` 선 삭제                  |
| POST   | `/teams/{id}/apply`                                    | 팀 가입 신청        | 로그인 | 기존 상태에 따라 예외 처리                 |
| POST   | `/teams/{teamId}/applications/{applicationId}/accept`  | 신청 수락           | 팀장   | `pending` → `active`, 인원수 증가          |
| POST   | `/teams/{teamId}/applications/{applicationId}/decline` | 신청 거절           | 팀장   | `pending` → `inactive`                     |

### 6.4 매치(match)

| 메서드 | 경로                           | 설명                      | 인증   | 비고                             |
| ------ | ------------------------------ | ------------------------- | ------ | -------------------------------- |
| GET    | `/match`                       | 매치 메인 → 목록 리디렉션 | 익명   |                                  |
| GET    | `/match/list`                  | 전체 매치 목록            | 선택   | `MatchService.getAllMatches`     |
| GET    | `/match/sport/{sportId}`       | 종목별 목록               | 선택   |                                  |
| GET    | `/match/{matchId}`             | 상세 조회                 | 선택   | 작성자 여부 플래그               |
| GET    | `/match/{matchId}/apply`       | 신청 폼                   | 선택   | 종목에 따라 다른 Mustache 템플릿 |
| POST   | `/match/{matchId}/apply`       | 신청 처리                 | 로그인 | 포지션 중복/정원 검증 후 승인    |
| GET    | `/match/stats`                 | 통계 페이지               | 선택   | 매치 수/스포츠/지역/상태 분포    |
| GET    | `/match/create`                | 매치 등록 폼              | 관리자 |                                  |
| POST   | `/match/create`                | 매치 등록                 | 관리자 | 날짜/시간 문자열 파싱            |
| GET    | `/match/{matchId}/edit`        | 수정 폼                   | 관리자 |                                  |
| POST   | `/match/{matchId}/edit`        | 수정 저장                 | 관리자 |                                  |
| POST   | `/match/{matchId}/delete`      | 매치 삭제                 | 관리자 | 연관 신청 삭제 후 매치 제거      |
| GET    | `/match/venue/{venueId}/sport` | 구장 → 종목 조회          | 선택   | JSON 반환                        |

### 6.5 커뮤니티(community)

| 메서드 | 경로                                    | 설명               | 인증          | 비고                            |
| ------ | --------------------------------------- | ------------------ | ------------- | ------------------------------- |
| GET    | `/community`                            | 공지사항 기본 노출 | 선택          | category/keyword/page/size      |
| GET    | `/community/{category}`                 | 카테고리별 목록    | 선택          | `notice`, `community` 등        |
| GET    | `/community/{category}/new`             | 새 글 폼           | 로그인        |                                 |
| POST   | `/community/{category}`                 | 글 저장            | 로그인        | `CommunityPostService.savePost` |
| GET    | `/community/{category}/{postId}`        | 상세               | 선택          | 조회수 증가                     |
| GET    | `/community/{category}/{postId}/edit`   | 수정 폼            | 작성자/관리자 | 세션 사용자 검사                |
| POST   | `/community/{category}/{postId}/update` | 글 수정            | 작성자/관리자 |                                 |
| POST   | `/community/{category}/{postId}/delete` | 삭제               | 작성자/관리자 | Soft delete 없음                |

### 6.6 MyPost(자기소개)

| 메서드 | 경로                                      | 설명                       | 인증   | 비고                                       |
| ------ | ----------------------------------------- | -------------------------- | ------ | ------------------------------------------ |
| GET    | `/mypost`                                 | 자기소개 목록(전체 스포츠) | 선택   | 정렬/검색 파라미터                         |
| GET    | `/mypost/sport/{sportId}/intro`           | 종목별 목록                | 선택   | `sportId`별 필터                           |
| GET    | `/mypost/sport/{sportId}/intro/new`       | 작성 폼                    | 로그인 |                                            |
| POST   | `/mypost/sport/{sportId}/intro`           | 작성 (파일 포함)           | 로그인 | `MyPostService.createBasic` 후 미디어 저장 |
| POST   | `/mypost/sport/{sportId}/intro/new`       | (대체) 작성                | 로그인 | 동일 기능, 중복 엔드포인트                 |
| GET    | `/mypost/{id}`                            | 상세                       | 선택   | 조회수 증가, 댓글/좋아요 상태              |
| GET    | `/mypost/{id}/edit`                       | 수정 폼                    | 작성자 |                                            |
| POST   | `/mypost/{id}/edit`                       | 수정 저장                  | 작성자 |                                            |
| POST   | `/mypost/{id}/delete`                     | 삭제                       | 작성자 |                                            |
| POST   | `/mypost/{id}/like`                       | 좋아요 토글                | 로그인 | 이미 좋아요 → 취소                         |
| POST   | `/mypost/{id}/comment`                    | 댓글 작성                  | 로그인 | 빈 문자열 체크, AbuseFilter 연동           |
| POST   | `/mypost/{id}/comment/{commentId}/delete` | 댓글 삭제                  | 작성자 | Soft delete (`isActive=false`)             |
| POST   | `/mypost/{id}/comments`                   | (대체) 댓글 작성           | 로그인 | `MyPostCommentController` → AbuseFilter    |
| POST   | `/mypost/comments/{commentId}/delete`     | (대체) 댓글 삭제           | 로그인 |                                            |

### 6.7 관리자(admin)

| 메서드 | 경로                                     | 설명             | 인증   | 비고                              |
| ------ | ---------------------------------------- | ---------------- | ------ | --------------------------------- |
| GET    | `/admin`                                 | 대시보드         | 관리자 | 전체 게시글/사용자 카운트         |
| GET    | `/admin/posts`                           | 게시글 관리      | 관리자 | 커뮤니티 + MyPost 리스트          |
| GET    | `/admin/posts/search`                    | 키워드 검색      | 관리자 |                                   |
| GET    | `/admin/posts/search/username`           | 작성자 검색      | 관리자 |                                   |
| POST   | `/admin/posts/{id}/delete`               | 통합 게시글 삭제 | 관리자 | `Post` 기반                       |
| POST   | `/admin/community-posts/{postId}/delete` | 커뮤니티 삭제    | 관리자 | AJAX (문자열 반환)                |
| POST   | `/admin/my-posts/{postId}/delete`        | MyPost 삭제      | 관리자 | AJAX                              |
| GET    | `/admin/users`                           | 사용자 관리      | 관리자 | 게시글 수 계산, 활성/비활성 표시  |
| POST   | `/admin/users/{id}/delete`               | 사용자 삭제      | 관리자 |                                   |
| POST   | `/admin/users/{id}/role`                 | 역할 업데이트    | 관리자 | `role=ADMIN` → isAdmin=true       |
| POST   | `/admin/users/{id}/force-withdraw`       | 강제 탈퇴        | 관리자 | `isActive=false`                  |
| POST   | `/admin/users/{id}/restore`              | 계정 복구        | 관리자 |                                   |
| GET    | `/admin/applications`                    | 매치 신청 목록   | 관리자 | `ApplicationStatus.PENDING` 필터  |
| POST   | `/admin/applications/{id}/approve`       | 신청 승인        | 관리자 | `MatchService.approveApplication` |
| POST   | `/admin/applications/{id}/reject`        | 신청 거절        | 관리자 |                                   |

### 6.8 Flask 욕설 필터 API

| 메서드 | 경로              | 설명           | 입력                      | 출력                                       |
| ------ | ----------------- | -------------- | ------------------------- | ------------------------------------------ |
| GET    | `/health`         | 헬스체크       | 없음                      | 상태 JSON (`status`, `abuse_db_loaded` 등) |
| POST   | `/predict`        | 단일 댓글 판별 | `{ "comment": "..." }`    | `is_hate_speech`, `predicted_labels`       |
| POST   | `/predict_batch`  | 다건 판별      | `{ "comments": ["..."] }` | `results` 배열                             |
| POST   | `/filter_comment` | 레거시 호환    | `{ "comment": "..." }`    | `/predict`와 동일                          |

## 7. 배포/운영 체크리스트

- DB 자격 증명, 업로드 경로(`app.upload-dir`)를 환경 변수로 외부화.
- Flask 자동 실행 스크립트는 macOS 경로에 종속 → Linux/배포 환경에서는 스크립트 수정 또는 Docker/시스템 서비스로 관리.
- 비밀번호는 `{noop}` 평문 저장이므로 **반드시 `BCryptPasswordEncoder` 등으로 전환**하고 `SecurityConfig` 조정 필요.
- 현재 CSRF 비활성화됨 → 실서비스 전 폼별 CSRF 토큰 적용 고려.
- 커뮤니티/자기소개 페이징은 메모리 기반 → 대용량 데이터 시 DB 페이지네이션으로 리팩터링 권장.
- 업로드된 미디어 파일 정리(삭제, 용량 모니터링)를 위한 배치나 S3 연동 고려.


## 8. 향후 개선 아이디어

- Spring Data `Page<T>` 기반 API로 목록/검색 고도화.
- 서비스/컨트롤러 단위 테스트, 통합 테스트 케이스 확충.
- REST API 제공 및 SPA/모바일 앱 연계.
- 욕설 필터를 메시지 큐/비동기 처리로 전환해 응답 지연 최소화.
- 업로드 파일 스토리지 추상화 및 S3/GCS 통합.
