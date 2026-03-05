# 경영학전공 AI 인터랙티브 페이지 — 개발 문서

**작성일:** 2026-03-04
**최종 수정:** 2026-03-06
**버전:** v3 (v3 단독 운영)

---

## 목차
1. [시스템 아키텍처](#1-시스템-아키텍처)
2. [기술 스택](#2-기술-스택)
3. [랜딩페이지 구조](#3-랜딩페이지-구조)
4. [인증 시스템 (auth.js)](#4-인증-시스템)
5. [아바타 시스템 (InteractiveAvatarNextJSDemo)](#5-아바타-시스템)
6. [API 명세](#6-api-명세)
7. [행동 추적 시스템](#7-행동-추적-시스템)
8. [설문조사 시스템](#8-설문조사-시스템)
9. [트러블슈팅](#9-트러블슈팅)
10. [배포 구조](#10-배포-구조)
11. [HeyGen 동시 접속 제한](#11-heygen-동시-접속-제한)
12. [교수진 정보](#12-교수진-정보)
13. [OG 메타태그](#13-og-메타태그-sns-공유-미리보기)
14. [마이크 정책](#14-마이크-정책)
15. [Google Forms 설문 (백업용)](#15-google-forms-설문-백업용)
16. [변경 이력](#16-변경-이력)

---

## 1. 시스템 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                   사용자 (브라우저)                    │
└──────────┬──────────────────────────┬────────────────┘
           │                          │
    ┌──────▼──────┐          ┌────────▼────────┐
    │ 랜딩페이지   │◄──────►│  아바타 iframe    │
    │ GitHub Pages │ postMsg │  Netlify 배포    │
    │ (index.html) │         │  (Next.js)       │
    └──────┬──────┘          └───┬─────────┬───┘
           │                     │         │
    ┌──────▼──────┐      ┌──────▼───┐ ┌───▼────────┐
    │ 백엔드 API   │      │ HeyGen   │ │ OpenAI     │
    │ PHP 5.4.45   │      │ Avatar   │ │ GPT-4o-mini│
    │ aiforalab.com│      │ API      │ │            │
    └──────┬──────┘      └──────────┘ └────────────┘
           │
    ┌──────▼──────┐
    │ MySQL DB     │
    │ business_db  │
    └─────────────┘
```

### 통신 흐름
1. **카카오 로그인**: 랜딩페이지 → Kakao SDK → 백엔드 API → DB
2. **아바타 연결**: 랜딩페이지 → postMessage → iframe(아바타) → HeyGen API
3. **대화**: 아바타 → /api/chat → OpenAI GPT-4o-mini → 아바타 TTS 발화
4. **섹션 스크롤**: 아바타 → postMessage(NAVIGATE_TAB) → 랜딩페이지 scrollIntoView
5. **행동 추적**: 랜딩페이지 → IntersectionObserver → 백엔드 log_batch API
6. **설문조사**: 랜딩페이지 → 백엔드 survey_submit API → DB

---

## 2. 기술 스택

| 구분 | 기술 | 비고 |
|------|------|------|
| 프론트엔드 | HTML/CSS/JS (바닐라) | 프레임워크 미사용 |
| 아바타 앱 | Next.js + TypeScript | Netlify 배포 |
| 아바타 SDK | @heygen/streaming-avatar | WebRTC 스트리밍 |
| AI (대화) | OpenAI GPT-4o-mini | temperature: 0, max_tokens: 400 |
| 음성 인식 | Web Speech API (Chrome) | ko-KR, continuous mode |
| 인증 | Kakao SDK v1 | JS Key: fc0a1313d895b1956f3830e5bf14307b |
| 백엔드 | PHP 5.4.45 | `??` 연산자 사용 불가 |
| DB | MySQL (business_db) | user: user2, pw: user2!! |
| 폰트 | Noto Sans KR + DM Serif Display | Google Fonts |
| 배포 (페이지) | GitHub Pages | main 브랜치 |
| 배포 (아바타) | Netlify | 자동 배포 |

---

## 3. 랜딩페이지 구조

### 3.1 파일 위치
- **v3**: `cha-biz-ai-v3/index.html` + `js/auth.js`
- v3 단독 운영 (v2는 더 이상 동기화하지 않음)

### 3.2 디자인 시스템

**컬러 팔레트 (웜톤 통일)**
```css
--charcoal: #1c1c1c        /* 다크 배경 */
--cream: #f5f0e8            /* 라이트 배경 */
--gold: #c9a27e             /* 주요 액센트 */
--gold-dark: #a07850        /* 강조 */
--beige: #d4b896            /* 서브 액센트 */
--warm-white: #faf8f4       /* 밝은 배경 */
/* 강조 다크: #6b4c30 (다크 브라운) */
/* 네이비 사용 금지 — 웜톤 통일 */
```

**타이포그래피**
- Hero h1: `clamp(34px, 9vw, 60px)` — DM Serif Display
- 섹션 h2: `clamp(26px, 7vw, 44px)` — DM Serif Display
- 바디: 15px, Noto Sans KR 400, line-height 1.8
- 섹션 태그: 11px, letter-spacing 4px, uppercase

**반응형 브레이크포인트**
- `768px`: 태블릿 (그리드 → 1열)
- `640px`: 스탯 그리드 2열
- `440px`: 모바일 최적화

### 3.3 섹션 구조

| # | ID | 제목 | 배경 |
|---|-----|------|------|
| 0 | hero | Hero | charcoal |
| 1 | research | 교육목표 + 융합형 인재 (병합) | cream |
| 2 | curriculum | 커리큘럼 | charcoal |
| 3 | ai | 차별성 & 복수전공 | beige |
| 4 | careers | 취업 & 진로 | charcoal |
| 5 | career-detail | 5대 진로 상세 | beige |
| 6 | only-cha | 산학연병 강점 | charcoal |
| 7 | experience | 실전 경험 | cream |
| 8 | faq | FAQ | warm-white |
| 9 | footer | Footer/CTA | charcoal gradient |
| 10 | survey | 설문조사 | cream (숨김) |

> **변경 이력 (03-06):** goals 섹션과 research 섹션을 하나의 research 섹션으로 병합. 교육목표 3가지 + 벤 다이어그램 + 통계카드 + 교육특성 칩이 한 페이지에 표시.

### 3.4 인터랙션 기능

**스크롤 프로그레스 바**
```javascript
window.addEventListener('scroll', () => {
  const h = document.documentElement.scrollHeight - window.innerHeight;
  spBar.style.transform = 'scaleX(' + (window.scrollY / h) + ')';
});
```

**Reveal 애니메이션 (IntersectionObserver)**
- `.reveal` 클래스에 `threshold: 0.2` 관찰
- 진입 시 `.visible` 추가 → CSS transition 발동
- `.rv1`~`.rv5` 지연 클래스 (100ms 간격)

**카운터 애니메이션**
- `data-count` 속성의 목표값으로 0부터 카운트업
- 2초 동안 easing: `1 - (1-t)^3`
- 소수점 지원: `parseFloat` + `toFixed(1)`

**3D 카드 틸트**
```javascript
card.addEventListener('mousemove', e => {
  const x = (e.clientX - r.left) / r.width - 0.5;
  const y = (e.clientY - r.top) / r.height - 0.5;
  card.style.transform = `perspective(600px) rotateX(${y*20}deg) rotateY(${x*-20}deg)`;
});
```

**FAQ 아코디언**
- `.open` 토글 → `max-height: 0` ↔ `scrollHeight`

**Nav Dots**
- 섹션별 IntersectionObserver → active dot 표시

### 3.5 시각적 다이어그램 (4종)

| 다이어그램 | 위치 | 기술 |
|-----------|------|------|
| 벤 다이어그램 | research | CSS absolute + border-radius 50% |
| 스텝 플로우 | curriculum | Flexbox + 연결선 |
| 콤보 테이블 | ai (복수전공) | CSS Grid + 뱃지 |
| 다이아몬드 허브 | only-cha | SVG line + absolute positioning |

### 3.6 PIP 아바타 컨테이너

```html
<div id="heygen-pip-container">
  <iframe id="heygen-pip"
    src="https://interactiveavatarnextjsdemo.netlify.app"
    allow="camera; microphone; autoplay; clipboard-write">
  </iframe>
  <!-- 드래그 핸들 (.pdb) -->
  <!-- 최소화/닫기 버튼 -->
</div>
```

**기능:**
- 드래그 이동 (마우스/터치)
- 최소화 (120x68px)
- 닫기: `CLOSE_AVATAR` postMessage → 아바타 세션 완전 종료
- 다시 열기 토글
- 반응형: 모바일에서 280x158px

---

## 4. 인증 시스템

### 4.1 카카오 로그인 플로우

```
사용자 → 동의 체크 → 카카오 로그인 → Kakao SDK
→ 액세스 토큰 → Kakao API (/v2/user/me)
→ kakao_id, nickname, email 획득
→ POST /business-api/api.php (action: kakao_login)
→ JWT 토큰 + user 객체 반환
→ localStorage 저장 (business_token, business_user)
→ UI 업데이트 + 아바타에 USER_INFO 전송
```

### 4.2 세션 관리

**LocalStorage 키:**
```
business_token   — JWT 토큰
business_user    — 유저 객체 JSON
business_session — 세션 ID (sess_timestamp_random)
```

**세션 복원 (페이지 로드 시):**
1. localStorage에서 토큰/유저 읽기
2. `?action=verify&token=...`로 유효성 검증
3. 유효 → UI 복원 + 추적 시작 + 아바타 전송 (6초 후)
4. 무효 → 세션 삭제 + 로그인 모달 표시 (1초 후)

### 4.3 아바타 통신 (USER_INFO)

**AudioContext 정책 대응:**
```javascript
// 사용자 첫 클릭/탭/키입력 감지
document.addEventListener('click', onFirstInteraction, true);
document.addEventListener('touchstart', onFirstInteraction, true);
document.addEventListener('keydown', onFirstInteraction, true);

function sendUserInfoToAvatar(user, token) {
  if (!_userHasInteracted) {
    _pendingSendArgs = { user, token };  // 대기
    return;
  }
  doSendUserInfo(user, token);  // 즉시 전송
}
```

**중복 방지:**
- `_userInfoSent` 플래그 — 한 번만 전송
- 아바타 앱에서도 `userInfoRef.current` 체크로 중복 처리 방지
- 타이머 폴백: 5초 → 10초 → 18초 재시도 (AVATAR_READY 대기)

---

## 5. 아바타 시스템

### 5.1 파일 구조
```
InteractiveAvatarNextJSDemo/
├── app/api/
│   ├── chat/route.ts              ← 대화 + 섹션 라우팅
│   └── get-access-token/route.ts  ← HeyGen 토큰 발급
├── app/lib/
│   ├── webSpeechAPI.ts            ← Web Speech 래퍼
│   └── constants.ts               ← 아바타 목록
├── components/
│   └── InteractiveAvatar.tsx      ← 메인 컴포넌트
└── .env                           ← API 키
```

### 5.2 아바타 설정

```typescript
const AVATAR_CONFIG = {
  quality: AvatarQuality.Low,
  avatarName: "e2eb35c947644f09820aa3a4f9c15488",  // 교수님 아바타
  voice: { voiceId: "", rate: 1.0, emotion: VoiceEmotion.FRIENDLY },
  language: "ko",
};
```

### 5.3 대화 흐름

```
[음성 입력] → Web Speech API (ko-KR) → 텍스트 변환
→ POST /api/chat { message, history }
→ GPT-4o-mini (SYSTEM_PROMPT + 대화 이력)
→ JSON { reply, action, tabId }
→ action=navigate? → NAVIGATE_TAB postMessage → 랜딩페이지 스크롤
→ speakWithAvatar(reply) → HeyGen TTS 발화
→ 발화 중 Web Speech 일시정지 (자기 소리 인식 방지)
→ 발화 종료 500ms 후 Web Speech 재개
```

### 5.4 섹션 기반 네비게이션

**route.ts 섹션 매핑:**
```typescript
const SECTION_INFO = {
  research:   { name: "전공 소개 & 교육목표", keywords: ["연구", "바이오", "헬스케어", "교육목표", "인재", "융합", ...] },
  curriculum: { name: "커리큘럼", keywords: ["수업", "과목", "학년", "캡스톤", "자격증", ...] },
  ai:         { name: "차별성 & 복수전공", keywords: ["복수전공", "교수", "액션러닝", "졸업인증", ...] },
  careers:    { name: "취업 & 진로", keywords: ["취업", "진로", "경영기획", "마케팅", "회계", ...] },
  "only-cha": { name: "차의과학대 강점", keywords: ["차대", "차병원", "산학연병", "RISE", ...] },
  experience: { name: "실전 경험", keywords: ["팀플", "해커톤", "창업", "경진대회", ...] },
  faq:        { name: "FAQ", keywords: ["수학", "수포자", "걱정", ...] },
};
```

**랜딩페이지 수신 (구 tab ID 호환):**
```javascript
var _tabToSection = {
  tab1:'research', tab2:'curriculum', tab3:'careers', tab4:'ai',
  tab5:'ai', tab6:'experience', tab7:'research', tab8:'only-cha',
  tab9:'careers', tab10:'careers', tab11:'only-cha', tab12:'ai',
  tab13:'faq', tab14:'faq'
};
```

### 5.5 PostMessage 프로토콜

| 타입 | 방향 | 페이로드 | 동작 |
|------|------|---------|------|
| USER_INFO | 랜딩→아바타 | `{user, token, history}` | 개인화 인사말 |
| START_AVATAR | 랜딩→아바타 | - | 세션 시작 |
| CLOSE_AVATAR | 랜딩→아바타 | - | 세션 완전 종료 (X 버튼) |
| AVATAR_READY | 아바타→랜딩 | - | 준비 완료 신호 |
| NAVIGATE_TAB | 아바타→랜딩 | `{tabId}` | 섹션 스크롤 |
| ASK_QUESTION | 랜딩→아바타 | `{question}` | 질문 전송 |

### 5.6 음성 인식 오류 대응 (SYSTEM_PROMPT)

```
"조수진" / "고수진" → 교수진
"취엄" → 취업
"복수정공" / "복스전공" → 복수전공
"해커쏜" / "핵커톤" → 해커톤
"캡스턴" / "켑스톤" → 캡스톤
"커리큘렘" → 커리큘럼
```
→ 문맥 추론으로 가장 가까운 경영학 관련 주제로 해석

### 5.7 TTS 발음 규칙

**프롬프트 규칙 + 후처리 강제 치환 (이중 대응)**

GPT 응답 반환 전 route.ts에서 강제 치환 수행 (GPT가 규칙 미준수 시에도 대응):

```
// 학교/학부명
"차의과학대학교" → "차 의과학 대학교" (3단어 분리)
"헬스케어융합학부" → "헬스케어 융합 학부"
"미래융합대학" → "미래 융합 대학"
"경영학전공" → "경영학 전공"

// 과목/분야명 (긴 합성어 분리)
"기술경영" → "기술 경영", "조직행동론" → "조직 행동론"
"경영학원론" → "경영학 원론", "캡스톤디자인" → "캡스톤 디자인"
"투자자산운용" → "투자 자산 운용", "기업지배구조" → "기업 지배 구조"

// 영어 약어 → 한글 발음
AI → "에이아이", R&D → "알앤디", ESG → "이에스지"
IT → "아이티", IR → "아이알", PR → "피알", CR → "씨알"
CRM → "씨알엠", CDO → "씨디오", VC → "브이씨"
ADsP → "에이디에스피", SQLD → "에스큐엘디", RISE → "라이즈"

// 기타
88% → "88 퍼센트"
"차병원" → "차 병원", "차병원그룹" → "차 병원 그룹"
```

### 5.8 Allowed Origins

```typescript
const allowedOrigins = [
  "https://sdkparkforbi.github.io",
  "https://sungbongju.github.io",
  "http://localhost",
  "http://127.0.0.1",
];
```

---

## 6. API 명세

### 6.1 백엔드 API (PHP)

**Base URL:** `https://aiforalab.com/business-api/api.php`

| Action | Method | 요청 | 응답 |
|--------|--------|------|------|
| kakao_login | POST | `{kakao_id, nickname, email}` | `{success, token, user}` |
| verify | GET | `?token=...` | `{success, valid}` |
| user_history | GET | `?user_id=...` | `{visit_count, recent_topics, ...}` |
| log_batch | POST | `{token, events[]}` | `{success}` |
| survey_submit | POST | `{satisfaction, reason, grade, gender, ...}` | `{success, id}` |
| save_chat | POST | `{message, response, session_id}` | `{success}` |

### 6.2 아바타 API (Next.js)

| 엔드포인트 | Method | 요청 | 응답 |
|-----------|--------|------|------|
| /api/chat | POST | `{message, history}` | `{reply, action, tabId}` |
| /api/chat | POST | `{type:"tab_explain", tabId}` | `{reply}` (고정 스크립트) |
| /api/get-access-token | POST | (없음) | HeyGen 토큰 문자열 |

### 6.3 환경 변수

```env
# 아바타 앱 (.env)
HEYGEN_API_KEY=<비공개>
OPENAI_API_KEY=<비공개>
NEXT_PUBLIC_BASE_API_URL=https://api.heygen.com
```

---

## 7. 행동 추적 시스템

### 7.1 추적 이벤트

| 이벤트 | 트리거 | 메타데이터 |
|--------|--------|-----------|
| section_view | 섹션 2초 이상 체류 후 이탈 | `{duration_seconds}` |
| scroll_depth | 10% 단위 스크롤 | `{depth_percent}` |
| cta_click | CTA 버튼 클릭 | `{button_text}` |
| tab_click | 탭/버튼 클릭 | `{button_text}` |
| page_total | 페이지 이탈 시 | `{total_seconds}` |

### 7.2 전송 방식

- 5개 이벤트마다 `fetch` POST 배치 전송
- 페이지 이탈 시 `navigator.sendBeacon` 사용 (전달 보장)
- 로그 형식: `{event_type, section_id, session_id, metadata, timestamp}`

---

## 8. 설문조사 시스템

### 8.1 설문 항목

- **만족도** (satisfaction): 1~5 Likert (매우 불만족~매우 만족)
- **방문 이유** (reason): 복수 선택 칩
- **학년** (grade): 1~5 Likert
- **성별** (gender): 선택
- **MBTI** (선택): 텍스트 입력
- **자유 의견** (feedback): 텍스트 입력

### 8.2 제출 플로우

```javascript
function submitSurvey() {
  // 필수 항목 검증 (만족도, 이유, 학년, 성별)
  // POST → api.php?action=survey_submit
  // 성공 → localStorage('survey_submitted') 저장
  // 중복 제출 방지
}
```

---

## 9. 트러블슈팅

### 9.1 아바타가 안 뜨는 경우

| 원인 | 해결 |
|------|------|
| Netlify cold start | iframe error 이벤트 시 자동 재시도 (최대 2회) |
| HeyGen API 토큰 만료 | 페이지 새로고침 |
| 동시 접속 제한 초과 | 자동 큐잉 (아래 11장 참조) |

### 9.2 인삿말 중복/안 나옴

| 원인 | 해결 |
|------|------|
| USER_INFO 중복 전송 | `_userInfoSent` 플래그 (auth.js) |
| 아바타 앱에서 중복 수신 | `userInfoRef.current` 체크 (InteractiveAvatar.tsx) |
| AudioContext 차단 | 사용자 첫 클릭/탭 후 전송 |
| AVATAR_READY 미수신 | 5/10/18초 폴백 타이머 |

### 9.3 스크롤 관련

| 원인 | 해결 |
|------|------|
| 모바일 스크롤 멈춤 | PIP 드래그 touchmove를 dragging 중에만 preventDefault |
| 답변 후 섹션 스크롤 안 됨 | 구 tab1~14 → 새 섹션 ID 매핑 테이블 추가 |

### 9.4 음성 인식 오류

| 원인 | 해결 |
|------|------|
| Web Speech API 오인식 | SYSTEM_PROMPT에 유사 발음 매핑 규칙 추가 |
| 전문 용어 인식 실패 | GPT에게 문맥 추론 지시 ("절대 알 수 없습니다로 답하지 마세요") |
| 근본 해결 | Azure Speech (5시간/월 무료) 또는 Whisper API로 교체 검토 |

### 9.5 카카오 로그인

| 원인 | 해결 |
|------|------|
| 카카오톡 인앱 브라우저 | Chrome으로 자동 리다이렉트 (Intent/scheme) |
| 토큰 URL 인코딩 | JWT의 `=` 문자 → `encodeURIComponent()` 사용 |
| 팝업 콜백 실패 | 1초 간격 토큰 폴링 (90초 타임아웃) |

---

## 10. 배포 구조

### 10.1 GitHub 레포

| 레포 | URL | 배포 |
|------|-----|------|
| cha-biz-ai-v2 | github.com/sungbongju/cha-biz-ai-v2 | GitHub Pages (main) |
| cha-biz-ai-v3 | github.com/sungbongju/cha-biz-ai-v3 | GitHub Pages (main) |
| InteractiveAvatarNextJSDemo | github.com/sungbongju/InteractiveAvatarNextJSDemo | Netlify 자동 배포 |

### 10.2 서버

| 서비스 | 주소 | 비고 |
|--------|------|------|
| 백엔드 API | aiforalab.com/business-api/api.php | PHP 5.4.45 |
| SSH | 106.247.236.2:10022 (user2/user2!!) | sudo 필요 |
| DB | business_db | MySQL |

### 10.3 CORS 허용 출처

```
sungbongju.github.io
sdkparkforbi.github.io
aiforalab.com
interactiveavatarnextjsdemo.netlify.app
localhost / 127.0.0.1
```

---

## 11. HeyGen 동시 접속 제한

### 11.1 공식 답변 (HeyGen 지원팀, 2026-03-04)

> "Your app won't crash! The API has a built-in queuing system.
> If you exceed your plan's concurrency limit, additional videos are
> automatically queued and processed as capacity becomes available."

### 11.2 플랜별 동시 세션

| 플랜 | 동시 세션 | 비고 |
|------|----------|------|
| Trial | 1개 | 무료 |
| API 기본 | 3~6개 | 유료 |
| Enterprise | 최대 20개 | 협의 |

### 11.3 40명 동시 접속 시나리오

- 3~6명: 즉시 아바타 연결
- 나머지 34~37명: **자동 큐 대기** (크래시 없음)
- 대기자는 아바타가 로딩 중으로 표시됨
- 앞선 세션 종료 시 순서대로 연결

### 11.4 권장 대응

1. **설문 우선 진행**: 아바타 대기 중에도 설문 작성 가능하도록 구성
2. **시차 접속 유도**: 조별로 5~10명씩 시간차 접속
3. **플랜 확인**: 현재 API 플랜의 동시 세션 수 확인 필요

---

## 12. 교수진 정보

| 이름 | 전공 분야 | 비고 |
|------|----------|------|
| 김주헌 | 국제경영학 | 학장 |
| 김억환 | 인사조직, 경영전략 | |
| 김용환 | 기술경제, 국제경제, 연구윤리 | |
| 김태동 | 회계세무 | |
| 박대근 | 재무금융 | 담당 교수 |
| 이희정 | 마케팅, 관광심리, 서비스 | |
| 김종석 | 기술경영(혁신관리·미래예측), 전략 | |

---

## 13. OG 메타태그 (SNS 공유 미리보기)

```html
<meta property="og:title" content="경영학전공 — 차의과학대학교"/>
<meta property="og:description" content="AI × 바이오헬스케어 × 비즈니스 융합형 인재를 양성합니다."/>
<meta property="og:image" content="https://sungbongju.github.io/cha-biz-ai-v3/img/og-thumbnail.jpg"/>
<meta property="og:url" content="https://sungbongju.github.io/cha-biz-ai-v3/"/>
```

- 이미지: lc.cha.ac.kr 경영학전공 배너 이미지 사용
- 카카오톡/SNS 공유 시 제목+설명+썸네일 미리보기 표시

---

## 14. 마이크 정책

- **자동 시작**: 아바타 세션 시작 시 Web Speech API 자동 초기화 + 시작
- **권한 거부 시**: alert 없이 조용히 실패 (console.error만 출력)
- **마이크 없는 환경**: 텍스트 입력으로 대화 가능
- **상태 텍스트**: 마이크 켜짐 → "듣는 중... 말씀하세요", 꺼짐 → "텍스트로 질문하세요"
- **마이크 토글**: 버튼 클릭으로 on/off 가능

---

## 15. Google Forms 설문 (백업용)

40명 동시 접속 데모 시 랜딩페이지 설문과 동일한 Google Forms를 병행 운영.

| 문항 | 유형 | 설정 |
|------|------|------|
| AI 아바타 상담 만족도 | 선형 배율 1~5 | 매우 불만족 ~ 매우 만족 |
| 만족/불만족 이유 | 객관식 (단일) | 6개 선택지 |
| 학년 | 객관식 | 1~4학년, 기타 |
| 성별 | 객관식 | 남성, 여성 |
| MBTI | 단답형 | 직접 입력 |

> 응답 데이터는 Google Sheets로 수집 후, 서버 DB에 일괄 INSERT하여 랜딩페이지 설문 결과와 통합 가능.

---

## 16. 변경 이력

| 날짜 | 내용 |
|------|------|
| 03-04 | 최초 작성 |
| 03-06 | 교수님 피드백 2차 반영: 회계·세무·금융→회계재무, goals+research 섹션 병합, experience 3칸 그리드, OG 메타태그 추가, 아바타 X→세션 종료, TTS 발음 후처리 대폭 확장, 마이크 자동시작+조용한 실패 처리, Google Forms 설문 병행 |
