# 리그 오브 레전드 전적 SNS DB 구조 정리

> 테이블명은 서비스 이름(`our.gg`)의 약자인 `og_` 접두사를 붙여 관리합니다.

## 사용자 정보 (`og_users` 테이블)

회원가입/프로필 화면에서 필요한 정보들:

| 필드 | 설명 |
|---|---|
| `id` | 사용자 고유번호 - 자동으로 1, 2, 3... 증가 |
| `email` | 로그인할 때 사용하는 이메일, **UNIQUE 제약 필요** |
| `password_hash` | 로그인 비밀번호 - bcrypt/argon2 등으로 해시한 값만 저장, 평문 저장 금지 *(수정: `password` → `password_hash`로 명확화)* |
| `nickname` | 앱 내 프로필 및 댓글에 표시되는 별명, **UNIQUE 제약 필요** |
| `birth_date` | 만 14세 미만 가입 제한 검증용 생년월일 |
| `profile_image_url` | 프로필 이미지 경로 |
| `bio` | 프로필에 보이는 자기소개글 |
| `riot_game_name` | 라이엇 게임 이름 - 예: `Hide on bush` |
| `riot_tag_line` | 라이엇 태그 라인 - 예: `KR1` |
| `puuid` | Riot API 전적 조회용 라이엇 고유 식별자, **UNIQUE 제약 필요** (앱 계정과 1:1 매핑) |
| `riot_region` | *(신규)* Riot API 라우팅용 리전/플랫폼 값 - 예: `kr`. 1차 개발은 KR 고정이라도 컬럼은 미리 확보 |
| `riot_verified` | *(신규)* Riot ID가 RSO(Riot Sign-On) OAuth로 본인 인증되었는지 여부. MVP는 단순 입력만 받더라도 추후 RSO 전환 시 구분 가능하도록 컬럼 확보 |
| `summoner_level` | 라이엇 소환사 레벨 |
| `tier_info` | 현재 티어 및 LP 요약 정보 — *(수정)* `tier` / `rank` / `league_points`로 세분화 저장 권장. 랭킹 정렬·필터링 시 문자열 하나보다 다루기 쉬움 |
| `terms_agreed_at` | *(신규)* 이용약관 동의 일시 — 회원가입 시 필수 동의 절차의 증빙 |
| `privacy_agreed_at` | *(신규)* 개인정보처리방침 동의 일시 |
| `deleted_at` | *(신규)* 회원 탈퇴 처리 일시 — NULL이면 활성 계정. 탈퇴 시 행을 바로 삭제하지 않고 이 값만 채운 뒤 개인정보를 익명화하는 소프트 삭제 방식 권장 (`og_posts`/`og_comments`가 `user_id`를 참조하므로 하드 삭제 시 데이터 정합성 깨짐) |
| `created_at` | 언제 가입했는지 |
| `updated_at` | *(신규)* 프로필 정보 마지막 수정 시각 |

## 게시물 정보 (`og_posts` 테이블)

SNS 피드 화면에서 필요한 정보들:

| 필드 | 설명 |
|---|---|
| `id` | 게시물 고유번호 - 자동 증가 |
| `user_id` | 작성자 ID - `og_users` 테이블과 연결 |
| `match_id` | 라이엇 API 게임 매치 고유 ID. 같은 매치를 여러 참가자가 각자 게시할 수 있어 `match_id` 자체는 유일하지 않음 → *(수정)* **`UNIQUE(user_id, match_id)`** 로 같은 유저의 동일 매치 중복 게시만 방지 |
| `caption` | 전적 공유 시 작성한 한 줄 소감/훈수·피드백 요청글 |
| `visibility` | 공개 범위 - 전체공개/친구공개/비공개 |
| `game_mode` | 게임 모드 - 솔로랭크/자유랭크/칼바람 등 |
| `game_duration` | 게임 진행 시간 - 초 단위 |
| `game_creation` | 게임을 플레이한 시각 |
| `is_win` | 승리 여부 - TRUE면 승리, FALSE면 패배 카드 색상 적용 |
| `champion_id` | *(수정)* 플레이한 챔피언의 **Riot Data Dragon 기준 숫자 ID**. 챔피언 이름/아이콘은 클라이언트에서 정적 데이터로 매핑해 표시 (기존엔 "이름/ID" 혼용 설명이었음) |
| `kills` / `deaths` / `assists` | 킬 / 데스 / 어시스트 수 |
| `damage_dealt` | 챔피언에게 가한 피해량 |
| `match_detail_json` | 10인 전체 KDA, 아이템, 룬 등 라이엇 상세 데이터 (API 재호출을 줄이기 위한 저장용) |
| `is_hidden` | *(신규)* 신고 누적 등으로 운영진이 숨김 처리했는지 여부 |
| `created_at` | 언제 피드에 올렸는지 |
| `updated_at` | *(신규)* 게시물 정보 수정 시각 |

### 게시물 작성 방식

1. 연동된 계정의 최근 매치 목록을 라이엇 API로 불러와 목록(Picker)으로 보여주기
2. 피드에 올릴 공유하고 싶은 게임 매치 1개 선택
3. 한 줄 소감/피드백 요청글(`caption`), 공개범위 선택 — ⚠️ *(수정)* 기존에 있던 "게임 타임라인 포인트"는 댓글(`og_comments.timeline_tag`) 기능이며 게시물 작성 단계와는 무관해 제거
4. 선택한 매치의 요약 및 상세 데이터와 작성 글을 `og_posts` 테이블에 저장

## 댓글 정보 (`og_comments` 테이블)

| 필드 | 설명 |
|---|---|
| `id` | 댓글 번호 - 자동 증가 |
| `post_id` | 어떤 게시물의 댓글인지 - `og_posts` 테이블과 연결 |
| `user_id` | 작성자 - `og_users` 테이블과 연결 |
| `content` | 댓글에 쓴 피드백/훈수 글 |
| `timeline_tag` | 특정 게임 시간대 지적용 타임라인 태그 - 예: `15:20` |
| `is_hidden` | *(신규)* 신고 처리로 숨김 여부 |
| `created_at` | 언제 댓글을 썼는지 |

## 감정표현 정보 (`og_post_reactions` 테이블)

| 필드 | 설명 |
|---|---|
| `id` | 반응 번호 - 자동 증가 |
| `post_id` | 어떤 게시물에 반응했는지 - `og_posts` 테이블과 연결 |
| `user_id` | 누가 반응했는지 - `og_users` 테이블과 연결 |
| `reaction_type` | 반응 종류 - 좋아요/캐리/버스/훈수필요 중 하나로 값 범위 제한 권장 |
| `created_at` | 언제 반응을 남겼는지 |

> ⚠️ *(수정)* **제약조건 명시**: `UNIQUE(post_id, user_id)` — 한 사용자가 한 게시물에 반응을 1개만 남길 수 있도록 강제. 기존 문서 하단 "연결 관계"에는 서술돼 있었지만 실제 제약조건으로는 누락돼 있었음.

## 친구/팔로우 정보 (`og_follows` 테이블)

| 필드 | 설명 |
|---|---|
| `id` | 팔로우 번호 - 자동 증가 |
| `follower_id` | 팔로우를 누른 사용자 - `og_users` 테이블과 연결 |
| `following_id` | 팔로우를 당한 사용자 - `og_users` 테이블과 연결 |
| `created_at` | 언제 팔로우했는지 |

> *(신규)* **제약조건**: `UNIQUE(follower_id, following_id)`로 중복 팔로우 방지, `CHECK(follower_id <> following_id)`로 자기 자신 팔로우 방지

## 알림 정보 (`og_notifications` 테이블)

| 필드 | 설명 |
|---|---|
| `id` | 알림 번호 - 자동 증가 |
| `receiver_id` | 알림을 받을 사용자 - `og_users` 테이블과 연결 |
| `actor_id` | 알림을 발생시킨 사용자 - `og_users` 테이블과 연결 |
| `type` | *(수정)* `follow` / `comment` / `reaction` / `tier_up` 중 하나로 값 범위 고정 (기존엔 예시 나열이었음) |
| `target_post_id` | 관련 게시물 번호 - `og_posts` 테이블과 연결. 팔로우 알림처럼 게시물과 무관한 경우 NULL 허용 |
| `is_read` | 알림 읽음 여부 |
| `created_at` | 언제 알림이 왔는지 |

## 신고 및 차단 정보 (`og_reports` / `og_blocks` 테이블)

**`og_blocks`**: `blocker_id`(차단한 유저) / `blocked_id`(차단당한 유저) / `created_at`(차단일)
> *(신규)* 제약조건: `UNIQUE(blocker_id, blocked_id)`, `CHECK(blocker_id <> blocked_id)`

**`og_reports`**: `reporter_id`(신고자) / **`target_type`** *(신규 — "post" 또는 "comment". 기존엔 `target_id`만 있어 대상이 게시물인지 댓글인지 구분 불가했음)* / `target_id`(신고 대상) / `reason`(신고 사유) / **`status`** *(신규 — "pending"/"reviewed"/"actioned", 운영자 검토 프로세스용)* / `created_at`(신고일) / **`resolved_at`** *(신규, 처리일)*

## 설정 정보 (`og_user_settings` 테이블) *(신규 추가 테이블)*

마이페이지 &gt; 설정 화면(`기획서.md` 3-11. 설정 페이지)에 대응:

| 필드 | 설명 |
|---|---|
| `user_id` | 설정 소유자 - `og_users`와 1:1 연결 (PK 겸 FK) |
| `notify_follow` | 팔로우 알림 수신 여부, 기본값 `TRUE` |
| `notify_comment` | 댓글/피드백 알림 수신 여부, 기본값 `TRUE` |
| `notify_reaction` | 좋아요·감정표현 알림 수신 여부, 기본값 `TRUE` |
| `notify_tier_change` | 승급/티어 변동 알림 수신 여부, 기본값 `TRUE` |
| `theme` | 라이트/다크 모드 선택, 기본값 `dark` |
| `updated_at` | 설정 마지막 변경 시각 |

## SNS 테이블 연결 관계 (단순화 버전)

- **사용자 ↔ 게시물**: 한 사용자가 여러 전적 게시물 작성 가능 (단, 동일 매치는 유저당 1개만 게시 — `UNIQUE(user_id, match_id)`)
- 게시물 작성 시 라이엇 API로 최근 매치 선택 → 선택한 매치 데이터와 글 내용을 `og_posts`에 저장
- **게시물 ↔ 감정표현**: 한 사용자가 한 게시물에 1개의 감정표현만 등록 가능 (`og_post_reactions` + `UNIQUE(post_id, user_id)`)
- **사용자 ↔ 댓글**: 한 사용자가 여러 게시물에 댓글/피드백 가능 (타임라인 태그 첨부 가능)
- **게시물 ↔ 댓글**: 한 게시물에 여러 사용자가 댓글 가능
- **사용자 ↔ 팔로우**: 한 사용자가 여러 사용자를 팔로우해 메인 피드에서 전적 모아보기 가능 (`og_follows` + `UNIQUE(follower_id, following_id)`)
- **사용자 ↔ 설정**: *(신규)* 한 사용자는 정확히 1개의 `og_user_settings` 레코드를 가짐 (1:1, 가입 시 기본값으로 자동 생성)
- **신고 처리 흐름**: *(신규)* 신고(`og_reports`)가 누적되면 우선 대상 `og_posts`/`og_comments`의 `is_hidden`을 `TRUE`로 전환하고, 운영자가 `status`를 검토해 최종 처리(반려/조치)

## 삭제/탈퇴 정책 *(신규 추가)*

- **회원 탈퇴**: `og_users` 행을 즉시 삭제하지 않고 `deleted_at`만 채우는 소프트 삭제 사용. `email`/`nickname`/`profile_image_url`/`bio` 등 개인 식별 정보는 익명화 처리하고 화면엔 "탈퇴한 사용자"로 표시. 작성했던 `og_posts`/`og_comments`는 유지해 다른 사용자의 피드/댓글 맥락이 깨지지 않도록 함
- **게시물/댓글 숨김**: 신고 처리 결과 `is_hidden = TRUE`로 전환된 콘텐츠는 일반 피드/상세 화면에서 제외하되, 실제 행 삭제(hard delete)는 운영자가 최종 확인 후에만 수행

## 인덱스 제안 *(신규 추가)*

- `og_posts(user_id)`, `og_posts(created_at DESC)`, `og_posts(visibility, created_at DESC)` — 메인 피드/마이페이지 조회 성능용
- `og_posts(created_at DESC, damage_dealt DESC)` — "최근 24시간 최고 딜량" 하이라이트 조회용
- `og_comments(post_id)`, `og_post_reactions(post_id)` — 게시물 상세 화면 집계용
- `og_follows(follower_id)`, `og_follows(following_id)` — 팔로잉/팔로워 목록 및 피드 조합용
- `og_notifications(receiver_id, is_read, created_at DESC)` — 알림 페이지 조회용

## 운영/동기화 참고사항 *(신규 추가)*

- `tier_info`(티어 정보)는 Riot API가 실시간으로 푸시해주지 않으므로, 주기적인 배치 작업으로 재조회 후 이전 값과 비교해 변경 시에만 승급 알림(`og_notifications.type = tier_up`)을 생성하는 방식 필요
- Riot Development API Key는 24시간마다 만료되므로, 배치/서버 작업이 이 정책의 영향을 받는지 별도 확인 필요 (`기획서.md` 4. 핵심 기능 정의 참고)
