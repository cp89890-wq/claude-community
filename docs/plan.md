# 개발 계획서 (Next.js + Supabase 구현)

`docs/requirements.md`(요구사항 정의서)와 `docs/design.md`(디자인 분석)를 바탕으로 한 실행 개발 계획.

---

## 1. 폴더 구조

```
community/
├── app/
│   ├── layout.tsx                     # 루트 레이아웃 (전역 Provider, 하단 탭 포함)
│   ├── page.tsx                       # 홈(피드) '/'
│   ├── (auth)/
│   │   ├── login/page.tsx             # 로그인 '/login'
│   │   └── signup/page.tsx            # 회원가입 '/signup'
│   ├── posts/
│   │   ├── new/page.tsx               # 글 작성 '/posts/new'
│   │   └── [id]/
│   │       ├── page.tsx               # 글 상세 '/posts/[id]'
│   │       └── edit/page.tsx          # 글 수정 '/posts/[id]/edit'
│   ├── profile/
│   │   ├── page.tsx                   # 내 프로필 '/profile'
│   │   └── edit/page.tsx              # 프로필 수정 '/profile/edit'
│   ├── users/
│   │   └── [id]/page.tsx              # 다른 유저 프로필 '/users/[id]'
│   ├── settings/page.tsx              # 설정 '/settings'
│   └── auth/callback/route.ts         # Supabase Auth 콜백 핸들러
├── components/
│   ├── layout/                        # BottomTab, Header
│   ├── feed/                          # FeedItem, FeedList, PostForm, LikeButton
│   ├── comment/                       # CommentList, CommentInput
│   ├── profile/                       # ProfileHeader, ProfileForm
│   └── ui/                            # Button, Input, Modal, ActionSheet 등 공통 UI
├── lib/
│   └── supabase/
│       ├── client.ts                  # 브라우저(Client Component)용 Supabase 클라이언트
│       ├── server.ts                  # 서버(Server Component/Action)용 Supabase 클라이언트
│       └── middleware.ts              # 세션 갱신 헬퍼
├── types/
│   └── database.types.ts              # Supabase CLI로 생성되는 DB 타입
├── middleware.ts                      # 인증 세션 갱신 + 라우트 가드
├── supabase/
│   └── migrations/                    # SQL 마이그레이션 파일
├── public/
├── docs/
│   ├── requirements.md
│   ├── design.md
│   └── plan.md
├── tailwind.config.ts
├── next.config.js
└── package.json
```

---

## 2. 페이지 / 라우트 목록

| 경로 | 화면 | 인증 필요 | 비고 |
|---|---|---|---|
| `/signup` | 회원가입 | X | 이메일/비밀번호/닉네임 |
| `/login` | 로그인 | X | 이메일/비밀번호 |
| `/` | 홈(피드) | 조회 X / 상호작용 O | 하단 탭: 홈 |
| `/posts/new` | 글 작성 | O | 글쓰기 FAB → 이동 |
| `/posts/[id]` | 글 상세 | 조회 X / 상호작용 O | 댓글·좋아요 |
| `/posts/[id]/edit` | 글 수정 | O (작성자 본인) | |
| `/profile` | 내 프로필 | O | 하단 탭: 내정보 |
| `/profile/edit` | 프로필 수정 | O | 닉네임/아바타 |
| `/users/[id]` | 다른 유저 프로필 | X (조회) | 수정 버튼 없음 |
| `/settings` | 설정 | O | 하단 탭: 설정, 로그아웃 |
| `/auth/callback` | Auth 콜백 | X | Supabase 인증 리다이렉트 처리용 Route Handler |

**공통 레이아웃**: 하단 탭바(홈/내정보/설정)는 `app/layout.tsx`에서 인증 상태에 따라 노출 여부 제어.

---

## 3. DB 테이블 개요

`docs/requirements.md` 5장 기준. 4개 테이블 + Storage 2버킷.

| 테이블 | 핵심 컬럼 | 관계 |
|---|---|---|
| `profiles` | id(PK/FK→auth.users), nickname(unique), avatar_url, bio, created_at | auth.users 1:1 |
| `posts` | id(PK), user_id(FK), title, content, image_url, created_at, updated_at | profiles 1:N |
| `comments` | id(PK), post_id(FK), user_id(FK), content, created_at | posts 1:N, profiles 1:N |
| `likes` | id(PK), post_id(FK), user_id(FK), created_at, unique(post_id, user_id) | posts 1:N, profiles 1:N |

**Storage 버킷**
| 버킷 | 용도 |
|---|---|
| `avatars` | 프로필 이미지 |
| `post-images` | 게시글 첨부 이미지 |

세부 컬럼/타입은 `docs/requirements.md` 5.1 참고. 신규 정의 없이 그대로 채택.

---

## 4. 구현 순서

### 0단계 — 프로젝트 셋업
- Next.js(App Router) 프로젝트 생성, Tailwind CSS 설정
- Supabase 프로젝트 생성, `supabase/migrations`로 4개 테이블 + Storage 버킷 생성
- `@supabase/ssr`, `@supabase/supabase-js` 설치, `lib/supabase/client.ts` / `server.ts` 작성
- 환경변수 설정(`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`)
- GitHub 저장소 ↔ Vercel 프로젝트 연결(프리뷰 배포 확인용, 실제 배포는 6단계)

### 1단계 — 인증 (FR-01~03)
- `/signup`, `/login` 페이지 및 폼 구현
- `auth.users` INSERT 시 `profiles` 자동 생성 트리거 작성
- `middleware.ts`에서 세션 갱신 + 비로그인 접근 가드
- 로그아웃 기능(설정 화면 연동)

### 2단계 — 피드 / 글 CRUD (FR-04~08)
- `posts` 목록(`/`), 상세(`/posts/[id]`), 작성(`/posts/new`), 수정(`/posts/[id]/edit`), 삭제
- 이미지 업로드(`post-images` 버킷 연동)
- 무한 스크롤 또는 페이지네이션

### 3단계 — 댓글 / 좋아요 (FR-09~11)
- `comments` 작성/삭제, 글 상세 화면에 연동
- `likes` 토글(추가/취소), unique 제약 기반 1인 1좋아요

### 4단계 — 프로필 (FR-12~14)
- 내 프로필(`/profile`), 프로필 수정(`/profile/edit`, 아바타 업로드)
- 다른 유저 프로필(`/users/[id]`)

### 5단계 — 설정 & 마무리
- `/settings` 로그아웃, 계정 관리(FR-15~16)
- 반응형/모바일 UI 다듬기, 로딩·에러·빈 상태 처리
- `docs/design.md` 색상/컴포넌트 톤 반영한 UI 폴리싱

### 6단계 — 배포
- Vercel 프로덕션 배포, 환경변수 등록
- Supabase 프로덕션 RLS 정책 최종 점검
- 도메인 연결(필요 시)

---

## 5. 주의점

### RLS (Row Level Security)
- 4개 테이블 모두 RLS 활성화 필수
- `posts`, `comments`: SELECT는 전체 공개, INSERT/UPDATE/DELETE는 `auth.uid() = user_id`만 허용
- `likes`: INSERT/DELETE는 `auth.uid() = user_id`만, SELECT는 공개(좋아요 수 집계용)
- `profiles`: SELECT는 공개, UPDATE는 본인만(`auth.uid() = id`)
- Storage 버킷도 RLS 정책 필요: 본인 소유 파일만 업로드/삭제 가능하도록 경로에 `user_id` 포함 권장(예: `avatars/{user_id}/...`)

### 로그인 / 세션 관리
- Next.js App Router에서는 `@supabase/ssr` 사용, 클라이언트/서버 쿠키 동기화 필요
- `middleware.ts`에서 매 요청마다 세션 갱신(refresh) 처리하지 않으면 서버 컴포넌트에서 세션이 유실될 수 있음
- 비로그인 사용자의 접근 범위: 피드 목록/상세, 다른 유저 프로필은 조회 가능 / 글쓰기·댓글·좋아요·내 프로필·설정은 로그인 필요 → 미들웨어 또는 페이지 레벨에서 이중 가드

### 회원가입 / profiles 동기화
- `profiles.nickname`은 unique 제약 → 가입 시 중복 에러 핸들링 필요
- `auth.users` → `profiles` 자동 생성은 DB 트리거(`on_auth_user_created`)로 처리해 애플리케이션 로직 누락 방지

### 좋아요 동시성
- 좋아요 토글은 `unique(post_id, user_id)` 제약 + upsert/delete 패턴으로 중복 클릭·동시 요청에 안전하게 처리

### 환경변수 / 보안
- `NEXT_PUBLIC_*` 키는 클라이언트에 노출되어도 되는 값(anon key)만 사용
- Service Role Key는 절대 클라이언트 코드/`NEXT_PUBLIC_*`에 넣지 않고, 서버 전용(관리자 작업 등) 용도로만 최소 사용

### 이미지 처리
- Next.js `<Image>` 사용 시 Supabase Storage 도메인을 `next.config.js`의 `images.remotePatterns`에 등록 필요
- 업로드 파일 크기/확장자 제한(클라이언트 검증 + Storage 정책)
