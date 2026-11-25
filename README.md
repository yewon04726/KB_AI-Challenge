# KB AI Challenge — 창업 입지·금융상품 추천 서비스

창업을 준비하는 가맹점주가 **대화로 조건을 입력하면**, 입지를 추천하고 **맞춤 금융상품 Top 3와 그 이유**를 알려주는 AI 서비스입니다.

- 팀 프로젝트 (KB AI Challenge 공모전)

<img width="1106" height="612" alt="IMG_1277" src="https://github.com/user-attachments/assets/1c1707ce-4c2c-4e8c-89ee-81561e79f2f0" />
<img width="1112" height="602" alt="IMG_1279" src="https://github.com/user-attachments/assets/9c4dcc25-3876-4d41-88e5-59cfa9ecee7c" />
<img width="1115" height="602" alt="IMG_1280" src="https://github.com/user-attachments/assets/1536d4ca-c22b-4ab4-97e4-25f859d338d4" />

---

## 서비스 흐름

```text
1. 대화형 정보 수집    사용자와 대화하며 창업 정보 9가지를 수집
        ↓
2. 입지 추천          수집한 정보로 상권을 분석해 입지 후보를 점수화
        ↓
3. 금융상품 추천      재무 조건으로 상품을 걸러내고 점수를 매겨 Top 3 추천
```

### 1. 대화형 정보 수집 (`POST /api/chat`)
- GPT-4o가 대화에서 **창업 목적, 업종, 자본금, 선호 지역, 매장 크기** 등 9개 항목을 추출합니다.
- 이전 대화에서 모은 정보는 유지하고 새 정보만 갱신합니다. 한 번에 여러 정보를 말해도 한꺼번에 채웁니다.
- 필수 5개가 모두 채워지면 선택 항목(타겟층, 우선순위, 주차, 대중교통)을 안내하고, 사용자가 원하지 않으면 바로 추천 단계로 넘어갑니다.
- "500원으로 창업할래" 같은 비현실적인 입력은 다시 확인하도록 유도합니다.
- 출력은 정해진 JSON 구조를 따르고, 종료 여부(`ready_to_proceed`)와 다음 질문(`next_question`)을 함께 돌려줍니다.

### 2. 입지 추천 (`POST /api/location`)
- 백엔드가 Python 입지 분석 엔진(`location-scoring/`)을 실행하고, JSON으로 입력과 결과를 주고받습니다.
- 서울 상권 데이터를 바탕으로 지표를 정규화하고, 산업별 AHP 가중치와 사용자의 우선순위를 반영해 입지를 랭킹합니다.
- 자세한 내용은 [location-scoring/README.md](location-scoring/README.md)를 참고하세요.

### 3. 금융상품 추천 (`POST /api/recommend`)

**판단은 규칙으로, 설명은 AI로** 나눈 구조입니다.

```text
사용자 재무 정보
  → [1단계] 필수 자격 필터링 (Hard Filter): 청년 여부, 자금 용도, 담보 여력, 특화 대상
  → [2단계] 금리 정규화를 위한 최솟값·최댓값 계산
  → [3단계] 가중치 기반 적합도 점수 계산
  → [4단계] 점수순 정렬, Top 3 선정
  → [5단계] AI가 점수 세부 내역을 바탕으로 추천 이유를 작성
```

| 평가 항목 | 배점 |
| --- | --- |
| 창업자 조건 적합도 | 25 |
| 신용·담보 적합도 | 20 |
| 자금 용도 적합도 | 20 |
| 필요금액 충족도 | 15 |
| 금리 경쟁력 | 15 |
| 우대조건 적합도 | 5 |

- 상품 순위는 **코드가 계산한 점수**로만 정해집니다. AI(gpt-4o-mini)는 순위를 바꾸지 않고, 항목별 점수(`score_detail`)를 받아 **추천 이유를 1~2줄로 설명**합니다.
- 응답에 항목별 점수를 함께 내려 프론트에서 차트로 보여줄 수 있습니다.
- AI 호출이 실패해도 서버가 죽지 않고 기본 문구로 대체합니다.

---

## 구조

```text
├─ frontend/           React (Vite) 화면: screens, components, api.js
├─ backend/            Node.js (Express) API 서버
│  ├─ server.js
│  └─ routes/
│     ├─ chat.js       대화형 정보 수집 (OpenAI)
│     ├─ location.js   Python 입지 엔진 실행
│     └─ recommend.js  금융상품 필터링·점수·Top 3·AI 코멘트 (Supabase + OpenAI)
├─ location-scoring/   Python 입지 분석 엔진 (AHP 스코어링, 데이터 수집, 분석 스크립트)
├─ docker-compose.yml  프론트(80)와 백엔드(8080)를 함께 실행
└─ requirements.txt    초기 프로토타입용 Python 의존성
```

```text
[React] → [Express :8080] ─┬→ OpenAI (chat)
                            ├→ Python 입지 엔진 (stdin/stdout JSON)
                            └→ Supabase(금융상품) + OpenAI (recommend)
```

---

## 기술 스택

| 영역 | 구성 |
| --- | --- |
| Frontend | React 18, Vite |
| Backend | Node.js, Express |
| AI | OpenAI (gpt-4o: 대화 수집, gpt-4o-mini: 추천 설명) |
| Database | Supabase (PostgreSQL) |
| 입지 분석 | Python (AHP 가중치, 서울 상권 공공데이터, 폐업 예측 분석) |
| 배포·실행 | Docker, Docker Compose |

---

## 실행 방법

### Docker로 한 번에 실행

```bash
docker compose up --build
```

- 프론트엔드: `http://localhost` (80번 포트)
- 백엔드: `http://localhost:8080`

### 환경 변수

`backend/.env`를 만듭니다. 실제 값은 Git에 올리지 않습니다.

```env
PORT=8080
SUPABASE_URL=
SUPABASE_ANON_KEY=
OPENAI_API_KEY=
# PYTHON_BIN=python3   (선택)
```

입지 엔진용 키는 `location-scoring/.env.example`을 복사해 설정합니다. (서울 열린데이터광장, 공공데이터포털 API 키)

---

## 내 역할
- **백엔드**
- **자료수집**: 입지 추천 스코어링을 위한 AHP논문 수집 및 정리 
- **대화형 정보 수집 에이전트**: 초기 Python 프로토타입(GPT-4o + 데이터 검증)을 만들고, 이 로직을 백엔드 `chat` 라우트로 옮김
- **연동·배포**: 네이버 API 연동, Docker 구성


팀원 담당: 입지 분석 엔진(`location-scoring/`), 프론트엔드 화면

---

## 알려진 한계

- 연령 조건은 청년 여부로 대체해 산출하는 **예시값**을 사용합니다.
- 금융상품 데이터는 Supabase 테이블에 의존하며, 상품 갱신 자동화는 없습니다.
- 입지 엔진은 일부 단계가 프로토타입입니다. 자세한 내용은 `location-scoring/README.md`를 참고하세요.

---

원본 저장소: [yunsunghyeon0/KB-AI-Challenge](https://github.com/yunsunghyeon0/KB-AI-Challenge)
