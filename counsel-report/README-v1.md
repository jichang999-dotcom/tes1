# [SOP] AI 리포트 제작 및 발송 자동화 시스템

## 1. 문서 개요

- **목적**: Google Sheets에 등록된 "발송 대상자"에게 주간 AI 분석 HTML 리포트 링크를 자동 발송하는 시스템
- **실행 주기**: 화~토 오후 19:00 (KST), 월요일 01:00 (KST) 보강 배치, 일요일 미실행
- **데이터 범위**: 실행일 기준 최근 30일 데이터 (D-30 ~ D-0, KST 기준)
- **시간 기준**: 모든 날짜/시간은 **KST (한국 표준시, UTC+9)** 기준
- **분류 체계**: **B안** (Admin → Strong → Weak → 미분류)

---

## 2. Step별 상세

| Step | 파일명 | 역할 | 입력 | 출력 | 중요도 |
|------|--------|------|------|------|--------|
| 1 | fetch_data.js | MariaDB에서 최근 30일 상담 데이터 추출 | MariaDB (db_square) | data/counsel_data.json | 필수 |
| 2 | analyze_sentiment.js | **4단계 분류**: Admin 제외 → Strong 즉시집계 → Weak+미분류 AI 후보 생성 | counsel_data.json, keywords.json, config/filter_rules.json | data/sentiment_results.json, data/ai_candidates.jsonl | 필수 |
| 3 | claude_enhance.js | **Weak+미분류** 대상 Claude AI 문맥분석. 캐시 대조 후 **신규(D-0~D-1) + 내용수정(text_hash 불일치) + 이전 FAILED 건만** AI 호출. **라운드 로빈 방식**으로 센터별 균등 배분. **할루시네이션 방지**: source_pk 검증 + 12개 카테고리 외 필터링 + D-0 부정 보정 + 학생 이름 보완. 나머지는 캐시 결과 자동 반영 | ai_candidates.jsonl, sentiment_results.json, cache/ai_review_cache.json | sentiment_results.json (보강) + suggested_keywords | 선택 |
| 4 | update_keywords.js | Step 3의 제안 키워드를 keywords.json에 반영 → 다음날 Step 2 강화 | sentiment_results.json | keywords.json (갱신) | 선택 |
| 5 | build_html.js | 센터별 HTML 리포트 생성. 학생별 부정건수를 센터 평균과 비교하여 **등급 부여** (🚨위험: 평균×2 초과 / 🟠주의: 평균×1.5 초과 / 🟢관심: 평균 초과). NEW 뱃지 판별 | sentiment_results.json, sentiment_previous.json | reports/\<cmp_cd\>.html | 필수 |
| 6 | generate_and_upload.js | GitHub Pages에 리포트 업로드 | reports/*.html | report_links.csv | 필수 |
| 7 | update_sheets.js | Google Sheets에 리포트 링크 삽입 | report_links.csv | Google Sheets 셀 갱신 | 선택 |
| 8 | update_raw_sheet.js | 상담 원데이터 + 키워드 매칭 결과를 Sheets에 누적 | counsel_data.json, keywords.json | Google Sheets "키워드raw" 시트 | 선택 |
| 9 | trigger_webhook.js | n8n 웹훅으로 완료 알림 전송 | 없음 | HTTP POST → n8n | 선택 |

---

## 3. 시스템 아키텍처

본 시스템은 **로컬 배치 프로그램(run_report.bat)**과 **n8n 클라우드 워크플로우**의 2단 구조로 운영됩니다.

### 2-1. 역할 분담

| 구분 | 담당 | 실행 환경 | 사유 |
|------|------|-----------|------|
| MariaDB 데이터 조회 | `fetch_data.js` | Windows PC | n8n 클라우드에서 MariaDB 직접 연결 불가 |
| 4단계 분류 (Admin→Strong→Weak→미분류) + AI 후보 생성 | `analyze_sentiment.js` | Windows PC | Admin 제외, Strong 즉시집계, Weak+미분류 AI 후보 파일(`ai_candidates.jsonl`) 생성 |
| Weak+미분류 Claude AI 문맥분석 보강 | `claude_enhance.js` | Windows PC (Claude Code CLI) | AI 후보 파일 기반 배치 처리, Weak 오탐 필터링, 캐시/lock 보호 |
| AI 제안 키워드 반영 | `update_keywords.js` | Windows PC | AI 제안 키워드를 `keywords.json`에 자동 반영 |
| HTML 리포트 생성 + 학생 등급 부여 | `build_html.js` | Windows PC | sample_report.html 디자인 기반 생성, 학생별 등급 (위험/주의/관심) 부여 |
| GitHub 업로드 | `generate_and_upload.js` | Windows PC | GitHub API 호출 |
| Google Sheets 발송링크 업데이트 | `update_sheets.js` | Windows PC | Google Sheets API (서비스 계정) |
| 원시 데이터 업로드 | `update_raw_sheet.js` | Windows PC | Google Sheets 키워드raw 시트 |
| 구글챗 DM 발송 | n8n 워크플로우 | n8n Cloud | Google Chat API 연동 |
| 사업부 이메일 발송 | n8n 워크플로우 | n8n Cloud | Gmail API 연동 |
| 발송 로그 기록 | n8n 워크플로우 | n8n Cloud | Google Sheets API 연동 |

### 2-2. 실행 흐름

```
[일일 배치] 화~토 19:00 KST (Windows 작업 스케줄러, 일/월 제외)
  │
  ▼
run_report.bat (9단계 배치 실행)
  ├─ [1/9] node fetch_data.js           ← MariaDB 쿼리 (D-30 ~ D-0, KST 기준), source_pk 부여
  ├─ [2/9] node analyze_sentiment.js    ← 4단계 분류: Admin제외→Strong즉시집계→Weak+미분류 AI후보 생성
  ├─ [3/9] node claude_enhance.js       ← Weak+미분류 AI 문맥분석. 라운드 로빈 센터 균등 배분, 캐시 미스만 AI 호출, 할루시네이션 방지 검증(source_pk/카테고리/D-0보정/학생명)
  ├─ [4/9] node update_keywords.js      ← AI 제안 키워드 자동 반영
  ├─ [5/9] node build_html.js           ← HTML 생성 + 학생 등급 부여 (위험/주의/관심, 센터 평균 대비)
  ├─ [6/9] node generate_and_upload.js  ← GitHub 업로드 + report_links.csv
  ├─ [7/9] node update_sheets.js        ← Google Sheets 발송링크 컬럼 업데이트
  ├─ [8/9] node update_raw_sheet.js     ← 원시 데이터 Google Sheets 업로드
  └─ [9/9] node trigger_webhook.js      ← n8n Webhook 호출 (발송 트리거)
        │
        ▼
n8n 워크플로우 (Webhook 트리거)
  ├─ 수신자 매칭 → 구글챗 DM 발송
  ├─ 사업부 이메일 발송
  └─ "발송 로그" 시트에 기록

[월요일 보강 배치] 매주 월요일 추가 실행
  └→ 지난주 실패/미전송/후순위 밀린 건 재처리 + 키워드 보강 검토
```

---

## 4. 관련 문서 및 서비스 링크

- 리포트 제작 관리 시트: [gid=1296826526](https://docs.google.com/spreadsheets/d/1lg75mmON79TwwzOwQCmt6l_3X8xxjj7Bze7Vz9DUcMY/edit?gid=1296826526#gid=1296826526)
- 발송 대상자 명단 시트: [gid=435721721](https://docs.google.com/spreadsheets/d/1lg75mmON79TwwzOwQCmt6l_3X8xxjj7Bze7Vz9DUcMY/edit?gid=435721721#gid=435721721)
- 발송 로그 기록 시트: [gid=899713549](https://docs.google.com/spreadsheets/d/1lg75mmON79TwwzOwQCmt6l_3X8xxjj7Bze7Vz9DUcMY/edit?gid=899713549#gid=899713549)
- n8n 워크플로우 - AI Report Automation v4 (발송 전용): Webhook 트리거 방식
- GitHub 리포트 저장소: [jichang999-dotcom/tes1](https://github.com/jichang999-dotcom/tes1)
- GitHub Pages 리포트 호스팅: [counsel-report](https://jichang999-dotcom.github.io/tes1/counsel-report/)

### 로컬 파일 구성

| 파일 | 용도 |
|------|------|
| `run_report.bat` | 메인 배치 프로그램 (9단계 자동 실행) |
| `fetch_data.js` | 1단계: MariaDB 데이터 추출 → `data/counsel_data.json` (각 row에 `source_pk` 포함) |
| `analyze_sentiment.js` | 2단계: 4단계 분류(Admin→Strong→Weak→미분류) → `data/sentiment_results.json` + AI 후보 파일 생성 |
| `claude_enhance.js` | 3단계: Weak+미분류 AI 문맥분석 (오탐 필터링 + 부정 추가 감지, 캐시/lock 보호) |
| `update_keywords.js` | 4단계: AI 제안 키워드를 `keywords.json`에 자동 반영 |
| `build_html.js` | 5단계: HTML 생성 + 학생 등급 부여 (🚨위험: 평균×2초과 / 🟠주의: 평균×1.5초과 / 🟢관심: 평균초과) → `reports/` |
| `generate_and_upload.js` | 6단계: GitHub 업로드 + report_links.csv |
| `update_sheets.js` | 7단계: Google Sheets "리포트 제작 관리 시트" 발송링크 업데이트 |
| `update_raw_sheet.js` | 8단계: 원시 데이터 Google Sheets 업로드 |
| `trigger_webhook.js` | 9단계: n8n Webhook 호출 (발송 트리거) |
| `lib/utils.js` | 공통 유틸리티 (log, normalizeText, generateTextHash, generateKeywordsHash) |
| `config/filter_rules.json` | 전처리 필터 설정 (최소 텍스트 길이, Admin rescue 설정) |
| `keywords.json` | 부정 감성 키워드 사전 (**3단계 구조: Admin 22개 / Strong 4카테고리 42개 / Weak 11카테고리 180+개**) |
| `credentials.json` | Google 서비스 계정 인증 키 (`.gitignore`에 추가 필요) |
| `claude_prompt.txt` | Claude AI 분석용 프롬프트 (참고용, 3단계에서는 내장 프롬프트 사용) |
| `sample_report.html` | HTML 리포트 디자인 템플릿 (이 파일의 CSS/구조를 그대로 사용) |
| `.env` | 환경변수 (DB/GitHub 접속 정보) |
| `setup_scheduler.bat` | Windows 작업 스케줄러 등록 (관리자 권한 실행) |
| `run_daily.js` | Node.js 대체 오케스트레이터 (레거시) |
| `data/` | JSON 데이터 저장 |
| `data/counsel_data.json` | 1단계 출력: DB 추출 데이터 (각 row에 `source_pk` 포함) |
| `data/sentiment_results.json` | 2~3단계 출력: 감성분석 결과 (Step 2 Strong + Step 3 AI 보강) |
| `data/ai_candidates.jsonl` | 2단계 출력: AI 재분석 후보 (Weak + 미분류, 필터 통과 건) |
| `data/ai_candidates.meta.json` | 2단계 출력: 후보 생성 메타데이터 (admin/strong/weak/unclassified 건수) |
| `data/sentiment_previous.json` | 이전 실행 결과 백업 (NEW 뱃지 비교용, 하루 1회만 갱신) |
| `cache/` | 캐시 저장소 |
| `cache/ai_review_cache.json` | 3단계 캐시: `{ source_pk: { text_hash, analyzed_at, categories } }` 기반 중복 분석 방지 + 이전 AI 부정 결과 재반영 |
| `reports/` | 생성된 HTML 리포트 로컬 보관 폴더 |
| `logs/` | 실행 로그 저장 폴더 (KST 기준 날짜) |

**사용 서비스**: MariaDB (데이터 소스), **Claude Code CLI** (AI 감성분석, 구독형), GitHub (HTML 호스팅), Google Sheets (트리거/로깅), Google Chat (DM 발송), Gmail (이메일 발송), n8n (발송 워크플로우), Windows 작업 스케줄러 (스케줄링)

---

## 5. 데이터 흐름도

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
│  │ 4단계 판정 순서:            │   ★피드백 루프★
│  │ 1. Admin 제외 (행정/운영)   │   (키워드 보강으로 Weak 감지율 향상)
│  │ 2. Strong → 즉시 부정 집계  │        │
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
  │  캐시 대조 → 캐시 미스만 AI 호출
  │  ┌────────────────────────────────────────────┐
  │  │ ★ 라운드 로빈 (센터 균등 배분)             │
  │  │ AAA→AAM→ACL→...→AAA→AAM→... (500건/청크)  │
  │  │ Rate limit 시 모든 센터 최소 1회 분석 보장  │
  │  └────────────────────────────────────────────┘
  │
  │  ┌────────────────────────────────────────────┐
  │  │ ★ 할루시네이션 방지 검증                   │
  │  │ ① source_pk가 입력에 없으면 제거           │
  │  │ ② 허용 12개 카테고리 외 자동 필터링        │
  │  │ ③ D-0 부정 건 → d0_negative_count 반영     │
  │  │ ④ AI 감지 학생 → student_names 보완        │
  │  └────────────────────────────────────────────┘
  │
  │  Weak: 문맥 검증으로 오탐 필터링
  │  미분류: 새로운 부정 감지
  │  → suggested_keywords 수집
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
  │  학생별 부정건수 vs 센터 평균 비교 → 등급 부여
  │  ┌──────────────────────────────────────┐
  │  │ 🚨 위험: 평균 × 2 초과              │
  │  │ 🟠 주의: 평균 × 1.5 초과 (2배 이하) │
  │  │ 🟢 관심: 평균 초과 (1.5배 이하)     │
  │  │ ※ 평균 이하 학생은 리스트 미포함    │
  │  └──────────────────────────────────────┘
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

## 6. 환경 설정

### 5-1. 로컬 환경변수 (`.env`)

| 변수 | 설명 |
|------|------|
| `MYSQL_HOST` | MariaDB 호스트 (61.74.65.40) |
| `MYSQL_PORT` | MariaDB 포트 (3306) |
| `MYSQL_USER` | MariaDB 접속 계정 |
| `MYSQL_PASSWORD` | MariaDB 접속 비밀번호 |
| `MYSQL_DB` | 데이터베이스명 (db_square) |
| `GITHUB_TOKEN` | GitHub Personal Access Token (repo 권한) |
| `GITHUB_OWNER` | GitHub 사용자명 (jichang999-dotcom) |
| `GITHUB_REPO` | GitHub 저장소명 (tes1) |
| `N8N_WEBHOOK_URL` | n8n 워크플로우 Webhook URL |
| `GOOGLE_CREDENTIALS_PATH` | Google 서비스 계정 JSON 키 파일 경로 (기본값: `./credentials.json`) |

> **Anthropic API 키는 불필요합니다.** Claude Code CLI(구독형)를 통해 AI 분석을 수행합니다.
> Google Sheets API 사용을 위해 서비스 계정(`credentials.json`)이 필요합니다. 해당 시트에 서비스 계정 이메일을 편집자로 공유해야 합니다.

### 5-2. Windows 작업 스케줄러

#### 일일 배치 (화~토 19:00, 일/월 제외)

- **작업 이름**: `AI_Report_Daily`
- **실행 시간**: 화~토 19:00 (KST), 일/월 미실행
- **실행 대상**: `run_report.bat`
- **등록 방법**: `setup_scheduler.bat`을 관리자 권한으로 실행
- **확인 명령**: `schtasks /query /tn "AI_Report_Daily" /v /fo list`

#### 월요일 보강 배치 (매주 월요일)

- **작업 이름**: `AI_Report_Monday`
- **실행 시간**: 매주 월요일 01:00 (KST)
- **실행 대상**: `run_report_monday.bat`
- **목적**: 지난주 실패/미전송/후순위 밀린 건 재처리 + suggested_keywords 기반 키워드 보강 검토
- **정상 운영 시**: 100~300건
- **맥시멈** (주간 전체 장애 시): ~1,362건

#### 스케줄러 등록 스크립트 (`setup_scheduler.bat`)

```batch
@echo off
chcp 65001 >nul
echo [스케줄러 등록] AI 리포트 자동화

REM === 일일 배치 (화~토 19:00 KST, 일/월 제외) ===
schtasks /create /tn "AI_Report_Daily" /tr "\"%~dp0run_report.bat\"" /sc weekly /d TUE,WED,THU,FRI,SAT /st 19:00 /f /rl highest
echo [완료] AI_Report_Daily - 화~토 19:00 KST

REM === 월요일 보강 배치 (매주 월요일 01:00) ===
schtasks /create /tn "AI_Report_Monday" /tr "\"%~dp0run_report_monday.bat\"" /sc weekly /d MON /st 01:00 /f /rl highest
echo [완료] AI_Report_Monday - 매주 월요일 01:00

echo.
echo [확인] 등록된 작업 목록:
schtasks /query /tn "AI_Report_Daily" /fo list | findstr "작업 이름 다음 실행 시간 상태"
schtasks /query /tn "AI_Report_Monday" /fo list | findstr "작업 이름 다음 실행 시간 상태"
pause
```

### 5-3. n8n 워크플로우 설정

1. n8n 워크플로우 **AI Report Automation v4** (ID: `W8Hhg3JbSVkL0xHL`)가 Active 상태인지 확인
2. Webhook 노드의 Production URL 확인
3. `.env`의 `N8N_WEBHOOK_URL`에 해당 URL 입력

---

## 7. 업무 프로세스 (Workflow)

### 1단계: 데이터 추출 (`fetch_data.js`)

- **데이터 조회**: MariaDB(db_square)에 접속하여 최근 30일(D-30 ~ D-0, KST 기준) 상담 데이터를 추출
  - 조회 테이블: `tb_counsel`, `tb_counsel_employ`, `tb_company`, `tb_employ`, `tb_student` (JOIN)
  - 조회 필드: `cns_cd`, `cmp_cd`, `cmp_nm`, `std_cd`, `std_nm`(학생명), `emp_cd`, `reg_dt`, `cns_st`, `cns_req_ct`
  - 필터 조건: `fc_cls_sn = '1'`, `brn_cls = 'BC0002'`, 제외 cmp_cd: `AFA, AKU, CAY, CCA`
- **source_pk 부여**: 각 row에 `source_pk = cns_cd`로 고유 식별자 부여. 이후 모든 단계에서 동일 식별자로 추적
- **cmp_cd별 그룹핑**: 조회 결과를 cmp_cd(센터) 단위로 분류
- **저장**: `data/counsel_data.json`

### 2단계: 4단계 분류 — Admin/Strong/Weak/미분류 (`analyze_sentiment.js`)

#### 분류 흐름 (B안)

```
전체 11,493건
  │
  ├─ [1] Admin 제외:      1,701건 (14.8%)  ← 행정/운영 기록
  ├─ [2] Strong 즉시집계:   848건 (7.4%)   ← 확정 부정, AI 불필요
  ├─ [3] Weak → AI 후보:  2,896건 (25.2%)  ← 문맥 검증 필요
  └─ [4] 미분류 → AI 후보: 6,048건 (52.6%) ← 키워드 미매칭
```

#### B안 선택 근거

| 비교 | A안 (Weak 즉시집계) | **B안 (Weak→AI 검증)** | C안 (절충) |
|------|--------------------|-----------------------|-----------|
| Weak 처리 | 즉시 집계 (오탐 포함) | **AI 문맥 검증 후 집계** | 즉시 집계 + 월1회 샘플 |
| AI 일평균 (D-0) | ~141건 | **~228건** | ~141건 |
| 퀄리티 | 낮음 | **높음** | 중간 |
| 오탐 대응 | 없음 | **실시간** | 월1회 |

**B안 선택 이유**: AI 처리량이 감당 가능한 수준(일평균 228건)이므로 퀄리티 우선

#### 판정 상세

1. **Admin 제외**: 제목(cns_st)에 Admin 키워드(부재, 통화중, 설명회 등 22개) 매칭 + 본문(cns_req_ct) 10자 미만 → 제외. 단, 본문에 Strong 키워드가 있으면 rescue하여 Strong으로 복원
2. **Strong 즉시 집계**: 확정적 부정 표현(환불, 퇴소, 컴플레인, 자해 등 42개) → AI 불필요, 즉시 학생별 부정 건수 반영
3. **Weak → AI 후보**: 문맥 검증이 필요한 표현(힘들, 어렵, 속상, 결석 등 180+개) → AI가 문맥 분석하여 오탐 필터링
4. **미분류 → AI 후보**: 키워드 미매칭 건 → AI가 새로운 부정 감지

#### AI 후보 파일 생성

- Weak + 미분류 중 전처리 필터 통과 건 → `data/ai_candidates.jsonl`
- 각 후보 row: `source_pk`, `cmp_cd`, `cns_cd`, `std_cd`, `reg_dt`, `cns_st`, `cns_req_ct`, `text_hash`, `classification`(weak/unclassified), `weak_category`
- 메타데이터(`data/ai_candidates.meta.json`)에 admin/strong/weak/미분류/필터제외/후보 건수 기록

#### 저장

- `data/sentiment_results.json` (Strong만 반영, AI 보강 전)

### 3단계: AI 문맥분석 — Weak+미분류 대상 (`claude_enhance.js`)

#### 목적

- **Weak**: 키워드 매칭됐지만 문맥상 긍정/중립인 건을 AI가 걸러냄 (예: "어려운 문제도 잘 풀었습니다" → 부정 아님)
- **미분류**: 키워드 사전에 없는 부정 표현을 AI가 추가 탐지

#### AI 프롬프트 핵심 지시

```
Weak 키워드가 포함되어 있어도 문맥상 긍정/중립이면 부정으로 판정하지 마세요.
예) "어려운 문제도 잘 풀었습니다" → 긍정 (부정 아님)
예) "힘들었지만 잘 해냈습니다" → 긍정 (부정 아님)
예) "결석 연락 드렸습니다" → 행정 기록 (부정 아님)
```

#### Claude Code CLI 호출 방식

```javascript
execSync('claude -p --allowedTools "Read,Write,Glob,Grep" --max-turns 30', {
  input: prompt,  // stdin으로 프롬프트 전달
  timeout: 600000,
});
```

- 센터(cmp_cd)별로 묶어서 호출 (같은 센터 맥락 유지 → 품질 향상)
- 500건 초과 센터는 500건씩 분할 (MAX_PER_CALL)
- **라운드 로빈 방식**: 센터별 청크를 번갈아 호출 (AAA→AAM→ACL→...→AAA→AAM→...)
  - Rate limit 시에도 모든 센터가 균등하게 분석된 상태로 중단
  - 특정 센터만 분석 누락되는 상황 방지

#### 캐시 구조 (`cache/ai_review_cache.json`)

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
- **text_hash**: 상담 내용(제목+본문)을 해시 함수로 변환한 고정 길이 지문(fingerprint). **상담 원문은 캐시에 저장하지 않음** (개인정보 보호 + 용량 절감)
- **categories**: 이전 AI 부정 분석 결과 저장 → 다음 실행 시 재반영
- **TTL**: 60일 보관, 만료 시 자동 제거

#### text_hash 동작 원리

```
원문: "너무 힘들어서 수업 가기 싫다고 합니다"
  → hash 함수 → "be79251a9f32471281fda84fbf875824" (32자 고정)

- 원문이 같으면 → 항상 같은 hash
- 원문이 한 글자라도 바뀌면 → 완전히 다른 hash
```

#### 캐시 판단 흐름

```
매일 Step 3 실행 시:
  DB에서 CN002082650의 상담 내용을 가져옴
  → hash 계산 → "be79251a..."
  → 캐시에서 CN002082650 조회 → text_hash 일치?

  ✅ 일치 (캐시 히트)
    → 내용 안 바뀜 → AI 재호출 불필요
    → 캐시의 categories를 sentiment_results.json에 자동 반영
    → 부정 건수에 카운트됨

  ❌ 불일치 (캐시 미스)
    → 내용이 수정됨 → 당일 Step 3에서 AI 즉시 재분석
    → 새 결과로 캐시 갱신

  ⚠️ AI 재분석 실패 (Rate limit 등)
    → FAILED 처리 → 월요일 보강 배치에서 재처리
```

#### AI 실제 호출 대상 (캐시 미스 항목만)

ai_candidates.jsonl 전체를 캐시(`ai_review_cache.json`)와 대조한 후, 아래 3가지 경우에만 AI를 호출합니다:

| # | 조건 | 설명 |
|---|------|------|
| ① | **신규 데이터 (D-0~D-1)** | 캐시에 source_pk 자체가 없는 건 |
| ② | **내용 수정 건** | source_pk는 있으나 text_hash가 불일치 (상담 내용이 변경됨) |
| ③ | **이전 FAILED 건** | Rate limit 등으로 AI 분석 실패 → 캐시에 미저장 |

- **캐시 히트** (source_pk + text_hash 일치): AI 호출 없이 캐시의 `categories`를 `sentiment_results.json`에 자동 반영
- 30일치 데이터를 DB에서 가져오지만, AI 호출은 위 3가지 캐시 미스 건에만 발생 → **비용 및 시간 효율화**

#### 보호 장치

| 장치 | 설명 |
|------|------|
| 캐시 중복 방지 | source_pk + text_hash 기반 (60일 보관) |
| 이전 결과 재반영 | 캐시에 categories 저장 → 재분석 없이 복원 |
| Rate limit 감지 | 한도 도달 시 즉시 중단, 다음 실행에서 이어서 처리 |
| 2회 재시도 | 5초 간격, 일시적 오류 대응 |
| Lock 파일 | PID 기반 중복 실행 방지 (60분 타임아웃) |
| 상태 관리 | PENDING → PROCESSING → DONE/FAILED |
| **source_pk 검증** | AI 반환 source_pk가 입력 데이터에 없으면 제거 (할루시네이션 방지) |
| **카테고리 검증** | 허용된 12개 카테고리 외 항목 자동 필터링 |
| **키워드 포맷 검증** | AI 제안 키워드의 타입/공백/길이/중복 검증 후 반영 |
| **D-0 부정 건수 보정** | AI 감지 부정 건의 reg_dt로 당일 여부 판별 → d0_negative_count 반영 |
| **학생 이름 보완** | AI로만 감지된 학생도 std_nm을 student_names에 반영 |

### 핵심: Step 2↔3↔4 피드백 루프

```
Day 1:
    Step 2: Admin/Strong/Weak 분류 → Weak+미분류 ~300건 AI 전송
    Step 3: Claude가 문맥분석 → Weak 오탐 필터링 + 미분류 부정 감지 + 키워드 15개 제안
    Step 4: 15개 중 신규 10개 → keywords.json 'AI_보강_키워드'에 추가

Day 2:
    Step 2: 보강된 키워드로 미분류 감소 → AI 전송량 감소
    Step 3: 캐시 히트로 기처리분 스킵 + 신규만 분석

Day N:
    Step 2: 충분히 보강된 키워드로 대부분 감지 → 미분류 극소
    Step 3: Claude 처리 시간 대폭 단축
```

**목적**: AI 처리량을 점진적으로 줄이면서 분석 퀄리티는 유지하는 자가 학습 구조

### 4단계: 키워드 반영 (`update_keywords.js`)

- Step 3의 제안 키워드를 `keywords.json`의 Weak `AI_보강_키워드` 카테고리에 자동 추가

### 5단계: HTML 리포트 생성 (`build_html.js`)

- **디자인 기준**: `sample_report.html`의 CSS/HTML 구조를 그대로 사용 (max-width: 480px 모바일 레이아웃)
- **HTML 구성 요소**:
  1. **topbar**: 센터명 + 리포트 기간
  2. **기간 요약 카드**: periodRow + compare-table (D-30 누적 / D-0 누적 비교 테이블)
  3. **부정 감성 키워드 빈도**: 상위 8개, 이모지 포함, 바차트(bar-bg/bar-fill). 빈도 합산 = 부정 감성 건수
  4. **위험 학생 알림 배너** (alertBanner): 등급이 "위험"인 학생 수만 카운트하여 표시
  5. **평균 초과 학생 리스트**: data-table (table-layout: fixed, colgroup으로 열 너비 고정)
     - 컬럼: **학생명** / 학생코드 / 부정 건수 / 초과율(정수 반올림, %) / 등급 / 신규
     - 위험등급: 평균×2 초과 = 위험, 평균×1.5 초과 = 주의, 그 외 = 관심
     - **NEW 태그**: 이전 실행(`data/sentiment_previous.json`)과 비교하여 신규 평균초과 학생에 표시
  6. **푸터**: 생성일시 (KST)
- **로컬 저장**: `reports/{cmp_cd}.html`
- **이전 결과 백업**: 현재 분석 결과를 `data/sentiment_previous.json`으로 복사 (다음 실행 시 NEW 판별용)

### 6단계: GitHub 업로드 (`generate_and_upload.js`)

- **GitHub 업로드**: `counsel-report/{cmp_cd}.html` 경로에 업로드 (덮어쓰기 — sha 조회 후 갱신)
- **저장소**: `jichang999-dotcom/tes1` (main 브랜치)
- **링크 생성**: `https://jichang999-dotcom.github.io/tes1/counsel-report/{cmp_cd}.html`
- **링크 목록 저장**: `report_links.csv`에 전체 결과 기록

### 7단계: Google Sheets 업데이트 (`update_sheets.js`)

- Google 서비스 계정으로 "리포트 제작" 시트에서 **PK = "A1"인 행**만 필터링하여 cmp_cd 매칭 행의 발송링크(E열) 업데이트

### 8단계: 원시 데이터 업로드 (`update_raw_sheet.js`)

- 상담 원데이터 + 키워드 매칭 결과를 Google Sheets "키워드raw" 시트에 누적

### 9단계: n8n Webhook 호출 (`trigger_webhook.js`)

- 업로드 성공한 리포트 목록(cmp_cd, 센터명, 링크, 상담건수)을 n8n Webhook으로 POST 전송

### n8n 워크플로우: 구글챗 발송 + 사업부 이메일 발송

#### 노드 구성

```
Webhook (POST) → Read Report Config (read: sheet) → Filter1 → Create a message (create: message)
  → Log Chat Send (append: sheet) → Read Business Units (read: sheet) → Prepare BU Email → Send BU Email (send: message)
```

#### 처리 흐름

1. **Webhook** — 로컬 배치에서 전송한 POST 수신
2. **Read Report Config** — "발송 대상자" 시트 읽기
3. **Filter1** — 이메일(구글챗ID) + 발송링크 모두 있는 행만 통과
4. **Create a message** — 구글챗 DM 발송: `안녕하세요 {{이름}}님 {{cmp_nm}} 리포트 링크입니다. {{발송링크}}`
5. **Log Chat Send** — "발송 로그" 시트에 발송일시(KST), 구글챗ID, 발송내용 기록
6. **Read Business Units** — "사업부" 시트에서 이메일 주소 읽기
7. **Prepare BU Email** — 센터별 리포트 링크 목록 + 발송일시로 메일 본문 생성
8. **Send BU Email** — Gmail API로 사업부 담당자에게 발송

---

## 8. 키워드 사전 (keywords.json) — 3단계 구조

### Admin 키워드 (22개) — 제외 대상

운영/행정성 기록. 제목에 매칭 + 본문 10자 미만일 때 제외.

| 키워드 |
|--------|
| 부재, 통화중, 설명회, 온라인진단, 진단예약, 진단안내, 학급보강, 개인보강, 시간변경, 일정조율, 참석, 문자 안내, 연락 요청, 테스트, 삭제요청, 상담이관, 자동등록, 시스템, 데이터정리, 일괄등록, 일괄처리, 진단취소 |

### Strong 키워드 (4개 카테고리, 42개) — 즉시 부정 집계

| 카테고리 | 키워드 수 | 대표 키워드 |
|---------|----------|------------|
| 이탈/포기 의사 | 17 | 환불, 퇴소, 자퇴, 수업거부, 등원거부, 그만두, 휴회 |
| 불만/분노 | 10 | 컴플레인, 항의, 폭언, 욕설, 교사교체, 강사변경 |
| 위험행동 | 11 | 자해, 자살, 죽고싶, 도벽, 훔쳤, 가출 |
| 가정문제 | 4 | 이혼, 별거, 가정폭력, 가정불화 |

### Weak 키워드 (11개 카테고리, 180+개) — AI 문맥 검증 대상

| 카테고리 | 키워드 수 | 대표 키워드 |
|---------|----------|------------|
| 스트레스/압박감 | 14 | 스트레스, 부담, 힘들, 지쳐, 예민 |
| 우울/무기력 | 20 | 우울, 무기력, 위축, 자존감, 울음 |
| 불만/분노 | 17 | 불만, 짜증, 속상, 답답, 격앙 |
| 불안/걱정 | 14 | 불안, 걱정, 초조, 긴장, 눈물 |
| 대인관계 갈등 | 19 | 갈등, 따돌림, 왕따, 괴롭, 충돌 |
| 이탈/포기 의사 | 18 | 포기, 결석, 거부, 불참, 휴원 |
| 학습 어려움 | 23 | 어렵, 못하, 이해안, 집중력, 부진 |
| 부적응 | 11 | 적응못, 부적응, 안맞, 불편, 견디기 |
| 사춘기/반항 | 9 | 사춘기, 반항, 싫다고, 하기싫 |
| 건강문제 | 7 | 아파서, 두통, 복통, 통증 |
| AI_보강_키워드 | 45 | 오기싫어, 자신감이없, 숙제를안, 수업방해 등 |

---

## 9. 필터 조건

### Admin 판정 (`analyze_sentiment.js`)

| 조건 | 설명 |
|------|------|
| Admin 제목 매칭 | 제목(cns_st)에 Admin 키워드 포함 |
| 본문 짧음 | 내용(cns_req_ct)이 10자 미만 |
| Rescue | Admin 판정이어도 본문에 Strong 키워드 있으면 Strong으로 복원 |

### AI 후보 필터 (`passesPreFilter`)

| 조건 | 설명 |
|------|------|
| 빈 텍스트 | 제목(cns_st)과 내용(cns_req_ct) 모두 비어있으면 제외 |
| 최소 길이 미달 | 제목+내용 합쳐서 10자 미만이면 제외 |

---

## 10. 운영 모드

### 일일 배치 (화~토 19:00 KST, 일/월 제외)

| 항목 | 내용 |
|------|------|
| 대상 데이터 | D-0~D-1 |
| Admin 제외 | 행정/운영 기록 자동 제외 |
| Strong | 즉시 학생별 부정 건수 반영 |
| Weak + 미분류 | AI 전송 → 문맥 분석 후 집계 |
| 일평균 AI 처리량 | D-0: ~228건 / D-0~D-1: ~427건 |

### 월요일 보강 배치

| 항목 | 내용 |
|------|------|
| 대상 | 지난주 실패/미전송/후순위 밀린 건 |
| 정상 운영 시 | 100~300건 |
| 맥시멈 (전체 장애) | ~1,362건 (주간 전체) |
| 추가 작업 | suggested_keywords 기반 키워드 보강 검토/자동 반영 |

### D-0 vs D-0~D-1 AI처리량 (실측 데이터)

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

## 11. 분석 대상 센터 (13개)

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

## 12. 에러 처리 및 로깅

### 에러 처리 전략

- **Step 1, 2, 5, 6** 실패 시 → **배치 중단** (핵심 리포트 생성/배포)
- **Step 3, 4, 7, 8, 9** 실패 시 → **경고 후 계속** (보조 기능)

Step 3이 실패해도 Step 2의 Strong 키워드 분석 결과만으로 리포트가 정상 생성됩니다.
**월요일 보강 배치에서 주간 미처리 건 일괄 재처리.**

### Step 3 Rate Limit 대응

- Rate limit 감지 시 남은 호출 즉시 중단 (불필요한 재시도 방지)
- 성공한 건은 캐시에 결과(categories) 포함 저장
- 다음 실행 시 캐시 히트된 건은 AI 호출 없이 결과 재반영
- 미처리 건만 신규 호출 → 여러 날에 걸쳐 점진적 완료
- **월요일 보강 배치에서 주간 미처리 건 일괄 재처리**

### n8n 워크플로우 에러 처리

- 구글챗 발송 실패: 개별 실패가 전체 워크플로우를 중단시키지 않음
- 사업부 메일 발송 실패: n8n 에러 핸들러 처리
- 장애 발생 보고: 담당자(`jichang999@askwhy.co.kr`)에게 에러 알림 메일 발송

### 실행 로그

- `logs/run_YYYY-MM-DD.log` 파일에 단계별 진행 상황 기록 (KST 기준 타임스탬프)

---

## 13. 예상 소요 시간

| Step | 소요 시간 | 비고 |
|------|----------|------|
| 1 (DB 추출) | 1~3분 | |
| 2 (4단계 분류) | 1~2분 | Admin/Strong/Weak/미분류 판정 |
| 3 (Claude AI) | 10~25분 → 점차 단축 | Weak+미분류 문맥분석 + 피드백 루프 효과 + 캐시 재반영 |
| 4 (키워드 갱신) | <1분 | |
| 5 (HTML 생성) | 1~2분 | |
| 6 (GitHub 업로드) | 2~5분 | |
| 7 (Sheets 링크) | 1~2분 | |
| 8 (Raw 데이터) | 2~5분 | |
| 9 (웹훅 알림) | <1분 | |
| **합계** | **20~45분 → 점차 단축** | |

---

## 14. Claude Code 사용량 및 비용

### Claude Code는 구독형 (API 과금 아님)

Claude Code CLI는 **Anthropic 구독 플랜** 기반으로 동작합니다. API 토큰 과금이 아닙니다.

| 플랜 | 월 비용 | 사용량 | 비고 |
|------|---------|--------|------|
| **Pro** | $20/월 | 기본 사용량 | 일 500건 처리 시 Rate limit 가능성 높음 |
| **Max (5x)** | $100/월 | Pro 대비 5배 | 일 500건 처리 권장 플랜 |
| **Max (20x)** | $200/월 | Pro 대비 20배 | 여유 있는 운영 |

### 일 500건 분석 시 사용량 추산

| 항목 | 추산 |
|------|------|
| 일일 상담 건수 | ~500건 |
| Admin 제외 후 | ~350~400건 |
| Strong 즉시 집계 (AI 불필요) | ~50~100건 |
| **AI 분석 대상 (Weak + 미분류)** | **~250~350건** |
| 센터별 배치 호출 횟수 | ~15~25회/일 |
| 키워드 보강 안정화 후 | AI 대상 ~200건/일 수준으로 감소 |

### 권장 플랜

- **초기 운영 (안정화 전)**: **Max 5x ($100/월)** — AI 대상 건수가 많고, 재시도/월요일 보강까지 고려
- **안정화 후**: **Pro ($20/월)** 가능성 — 키워드 보강으로 AI 대상 감소 후 재평가
- Rate limit 발생 시: 성공 건은 캐시 보존, 미처리 건은 다음 실행 시 자동 소화

---

## 15. 외부 연동

| 서비스 | 용도 | 접속 정보 |
|--------|------|----------|
| MariaDB | 상담 데이터 원본 | db_square (61.74.65.40:3306) |
| Claude CLI | AI 문맥 감성 분석 | claude -p (stdin 기반) |
| GitHub Pages | HTML 리포트 배포 | jichang999-dotcom.github.io/tes1/counsel-report/ |
| Google Sheets | 리포트 링크 + 원데이터 | 서비스 계정 (credentials.json) |
| n8n Webhook | 완료 알림 | ctdxteam1.app.n8n.cloud |

---

## 16. 운영 가이드

### 수동 실행

```bash
cd C:\Users\User\Desktop\test
run_report.bat
```

월요일 보강 배치 수동 실행:
```bash
run_report_monday.bat
```

개별 단계 실행:
```bash
node fetch_data.js
node analyze_sentiment.js
node claude_enhance.js
node update_keywords.js
node build_html.js
node generate_and_upload.js
node update_sheets.js
node update_raw_sheet.js
node trigger_webhook.js
```

### 스케줄러 상태 확인

```bash
schtasks /query /tn "AI_Report_Daily" /v /fo list
schtasks /query /tn "AI_Report_Monday" /v /fo list
```

### 주의사항

1. **PC 전원**: Windows 작업 스케줄러는 PC가 켜져 있어야 실행됩니다. 19:00에 PC가 꺼져 있으면 해당 일자 리포트가 생성되지 않습니다.
2. **네트워크**: MariaDB(61.74.65.40), GitHub API, n8n Cloud에 모두 접근 가능한 네트워크 환경이어야 합니다.
3. **환경변수**: `.env` 파일에 모든 접속 정보가 정확히 입력되어야 합니다. `.env`와 `credentials.json` 파일은 절대 Git에 커밋하지 마세요.
4. **n8n 워크플로우**: AI Report Automation v4가 Active 상태여야 Webhook 호출이 정상 작동합니다.
5. **Claude Code**: `claude` CLI 명령이 사용 가능해야 합니다. Rate limit 발생 시 월요일 보강 배치에서 자동 재처리.
6. **감성분석 키워드**: `keywords.json`이 3단계 구조(Admin/Strong/Weak). Strong은 보수적(확정 부정만), Weak는 넓게 설정.
7. **전처리 필터**: `config/filter_rules.json`에서 최소 텍스트 길이, Admin rescue 설정을 관리.
8. **시간 기준**: 모든 날짜/시간은 KST(UTC+9) 기준.
9. **캐시 관리**: `cache/ai_review_cache.json`은 60일 초과 항목 자동 정리. 수동 초기화 시 해당 파일 삭제 (다음 실행 시 전체 재분석).
10. **Lock 파일**: `data/.claude_enhance.lock` — 비정상 종료 시 60분 후 자동 해제, 수동 삭제 가능.

---

## 17. 기술 스택

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
