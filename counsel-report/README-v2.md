# [SOP] AI 리포트2 제작 및 발송 자동화 시스템 (D-0 분석 + 30일 누적)

## 1. 문서 개요

- **목적**: Google Sheets에 등록된 "발송 대상자"에게 AI 분석 HTML 리포트 링크를 자동 발송하는 시스템
- **핵심 컨셉**: **매일 D-0(당일) 데이터만 분석**하고, 분석 결과를 **로컬에 누적 저장**하여 최대 30일치 누적 데이터로 리포트를 생성
- **실행 주기**: 화~토 오후 19:00 (KST), 월요일 01:00 (KST) 보강 배치, 일요일 미실행
- **데이터 범위**: DB 조회는 **D-0(당일)만**, 리포트 표시는 **누적 최대 30일**
- **시간 기준**: 모든 날짜/시간은 **KST (한국 표준시, UTC+9)** 기준
- **분류 체계**: **B안** (Admin → Strong → Weak → 미분류)

### 기존 시스템(리포트1)과의 차이

| 비교 항목 | 리포트1 (기존) | **리포트2 (본 문서)** |
|-----------|---------------|----------------------|
| DB 조회 범위 | 매일 D-30 ~ D-0 (30일치 전체) | **매일 D-0만 (당일 데이터만)** |
| AI 분석 대상 | D-0~D-1 신규 + 캐시 미스 건 | **D-0 당일 건만** |
| 30일 데이터 확보 | 매번 DB에서 30일치 재조회 | **로컬 누적 파일에서 30일치 확보** |
| DB 부하 | 높음 (매일 30일치 전체 조회) | **낮음 (당일 데이터만 조회)** |
| AI 처리량 | 일평균 ~228건 (캐시 미스 기준) | **일평균 ~228건 (D-0 전체)** |
| 데이터 정합성 | DB 원본 기준 (매번 재조회) | **분석 시점 확정 (D-0 분석 결과를 그대로 누적)** |

---

## 2. Step별 상세

| Step | 파일명 | 역할 | 입력 | 출력 | 중요도 |
|------|--------|------|------|------|--------|
| 1 | fetch_data.js | MariaDB에서 **D-0(당일) 상담 데이터만** 추출 | MariaDB (db_square) | data/counsel_data_d0.json | 필수 |
| 2 | analyze_sentiment.js | **4단계 분류**: Admin 제외 → Strong 즉시집계 → Weak+미분류 AI 후보 생성 (D-0 데이터만 대상) | counsel_data_d0.json, keywords.json, config/filter_rules.json | data/sentiment_d0.json, data/ai_candidates.jsonl | 필수 |
| 3 | claude_enhance.js | **Weak+미분류** 대상 Claude AI 문맥분석. 라운드 로빈 센터 균등 배분. 할루시네이션 방지 검증. D-0 데이터만 처리하므로 캐시 불필요 | ai_candidates.jsonl, sentiment_d0.json | sentiment_d0.json (보강) + suggested_keywords | 선택 |
| 4 | update_keywords.js | Step 3의 제안 키워드를 keywords.json에 반영 → 다음날 Step 2 강화 | sentiment_d0.json | keywords.json (갱신) | 선택 |
| 5 | accumulate_results.js | D-0 분석 결과를 **누적 파일에 병합**. 30일 초과 데이터 자동 정리 | sentiment_d0.json, data/sentiment_accumulated.json | data/sentiment_accumulated.json (갱신) | 필수 |
| 6 | build_html.js | **누적 데이터(최대 30일)** 기반 센터별 HTML 리포트 생성. 학생별 등급 부여. NEW 뱃지 판별 | sentiment_accumulated.json, sentiment_previous.json | reports/\<cmp_cd\>.html | 필수 |
| 7 | generate_and_upload.js | GitHub Pages에 리포트 업로드 | reports/*.html | report_links.csv | 필수 |
| 8 | update_sheets.js | Google Sheets에 리포트 링크 삽입 | report_links.csv | Google Sheets 셀 갱신 | 선택 |
| 9 | update_raw_sheet.js | D-0 상담 원데이터 + 키워드 매칭 결과를 Sheets에 누적 | counsel_data_d0.json, keywords.json | Google Sheets "키워드raw" 시트 | 선택 |
| 10 | trigger_webhook.js | n8n 웹훅으로 완료 알림 전송 | 없음 | HTTP POST → n8n | 선택 |

---

## 3. 시스템 아키텍처

본 시스템은 **로컬 배치 프로그램(run_report2.bat)**과 **n8n 클라우드 워크플로우**의 2단 구조로 운영됩니다.

### 3-1. 역할 분담

| 구분 | 담당 | 실행 환경 | 사유 |
|------|------|-----------|------|
| MariaDB D-0 데이터 조회 | `fetch_data.js` | Windows PC | n8n 클라우드에서 MariaDB 직접 연결 불가 |
| 4단계 분류 + AI 후보 생성 | `analyze_sentiment.js` | Windows PC | D-0 데이터에 대해 Admin/Strong/Weak/미분류 판정 |
| Weak+미분류 Claude AI 문맥분석 | `claude_enhance.js` | Windows PC (Claude Code CLI) | D-0 AI 후보 배치 처리, 할루시네이션 방지 검증 |
| AI 제안 키워드 반영 | `update_keywords.js` | Windows PC | AI 제안 키워드를 `keywords.json`에 자동 반영 |
| **D-0 결과 누적 병합** | `accumulate_results.js` | Windows PC | **D-0 분석 결과를 30일 누적 파일에 병합 + 만료 정리** |
| HTML 리포트 생성 + 학생 등급 | `build_html.js` | Windows PC | **누적 데이터(최대 30일) 기반** 리포트 생성 |
| GitHub 업로드 | `generate_and_upload.js` | Windows PC | GitHub API 호출 |
| Google Sheets 발송링크 업데이트 | `update_sheets.js` | Windows PC | Google Sheets API (서비스 계정) |
| 원시 데이터 업로드 | `update_raw_sheet.js` | Windows PC | Google Sheets 키워드raw 시트 |
| 구글챗 DM 발송 | n8n 워크플로우 | n8n Cloud | Google Chat API 연동 |
| 사업부 이메일 발송 | n8n 워크플로우 | n8n Cloud | Gmail API 연동 |
| 발송 로그 기록 | n8n 워크플로우 | n8n Cloud | Google Sheets API 연동 |

### 3-2. 실행 흐름

```
[일일 배치] 화~토 19:00 KST (Windows 작업 스케줄러, 일/월 제외)
  │
  ▼
run_report2.bat (10단계 배치 실행)
  ├─ [1/10] node fetch_data.js            ← MariaDB에서 D-0(당일) 데이터만 조회
  ├─ [2/10] node analyze_sentiment.js     ← D-0 데이터 4단계 분류
  ├─ [3/10] node claude_enhance.js        ← D-0 Weak+미분류 AI 문맥분석, 할루시네이션 방지 검증
  ├─ [4/10] node update_keywords.js       ← AI 제안 키워드 자동 반영
  ├─ [5/10] node accumulate_results.js    ← ★ D-0 결과를 누적 파일에 병합 (30일 초과 자동 정리)
  ├─ [6/10] node build_html.js            ← 누적 데이터(최대 30일) 기반 HTML 생성 + 학생 등급
  ├─ [7/10] node generate_and_upload.js   ← GitHub 업로드 + report_links.csv
  ├─ [8/10] node update_sheets.js         ← Google Sheets 발송링크 컬럼 업데이트
  ├─ [9/10] node update_raw_sheet.js      ← D-0 원시 데이터 Google Sheets 업로드
  └─ [10/10] node trigger_webhook.js      ← n8n Webhook 호출 (발송 트리거)
        │
        ▼
n8n 워크플로우 (Webhook 트리거)
  ├─ 수신자 매칭 → 구글챗 DM 발송
  ├─ 사업부 이메일 발송
  └─ "발송 로그" 시트에 기록

[월요일 보강 배치] 매주 월요일 01:00 KST
  └→ 지난주 실패/미전송 건 재처리 + 키워드 보강 검토
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
| `run_report2.bat` | 메인 배치 프로그램 (10단계 자동 실행) |
| `fetch_data.js` | 1단계: MariaDB D-0 데이터 추출 → `data/counsel_data_d0.json` |
| `analyze_sentiment.js` | 2단계: D-0 데이터 4단계 분류 → `data/sentiment_d0.json` + AI 후보 파일 |
| `claude_enhance.js` | 3단계: D-0 Weak+미분류 AI 문맥분석 (할루시네이션 방지 검증) |
| `update_keywords.js` | 4단계: AI 제안 키워드를 `keywords.json`에 자동 반영 |
| `accumulate_results.js` | **5단계: D-0 결과를 누적 파일에 병합 + 30일 초과 정리** |
| `build_html.js` | 6단계: 누적 데이터 기반 HTML 생성 + 학생 등급 부여 |
| `generate_and_upload.js` | 7단계: GitHub 업로드 + report_links.csv |
| `update_sheets.js` | 8단계: Google Sheets 발송링크 업데이트 |
| `update_raw_sheet.js` | 9단계: D-0 원시 데이터 Google Sheets 업로드 |
| `trigger_webhook.js` | 10단계: n8n Webhook 호출 (발송 트리거) |
| `lib/utils.js` | 공통 유틸리티 (log, normalizeText, generateTextHash) |
| `config/filter_rules.json` | 전처리 필터 설정 (최소 텍스트 길이, Admin rescue 설정) |
| `keywords.json` | 부정 감성 키워드 사전 (**3단계 구조: Admin 22개 / Strong 4카테고리 42개 / Weak 11카테고리 180+개**) |
| `credentials.json` | Google 서비스 계정 인증 키 (`.gitignore`에 추가 필요) |
| `sample_report.html` | HTML 리포트 디자인 템플릿 |
| `.env` | 환경변수 (DB/GitHub 접속 정보) |
| `setup_scheduler.bat` | Windows 작업 스케줄러 등록 (관리자 권한 실행) |

### 데이터 파일

| 파일 | 용도 |
|------|------|
| `data/counsel_data_d0.json` | 1단계 출력: D-0 DB 추출 데이터 (각 row에 `source_pk` 포함) |
| `data/sentiment_d0.json` | 2~3단계 출력: D-0 감성분석 결과 |
| `data/ai_candidates.jsonl` | 2단계 출력: D-0 AI 재분석 후보 (Weak + 미분류) |
| `data/ai_candidates.meta.json` | 2단계 출력: 후보 생성 메타데이터 |
| **`data/sentiment_accumulated.json`** | **5단계 출력: 최대 30일 누적 분석 결과 (핵심 파일)** |
| `data/sentiment_previous.json` | 이전 실행 결과 백업 (NEW 뱃지 비교용) |
| `reports/` | 생성된 HTML 리포트 로컬 보관 폴더 |
| `logs/` | 실행 로그 저장 폴더 (KST 기준 날짜) |

**사용 서비스**: MariaDB (데이터 소스), **Claude Code CLI** (AI 감성분석, 구독형), GitHub (HTML 호스팅), Google Sheets (트리거/로깅), Google Chat (DM 발송), Gmail (이메일 발송), n8n (발송 워크플로우), Windows 작업 스케줄러 (스케줄링)

---

## 5. 핵심 컨셉: D-0 분석 + 30일 누적

### 5-1. 기본 원리

```
┌─────────────────────────────────────────────────────────────┐
│  기존 (리포트1): 매일 DB에서 30일치 전체를 다시 가져옴      │
│                                                             │
│  Day 1: DB → [D-30 ~ D-0] 11,000건 조회 → 분석 → 리포트   │
│  Day 2: DB → [D-30 ~ D-0] 11,000건 조회 → 분석 → 리포트   │
│  Day 3: DB → [D-30 ~ D-0] 11,000건 조회 → 분석 → 리포트   │
│  ※ 매일 11,000건 조회, 대부분은 어제와 동일한 데이터       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  리포트2 (본 시스템): 당일 데이터만 가져와서 누적           │
│                                                             │
│  Day 1: DB → [D-0] ~400건 조회 → 분석 → 누적 파일에 추가   │
│  Day 2: DB → [D-0] ~400건 조회 → 분석 → 누적 파일에 추가   │
│  Day 3: DB → [D-0] ~400건 조회 → 분석 → 누적 파일에 추가   │
│  ...                                                        │
│  Day 30: 누적 파일에 30일치 ~12,000건 보유 → 리포트 생성    │
│  Day 31: D-1(가장 오래된 날) 자동 정리 → 항상 최대 30일 유지│
│  ※ 매일 ~400건만 조회, DB 부하 대폭 감소                   │
└─────────────────────────────────────────────────────────────┘
```

### 5-2. 누적 파일 구조 (`data/sentiment_accumulated.json`)

```json
{
  "meta": {
    "last_updated": "2026-03-28T19:15:00+09:00",
    "date_range": {
      "oldest": "2026-02-27",
      "newest": "2026-03-28"
    },
    "total_records": 11842,
    "days_accumulated": 30
  },
  "daily_entries": {
    "2026-03-28": {
      "fetched_at": "2026-03-28T19:01:23+09:00",
      "record_count": 412,
      "centers": {
        "AAA": {
          "total_count": 85,
          "negative_count": 12,
          "students": {
            "STD001": {
              "std_nm": "김학생",
              "negative_count": 3,
              "categories": ["스트레스/압박감", "학습 어려움"],
              "keywords": ["힘들", "어렵"]
            }
          }
        }
      }
    },
    "2026-03-27": { ... },
    "2026-02-27": { ... }
  }
}
```

### 5-3. 누적 동작 흐름

```
매일 Step 5 (accumulate_results.js) 실행 시:

  1. sentiment_accumulated.json 로드 (없으면 빈 구조 생성)
  2. sentiment_d0.json (오늘 분석 결과) 로드
  3. 오늘 날짜 키로 daily_entries에 추가
     - 동일 날짜 키가 이미 있으면 → 덮어쓰기 (재실행 대응)
  4. 30일 초과 데이터 정리
     - daily_entries의 날짜 키를 정렬
     - 오늘 기준 30일 이전 키 → 삭제
  5. meta 정보 갱신 (last_updated, date_range, total_records, days_accumulated)
  6. sentiment_accumulated.json 저장
```

### 5-4. 누적 방식의 장점과 한계

#### 장점

| 항목 | 효과 |
|------|------|
| **DB 부하 감소** | 매일 ~400건만 조회 (기존 ~11,000건 → 96% 감소) |
| **AI 처리량 동일** | D-0 전량 분석이므로 AI 품질 유지 |
| **실행 시간 단축** | DB 조회 시간 대폭 감소, 전체 배치 시간 단축 |
| **네트워크 부하 감소** | DB 전송량 감소 |
| **캐시 불필요** | D-0만 처리하므로 AI 캐시 로직 제거 가능 (단순화) |

#### 한계 및 대응

| 한계 | 설명 | 대응 방안 |
|------|------|-----------|
| **분석 시점 확정** | D-0 분석 결과는 확정값으로 누적. DB에서 과거 상담이 수정되어도 이미 누적된 결과는 변경하지 않음 (설계 의도) | 해당 없음 |
| **초기 30일 미달** | 시스템 최초 가동 시 누적 데이터 부족 | 초기 1회 30일치 일괄 투입 (bootstrap) |
| **누적 파일 손상** | 파일 손상/삭제 시 30일치 데이터 손실 | 일별 백업 + bootstrap 재실행 |
| **일요일 공백** | 일요일 미실행 → 해당일 데이터 누락 | 월요일 보강 배치에서 일요일분 포함 처리 |

---

## 6. 데이터 흐름도

```
MariaDB (db_square, D-0 당일 상담만)
│
▼
[Step 1] fetch_data.js ← D-0(당일) 데이터만 조회
│
▼
counsel_data_d0.json (~400건/일)
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
├─ sentiment_d0.json (Strong만 반영)      │
│                                         │
├─ ai_candidates.jsonl (Weak + 미분류)    │
│                                         │
▼                                         │
[Step 3] claude_enhance.js ───────────────┘
  │
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
  │  │ ③ AI 감지 학생 → student_names 보완        │
  │  └────────────────────────────────────────────┘
  │
  │  Weak: 문맥 검증으로 오탐 필터링
  │  미분류: 새로운 부정 감지
  │  → suggested_keywords 수집
│
▼
sentiment_d0.json (보강됨: Strong + AI 검증 Weak + AI 신규 감지)
│
▼
[Step 4] update_keywords.js
  (제안 키워드 → 'AI_보강_키워드' 카테고리에 추가)
│
▼
[Step 5] accumulate_results.js ★ 핵심 단계 ★
  │  ┌────────────────────────────────────────────┐
  │  │ sentiment_accumulated.json 로드             │
  │  │ + sentiment_d0.json 병합                    │
  │  │ + 30일 초과 데이터 자동 정리                │
  │  │ = 최대 30일 누적 데이터 유지                │
  │  └────────────────────────────────────────────┘
│
▼
sentiment_accumulated.json (최대 30일 누적)
│
▼
[Step 6] build_html.js + sentiment_previous.json (전일 비교 → NEW 뱃지)
  │  누적 데이터에서 학생별 부정건수 합산 vs 센터 평균 비교 → 등급 부여
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
├─▶ [Step 7] generate_and_upload.js → GitHub Pages + report_links.csv
│     │
│     ├─▶ [Step 8] update_sheets.js → Google Sheets 링크 갱신
│     │
│     └─▶ [Step 9] update_raw_sheet.js → Google Sheets D-0 원데이터 누적
│
└─▶ [Step 10] trigger_webhook.js → n8n 완료 알림
```

---

## 7. 환경 설정

### 7-1. 로컬 환경변수 (`.env`)

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

### 7-2. Windows 작업 스케줄러

#### 일일 배치 (화~토 19:00, 일/월 제외)

- **작업 이름**: `AI_Report2_Daily`
- **실행 시간**: 화~토 19:00 (KST), 일/월 미실행
- **실행 대상**: `run_report2.bat`

#### 월요일 보강 배치 (매주 월요일)

- **작업 이름**: `AI_Report2_Monday`
- **실행 시간**: 매주 월요일 01:00 (KST)
- **실행 대상**: `run_report2_monday.bat`
- **목적**: 지난주 실패/미전송 건 재처리 + 일요일분 D-0 데이터 추가 수집 + 키워드 보강 검토

#### 스케줄러 등록 스크립트 (`setup_scheduler.bat`)

```batch
@echo off
chcp 65001 >nul
echo [스케줄러 등록] AI 리포트2 자동화

REM === 일일 배치 (화~토 19:00 KST, 일/월 제외) ===
schtasks /create /tn "AI_Report2_Daily" /tr "\"%~dp0run_report2.bat\"" /sc weekly /d TUE,WED,THU,FRI,SAT /st 19:00 /f /rl highest
echo [완료] AI_Report2_Daily - 화~토 19:00 KST

REM === 월요일 보강 배치 (매주 월요일 01:00) ===
schtasks /create /tn "AI_Report2_Monday" /tr "\"%~dp0run_report2_monday.bat\"" /sc weekly /d MON /st 01:00 /f /rl highest
echo [완료] AI_Report2_Monday - 매주 월요일 01:00

echo.
echo [확인] 등록된 작업 목록:
schtasks /query /tn "AI_Report2_Daily" /fo list | findstr "작업 이름 다음 실행 시간 상태"
schtasks /query /tn "AI_Report2_Monday" /fo list | findstr "작업 이름 다음 실행 시간 상태"
pause
```

### 7-3. n8n 워크플로우 설정

1. n8n 워크플로우 **AI Report Automation v4** (발송 전용)가 Active 상태인지 확인
2. Webhook 노드의 Production URL 확인
3. `.env`의 `N8N_WEBHOOK_URL`에 해당 URL 입력

---

## 8. 업무 프로세스 (Workflow)

### 1단계: D-0 데이터 추출 (`fetch_data.js`)

- **데이터 조회**: MariaDB(db_square)에 접속하여 **D-0(당일) 상담 데이터만** 추출
  - 조회 테이블: `tb_counsel`, `tb_counsel_employ`, `tb_company`, `tb_employ`, `tb_student` (JOIN)
  - 조회 필드: `cns_cd`, `cmp_cd`, `cmp_nm`, `std_cd`, `std_nm`(학생명), `emp_cd`, `reg_dt`, `cns_st`, `cns_req_ct`
  - 필터 조건: `fc_cls_sn = '1'`, `brn_cls = 'BC0002'`, 제외 cmp_cd: `AFA, AKU, CAY, CCA`
  - **날짜 조건**: `reg_dt >= 오늘 00:00:00 AND reg_dt <= 오늘 23:59:59` (KST 기준)
- **source_pk 부여**: 각 row에 `source_pk = cns_cd`로 고유 식별자 부여
- **cmp_cd별 그룹핑**: 조회 결과를 cmp_cd(센터) 단위로 분류
- **저장**: `data/counsel_data_d0.json`

### 2단계: 4단계 분류 — Admin/Strong/Weak/미분류 (`analyze_sentiment.js`)

기존 리포트1과 동일한 분류 로직을 D-0 데이터에 적용한다.

#### 분류 흐름 (B안)

```
D-0 데이터 (~400건/일 기준)
  │
  ├─ [1] Admin 제외:      ~60건 (14.8%)   ← 행정/운영 기록
  ├─ [2] Strong 즉시집계:  ~30건 (7.4%)   ← 확정 부정, AI 불필요
  ├─ [3] Weak → AI 후보:  ~100건 (25.2%)  ← 문맥 검증 필요
  └─ [4] 미분류 → AI 후보: ~210건 (52.6%) ← 키워드 미매칭
```

#### 판정 상세

1. **Admin 제외**: 제목(cns_st)에 Admin 키워드(부재, 통화중, 설명회 등 22개) 매칭 + 본문(cns_req_ct) 10자 미만 → 제외. 단, 본문에 Strong 키워드가 있으면 rescue하여 Strong으로 복원
2. **Strong 즉시 집계**: 확정적 부정 표현(환불, 퇴소, 컴플레인, 자해 등 42개) → AI 불필요, 즉시 학생별 부정 건수 반영
3. **Weak → AI 후보**: 문맥 검증이 필요한 표현(힘들, 어렵, 속상, 결석 등 180+개) → AI가 문맥 분석하여 오탐 필터링
4. **미분류 → AI 후보**: 키워드 미매칭 건 → AI가 새로운 부정 감지

#### 저장

- `data/sentiment_d0.json` (D-0 Strong만 반영, AI 보강 전)
- `data/ai_candidates.jsonl` (D-0 Weak + 미분류)

### 3단계: AI 문맥분석 — D-0 Weak+미분류 대상 (`claude_enhance.js`)

#### 목적

- **Weak**: 키워드 매칭됐지만 문맥상 긍정/중립인 건을 AI가 걸러냄
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

#### 리포트2에서의 캐시 정책

D-0 데이터만 처리하므로 **AI 캐시(`ai_review_cache.json`)는 사용하지 않는다**. 매일 당일 데이터 전량을 분석하며, 과거 데이터를 재분석할 일이 없으므로 캐시 히트/미스 판단이 불필요하다.

단, 월요일 보강 배치에서 실패 건 재처리 시에는 해당 일자의 데이터를 다시 가져와 분석한다.

#### 할루시네이션 방지 검증

| 장치 | 설명 |
|------|------|
| **source_pk 검증** | AI 반환 source_pk가 입력 데이터에 없으면 제거 |
| **카테고리 검증** | 허용된 12개 카테고리 외 항목 자동 필터링 |
| **키워드 포맷 검증** | AI 제안 키워드의 타입/공백/길이/중복 검증 후 반영 |
| **학생 이름 보완** | AI로만 감지된 학생도 std_nm을 student_names에 반영 |

#### 기타 보호 장치

| 장치 | 설명 |
|------|------|
| Rate limit 감지 | 한도 도달 시 즉시 중단, 월요일 보강 배치에서 재처리 |
| 2회 재시도 | 5초 간격, 일시적 오류 대응 |
| Lock 파일 | PID 기반 중복 실행 방지 (60분 타임아웃) |
| 상태 관리 | PENDING → PROCESSING → DONE/FAILED |

### 핵심: Step 2↔3↔4 피드백 루프

```
Day 1:
    Step 2: D-0 데이터 Admin/Strong/Weak 분류 → Weak+미분류 ~310건 AI 전송
    Step 3: Claude가 문맥분석 → Weak 오탐 필터링 + 미분류 부정 감지 + 키워드 15개 제안
    Step 4: 15개 중 신규 10개 → keywords.json 'AI_보강_키워드'에 추가

Day 2:
    Step 2: 보강된 키워드로 미분류 감소 → AI 전송량 감소
    Step 3: 신규 D-0 데이터만 분석

Day N:
    Step 2: 충분히 보강된 키워드로 대부분 감지 → 미분류 극소
    Step 3: Claude 처리 시간 대폭 단축
```

### 4단계: 키워드 반영 (`update_keywords.js`)

- Step 3의 제안 키워드를 `keywords.json`의 Weak `AI_보강_키워드` 카테고리에 자동 추가

### 5단계: 누적 병합 (`accumulate_results.js`) ★ 핵심 ★

#### 처리 흐름

```
1. sentiment_accumulated.json 로드
   └─ 파일 없으면 → 빈 구조 { meta: {}, daily_entries: {} } 생성

2. sentiment_d0.json (오늘 분석 결과) 로드

3. daily_entries에 오늘 날짜 키로 추가
   └─ 동일 날짜 키가 이미 있으면 → 덮어쓰기 (당일 재실행 대응)

4. 30일 초과 데이터 정리
   └─ daily_entries의 날짜 키를 정렬 → 오늘 기준 30일 이전 키 삭제

5. meta 정보 갱신
   └─ last_updated, date_range, total_records, days_accumulated

6. sentiment_accumulated.json 저장

7. sentiment_previous.json 갱신 (NEW 뱃지 비교용)
```

#### 보호 장치

| 장치 | 설명 |
|------|------|
| **원자적 쓰기** | 임시 파일에 먼저 쓰고 rename (쓰기 중 비정상 종료 대응) |
| **일별 백업** | 병합 전 `data/backup/accumulated_YYYY-MM-DD.json`에 백업 |
| **중복 병합 방지** | 동일 날짜 키 존재 시 덮어쓰기 (멱등성 보장) |
| **무결성 검증** | 로드 시 JSON 파싱 실패 → 백업에서 복원 시도 |

### 6단계: HTML 리포트 생성 (`build_html.js`)

- **데이터 소스**: `data/sentiment_accumulated.json` (누적 최대 30일)
- **집계 방식**: `daily_entries`의 전체 날짜를 순회하며 센터별/학생별 부정 건수 합산
- **디자인 기준**: `sample_report.html`의 CSS/HTML 구조를 그대로 사용 (max-width: 480px 모바일 레이아웃)
- **HTML 구성 요소**:
  1. **topbar**: 센터명 + 리포트 기간 (누적 시작일 ~ 최신일)
  2. **기간 요약 카드**: 누적 기간 / D-0 신규 건수 비교 테이블
  3. **부정 감성 키워드 빈도**: 상위 8개, 이모지 포함, 바차트. 빈도 합산 = 부정 감성 건수
  4. **위험 학생 알림 배너** (alertBanner): 등급이 "위험"인 학생 수만 카운트하여 표시
  5. **평균 초과 학생 리스트**: data-table
     - 컬럼: **학생명** / 학생코드 / 부정 건수 / 초과율(%) / 등급 / 신규
     - 위험등급: 평균×2 초과 = 위험, 평균×1.5 초과 = 주의, 그 외 = 관심
     - **NEW 태그**: 이전 실행(`data/sentiment_previous.json`)과 비교하여 신규 평균초과 학생에 표시
  6. **푸터**: 생성일시 (KST) + 누적 일수 표시
- **로컬 저장**: `reports/{cmp_cd}.html`

### 7단계: GitHub 업로드 (`generate_and_upload.js`)

- **GitHub 업로드**: `counsel-report/{cmp_cd}.html` 경로에 업로드 (덮어쓰기 — sha 조회 후 갱신)
- **저장소**: `jichang999-dotcom/tes1` (main 브랜치)
- **링크 생성**: `https://jichang999-dotcom.github.io/tes1/counsel-report/{cmp_cd}.html`
- **링크 목록 저장**: `report_links.csv`에 전체 결과 기록

### 8단계: Google Sheets 업데이트 (`update_sheets.js`)

- Google 서비스 계정으로 "리포트 제작" 시트에서 **PK = "A1"인 행**만 필터링하여 cmp_cd 매칭 행의 발송링크(E열) 업데이트

### 9단계: 원시 데이터 업로드 (`update_raw_sheet.js`)

- D-0 상담 원데이터 + 키워드 매칭 결과를 Google Sheets "키워드raw" 시트에 누적

### 10단계: n8n Webhook 호출 (`trigger_webhook.js`)

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
4. **Create a message** — 구글챗 DM 발송
5. **Log Chat Send** — "발송 로그" 시트에 발송일시(KST), 구글챗ID, 발송내용 기록
6. **Read Business Units** — "사업부" 시트에서 이메일 주소 읽기
7. **Prepare BU Email** — 센터별 리포트 링크 목록 + 발송일시로 메일 본문 생성
8. **Send BU Email** — Gmail API로 사업부 담당자에게 발송

---

## 9. 키워드 사전 (keywords.json) — 3단계 구조

기존 리포트1과 동일한 키워드 사전을 공유한다.

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

## 10. 필터 조건

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

## 11. 운영 모드

### 일일 배치 (화~토 19:00 KST, 일/월 제외)

| 항목 | 내용 |
|------|------|
| DB 조회 대상 | **D-0 (당일) 데이터만** |
| 일평균 DB 조회량 | **~400건** (기존 ~11,000건 대비 96% 감소) |
| Admin 제외 | ~60건 자동 제외 |
| Strong | ~30건 즉시 부정 집계 |
| Weak + 미분류 | ~310건 AI 전송 → 문맥 분석 후 집계 |
| 누적 병합 | D-0 결과를 누적 파일에 추가, 30일 초과 데이터 정리 |
| 리포트 생성 | 누적 데이터(최대 30일) 기반 |

### 월요일 보강 배치

| 항목 | 내용 |
|------|------|
| 대상 | 지난주 실패/미전송 건 + **일요일분 D-0 데이터** |
| 정상 운영 시 | 100~300건 |
| 추가 작업 | suggested_keywords 기반 키워드 보강 검토/자동 반영 |

### 초기 투입 (Bootstrap)

시스템 최초 가동 또는 누적 파일 손실 시, 30일치 데이터를 일괄 투입한다.

```
node bootstrap_accumulate.js
  → MariaDB에서 D-30 ~ D-0 전체 조회
  → 일자별로 분리하여 각각 분류 + AI 분석
  → sentiment_accumulated.json에 30일치 일괄 기록
  → 이후 일일 배치로 정상 운영 전환
```

---

## 12. 분석 대상 센터 (13개)

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

## 13. 에러 처리 및 로깅

### 에러 처리 전략

- **Step 1, 2, 5, 6, 7** 실패 시 → **배치 중단** (핵심: 데이터 추출, 분류, 누적, 리포트 생성, 업로드)
- **Step 3, 4, 8, 9, 10** 실패 시 → **경고 후 계속** (보조 기능)

Step 3이 실패해도 Step 2의 Strong 키워드 분석 결과만으로 D-0 데이터가 누적되고 리포트가 정상 생성됩니다.

**Step 5 (누적 병합) 실패 시**: 가장 치명적. 원자적 쓰기 + 백업으로 데이터 손실 방지.

### Step 3 Rate Limit 대응

- Rate limit 감지 시 남은 호출 즉시 중단
- 성공 건까지의 결과만 D-0 결과에 반영
- 미처리 건은 월요일 보강 배치에서 재처리

### 누적 파일 장애 대응

| 장애 유형 | 대응 |
|-----------|------|
| JSON 파싱 실패 | 직전 일별 백업에서 복원 |
| 파일 삭제/손실 | bootstrap 재실행 (30일치 일괄 투입) |
| 디스크 용량 부족 | 30일 초과 백업 자동 정리 (최근 7일 백업만 유지) |

### n8n 워크플로우 에러 처리

- 구글챗 발송 실패: 개별 실패가 전체 워크플로우를 중단시키지 않음
- 사업부 메일 발송 실패: n8n 에러 핸들러 처리
- 장애 발생 보고: 담당자(`jichang999@askwhy.co.kr`)에게 에러 알림 메일 발송

### 실행 로그

- `logs/run_YYYY-MM-DD.log` 파일에 단계별 진행 상황 기록 (KST 기준 타임스탬프)

---

## 14. 예상 소요 시간

| Step | 소요 시간 | 비고 |
|------|----------|------|
| 1 (DB D-0 추출) | **< 30초** | D-0만 조회 → 기존 1~3분에서 대폭 단축 |
| 2 (4단계 분류) | < 30초 | D-0 ~400건 대상 |
| 3 (Claude AI) | 5~15분 → 점차 단축 | D-0 Weak+미분류만 분석 + 피드백 루프 효과 |
| 4 (키워드 갱신) | < 1분 | |
| 5 (누적 병합) | **< 10초** | JSON 병합 + 30일 정리 |
| 6 (HTML 생성) | 1~2분 | 누적 데이터 기반 |
| 7 (GitHub 업로드) | 2~5분 | |
| 8 (Sheets 링크) | 1~2분 | |
| 9 (Raw 데이터) | 1~2분 | D-0 데이터만 업로드 |
| 10 (웹훅 알림) | < 1분 | |
| **합계** | **12~30분 → 점차 단축** | 기존 20~45분 대비 단축 |

---

## 15. Claude Code 사용량 및 비용

### Claude Code는 구독형 (API 과금 아님)

| 플랜 | 월 비용 | 사용량 | 비고 |
|------|---------|--------|------|
| **Pro** | $20/월 | 기본 사용량 | D-0 ~310건/일 처리 시 Rate limit 가능성 |
| **Max (5x)** | $100/월 | Pro 대비 5배 | 권장 플랜 |
| **Max (20x)** | $200/월 | Pro 대비 20배 | 여유 있는 운영 |

### 리포트1 대비 AI 사용량 비교

| 항목 | 리포트1 | **리포트2** |
|------|---------|------------|
| AI 분석 대상 | D-0~D-1 캐시 미스 건 | **D-0 전량** |
| 일평균 AI 호출 | ~228건 | **~310건 (D-0 Weak+미분류 전량)** |
| 캐시 효과 | 있음 (기분석 건 스킵) | **없음 (매일 D-0 전량 분석)** |
| AI 품질 | 동일 | 동일 |

> 리포트2는 캐시를 사용하지 않으므로 일평균 AI 호출량이 다소 증가할 수 있으나, DB 부하 감소와 실행 시간 단축으로 상쇄됩니다.

---

## 16. 외부 연동

| 서비스 | 용도 | 접속 정보 |
|--------|------|----------|
| MariaDB | 상담 데이터 원본 (D-0만 조회) | db_square (61.74.65.40:3306) |
| Claude CLI | AI 문맥 감성 분석 | claude -p (stdin 기반) |
| GitHub Pages | HTML 리포트 배포 | jichang999-dotcom.github.io/tes1/counsel-report/ |
| Google Sheets | 리포트 링크 + 원데이터 | 서비스 계정 (credentials.json) |
| n8n Webhook | 완료 알림 | ctdxteam1.app.n8n.cloud |

---

## 17. 운영 가이드

### 수동 실행

```bash
cd C:\Users\User\Desktop\test
run_report2.bat
```

월요일 보강 배치 수동 실행:
```bash
run_report2_monday.bat
```

초기 투입 (30일치 일괄):
```bash
node bootstrap_accumulate.js
```

개별 단계 실행:
```bash
node fetch_data.js
node analyze_sentiment.js
node claude_enhance.js
node update_keywords.js
node accumulate_results.js
node build_html.js
node generate_and_upload.js
node update_sheets.js
node update_raw_sheet.js
node trigger_webhook.js
```

### 스케줄러 상태 확인

```bash
schtasks /query /tn "AI_Report2_Daily" /v /fo list
schtasks /query /tn "AI_Report2_Monday" /v /fo list
```

### 주의사항

1. **PC 전원**: Windows 작업 스케줄러는 PC가 켜져 있어야 실행됩니다.
2. **네트워크**: MariaDB, GitHub API, n8n Cloud에 접근 가능한 환경 필요.
3. **환경변수**: `.env`와 `credentials.json` 파일은 절대 Git에 커밋하지 마세요.
4. **n8n 워크플로우**: AI Report Automation v4가 Active 상태여야 Webhook 호출이 정상 작동합니다.
5. **Claude Code**: `claude` CLI 명령이 사용 가능해야 합니다.
6. **누적 파일 관리**: `data/sentiment_accumulated.json`이 핵심 데이터. **절대 수동 삭제 금지**. 손실 시 `bootstrap_accumulate.js` 실행.
7. **백업 확인**: `data/backup/` 폴더에 일별 백업이 정상 생성되는지 주기적 확인.
8. **일요일 공백**: 일요일 미실행으로 해당일 데이터 누락. 월요일 보강 배치에서 자동 보충.
9. **분석 확정 원칙**: D-0 분석 결과는 확정값으로 누적됨. DB에서 과거 상담이 수정되더라도 이미 누적된 결과는 변경하지 않는 것이 본 시스템의 설계 의도.
10. **Lock 파일**: `data/.claude_enhance.lock` — 비정상 종료 시 60분 후 자동 해제, 수동 삭제 가능.

---

## 18. 기술 스택

| 항목 | 상세 |
|------|------|
| 런타임 | Node.js |
| DB 클라이언트 | mysql2 |
| AI | Claude Code CLI (stdin 파이프, 구독형) |
| Google API | googleapis |
| 환경변수 | dotenv |
| 자동화 | Windows 작업 스케줄러 + run_report2.bat |
| 리포트 배포 | GitHub Pages |
| 발송 | n8n Cloud (구글챗 DM + Gmail) |
| 알림 | n8n Webhook |
