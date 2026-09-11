# 커뮤니티 사이트 DB 구조 정리

> 테이블명은 전부 `ls_` 접두사를 붙임 (Leetudy Study 약자)
> 로그인/비밀번호는 Supabase Auth(`auth.users`)가 자체적으로 관리하므로, 아래 테이블에는 비밀번호를 따로 저장하지 않음. (`기획서_보안점.md` 참고)

## 사용자 정보 (`ls_users`)

회원가입 화면 + 마이페이지에서 필요한 정보들

| 컬럼 | 설명 |
|---|---|
| `id` | 사용자 고유 번호 (Supabase Auth의 `auth.users.id`와 동일한 값을 사용, uuid) |
| `email` | 이메일 (로그인 계정, 가입 시 중복 검사만 진행 / 실제 인증은 `auth.users`가 관리) |
| `nickname` | 닉네임(이름) |
| `phone` | 전화번호 |
| `profile_image_url` | 프로필 사진 (마이페이지에서 수정 가능) |
| `created_at` | 가입일 (자동 기록) |
| `updated_at` | 정보 마지막 수정일 |

## 카테고리 정보 (`ls_categories`)

스터디 주제별 카테고리 분류(태그) 기능에 필요

| 컬럼 | 설명 |
|---|---|
| `id` | 카테고리 번호 (자동으로 1, 2, 3... 증가) |
| `name` | 카테고리 이름 (예: 어학, 자격증, 코딩, 취업 등) |
| `created_at` | 등록일 |

## 게시물 정보 (`ls_posts`)

게시물 목록/상세, 글쓰기 화면에서 필요한 정보들

| 컬럼 | 설명 |
|---|---|
| `id` | 게시물 번호 (자동으로 1, 2, 3... 증가) |
| `user_id` | 작성자 (어떤 사용자가 썼는지 - `ls_users` 테이블과 연결) |
| `category_id` | 스터디 주제 (어떤 카테고리에 속하는지 - `ls_categories` 테이블과 연결) |
| `title` | 제목 |
| `content` | 내용(본문) |
| `memo` | 메모칸 |
| `like_count` | 좋아요 수 (목록 카드에 바로 보여주기 위한 캐시용 숫자, `ls_post_likes` 개수와 동기화) |
| `comment_count` | 댓글 수 (목록 카드에 바로 보여주기 위한 캐시용 숫자, `ls_comments` 개수와 동기화) |
| `created_at` | 작성일 |
| `updated_at` | 수정일 (마지막으로 언제 수정했는지) |

## 게시물 사진 정보 (`ls_post_images`)

게시물에 사진을 여러 장 업로드할 수 있도록 별도 테이블로 분리

| 컬럼 | 설명 |
|---|---|
| `id` | 사진 번호 (자동으로 1, 2, 3... 증가) |
| `post_id` | 어떤 게시물에 첨부된 사진인지 (`ls_posts` 테이블과 연결) |
| `image_url` | 사진 파일 경로 (Supabase Storage 주소) |
| `created_at` | 업로드일 |

## 댓글 정보 (`ls_comments`)

댓글 + 대댓글 화면에서 필요한 정보들

| 컬럼 | 설명 |
|---|---|
| `id` | 댓글 번호 (자동으로 1, 2, 3... 증가) |
| `post_id` | 게시물 (어떤 게시물에 달린 댓글인지 - `ls_posts` 테이블과 연결) |
| `user_id` | 작성자 (어떤 사용자가 썼는지 - `ls_users` 테이블과 연결) |
| `parent_comment_id` | 부모 댓글 번호 (대댓글일 경우에만 값이 있음, 없으면 최상위 댓글 / `ls_comments` 자기 자신과 연결) |
| `content` | 댓글 내용 |
| `like_count` | 댓글 좋아요 수 (캐시용 숫자, `ls_comment_likes` 개수와 동기화) |
| `created_at` | 작성일 |
| `updated_at` | 수정일 |

## 게시물 좋아요 정보 (`ls_post_likes`)

누가 어떤 게시물에 좋아요를 눌렀는지 기록 (중복 좋아요 방지용)

| 컬럼 | 설명 |
|---|---|
| `id` | 좋아요 번호 (자동으로 1, 2, 3... 증가) |
| `post_id` | 좋아요 누른 게시물 (`ls_posts` 테이블과 연결) |
| `user_id` | 좋아요 누른 사용자 (`ls_users` 테이블과 연결) |
| `created_at` | 좋아요 누른 시간 |

> `(post_id, user_id)` 조합은 중복 저장 불가 (한 사람이 같은 글에 좋아요 1번만)

## 댓글 좋아요 정보 (`ls_comment_likes`)

누가 어떤 댓글에 좋아요를 눌렀는지 기록 (중복 좋아요 방지용)

| 컬럼 | 설명 |
|---|---|
| `id` | 좋아요 번호 (자동으로 1, 2, 3... 증가) |
| `comment_id` | 좋아요 누른 댓글 (`ls_comments` 테이블과 연결) |
| `user_id` | 좋아요 누른 사용자 (`ls_users` 테이블과 연결) |
| `created_at` | 좋아요 누른 시간 |

> `(comment_id, user_id)` 조합은 중복 저장 불가 (한 사람이 같은 댓글에 좋아요 1번만)

## 테이블 연결 관계

- 사용자 → 게시물: 한 사용자가 여러 게시물 작성 가능 (`ls_users` 1 : N `ls_posts`)
- 사용자 → 댓글: 한 사용자가 여러 댓글 작성 가능 (`ls_users` 1 : N `ls_comments`)
- 게시물 → 댓글: 한 게시물에 여러 댓글 작성 가능 (`ls_posts` 1 : N `ls_comments`)
- 댓글 → 대댓글: 한 댓글에 여러 대댓글 작성 가능 (`ls_comments` 1 : N `ls_comments`, 자기참조)
- 게시물 → 사진: 한 게시물에 여러 사진 첨부 가능 (`ls_posts` 1 : N `ls_post_images`)
- 카테고리 → 게시물: 한 카테고리에 여러 게시물 소속 가능 (`ls_categories` 1 : N `ls_posts`)
- 사용자 ↔ 게시물 (좋아요): 좋아요 기록으로 다대다 관계 연결 (`ls_post_likes`)
- 사용자 ↔ 댓글 (좋아요): 좋아요 기록으로 다대다 관계 연결 (`ls_comment_likes`)

## 마이페이지 화면 연결 방법 (참고)

- **내가 쓴 글**: `ls_posts`에서 `user_id = 내 id`로 조회
- **내가 쓴 댓글**: `ls_comments`에서 `user_id = 내 id`로 조회
- **좋아요한 게시글**: `ls_post_likes`에서 `user_id = 내 id`인 것을 찾은 뒤, 연결된 `ls_posts` 조회

## 보안 관련 참고사항

(`기획서_보안점.md` 요약)

- 모든 테이블은 **RLS(Row Level Security)** 적용: 본인 데이터(`user_id = auth.uid()`)만 수정/삭제 가능하도록 DB 레벨에서 강제
- `ls_post_images`는 Storage 버킷 업로드 권한을 로그인 사용자로 제한, 이미지 확장자/용량 검증
