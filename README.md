# 자체 감사 결과 AI 분류·조회 시스템

> 공개 감사 결과 약 **1만 5천 건**을 AI로 분류해, 실무자가 분야·업무 기준으로 바로 찾고 통계까지 볼 수 있게 만든 시스템

감사원 연구 과제로 개발했습니다. 한림대학교 지능형의사결정시스템 연구실 (LIT LAB)

| | |
|---|---|
| **기간** | 2025.07 ~ 2026.01 |
| **팀 구성** | 3명 |
| **본인 담당** | 백엔드 · 데이터 파이프라인 (수집·정제, 분류 체계, API, S3 연동, 배포) |
| **운영** | AWS 배포 후 약 6개월 운영, 소스코드와 실행 절차 인계 완료 |
| **협업 저장소** | [lit-ai-lab/pap2025_viewer](https://github.com/lit-ai-lab/pap2025_viewer) |

---

## 📌 무엇을 해결하나

감사 실무자가 과거 사례를 찾을 때 **분야나 업무 기준이 없어** 선별에 시간이 걸렸습니다. 감사 결과는 공개돼 있지만 기관별·연도별로 흩어져 있고, 제목만으로는 어떤 업무에 관한 사례인지 알기 어렵습니다.

그래서 흩어진 감사 결과를 모아 **일관된 분류 기준으로 라벨을 붙이고**, 비전공자도 쓸 수 있는 조회·통계 화면을 만들었습니다.

---

## 🔄 데이터 파이프라인

```
공개 감사 결과 수집
   │
   ├─ 정제 · 중복 제거 ──── 파일 해시 기반. 같은 문서가 여러 경로로 들어와도 한 건으로
   │
   ├─ AI 자동 분류 ──────── GPT-4.1-mini를 배치 스크립트로 일괄 호출
   │                       분야(category) → 업무(task) → 세부 업무 3단계
   │
   ├─ 이중 저장 ─────────── AI 분류 결과와 실무자 라벨링 원본을 컬럼으로 분리
   │                       원본을 덮어쓰지 않아 언제든 대조 가능
   │
   └─ 피드백 반영 ───────── 실무자 검토 결과로 분류 체계 자체를 개정
   │
   ▼
PostgreSQL 적재 · 원문 PDF는 S3 보관
```

### 왜 원본을 따로 남겼나

AI가 붙인 라벨이 실무자 판단과 다를 수 있습니다. 한 컬럼에 덮어쓰면 **어디가 어떻게 틀렸는지 추적할 수 없어** 분류 체계를 개선할 근거가 사라집니다.

두 값을 나란히 보관했더니 실무자 피드백을 모아 **분류 기준 자체를 고치는** 작업이 가능해졌습니다.

### 왜 배치인가

1만 5천 건을 건건이 호출하면 시간과 비용이 모두 커집니다. 문서를 묶어 배치 스크립트로 일괄 처리하고, 중단 지점부터 이어서 돌릴 수 있게 구성했습니다.

---

## 📊 조회와 통계

분류가 끝난 데이터를 실무자가 바로 쓸 수 있는 형태로 제공합니다.

| 기능 | 설명 |
|---|---|
| **조건 조회** | 분야 · 업무 · 감사 유형 · 기관 기준 필터링 |
| **분야·업무별 차트** | 건수 집계를 막대/원형, 드릴다운 도넛으로 시각화 |
| **지역별 지도 통계** | 시도·시군구 단위 분포를 지도에 표시 |
| **비교 보기** | 여러 조건의 결과를 나란히 대조 |
| **원문 확인** | 감사 결과 PDF를 화면에서 바로 열람 |
| **엑셀 다운로드** | 조회 결과를 파일로 내보내기 |

집계는 백엔드에서 처리합니다 (`crud/map.py`의 `get_category_task_summary`, `get_tasks_by_region`).

---

## 🛠 기술 스택

| 계층 | 기술 |
|---|---|
| **Backend** | FastAPI · SQLAlchemy · Pydantic Settings |
| **DB** | PostgreSQL · Alembic (마이그레이션) |
| **분류** | OpenAI GPT-4.1-mini (배치 호출) |
| **스토리지** | AWS S3 (감사 결과 PDF) |
| **Frontend** | React · Vite |
| **시각화** | 막대·원형 차트, 드릴다운 도넛, 지도(GeoJSON) |
| **인프라** | AWS 배포 · 도메인 연결 |

---

## 📁 프로젝트 구조

```
.
├── main.py                       # FastAPI 진입점
├── config.py                     # 설정 (DB · AWS 자격증명은 .env에서 로드)
├── database.py                   # 세션 · 엔진
├── models.py / schemas.py        # ORM 모델 · Pydantic 스키마
├── loader.py                     # 수집 데이터 적재
├── init_db.py                    # 초기 스키마 생성
│
├── domain/
│   ├── viewer/                   # 감사 사례 조회 (router · service)
│   ├── map/                      # 분야·업무·지역별 집계
│   ├── metadata/                 # 분류 체계 · 기관 메타데이터
│   └── pdf/                      # 원문 PDF 조회
│
├── crud/
│   ├── viewer.py                 # 조회 쿼리
│   ├── map.py                    # 집계 쿼리 (카테고리별 · 지역별)
│   └── metadata.py               # 메타데이터 조회
│
├── utils/s3_client.py            # S3 연동
├── alembic/                      # 마이그레이션 이력
│
└── Front/FE/src/
    ├── component/
    │   ├── MainPage.jsx          # 메인 조회 화면
    │   ├── MapPage.jsx           # 지역별 지도 통계
    │   ├── TaskPage.jsx          # 업무별 조회
    │   ├── ChartTabs.jsx         # 차트 탭
    │   ├── BarPieChart.jsx       # 막대 · 원형 차트
    │   ├── DrillDownDonutChart.jsx  # 드릴다운 도넛
    │   ├── RegionTaskTable.jsx   # 지역·업무 교차표
    │   ├── ComparisonGrid.jsx    # 조건 비교
    │   ├── DataTable.jsx         # 결과 표 · 엑셀 내보내기
    │   ├── PdfViewer.jsx         # 원문 PDF 뷰어
    │   └── Filtering.jsx         # 조건 필터
    └── data/                     # 분류 체계 · 기관 목록 · 지도 GeoJSON
```

---

## 🚀 실행 방법

### 사전 요구사항

| 항목 | 버전 |
|---|---|
| Python | 3.11 이상 |
| Node.js | 20 이상 |
| PostgreSQL | 16 |

### 1. 환경변수

루트에 `.env`를 만듭니다.

```ini
DATABASE_URL=postgresql://<user>:<password>@localhost:5432/<db>
AWS_ACCESS_KEY_ID=<key>
AWS_SECRET_ACCESS_KEY=<secret>
AWS_DEFAULT_REGION=ap-northeast-2
S3_BUCKET_NAME=<bucket>
```

> ⚠️ `.env`는 `.gitignore`에 포함돼 있습니다. 실제 자격증명은 커밋하지 마세요.

### 2. 백엔드

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
python init_db.py
uvicorn main:app --reload --port 8000
```

API 문서: http://127.0.0.1:8000/docs

### 3. 데이터 적재

```bash
python loader.py
```

### 4. 프론트엔드

```bash
cd Front/FE
npm install
npm run dev
```

---

## 📌 공개 범위

이 저장소는 **시스템 구조와 구현 방식**을 공개합니다. 다음은 포함되어 있지 않습니다.

- 수집·분류한 감사 결과 데이터
- 실제 배포 환경의 접속 정보 및 자격증명

`Front/FE/src/data/`의 JSON은 분류 체계·기관 목록·지도 좌표 같은 **메타데이터**이며, 감사 사례 본문이 아닙니다.

---

## 👥 기여

3인 팀 프로젝트입니다. 본 저장소는 원 협업 저장소 [lit-ai-lab/pap2025_viewer](https://github.com/lit-ai-lab/pap2025_viewer)의 포크이며, 저는 **백엔드와 데이터 파이프라인**을 담당했습니다 — 수집·정제, 분류 체계 수립과 배치 분류, API 설계, S3 연동, AWS 배포.
