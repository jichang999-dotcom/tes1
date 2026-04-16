# 조치완료 리셋 플로우 (테스트용)

> 상담사가 HTML 리포트에서 "조치완료" 버튼을 누르면, 해당 학생의 **과거 상담 이력이 리셋**되고 **그 이후 상담만 새로 누적**되는 구조의 프로토타입.
> 기존 리포트 파일은 건드리지 않고, 본 `update/` 폴더에서만 실험합니다.

---

## 폴더 구조

```
update/
├── README.md              ← 본 문서
└── ACV_리셋테스트.html     ← 단일 테스트 페이지 (백엔드 없음)
```

현재는 **HTML 1개 파일 안에서 전체 플로우를 시뮬레이션**합니다.
- 상담 데이터: 코드 안에 시드 데이터로 포함
- 조치 기록 저장: `localStorage` (= Google Sheets 역할)
- 배치 재빌드: "🔄 배치 시뮬레이션" 버튼 (= 로컬 19시 배치 역할)

n8n·시트·Node 스크립트가 **아직 없이도** 전체 UX와 리셋 동작을 확인할 수 있습니다.

---

## 사용 방법

1. 탐색기에서 `ACV_리셋테스트.html`을 더블클릭 → 브라우저로 열림
2. 상단 `상담사: 미등록` 옆 **[변경]** → 이메일 입력
3. 아래 시나리오 순서로 테스트

### 시나리오 A — 기본 조치완료

| 단계 | 동작 | 확인 포인트 |
|---|---|---|
| 1 | 임의 학생 카드의 **[✅ 조치완료]** 클릭 | 모달 오픈 |
| 2 | 메모 입력 후 **[등록]** | 토스트 뜸 + localStorage 저장 |
| 3 | 왼쪽 원본 테이블 확인 | 해당 학생의 과거 상담이 **회색 취소선** |
| 4 | 상단 **[🔄 19시 배치 시뮬레이션]** | 학생 카드가 **아카이브**로 이동, 평균 초과 카운트 감소 |
| 5 | 하단 **action_log.json** 뷰어 | 리셋 맵에 해당 학생 등록된 것 확인 |

### 시나리오 B — 조치 후 재발생

| 단계 | 동작 | 확인 포인트 |
|---|---|---|
| 1 | 시나리오 A 수행 상태 | 학생 1명 이상 아카이브에 있어야 함 |
| 2 | **[🔁 재발 상담 추가]** 클릭 | 아카이브된 학생 중 랜덤 1명에게 "오늘" 새 부정 상담 추가 |
| 3 | **[🔄 배치 시뮬레이션]** | 해당 학생이 **"조치 후 재발생" 경고**와 함께 리스트로 복귀 |
| 4 | 상단에 🔁 재발 배너 표시 | 관리자 알림용 UI |

### 시나리오 C — 조치 취소

| 단계 | 동작 | 확인 포인트 |
|---|---|---|
| 1 | 아카이브 카드의 **[↩️ 취소]** 클릭 | 확인 팝업 |
| 2 | 확인 | 과거 상담 이력 전체 복원 |

### 기타 버튼

- **[📅 하루 지남]** — 가상 "현재 시각"을 하루 전진시킴. 시간 경과에 따른 상담 누적 실험용
- **[♻️ 전체 초기화]** — 모든 테스트 상태 삭제 후 초기 상태로 복귀

---

## 핵심 로직 (HTML 내부 발췌)

### action_log.json 생성
```javascript
function buildActionLog() {
  const reset_map = {};
  for (const [code, st] of Object.entries(STATE)) {
    if (st.status === 'active') {
      reset_map[code] = {
        reset_at: st.reset_at,
        상담사: st.상담사,
        메모: st.메모,
        requestId: st.requestId
      };
    }
  }
  return { generated_at: NOW, reset_map };
}
```

### 리셋 필터
```javascript
function filterByReset(records, code, reset_map) {
  const reset = reset_map[code];
  if (!reset) return { effective: records, preReset: [] };
  return {
    effective: records.filter(r => r.cns_dt >  reset.reset_at),
    preReset : records.filter(r => r.cns_dt <= reset.reset_at)
  };
}
```

### 학생 카드 분기
| 조건 | 표시 |
|---|---|
| 조치 없음 | 일반 카드 |
| 조치 O · 이후 신규 없음 | **아카이브 탭으로 이동** (리스트에서 숨김) |
| 조치 O · 이후 신규 있음 | 리스트 유지 + **🔁 재발생 경고** + 상단 배너 |

---

## 실제 운영으로 옮길 때 매핑

| 테스트 요소 | 실제 운영 |
|---|---|
| HTML 내부 `SEED_COUNSELS` | `fetch_data.js`가 만드는 상담 레코드 JSON |
| `localStorage` STATE | Google Sheets "조치_내역" 시트 |
| `buildActionLog()` | `fetch_action_log.js` (Step 5.5) |
| `filterByReset()` | `build_html.js` 안의 필터 함수 |
| [🔄 배치 시뮬레이션] 버튼 | 19:00 `run_report2.bat` |
| [✅ 조치완료] 버튼 → STATE 갱신 | 버튼 → GAS Web App POST → 시트 append |

즉 **HTML 쪽 로직은 그대로 유지**하고, 저장소만 `localStorage` → `Google Sheets`로 바꾸면 운영 버전이 됩니다.

---

## 다음 단계 (승인 후)

1. 시드 데이터 대신 **실제 ACV 15일치 원본 상담 JSON**을 읽도록 수정
2. 기존 `reports/ACV.html` 생성 파이프라인 (`build_html.js`)에 리셋 필터만 이식
3. 저장소를 Google Sheets로 교체 (GAS Web App endpoint 1개)
4. 타 센터로 확대 (`CENTER_CODE` 변수만 바꾸면 재사용 가능)

---

## 디버깅 팁

- **상태 확인**: 브라우저 F12 → Application → Local Storage → 파일 URL 선택
  - `acv_reset_test_state_v1` — 조치 기록
  - `acv_reset_test_user` — 상담사 이메일
  - `acv_reset_test_clock` — 가상 현재 시각
  - `acv_reset_test_extra_counsels` — 재발 시나리오 추가 상담
- **완전 초기화**: 페이지에서 **[♻️ 전체 초기화]** 또는 F12 콘솔 `localStorage.clear(); location.reload();`
