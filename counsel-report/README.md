# [SOP] AI 리포트 제작 및 발송 자동화 시스템

## 1. 문서 개요

- **목적**: Google Sheets에 등록된 "발송 대상자"에게 주간 AI 분석 HTML 리포트 링크를 자동 발송하는 시스템.
- **실행 주기**: 매주 월요일 오전 09:00 발송 (데이터 생성 등 선행 작업은 08:30 시작).
- **데이터 범위**: 실행일 기준 최근 30일 데이터 (D-30 ~ D-1)

---

## 2. 관련 문서 및 서비스 링크

- 리포트 제작 관리 시트: [gid=1296826526](https://docs.google.com/spreadsheets/d/1lg75mmON79TwwzOwQCmt6l_3X8xxjj7Bze7Vz9DUcMY/edit?gid=1296826526#gid=1296826526)
- 발송 대상자 명단 시트: [gid=435721721](https://docs.google.com/spreadsheets/d/1lg75mmON79TwwzOwQCmt6l_3X8xxjj7Bze7Vz9DUcMY/edit?gid=435721721#gid=435721721)
- 발송 로그 기록 시트: [gid=899713549](https://docs.google.com/spreadsheets/d/1lg75mmON79TwwzOwQCmt6l_3X8xxjj7Bze7Vz9DUcMY/edit?gid=899713549#gid=899713549)
- n8n 워크플로우 - AI Report Generator: [OLqsHdJ3oLDj7JXy](https://ctdxteam1.app.n8n.cloud/workflow/OLqsHdJ3oLDj7JXy)
- n8n 워크플로우 - AI Report Sender: [f4WJsxwi5shTnKsn](https://ctdxteam1.app.n8n.cloud/workflow/f4WJsxwi5shTnKsn)
- n8n 워크플로우 - AI Report Error Alert: [Z8Ys4lheaQPtLacu](https://ctdxteam1.app.n8n.cloud/workflow/Z8Ys4lheaQPtLacu)
- GitHub 리포트 저장소: [jichang999-dotcom/tes1](https://github.com/jichang999-dotcom/tes1)
- GitHub Pages 리포트 호스팅: [counsel-report](https://jichang999-dotcom.github.io/tes1/counsel-report/)

**사용 서비스**: Google Sheets (트리거/로깅), Claude (AI HTML 업데이트), MariaDB (데이터 소스), Google Drive (HTML 저장), GitHub (HTML 호스팅), Gmail (이메일 발송), n8n (워크플로우 자동화)

---

## 4. 업무 프로세스 (Workflow)

### 1단계: 데이터 추출 및 HTML 생성 (선행 작업 - 월요일 08:30 실행)

- **데이터 조회**: n8n 워크플로우에서 MariaDB(db_square)에 접속하여 cmp_cd(회사/캠페인 코드)별로 월 데이터를 추출합니다.
  - 조회 범위: 실행일 기준 D-30 ~ D-1
  - 쿼리문은 n8n 워크플로우 `AI 리포트 생성`(ID: `OLqsHdJ3oLDj7JXy`)의 `MariaDB 데이터 추출` 노드에서 관리합니다. 쿼리 변경 시 해당 노드를 직접 수정하세요.
- **AI 감성 분석**: Claude API를 통해 `cns_emp_cd`(상담사 기록), `cns_req_ct`(상담 요청 내용)의 부정적 텍스트를 감성 분석합니다.
  - cmp_cd별로 부정 감성 건수를 1개월간 누적 카운트
  - cmp_cd 내 학생(`des_usr_act_cd`)별 평균 부정 건수를 산출하고, 평균 초과 학생을 식별
- **HTML 생성**: 분석 결과를 바탕으로 cmp_cd별 주간 리포트 HTML 파일을 각각 생성합니다. 리포트에 포함되는 항목:
  - 기간 요약 (조회 기간, 총 상담 건수, 부정 감성 건수, 부정 비율)
  - 부정 감성 키워드 빈도 통계
  - 평균 초과 학생 리스트 (학생코드, 부정 건수, 초과율, 위험등급)
  - 상담빈도 초과 리스트 (학생코드, 상담 횟수, 초과율)
  - 상담사별 현황 (상담사 코드, 총 상담, 부정 감성 비율)
  - 위험 학생 상세 내역 (부정 상담 일자, 상담사, 감성, 키워드)

### 2단계: GitHub 업로드 및 배포 링크 생성

- **업데이트 확인**: Google Sheets "리포트 제작" 시트의 Pk 컬림에 A1, 업데이트 컬럼이 Y인지 확인하여 업데이트를 시작 합니다.
- **GitHub 업로드**: n8n Code 노드에서 GitHub API를 호출하여 1단계에서 생성된 HTML 파일을 cmp_cd별로 Public GitHub Repository의 `counsel-report` 폴더에 업로드합니다.
  - 저장소: `jichang999-dotcom/tes1` (main 브랜치)
  - 업로드 경로: `counsel-report/{cmp_cd}.html` (예: counsel-report/CMP001.html)
  - 업로드 방식: **덮어쓰기** — 기존 파일이 있으면 sha 값을 조회하여 같은 파일명으로 최신 내용을 덮어씁니다. 매주 cmp_cd당 1개의 HTML 파일만 유지되며, 이전 주 리포트는 최신 리포트로 교체됩니다.
  - 인증: `GITHUB_TOKEN` 환경변수 사용
- **링크 생성 및 시트 기록**: 업로드 완료 후 `https://jichang999-dotcom.github.io/tes1/counsel-report/{cmp_cd}.html` 형태의 cmp_cd별 배포용 리포트 링크를 생성합니다. 생성된 링크는 "리포트 제작" 시트의 PK 컬럼에서 A1 값을 찾고, 해당 행들 중 cmp_cd가 일치하는 행의 HTML업데이트 컬럼에 링크를 기록합니다.

### 3단계: 발송 대상자 매핑

- **대상자 식별**: "리포트 제작" 시트와 "발송 대상자" 시트를 대조하여 PK와 cmp_cd가 모두 일치하는 수신자를 매칭합니다.
- **링크 매칭**: 매칭된 수신자에게 해당 cmp_cd의 배포 링크를 할당합니다.
- **시트 업데이트**: 할당된 링크를 "발송 대상자" 시트에 발송링크 컬럼에 링크주소를 업데이트 합니다.
- **수신자 정보 확보**: "발송 대상자" 시트에서 최종 발송할 이메일 주소를 추출합니다.

### 4단계: 이메일 발송 처리 (월요일 09:00 실행)

- **발송 조건 확인**: Google Sheets "발송 대상자" 시트의 발송여부 컬럼이 Y인지 확인하여 트리거를 시작합니다.
- **대상자 발송**: 최종 확인된 대상자들에게 매핑된 리포트 링크가 포함된 이메일을 발송합니다.

---

## 5. 에러 처리 및 로깅 (Error Handling & Logging)

### 에러 처리 및 알림

- **재시도 로직**: AI 노드(데이터 추출/HTML 생성 등) 실패 시 최대 3회까지 재시도합니다.
- **장애 발생 보고**: 3회 재시도 후에도 최종 실패할 경우, 담당자(jichang999@askwhy.co.kr)에게 즉시 에러 알림 메일을 발송합니다.

### 발송 로깅

- **기록 위치**: 모든 발송 내역 및 문의 사항은 "발송 로그" Google Sheets에 기록됩니다.
- **기록 항목**: 발송일시, 이름, 메일주소, PK, 발송여부 (성공/실패 상태).
