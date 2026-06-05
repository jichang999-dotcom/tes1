# 사내 데이터 질의응답 AI 구축 기획안  
## 대시보드 쿼리 매칭형에서 판다형 AI 데이터 Assistant로 고도화하는 방향

- 작성일: 2026-06-02
- 작성 목적: 토스플레이스 데이터봇 `판다(PANDA)` 사례를 참고하여, 사내 데이터 질의응답 AI를 단계적으로 구축하기 위한 개발 방향 정리
- 참고 사례: 토스테크 아티클 「토스플레이스 데이터봇 ‘판다(PANDA)’를 소개합니다 : 모든 팀원이 데이터 전문가처럼 일하는 방법」
- 참고 URL: https://toss.tech/article/da-assistant-panda

---

# 1. 기획 배경

사내에서는 운영팀, 결제팀, 평가팀, CS팀, 영업팀 등 여러 사업부가 각자의 대시보드를 사용하고 있다.

하지만 실제 업무에서는 다음과 같은 요청이 반복된다.

```text
이번 달 미납 학생 수 알려줘
센터별 정기결제 가입자 수 알려줘
지난달 대비 신규생이 줄어든 센터 알려줘
무응시 학생이 많은 센터 알려줘
알림톡 실패가 많은 센터 알려줘
```

현재는 이런 요청을 처리하기 위해 담당자가 대시보드를 직접 확인하거나, SQL을 작성하거나, 데이터 담당자에게 요청해야 한다.

이 문제를 해결하기 위해 사내 데이터 질의응답 AI를 구축한다.

목표는 다음과 같다.

```text
사용자가 자연어로 질문
→ AI가 질문 의도와 지표를 해석
→ 검증된 쿼리 또는 표준 마트를 이용해 조회
→ 결과, 조회 기준, 인사이트를 함께 제공
→ 답변할 수 없는 요청은 관리자 검토 큐로 전달
```

---

# 2. 토스 판다(PANDA) 사례 핵심 요약

토스플레이스의 판다(PANDA)는 사내 구성원이 자연어로 데이터를 질문하면, 보안 등급에 따라 허용된 범위 안에서 데이터를 조회하고 분석 결과를 제공하는 데이터 분석 Assistant이다.

## 2.1 판다의 핵심 문제의식

토스플레이스 사례에서 확인할 수 있는 핵심 문제는 다음과 같다.

| 문제 | 설명 |
|---|---|
| 단순 데이터 요청 증가 | 전체 데이터 요청 중 상당수가 복잡한 분석이 아니라 단순 수치 확인 요청 |
| 데이터 분석가 리소스 소모 | 데이터 분석가가 반복적인 조회 요청을 처리하느라 심층 분석에 집중하기 어려움 |
| 대시보드 탐색 불편 | 사용자가 어떤 대시보드에서 어떤 지표를 봐야 하는지 알기 어려움 |
| AI의 잘못된 테이블 선택 | AI가 수많은 테이블 중 매번 다른 테이블을 선택하면 결과가 달라짐 |
| 비즈니스 기준 오류 | “활성”, “미납”, “신규” 같은 용어의 업무 기준을 모르면 결과가 틀림 |
| DW 비용 증가 | 비효율적인 쿼리 실행으로 데이터 웨어하우스 비용 증가 가능 |

---

## 2.2 판다의 해결 방식

판다는 단순히 LLM을 붙인 챗봇이 아니라, 데이터와 답변 구조를 함께 정비한 시스템으로 볼 수 있다.

핵심 구성은 다음과 같다.

| 구성 | 설명 |
|---|---|
| 표준 데이터 마트, SSOT | AI가 믿고 참고할 수 있는 기준 데이터 정비 |
| 테이블/컬럼 Description | AI가 테이블과 컬럼 의미를 이해하도록 설명 정비 |
| 비즈니스 용어 사전 | “활성 매장”, “업종 분류” 같은 업무 용어를 데이터 기준과 연결 |
| Scoring & Ranking | 질문과 가장 관련 있는 테이블을 일관되게 선택하는 점수 체계 |
| Agentic Loop | 테이블 탐색, SQL 생성, 실행, 결과 검증, 재시도를 반복 |
| 답변 구조화 | 결과, 조회 기준, 인사이트를 함께 제공 |

---

# 3. 사용자 질문을 1개의 메타 테이블로 모두 커버하는가?

결론부터 말하면, 사용자 질문 전체를 **1개의 메타 테이블 하나로만 커버한다**고 보기는 어렵다.

보다 정확한 구조는 다음과 같다.

```text
사용자 질문
→ 비즈니스 용어/지표 정의 확인
→ 관리 대상 데이터 마트 후보 탐색
→ 테이블/컬럼 Description 기반 Scoring & Ranking
→ SQL 생성
→ 실행/검증/재시도
→ 결과 + 조회 기준 + 인사이트 답변
```

즉, 메타 테이블은 답 데이터를 저장하는 곳이 아니라, 질문을 올바른 데이터와 지표로 연결하는 **지도** 역할을 한다.

---

## 3.1 메타 테이블의 역할

예를 들어 사용자가 이렇게 질문한다.

```text
이번 달 미납 학생 수 알려줘
```

AI는 메타 정보에서 다음 내용을 찾아야 한다.

| 구분 | 메타에서 찾는 내용 |
|---|---|
| 도메인 | 결제 / 청구 / 정기결제 |
| 지표명 | 미납 학생 수 |
| 기준 테이블 | `mart_payment_unpaid_student` |
| 기준 컬럼 | `payment_ym`, `std_cd`, `unpaid_amount`, `order_state` |
| 미납 기준 | `unpaid_amount > 0` |
| 제외 조건 | 취소, 환불 완료, 청구 취소 제외 |
| 권한 | 본사는 전체, 센터는 자기 센터만 |
| 기본 기간 | 현재 청구월 |

---

# 4. 제안하는 개발 방향

처음부터 판다형 AI Agent를 만드는 것은 위험하다.

사용자 회사의 업무 데이터는 정기결제, 미납, 학생, 평가, 알림톡, 센터, 수강 상태 등 업무 기준이 복잡하다.

따라서 아래처럼 단계적으로 개발하는 것이 좋다.

```text
1단계: 대시보드 쿼리 매칭형
   ↓
2단계: 쿼리 템플릿 + 지표/용어 사전 기반 확장형
   ↓
3단계: 표준 마트/VIEW 정비
   ↓
4단계: 제한적 AI SQL 생성형
   ↓
5단계: 판다형 Agent
```

---

# 5. 1단계: 대시보드 쿼리 매칭형

## 5.1 개념

사업부별로 이미 사용 중인 대시보드의 검증된 쿼리 또는 지표를 등록하고, AI가 사용자의 질문을 해당 쿼리 템플릿에 매칭하여 답변하는 방식이다.

```text
사용자 질문
→ 질문 의도 분석
→ 등록된 지표/쿼리 템플릿 검색
→ 파라미터 적용
→ SELECT 실행
→ 결과 답변
```

---

## 5.2 AI의 역할

이 단계에서 AI는 SQL을 직접 만들지 않는다.

AI의 역할은 다음과 같다.

| 역할 | 설명 |
|---|---|
| 질문 분류 | 결제, 학생, 평가, 알림톡 등 도메인 판단 |
| 지표 매칭 | 미납 금액, 신규생 수, 응시율 등 선택 |
| 파라미터 추출 | 이번 달, 지난달, 특정 센터 등 추출 |
| 쿼리 템플릿 선택 | 등록된 검증 쿼리 중 선택 |
| 답변 생성 | 조회 결과를 사람이 이해하기 쉽게 설명 |
| 답변 불가 판단 | 매칭되는 쿼리가 없으면 관리자 큐로 보냄 |

---

## 5.3 예시

사용자 질문:

```text
이번 달 센터별 미납 금액 알려줘
```

AI 판단:

| 항목 | 판단 |
|---|---|
| 도메인 | 결제 / 정기결제 |
| 지표 | 미납 금액 |
| 집계 기준 | 센터별 |
| 기간 | 이번 달 |
| 사용할 쿼리 | `query_template_id = 12` |
| 파라미터 | `payment_ym = 202606` |

쿼리 템플릿 예시:

```sql
SELECT
    cmp_cd,
    cmp_nm,
    SUM(unpaid_amount) AS total_unpaid_amount
FROM mart_payment_unpaid_student
WHERE payment_ym = #{paymentYm}
  AND unpaid_amount > 0
  AND cancel_yn = 'N'
  AND refund_yn = 'N'
GROUP BY cmp_cd, cmp_nm
ORDER BY total_unpaid_amount DESC;
```

---

## 5.4 장점

| 장점 | 설명 |
|---|---|
| 구축이 빠름 | 기존 대시보드 쿼리를 활용 가능 |
| 안정성이 높음 | 사람이 검증한 쿼리만 사용 |
| 보안 통제가 쉬움 | 허용된 쿼리만 실행 |
| 내부 설득이 쉬움 | AI가 임의로 DB를 조회하지 않음 |
| 운영 개선 자료 확보 | 답변 불가 질문이 신규 지표 후보가 됨 |

---

# 6. 2단계: 지표/용어 사전 기반 확장형

## 6.1 개념

사용자가 정확한 지표명을 말하지 않아도 AI가 사용자 표현을 내부 업무 지표로 변환하는 단계이다.

예를 들어 다음과 같이 해석한다.

| 사용자 표현 | 내부 지표 |
|---|---|
| 결제 안 된 학생 | 미납 학생 |
| 자동결제 가입자 | 정기결제 가입자 |
| 시험 안 본 학생 | 무응시 학생 |
| 새로 들어온 학생 | 신규생 |
| 다시 온 학생 | 복귀생 |

---

## 6.2 AI의 역할

| 역할 | 설명 |
|---|---|
| 유사 표현 해석 | “결제 안 된 학생” → “미납 학생” |
| 업무 용어 변환 | “다시 온 학생” → “복귀생” |
| 지표 기준 확인 | 미납 기준, 신규생 기준, 응시율 기준 확인 |
| 질문 보정 | 애매한 질문을 내부 지표 기준으로 변환 |
| 유사 질문 학습 | 기존 질문 로그를 참고해 매칭 정확도 향상 |
| 관리자 개선 자료 생성 | 답변 불가 질문을 지표 후보로 정리 |

---

## 6.3 예시

사용자 질문:

```text
결제 안 된 학생이 많은 센터 알려줘
```

AI 해석:

```text
결제 안 된 학생
= 미납 학생
= unpaid_amount > 0
= COUNT(DISTINCT std_cd)
```

이후 1단계의 쿼리 템플릿 또는 3단계의 표준 마트를 사용해 조회한다.

---

# 7. 3단계: 표준 마트/VIEW 기반 단계

## 7.1 개념

AI가 복잡한 원천 운영 테이블을 직접 JOIN하지 않도록, 조회 전용 표준 마트 또는 VIEW를 만든다.

AI는 이 표준 마트 안에서만 데이터를 조회하도록 제한한다.

---

## 7.2 추천 표준 마트

| 마트명 | 목적 |
|---|---|
| `mart_payment_recurring_status` | 정기결제 가입, 해지, 결제 상태 요약 |
| `mart_payment_unpaid_student` | 학생별 미납 현황 |
| `mart_student_monthly_status` | 월별 학생 재원, 신규, 퇴원, 복귀 상태 |
| `mart_test_result_summary` | 평가 응시, 무응시, 평균 점수 요약 |
| `mart_message_send_result` | 알림톡/문자 발송 성공, 실패 현황 |

---

## 7.3 AI의 역할

| 역할 | 설명 |
|---|---|
| 적합한 마트 선택 | 질문에 맞는 조회용 VIEW/마트 선택 |
| 컬럼 의미 해석 | `payment_ym`, `cmp_cd`, `std_cd` 의미 파악 |
| 지표-테이블 연결 | 미납 금액 → `mart_payment_unpaid_student` |
| 권한 범위 확인 | 본사/센터/사업부 권한에 따라 조회 범위 제한 |
| 원천 테이블 접근 제한 | 복잡한 운영 테이블 직접 조회 방지 |
| 데이터 기준 설명 | 어떤 마트 기준으로 조회했는지 답변에 표시 |

---

# 8. 4단계: 제한적 AI SQL 생성형

## 8.1 개념

이 단계부터 AI가 SELECT 문을 직접 생성한다.

단, 모든 테이블을 자유롭게 조회하는 것이 아니라 승인된 마트/VIEW 안에서만 SQL을 생성한다.

---

## 8.2 허용/금지 기준

| 구분 | 내용 |
|---|---|
| 허용 | 승인된 표준 마트/VIEW |
| 허용 | SELECT 문 |
| 허용 | 승인된 컬럼 |
| 허용 | 사용자 권한 범위 내 조회 |
| 금지 | INSERT, UPDATE, DELETE, DROP, ALTER |
| 금지 | 원천 운영 테이블 무제한 접근 |
| 금지 | 개인정보 컬럼 직접 조회 |
| 금지 | 기간 조건 없는 대량 조회 |
| 금지 | 권한 없는 센터 데이터 조회 |

---

## 8.3 AI의 역할

| 역할 | 설명 |
|---|---|
| SELECT 생성 | 승인된 테이블 기준으로 SQL 작성 |
| 조건 자동 적용 | 기간, 센터, 권한 조건 추가 |
| 기본 제외 조건 적용 | 취소/환불/비활성 데이터 제외 |
| SQL 안전성 검사 | DML/DDL 금지, 허용 테이블만 사용 |
| 결과 제한 | LIMIT, 기간 조건, 권한 조건 강제 |
| 오류 수정 | 컬럼 오류, 조건 오류 발생 시 수정 |
| 답변 기준 생성 | SQL 기준을 자연어로 설명 |

---

## 8.4 예시

사용자 질문:

```text
최근 3개월간 센터별 미납 금액 추이를 보여줘
```

AI가 생성할 수 있는 SQL:

```sql
SELECT
    payment_ym,
    cmp_cd,
    cmp_nm,
    SUM(unpaid_amount) AS total_unpaid_amount
FROM mart_payment_unpaid_student
WHERE payment_ym BETWEEN '202604' AND '202606'
  AND unpaid_amount > 0
  AND cancel_yn = 'N'
  AND refund_yn = 'N'
GROUP BY payment_ym, cmp_cd, cmp_nm
ORDER BY payment_ym, total_unpaid_amount DESC;
```

---

# 9. 5단계: 판다형 Agent 단계

## 9.1 개념

최종 단계에서는 AI가 단순 SQL 생성기를 넘어 데이터 분석 Agent로 동작한다.

```text
질문 이해
→ 질문 분해
→ 필요한 데이터 판단
→ 테이블 후보 탐색
→ SQL 생성
→ 실행
→ 결과 검증
→ 필요 시 재시도
→ 인사이트 생성
→ 답변
```

---

## 9.2 AI의 역할

| 역할 | 설명 |
|---|---|
| 질문 분해 | 복합 질문을 여러 분석 단위로 나눔 |
| 분석 계획 수립 | 어떤 데이터를 어떤 순서로 조회할지 결정 |
| 테이블 후보 탐색 | 메타데이터 기반으로 적합한 테이블 찾기 |
| SQL 생성 | 필요한 분석 쿼리 작성 |
| SQL 실행 | 조회 전용 환경에서 실행 |
| 결과 검증 | 값이 비정상인지, 결과가 없는지 확인 |
| 재시도 | 실패 시 다른 조건/테이블로 재생성 |
| 인사이트 도출 | 증가/감소, 이상치, 순위, 추세 설명 |
| 사용자 재질문 | 기준이 애매하면 필요한 조건 질문 |
| 관리자 알림 | 답변 불가/기준 없음/권한 오류 시 검토 큐 등록 |

---

## 9.3 예시

사용자 질문:

```text
정기결제 가입자는 늘었는데 미납도 같이 증가한 센터가 있어?
```

AI는 이 질문을 다음처럼 분해한다.

```text
1. 정기결제 가입자 수 추이 필요
2. 미납 금액 또는 미납 학생 수 추이 필요
3. 센터별 비교 필요
4. 최근 기간 기준 필요
5. 가입자 증가 + 미납 증가 조건 필요
```

사용할 수 있는 데이터:

```text
mart_payment_recurring_status
mart_payment_unpaid_student
mart_student_monthly_status
```

AI가 수행할 분석:

```text
센터별 월별 정기결제 가입자 수 계산
센터별 월별 미납 금액 계산
전월 대비 증감률 계산
가입자 증가율 > 0
미납 금액 증가율 > 0
해당 센터 추출
```

---

# 10. 단계별 AI 역할 요약

| 단계 | 방식 | AI의 핵심 역할 | SQL 직접 생성 여부 |
|---|---|---|---|
| 1단계 | 대시보드 쿼리 매칭형 | 질문을 분류하고 기존 쿼리 템플릿에 매칭 | 아니오 |
| 2단계 | 지표/용어 사전 기반 | 사용자 표현을 내부 지표와 연결 | 아니오 또는 일부 보정 |
| 3단계 | 표준 마트/VIEW 기반 | 어떤 마트/VIEW를 사용할지 판단 | 일부 가능 |
| 4단계 | 제한적 SQL 생성형 | 승인된 테이블 안에서 SELECT 직접 생성 | 예 |
| 5단계 | 판다형 Agent | 테이블 탐색, SQL 생성, 실행, 검증, 재시도, 인사이트 도출 | 예 |

---

# 11. 답변 불가 요청 처리 방식

AI가 모든 질문에 답변해서는 안 된다.

다음과 같은 요청은 관리자 검토 큐로 보내야 한다.

| 답변 불가 사유 | 예시 |
|---|---|
| 지표 정의 없음 | “위험한 센터 알려줘” |
| 기간 기준 불명확 | “요즘 줄어든 센터 알려줘” |
| 권한 없음 | 센터 사용자가 전체 센터 매출 요청 |
| 등록된 쿼리 없음 | 대시보드에 없는 신규 분석 요청 |
| 쿼리 실패 | 컬럼 변경, 테이블 오류 |
| 결과 신뢰도 낮음 | 후보 지표가 여러 개로 충돌 |
| 개인정보 위험 | 학생 연락처 전체 요청 |

---

## 11.1 답변 불가 처리 흐름

```text
1. 답변 불가 요청 저장
2. 실패 사유 분류
3. 유사 질문 묶기
4. 관리자에게 요약 알림
5. 관리자가 지표/쿼리 템플릿 추가
6. 다음부터 AI가 답변 가능
```

---

## 11.2 관리자 큐 테이블 예시

```sql
CREATE TABLE tb_ai_unanswered_request (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,

    user_id            VARCHAR(50),
    user_name          VARCHAR(50),
    business_unit      VARCHAR(50),

    question           TEXT NOT NULL,
    fail_reason        VARCHAR(100),
    fail_detail        TEXT,

    matched_dashboard  VARCHAR(100),
    matched_metric     VARCHAR(100),

    status             VARCHAR(20) DEFAULT 'WAITING' COMMENT 'WAITING, REVIEWING, DONE, REJECTED',
    admin_comment      TEXT,

    reg_dt             DATETIME DEFAULT CURRENT_TIMESTAMP,
    mod_dt             DATETIME NULL
);
```

---

# 12. 주요 메타 테이블 설계 예시

## 12.1 대시보드 메타 테이블

```sql
CREATE TABLE tb_ai_dashboard_catalog (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,

    business_unit      VARCHAR(50) NOT NULL COMMENT '사업부/팀',
    dashboard_name     VARCHAR(100) NOT NULL COMMENT '대시보드명',
    dashboard_desc     TEXT COMMENT '대시보드 설명',

    domain_name        VARCHAR(50) COMMENT 'student/payment/test/message 등',
    main_metrics       TEXT COMMENT '주요 지표 목록',
    sample_questions   TEXT COMMENT '이 대시보드로 답변 가능한 질문 예시',

    use_yn             CHAR(1) DEFAULT 'Y',
    reg_dt             DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 12.2 지표/쿼리 템플릿 테이블

```sql
CREATE TABLE tb_ai_query_template (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,

    dashboard_id       BIGINT NOT NULL,
    metric_name        VARCHAR(100) NOT NULL COMMENT '지표명',
    metric_desc        TEXT COMMENT '지표 설명',

    question_keywords  TEXT COMMENT '사용자 질문 매칭 키워드',
    query_template     LONGTEXT NOT NULL COMMENT '실행 가능한 SELECT 쿼리 템플릿',

    required_params    TEXT COMMENT '필수 파라미터: paymentYm, cmpCd 등',
    default_params     TEXT COMMENT '기본값 규칙: 현재월, 전월 등',

    answer_format      TEXT COMMENT '답변 포맷',
    caution_text       TEXT COMMENT '주의사항/기준 설명',

    allow_group_by     TEXT COMMENT '센터별, 월별, 학년별 등 허용 그룹',
    use_yn             CHAR(1) DEFAULT 'Y',

    reg_dt             DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 12.3 비즈니스 용어 사전

```sql
CREATE TABLE tb_ai_business_term (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,

    term_name          VARCHAR(100) NOT NULL COMMENT '사용자 표현 또는 업무 용어',
    standard_metric    VARCHAR(100) COMMENT '연결되는 표준 지표명',
    domain_name        VARCHAR(50) COMMENT '도메인',
    term_desc          TEXT COMMENT '용어 설명',
    calculation_rule   TEXT COMMENT '계산/판단 기준',

    use_yn             CHAR(1) DEFAULT 'Y',
    reg_dt             DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

예시:

| term_name | standard_metric | calculation_rule |
|---|---|---|
| 결제 안 된 학생 | 미납 학생 수 | `unpaid_amount > 0` |
| 시험 안 본 학생 | 무응시 학생 수 | 대상자 중 시험 결과 없음 |
| 다시 온 학생 | 복귀생 수 | 이전월 공백 후 재등록 |
| 새로 들어온 학생 | 신규생 수 | 해당월 최초 등록 |

---

## 12.4 질문 로그 테이블

```sql
CREATE TABLE tb_ai_question_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,

    user_id            VARCHAR(50),
    user_name          VARCHAR(50),
    question           TEXT NOT NULL,

    matched_domain     VARCHAR(50),
    matched_metric     VARCHAR(100),
    matched_template_id BIGINT,

    generated_sql      LONGTEXT,
    answer_summary     TEXT,

    success_yn         CHAR(1),
    fail_reason        VARCHAR(100),
    feedback           TEXT,

    reg_dt             DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

# 13. 테이블 정의서 예시

아래는 AI가 SELECT 문을 만들 수 있도록 작성한 테이블 정의서 예시이다.

---

## 13.1 테이블 기본 정보

| 항목 | 내용 |
|---|---|
| 테이블명 | `mart_payment_unpaid_student` |
| 한글명 | 정기결제 미납 학생 현황 |
| 테이블 유형 | 조회용 마트 / VIEW |
| 사용 목적 | 청구월 기준으로 정기결제 미납 학생 수, 미납 금액, 센터별 미납 현황 조회 |
| 주요 사용자 | 본사 운영팀, 결제 담당자, 센터 관리자, AI 데이터 조회봇 |
| 데이터 기준 | 정기결제 청구가 생성된 학생 중 미납 금액이 남아 있는 건 |
| 갱신 주기 | 매일 오전 1시 청구/결제 데이터 반영 후 갱신 |
| 개인정보 포함 여부 | 학생명 포함, 학부모 연락처 미포함 |
| AI 조회 허용 여부 | 허용 |
| 허용 쿼리 | SELECT만 허용 |

---

## 13.2 컬럼 정의

| 순번 | 컬럼명 | 한글명 | 데이터 타입 | NULL | 설명 | 예시 |
|---:|---|---|---|---|---|---|
| 1 | `payment_ym` | 청구월 | VARCHAR(6) | N | 정기결제 청구 기준 월. `YYYYMM` 형식 | `202606` |
| 2 | `cmp_cd` | 센터 코드 | VARCHAR(3) | N | 학생이 소속된 센터 코드 | `BFA` |
| 3 | `cmp_nm` | 센터명 | VARCHAR(100) | Y | 센터 이름 | `압구정센터` |
| 4 | `fc_cls_sn` | 직영/가맹 구분 | INT | Y | 직영/가맹 구분 코드. `1`은 직영으로 사용 | `1` |
| 5 | `std_cd` | 학생 코드 | VARCHAR(11) | N | 학생을 식별하는 고유 코드 | `ST260000001` |
| 6 | `std_nm` | 학생명 | VARCHAR(50) | Y | 학생 이름 | `홍길동` |
| 7 | `monthly_user_number` | 정기결제 고객번호 | VARCHAR(15) | Y | 먼쓸리 정기결제 고객번호 | `MU260600001` |
| 8 | `monthly_bill_number` | 정기결제 청구번호 | VARCHAR(30) | Y | 먼쓸리 청구서 번호 | `MB260600001` |
| 9 | `order_id` | 정기결제 주문 ID | BIGINT | N | `tb_student_recurring_order.id` 값 | `1024` |
| 10 | `order_state` | 주문 상태 | CHAR(1) | N | 정기결제 주문 상태 코드 | `3` |
| 11 | `order_state_nm` | 주문 상태명 | VARCHAR(50) | Y | 주문 상태 코드의 한글명 | `미납` |
| 12 | `invoice_type` | 청구서 유형 | VARCHAR(50) | Y | 수업료, 교재비, 기타청구서 구분 | `수업료 청구서_교재비` |
| 13 | `class_ogn_let_cd` | 수강/청구 항목 코드 | VARCHAR(11) | Y | 청구 상세 항목을 식별하는 코드 | `CL260600001` |
| 14 | `charge_amount` | 청구 금액 | DECIMAL(12,0) | N | 학생에게 청구된 금액 | `189000` |
| 15 | `paid_amount` | 결제 금액 | DECIMAL(12,0) | N | 실제 결제 완료된 금액 | `100000` |
| 16 | `unpaid_amount` | 미납 금액 | DECIMAL(12,0) | N | 청구 금액에서 결제 금액을 제외한 남은 금액 | `89000` |
| 17 | `cancel_yn` | 취소 여부 | CHAR(1) | N | 청구 또는 수강 취소 여부 | `N` |
| 18 | `refund_yn` | 환불 여부 | CHAR(1) | N | 환불 완료 여부 | `N` |
| 19 | `recurring_yn` | 정기결제 여부 | CHAR(1) | N | 정기결제 청구 대상 여부 | `Y` |
| 20 | `reg_dt` | 등록일시 | DATETIME | Y | 청구 데이터 생성 일시 | `2026-06-01 01:00:00` |
| 21 | `data_updated_dt` | 데이터 갱신일시 | DATETIME | Y | 마트 데이터 최종 갱신 일시 | `2026-06-02 01:10:00` |

---

## 13.3 업무 지표 정의

### 미납 학생 수

| 항목 | 내용 |
|---|---|
| 지표명 | 미납 학생 수 |
| 정의 | 청구월 기준 미납 금액이 남아 있는 학생 수 |
| 집계 방식 | `COUNT(DISTINCT std_cd)` |
| 기본 조건 | `unpaid_amount > 0` |
| 제외 조건 | `cancel_yn = 'Y'`, `refund_yn = 'Y'` |
| 기본 기간 | 사용자가 기간을 말하지 않으면 현재 청구월 기준 |

예시 SQL:

```sql
SELECT
    COUNT(DISTINCT std_cd) AS unpaid_student_count
FROM mart_payment_unpaid_student
WHERE payment_ym = DATE_FORMAT(CURRENT_DATE, '%Y%m')
  AND unpaid_amount > 0
  AND cancel_yn = 'N'
  AND refund_yn = 'N';
```

---

### 센터별 미납 금액

| 항목 | 내용 |
|---|---|
| 지표명 | 센터별 미납 금액 |
| 정의 | 센터별 미납 금액 합계 |
| 집계 방식 | `GROUP BY cmp_cd, cmp_nm` |
| 정렬 기준 | 미납 금액이 큰 센터 순 |

예시 SQL:

```sql
SELECT
    cmp_cd,
    cmp_nm,
    COUNT(DISTINCT std_cd) AS unpaid_student_count,
    SUM(unpaid_amount) AS total_unpaid_amount
FROM mart_payment_unpaid_student
WHERE payment_ym = DATE_FORMAT(CURRENT_DATE, '%Y%m')
  AND unpaid_amount > 0
  AND cancel_yn = 'N'
  AND refund_yn = 'N'
GROUP BY cmp_cd, cmp_nm
ORDER BY total_unpaid_amount DESC;
```

---

## 13.4 AI 사용 시 주의사항

1. 이 테이블은 미납 조회용 마트이므로 결제 성공률 전체 분석에는 사용하지 않는다.
2. 결제 성공/실패 전체 현황은 `mart_payment_recurring_status` 테이블을 사용한다.
3. 학생 개인정보 보호를 위해 학부모 연락처, 카드번호, 결제수단 상세 정보는 포함하지 않는다.
4. 사용자가 센터 권한 사용자라면 본인 센터의 `cmp_cd` 조건을 반드시 추가한다.
5. 사용자가 “전체 센터”를 요청하더라도 권한이 없으면 전체 조회를 허용하지 않는다.
6. `unpaid_amount`가 0보다 큰 경우만 미납으로 본다.
7. 취소 또는 환불 완료 건은 미납 집계에서 제외한다.

---

# 14. 관리자 화면에 필요한 기능

| 기능 | 설명 |
|---|---|
| 답변 불가 질문 목록 | 사용자가 물었지만 답변 못 한 질문 |
| 유사 질문 묶기 | “미납 학생”, “미수 학생”, “결제 안 한 학생” 통합 |
| 실패 사유 확인 | 지표 없음, 권한 없음, 쿼리 없음, 조건 불명확 |
| 관련 대시보드 연결 | 어느 대시보드로 답변할지 지정 |
| 쿼리 템플릿 등록 | SELECT 쿼리 등록 |
| 답변 기준 작성 | “미납 기준은 잔액 > 0” 같은 설명 |
| 테스트 실행 | 샘플 질문으로 답변 확인 |
| 사용 승인 | 승인 후 AI 답변 가능 처리 |

---

# 15. 보안 및 운영 안전장치

| 안전장치 | 설명 |
|---|---|
| SELECT만 허용 | INSERT, UPDATE, DELETE, DROP, ALTER 금지 |
| 허용 테이블 제한 | 표준 마트/VIEW만 조회 |
| 권한 조건 자동 추가 | 센터 사용자는 자기 센터만 조회 |
| 실행 시간 제한 | 무거운 쿼리 방지 |
| 결과 Row 제한 | 대량 개인정보 노출 방지 |
| SQL 로그 저장 | 사후 검토 및 감사 |
| 답변 기준 표시 | 사용자가 결과 기준을 이해할 수 있게 함 |
| 관리자 검토 큐 | 답변 불가/오답 개선 |
| 개인정보 마스킹 | 학생명, 연락처, 결제정보 보호 |

---

# 16. 추천 MVP 범위

처음부터 전사 데이터를 모두 연결하지 않고, 질문 빈도가 높고 기준을 정의하기 쉬운 도메인부터 시작한다.

## 16.1 1차 MVP 추천 도메인

| 도메인 | 우선 지표 |
|---|---|
| 정기결제 | 가입자 수, 해지자 수, 미납 학생 수, 청구 금액, 결제 성공/실패 |
| 학사관리 | 재원생 수, 신규생 수, 퇴원생 수, 복귀생 수 |
| 평가 | 응시자 수, 무응시자 수, 평균 점수, 합격/불합격 |
| 알림톡 | 발송 건수, 실패 건수, 실패 사유 |

---

## 16.2 MVP 산출물

```text
1. 사업부별 대시보드 목록
2. 핵심 질문 30~50개
3. 검증된 쿼리 템플릿
4. 답변 불가 요청 로그
5. 사용자 피드백 로그
6. 지표 정의서 초안
7. 테이블 정의서 초안
8. 관리자 검토 화면
```

---

# 17. 최종 개발 로드맵

| 단계 | 목표 | AI 역할 | 주요 산출물 |
|---|---|---|---|
| Phase 1 | 대시보드 쿼리 매칭형 PoC | 질문 분류, 템플릿 매칭, 답변 생성 | 쿼리 템플릿, 질문 로그 |
| Phase 2 | 지표/용어 사전 구축 | 사용자 표현을 내부 지표로 변환 | 용어 사전, 지표 정의서 |
| Phase 3 | 표준 마트/VIEW 정비 | 적합한 마트 선택 | 조회용 마트, 테이블 정의서 |
| Phase 4 | 제한적 SQL 생성 | 승인된 마트 안에서 SELECT 생성 | SQL 검증기, 권한 필터 |
| Phase 5 | 판다형 Agent 고도화 | 테이블 탐색, SQL 생성, 실행, 검증, 인사이트 도출 | Agent Loop, 품질 관리 체계 |

---

# 18. 내부 제안용 문구

아래 문구는 내부 보고서나 기획서에 그대로 사용할 수 있다.

```text
본 과제는 사내 데이터 질의응답 AI를 단계적으로 구축하는 것을 목표로 한다.

초기에는 사업부별 기존 대시보드에서 검증된 쿼리와 지표를 기반으로 사용자의 자연어 질문을 쿼리 템플릿에 매칭하여 답변한다.

답변이 불가능하거나 기준이 불명확한 질문은 관리자 검토 큐에 저장하여 지표 정의와 쿼리 템플릿을 지속적으로 확장한다.

이후 축적된 질문 로그, 지표 사전, 테이블 정의서, 표준 데이터 마트를 기반으로 AI가 제한된 범위 내에서 SELECT 쿼리를 직접 생성하고 검증하는 판다형 데이터 분석 Assistant로 고도화한다.
```

---

# 19. 결론

사용자 회사에서 가장 적합한 방향은 다음과 같다.

```text
대시보드 쿼리 매칭형으로 빠르게 시작
→ 답변 불가 질문을 모아 지표/용어 사전 구축
→ 자주 쓰는 업무 데이터를 표준 마트/VIEW로 정리
→ 승인된 마트 안에서만 AI SELECT 생성 허용
→ 장기적으로 판다형 Agent로 고도화
```

이 방식은 다음 장점이 있다.

| 장점 | 설명 |
|---|---|
| 빠른 시작 가능 | 기존 대시보드 쿼리 활용 |
| 정확도 확보 | 검증된 쿼리부터 사용 |
| 보안 통제 가능 | SELECT, 허용 테이블, 권한 제한 |
| 운영 개선 가능 | 답변 불가 질문이 지표 개선 자료가 됨 |
| 단계적 고도화 가능 | 템플릿형에서 Agent형으로 확장 가능 |

핵심은 AI가 처음부터 모든 것을 하게 만드는 것이 아니라, **사람이 검증한 지표와 쿼리에서 시작해 점진적으로 AI의 자율성을 높이는 것**이다.
