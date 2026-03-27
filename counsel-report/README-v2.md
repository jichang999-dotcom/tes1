# AI 부정감성 분석 리포트 자동화 시스템

**최종 업데이트: 2026-03-27**
**분류 체계: 3안 B안 (Admin → Strong → Weak → 미분류)**

---

## 전체 구조

```
[일일 배치] run_report.bat (진입점, 매일 19:00 Windows 작업 스케줄러)
  └→ Step 1~9 순차 실행

[월요일 보강 배치] 매주 월요일 추가 실행
  └→ 지난주 실패/미전송/후순위 밀린 건 재처리 + 키워드 보강 검토
```

---

## Step별 상세

| Step | 파일명 | 역할 | 입력 | 출력 | 중요도 |
|------|--------|------|------|------|--------|
| 1 | fetch_data.js | MariaDB에서 최근 30일 상담 데이터 추출 | MariaDB (db_square) | data/counsel_data.json | 필수 |
| 2 | analyze_sentiment.js | **4단계 분류**: Admin 제외 → Strong 즉시집계 → Weak+미분류 AI 후보 생성 | counsel_data.json, keywords.json, config/filter_rules.json | data/sentiment_results.json, data/ai_candidates.jsonl | 필수 |
| 3 | claude_enhance.js | **Weak+미분류** 대상 Claude AI 문맥분석 (오탐 필터링 + 부정 추가 감지) | ai_candidates.jsonl, sentiment_results.json, cache/ai_review_cache.json | sentiment_results.json (보강) + suggested_keywords | 선택 |
| 4 | update_keywords.js | Step 3의 제안 키워드를 keywords.json에 반영 → 다음날 Step 2 강화 | sentiment_results.json | keywords.json (갱신) | 선택 |
| 5 | build_html.js | 센터별 HTML 리포트 생성 + NEW 뱃지 판별 | sentiment_results.json, sentiment_previous.json | reports/\<cmp_cd\>.html | 필수 |
| 6 | generate_and_upload.js | GitHub Pages에 리포트 업로드 | reports/*.html | report_links.csv | 필수 |
| 7 | update_sheets.js | Google Sheets에 리포트 링크 삽입 | report_links.csv | Google Sheets 셀 갱신 | 선택 |
| 8 | update_raw_sheet.js | 상담 원데이터 + 키워드 매칭 결과를 Sheets에 누적 | counsel_data.json, keywords.json | Google Sheets "키워드raw" 시트 | 선택 |
| 9 | trigger_webhook.js | n8n 웹훅으로 완료 알림 전송 | 없음 | HTTP POST → n8n | 선택 |

---

## 데이터 흐름도

```
MariaDB (db_square, 최근 30일 상담)
│
▼
[Step 1] fetch_data.js
│
▼
counsel_data.json
│
▼
[Step 2] analyze_sentiment.js ◄──── keywords.json (Admin / Strong / Weak 3단계)
│                                         ▲
│  ┌─────────────────────────────┐        │
│  │ 4단계 판정 순서:            │        │
│  │ 1. Admin 제외 (행정/운영)   │   ★피드백 루프★
│  │ 2. Strong → 즉시 부정 집계  │   (키워드 보강으로 Weak 감지율 향상)
│  │ 3. Weak → AI 후보           │        │
│  │ 4. 미분류 → AI 후보         │        │
│  └─────────────────────────────┘        │
│                                         │
├─ sentiment_results.json (Strong만 반영) │
│                                         │
├─ ai_candidates.jsonl (Weak + 미분류)    │
│                                         │
▼                                         │
[Step 3] claude_enhance.js ───────────────┘
  (Weak: 문맥 검증으로 오탐 필터링)
  (미분류: 새로운 부정 감지)
  → suggested_keywords 수집
│
▼
sentiment_results.json (보강됨: Strong + AI 검증 Weak + AI 신규 감지)
│
▼
[Step 4] update_keywords.js
  (제안 키워드 → 'AI_보강_키워드' 카테고리에 추가)
│
▼
[Step 5] build_html.js + sentiment_previous.json (전일 비교 → NEW 뱃지)
│
▼
reports/<cmp_cd>.html
│
├─▶ [Step 6] generate_and_upload.js → GitHub Pages + report_links.csv
│     │
│     ├─▶ [Step 7] update_sheets.js → Google Sheets 링크 갱신
│     │
│     └─▶ [Step 8] update_raw_sheet.js → Google Sheets 원데이터 누적
│
└─▶ [Step 9] trigger_webhook.js → n8n 완료 알림
```

---

## 핵심: 3안 B안 분류 체계

### 판정 순서 (analyze_sentiment.js)

```
전체 데이터 (11,493건 기준)
│
├─ [1] Admin 제외: 1,701건 (14.8%)
│    행정/운영성 기록 (부재, 통화중, 설명회, 온라인진단 등)
│    ※ 제목이 Admin이어도 본문에 Strong 키워드 있으면 rescue
│
├─ [2] Strong 즉시 집계: 848건 (7.4%)
│    확정적 부정 (환불, 퇴소, 컴플레인, 자해, 이혼 등)
│    → AI 불필요, 즉시 학생별 부정 건수 반영
│
├─ [3] Weak → AI 후보: 2,896건 (25.2%)
│    문맥 검증 필요 (힘들, 어렵, 속상, 결석, 빠지 등)
│    → AI가 문맥 분석하여 오탐 필터링
│
└─ [4] 미분류 → AI 후보: 6,048건 (52.6%)
     키워드 미매칭
     → AI가 새로운 부정 감지
```

### 왜 B안인가?

| 비교 항목 | A안 (Weak 즉시집계) | **B안 (Weak→AI 검증)** | C안 (절충) |
|-----------|--------------------|-----------------------|-----------|
| Weak 처리 | 즉시 집계 (오탐 포함) | **AI 문맥 검증 후 집계** | 즉시 집계 + 월1회 샘플 |
| AI 일평균 | ~141건 | **~228건** | ~141건 |
| 퀄리티 | 낮음 | **높음** | 중간 |
| 오탐 대응 | 없음 | **실시간** | 월1회 |

**B안 선택 이유**: AI 처리량이 감당 가능한 수준(일평균 228건)이므로 퀄리티 우선

---

## 운영 모드

### 일일 배치 (매일 19:00)

```
대상: D-0~D-1 데이터
│
├─ Admin 제외
├─ Strong → 즉시 학생별 부정 건수 반영
├─ Weak + 미분류 → AI 전송
│    ├─ Weak: 문맥 검증 (오탐이면 제외, 부정이면 집계)
│    └─ 미분류: 새로운 부정 감지
└─ 학생별(std_cd) 최종 부정 건수 집계
```

### 월요일 보강 배치

```
대상: 지난주 일일 배치의 미처리 건
│
├─ 실패 건 (FAILED)
├─ 미전송 건 (캐시 미적중 신규)
├─ 후순위 밀린 건 (Rate limit으로 중단된 건)
│
├─ AI 문맥분석
├─ 결과 반영
└─ suggested_keywords 기반 키워드 보강 검토/자동 반영
```

**월요일 맥시멈**: ~1,362건 (주간 전체 장애 시), 정상 운영 시 100~300건

---

## Step 2↔3↔4 피드백 루프

```
Day 1:
    Step 2: Admin/Strong/Weak 분류 → Weak+미분류 300건 AI 전송
    Step 3: Claude가 300건 문맥분석 → Weak 오탐 50건 제외 + 미분류 30건 추가 감지
            + 키워드 15개 제안
    Step 4: 15개 중 신규 10개 → keywords.json 'AI_보강_키워드'에 추가

Day 2:
    Step 2: 보강된 키워드로 Weak 감지 증가 → 미분류 감소 → AI 전송 250건
    Step 3: 캐시 히트로 기처리분 스킵 + 신규만 분석
    Step 4: 추가 키워드 누적

Day N:
    Step 2: 충분히 보강된 키워드로 대부분 Weak로 감지 → 미분류 극소
    Step 3: Weak 오탐 필터링 + 소수 미분류만 처리 → 처리 시간 대폭 단축
```

**목적**: AI 처리량을 점진적으로 줄이면서 분석 퀄리티는 유지하는 자가 학습 구조

---

## 키워드 사전 (keywords.json) — 3단계 구조

### Admin 키워드 (22개) — 제외 대상

운영/행정성 기록. 제목에 매칭 + 본문 10자 미만일 때 제외.

| 키워드 |
|--------|
| 부재, 통화중, 설명회, 온라인진단, 진단예약, 진단안내, 학급보강, 개인보강, 시간변경, 일정조율, 참석, 문자 안내, 연락 요청, 테스트, 삭제요청, 상담이관, 자동등록, 시스템, 데이터정리, 일괄등록, 일괄처리, 진단취소 |

### Strong 키워드 (4개 카테고리, 42개) — 즉시 부정 집계

| # | 카테고리 | 키워드 수 | 대표 키워드 |
|---|---------|----------|------------|
| 1 | 이탈/포기 의사 | 17 | 환불, 퇴소, 자퇴, 수업거부, 등원거부, 그만두, 휴회 |
| 2 | 불만/분노 | 10 | 컴플레인, 항의, 폭언, 욕설, 교사교체, 강사변경 |
| 3 | 위험행동 | 11 | 자해, 자살, 죽고싶, 도벽, 훔쳤, 가출 |
| 4 | 가정문제 | 4 | 이혼, 별거, 가정폭력, 가정불화 |

### Weak 키워드 (11개 카테고리, 180+개) — AI 문맥 검증 대상

| # | 카테고리 | 키워드 수 | 대표 키워드 |
|---|---------|----------|------------|
| 1 | 스트레스/압박감 | 14 | 스트레스, 부담, 힘들, 지쳐, 예민 |
| 2 | 우울/무기력 | 20 | 우울, 무기력, 위축, 자존감, 울음 |
| 3 | 불만/분노 | 17 | 불만, 짜증, 속상, 답답, 격앙 |
| 4 | 불안/걱정 | 14 | 불안, 걱정, 초조, 긴장, 눈물 |
| 5 | 대인관계 갈등 | 19 | 갈등, 따돌림, 왕따, 괴롭, 충돌 |
| 6 | 이탈/포기 의사 | 18 | 포기, 결석, 거부, 불참, 휴원 |
| 7 | 학습 어려움 | 23 | 어렵, 못하, 이해안, 집중력, 부진 |
| 8 | 부적응 | 11 | 적응못, 부적응, 안맞, 불편, 견디기 |
| 9 | 사춘기/반항 | 9 | 사춘기, 반항, 싫다고, 하기싫 |
| 10 | 건강문제 | 7 | 아파서, 두통, 복통, 통증 |
| 11 | AI_보강_키워드 | 45 | 오기싫어, 자신감이없, 숙제를안, 수업방해 등 |

---

## Step 3 (claude_enhance.js) 상세

### 처리 전략
- 대상: **Weak + 미분류** (ai_candidates.jsonl에 classification 필드 포함)
- Weak: 키워드 매칭됐지만 문맥상 오탐 여부를 AI가 검증
- 미분류: 키워드 미매칭 건에서 새로운 부정 감지
- 센터(cmp_cd)별로 묶어서 호출 (같은 센터 맥락 유지 → 품질 향상)
- 500건 초과 센터만 500건씩 분할 (MAX_PER_CALL)
- Claude CLI를 stdin 기반으로 호출 (`execSync`의 `input` 옵션)

### AI 프롬프트 핵심 지시

```
Weak 키워드가 포함되어 있어도 문맥상 긍정/중립이면 부정으로 판정하지 마세요.
예) "어려운 문제도 잘 풀었습니다" → 긍정 (부정 아님)
예) "힘들었지만 잘 해냈습니다" → 긍정 (부정 아님)
예) "결석 연락 드렸습니다" → 행정 기록 (부정 아님)
```

### 캐시 구조 (cache/ai_review_cache.json)
```json
{
  "CN002082650": {
    "text_hash": "be79251a9f32...",
    "analyzed_at": "2026-03-26T06:22:12.961Z",
    "categories": ["학습 어려움", "스트레스/압박감"]
  }
}
```
- **key** (`CN002082650`): `cns_cd` (source_pk) — 어떤 상담 건인지 식별
- **text_hash**: 상담 내용(제목+본문)을 해시 함수로 변환한 고정 길이 지문(fingerprint). 상담 원문은 캐시에 저장하지 않음 (개인정보 보호 + 용량 절감)
  - 원문이 같으면 → 항상 같은 hash / 한 글자라도 바뀌면 → 완전히 다른 hash
- **categories**: 이전 AI 부정 분석 결과 저장 → 다음 실행 시 재반영
- **TTL**: 60일 보관, 만료 시 자동 제거

### 캐시 판단 흐름
```
매일 Step 3 실행 시:
  DB에서 CN002082650의 상담 내용을 가져옴
  → hash 계산 → "be79251a..."
  → 캐시에서 CN002082650 조회 → text_hash 일치?

  ✅ 일치 (캐시 히트)
    → 내용 안 바뀜 → AI 재호출 불필요
    → 캐시의 categories를 sentiment에 자동 반영

  ❌ 불일치 (캐시 미스)
    → 내용이 수정됨 → 당일 Step 3에서 AI 즉시 재분석
    → 새 결과로 캐시 갱신

  ⚠️ AI 재분석 실패 (Rate limit 등)
    → FAILED 처리 → 월요일 보강 배치에서 재처리
```

### 보호 장치

| 장치 | 설명 |
|------|------|
| 캐시 중복 방지 | source_pk + text_hash 기반 (60일 보관) |
| 이전 결과 재반영 | 캐시에 categories 저장 → 재분석 없이 복원 |
| Rate limit 감지 | 한도 도달 시 즉시 중단, 다음 실행에서 이어서 처리 |
| 2회 재시도 | 5초 간격, 일시적 오류 대응 |
| Lock 파일 | PID 기반 중복 실행 방지 (60분 타임아웃) |
| 상태 관리 | PENDING → PROCESSING → DONE/FAILED |

---

## 필터 조건

### Admin 판정 (analyze_sentiment.js)

| 조건 | 설명 |
|------|------|
| **Admin 제목 매칭** | 제목(cns_st)에 Admin 키워드 포함 |
| **본문 짧음** | 내용(cns_req_ct)이 10자 미만 |
| **Rescue** | Admin 판정이어도 본문에 Strong 키워드 있으면 Strong으로 복원 |

### AI 후보 필터 (passesPreFilter)

| 조건 | 설명 |
|------|------|
| **빈 텍스트** | 제목+내용 모두 비어있으면 제외 |
| **최소 길이 미달** | 제목+내용 합쳐서 10자 미만이면 제외 |

---

## 2단계 처리 흐름 (실측 데이터 기준)

```
전체 11,493건
  ├─ Admin 제외:      1,701건 (14.8%) → 행정/운영 기록
  ├─ Strong 즉시집계:   848건 (7.4%)  → AI 불필요, 확정 부정
  ├─ Weak (AI후보):   2,896건 (25.2%) → AI 문맥 검증
  └─ 미분류 (AI후보): 6,048건 (52.6%) → AI 부정 탐지
       ├─ 필터 제외:  일부 → 빈 텍스트/짧은 텍스트
       └─ AI 후보(최종): Weak+미분류 중 필터 통과 건
```

### D-0 vs D-0~D-1 AI처리량 (3/17~3/26 실측)

| 운영일 | D-0 AI대상 | D-0~D-1 AI대상 |
|--------|-----------|---------------|
| 03-17(월) | 287 | 296 |
| 03-18(화) | 224 | 511 |
| 03-19(수) | 259 | 483 |
| 03-20(목) | 253 | 512 |
| 03-21(금) | 315 | 568 |
| 03-22(토) | 2 | 317 |
| 03-23(일) | 22 | 24 |
| 03-24(월) | 292 | 314 |
| 03-25(화) | 325 | 617 |
| 03-26(수) | 300 | 625 |
| **일평균** | **228건** | **427건** |

---

## 주요 파일 목록

### 실행 파일

| 파일 | 설명 |
|------|------|
| run_report.bat | 마스터 배치 스크립트 (진입점, 19:00 스케줄러) |
| run_daily.js | Node.js 대체 오케스트레이터 (레거시) |

### 데이터 파일

| 파일 | 설명 |
|------|------|
| data/counsel_data.json | DB에서 추출한 상담 원데이터 (Step 1 출력) |
| data/sentiment_results.json | 감성 분석 결과 (Step 2 Strong + Step 3 AI 보강) |
| data/sentiment_previous.json | 전일 분석 결과 (NEW 뱃지 비교용, 하루 1회만 갱신) |
| data/ai_candidates.jsonl | AI 분석 후보 (Step 2 출력 — Weak + 미분류) |
| data/ai_candidates.meta.json | AI 후보 통계 메타데이터 (admin/strong/weak/unclassified 건수) |
| keywords.json | 부정 감성 키워드 사전 (**3단계: Admin / Strong / Weak**) |
| report_links.csv | GitHub 업로드 결과 (센터별 URL) |
| cache/ai_review_cache.json | AI 분석 캐시 (source_pk → text_hash + categories, 60일 보관) |

### 설정 파일

| 파일 | 설명 |
|------|------|
| .env | DB/GitHub/Google 인증 정보 |
| credentials.json | Google Sheets 서비스 계정 키 |
| config/filter_rules.json | Admin rescue 설정, 최소 텍스트 길이 |
| package.json | Node.js 의존성 (mysql2, googleapis, dotenv, @anthropic-ai/sdk) |

### 출력

| 파일 | 설명 |
|------|------|
| reports/\<cmp_cd\>.html | 센터별 HTML 리포트 (13개) |
| logs/run_YYYY-MM-DD.log | 일자별 실행 로그 |

---

## 분석 대상 센터 (13개)

| 코드 | 센터명 | 최근 상담건수 |
|------|--------|-------------|
| AAA | 와이즈만 서초센터 | 2,393 |
| AAM | 와이즈만 분당센터 [초등관] | 1,044 |
| ACL | 와이즈만 강서센터 | 144 |
| ACQ | 와이즈만 평촌센터 [초등관] | 1,396 |
| ACS | 와이즈만 해운대센텀센터 | 1,506 |
| ACV | 와이즈만 송파센터 [초등관] | 1,495 |
| ADO | 와이즈만 압구정센터 | 678 |
| AFF | 분당 영재입시센터 [중등관] | 322 |
| AFN | 와이즈만 대치센터 | 917 |
| AIP | 와이즈만 일산후곡센터 | 78 |
| AIU | 와이즈만 수원영통센터 | 589 |
| AJR | 평촌 영재입시센터 [중등관] | 381 |
| ALB | 송파 영재입시센터 [중등관] | 284 |

---

## 에러 처리 전략

- **Step 1, 2, 5, 6** 실패 시 → **배치 중단** (핵심 리포트 생성/배포)
- **Step 3, 4, 7, 8, 9** 실패 시 → **경고 후 계속** (보조 기능)

Step 3이 실패해도 Step 2의 Strong 키워드 분석 결과만으로 리포트가 정상 생성됩니다.

### Step 3 Rate Limit 대응
- Rate limit 감지 시 남은 호출 즉시 중단 (불필요한 재시도 방지)
- 성공한 건은 캐시에 결과(categories) 포함 저장
- 다음 실행 시 캐시 히트된 건은 AI 호출 없이 결과 재반영
- 미처리 건만 신규 호출 → 여러 날에 걸쳐 점진적 완료
- **월요일 보강 배치에서 주간 미처리 건 일괄 재처리**

---

## 예상 소요 시간

| Step | 소요 시간 | 비고 |
|------|----------|------|
| 1 (DB 추출) | 1~3분 | |
| 2 (4단계 분류) | 1~2분 | Admin/Strong/Weak/미분류 판정 |
| 3 (Claude AI) | 10~25분 → 점차 단축 | Weak+미분류 대상 문맥분석 + 캐시 효과 |
| 4 (키워드 갱신) | <1분 | |
| 5 (HTML 생성) | 1~2분 | |
| 6 (GitHub 업로드) | 2~5분 | |
| 7 (Sheets 링크) | 1~2분 | |
| 8 (Raw 데이터) | 2~5분 | |
| 9 (웹훅 알림) | <1분 | |
| **합계** | **20~45분 → 점차 단축** | |

---

## 외부 연동

| 서비스 | 용도 | 접속 정보 |
|--------|------|----------|
| MariaDB | 상담 데이터 원본 | db_square (61.74.65.40:3306) |
| Claude CLI | AI 문맥 감성 분석 | claude -p (stdin 기반) |
| GitHub Pages | HTML 리포트 배포 | jichang999-dotcom.github.io/tes1/counsel-report/ |
| Google Sheets | 리포트 링크 + 원데이터 | 서비스 계정 (credentials.json) |
| n8n Webhook | 완료 알림 | ctdxteam1.app.n8n.cloud |

---

## 기술 스택

| 항목 | 상세 |
|------|------|
| 런타임 | Node.js |
| DB 클라이언트 | mysql2 |
| AI | Claude CLI (stdin 파이프), @anthropic-ai/sdk |
| Google API | googleapis |
| 환경변수 | dotenv |
| 자동화 | Windows 작업 스케줄러 + run_report.bat |
| 리포트 배포 | GitHub Pages |
| 알림 | n8n Webhook |
