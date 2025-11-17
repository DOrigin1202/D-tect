# D-Tect: 실시간 사이버불링 탐지 시스템

[![FastAPI](https://img.shields.io/badge/FastAPI-0.112.0-009688?logo=fastapi)](https://fastapi.tiangolo.com/)
[![Transformers](https://img.shields.io/badge/Transformers-4.40+-FF6F00?logo=huggingface)](https://huggingface.co/transformers/)
[![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o-412991?logo=openai)](https://openai.com/)
[![Google Cloud](https://img.shields.io/badge/Google_Cloud-Vision_API-4285F4?logo=googlecloud)](https://cloud.google.com/vision)

게임 스트리밍 환경에서 실시간으로 채팅 스크린샷을 분석하여 사이버불링을 탐지하는 AI 기반 FastAPI 서버입니다.

## 📊 프로젝트 개요

- **목적**: 게임 스트리밍 플랫폼의 실시간 채팅 모니터링 및 사이버불링 자동 탐지
- **처리 방식**: 스크린샷 → OCR → 텍스트 분류 → 라벨링
- **아키텍처**: Google Vision API + UNSMILE + GPT-4o 하이브리드 모델
- **배포**: FastAPI 기반 RESTful API 서버

## 🎯 담당 역할

본 프로젝트에서 다음 작업을 수행했습니다:

- ✅ **OCR 파이프라인 구축**: Google Vision API를 활용한 스크린샷 텍스트 추출
- ✅ **다단계 분류 시스템 설계**: UNSMILE 1차 필터링 + GPT-4o 세부 분류
- ✅ **FastAPI 서버 구현**: 비동기 처리 및 백그라운드 태스크 관리
- ✅ **중복 억제 로직**: 캡처 주기로 인한 중복 탐지 방지 시스템
- ✅ **JSON 스키마 설계**: Structured Output을 위한 엄격한 스키마 정의

## 📁 프로젝트 구조

```
D-Tect/
├── main.py                 # FastAPI 서버 메인 (실제 배포 버전)
├── main_copy.py            # 개발/실험용 백업 버전
├── analyze_client.py       # API 테스트 클라이언트 (미사용)
├── requirements.txt        # 의존성 패키지
├── .gitignore             # Git 제외 파일 목록
├── credentials/           # API 키 저장 디렉토리
│   ├── *.json            # Google Cloud Vision 서비스 계정 키
│   └── openai.key        # OpenAI API 키
└── logs/                  # 분석 로그 (런타임 생성)
    ├── chat_log.json             # OCR 원문 로그 (JSON Lines)
    └── classified_chats.json     # 탐지/억제 결과 로그 (JSON Lines)
```

## 🔧 기술 스택

### 핵심 프레임워크
- **FastAPI 0.112.0**: 비동기 웹 서버
- **Uvicorn**: ASGI 서버
- **Pydantic 2.7.0**: 데이터 검증

### AI/ML 라이브러리
- **Transformers 4.40+**: Hugging Face 모델 로딩
- **LangChain (OpenAI)**: GPT-4o 체이닝
- **Google Cloud Vision 3.7.2+**: OCR 엔진

### 보조 라이브러리
- **Pillow**: 이미지 전처리 (크롭)
- **Tenacity**: LLM API 재시도 로직 (main_copy.py)

## 🧠 시스템 아키텍처

### 전체 처리 흐름

```
[스크린샷] → [이미지 크롭] → [Google Vision OCR] → [라인 분리]
              ↓
[UNSMILE 1차 필터] (score > 0.80 and label != 'clean')
              ↓
         [중복 억제 체크]
              ↓
    [GPT-4o 세부 분류] (6가지 라벨)
              ↓
[Spring Boot 콜백] + [로그 저장]
```

### 1. OCR 파이프라인

**이미지 전처리**
```python
def ocr_lines_from_image_bytes(
    img_bytes: bytes,
    crop_top_ratio: float = 0.15,    # 상단 15% 제거 (헤더)
    crop_bottom_ratio: float = 0.1,  # 하단 10% 제거 (입력창)
)
```

**채팅 포맷 파싱**
- 사용자명과 메시지 분리: `"username: message"` → `{"user": "username", "text": "message"}`
- 질문 형식 정규화: `"text?"` → `"text ?"` (띄어쓰기 추가)

### 2. 다단계 분류 시스템

#### Phase 1: UNSMILE 1차 필터링
- **모델**: `smilegate-ai/kor_unsmile` (Hugging Face)
- **역할**: 유해 표현 사전 필터링
- **기준**: `score > 0.80` and `label != 'clean'`
- **목적**: GPT-4o API 비용 절감 (유해 가능성 높은 텍스트만 전달)

```python
unsmile = clf(text)
label = unsmile[0]["label"]      # clean / offensive / hate / ...
score = float(unsmile[0]["score"])  # 0.0 ~ 1.0
```

#### Phase 2: GPT-4o 세부 분류
- **모델**: `gpt-4o` (OpenAI)
- **입력**: UNSMILE 통과 텍스트
- **출력**: Structured Output (JSON Schema)
- **라벨 후보** (6가지):

| 라벨 | 설명 | 예시 |
|-----|------|------|
| VIOLENCE | 폭력 위협, 자해 조장 | "죽어버려", "때려줄까?" |
| DEFAMATION | 명예훼손, 악의적 소문 | "저 사람 사기꾼임", "부정행위 했음" |
| SEXUAL | 성적 대상화, 성희롱 | "몸매 좋네", "야한 거 보여줘" |
| BULLYING | 따돌림, 모욕, 혐오 발언 | "너 같은 쓰레기", "꺼져" |
| CHANTAGE | 협박, 갈취 (비밀 폭로 위협) | "돈 안 주면 퍼트림", "비밀 알고 있음" |
| EXTORTION | 공갈, 강요 (직접적 위협) | "안 하면 디도스", "방송 끌어" |

**JSON Schema 구조**
```json
{
  "items": [
    {"label": "BULLYING", "count": 3},
    {"label": "SEXUAL", "count": 1}
  ]
}
```

- `label`: 6가지 라벨 중 1개 (enum)
- `count`: 심각도 (1~5)
- `items`: 최소 1개, 최대 5개 (한 메시지에 복수 라벨 가능)

### 3. 중복 억제 시스템

캡처 주기(예: 3초마다)로 인한 중복 탐지를 방지합니다:

**설정값**
```python
DEDUP_ENABLED = True              # 중복 억제 활성화
DEDUP_TTL_SEC = 12.0             # 12초 내 중복 무시
DEDUP_SIM_THRESHOLD = 0.985      # 98.5% 이상 유사 시 중복 판정
DEDUP_MAX_ENTRIES = 400          # 세션당 최대 캐시 크기
```

**동작 방식**
1. 텍스트 정규화: 소문자 변환 + 공백 정규화
2. 시간 기반 캐시: 최근 12초 이내 탐지 기록 유지
3. 유사도 검사: `difflib.SequenceMatcher`로 98.5% 이상 유사 시 억제
4. 메모리 관리: 세션별 최대 400개까지만 캐싱

**중요**: 지속적/반복적 가해는 모두 탐지됩니다. 오직 "캡처 타이밍으로 인한 동일 텍스트"만 억제합니다.

### 4. 콜백 시스템

탐지 결과를 Spring Boot 백엔드로 전송:

**콜백 URL**
```python
# URL 1: 라벨값을 tb_case 테이블에 저장
SPRING_CALLBACK_URL = "http://127.0.0.1:8081/api/analysis/callback-model"

# URL 2: 분석 테이블에 전체 결과 저장
SPRING_CALLBACK_URL_2 = "http://127.0.0.1:8081/api/analysis/callback"
```

**페이로드 구조**
```json
[
  {
    "user": "악성유저123",
    "text": "너 같은 쓰레기는 방송 그만해",
    "score": "0.95",
    "classification": [
      {"label": "BULLYING", "count": 4}
    ]
  }
]
```

## 🚀 설치 및 실행

### 1. 환경 설정

```bash
# Python 3.12+ 권장
pip install -r requirements.txt
```

### 2. API 키 설정

**Google Cloud Vision**
```bash
# 방법 1: 환경변수
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"

# 방법 2: credentials 디렉토리
mkdir -p credentials
cp service-account.json credentials/
```

**OpenAI API**
```bash
# 방법 1: 환경변수
export OPENAI_API_KEY="sk-..."

# 방법 2: 파일로 저장
echo "sk-..." > credentials/openai.key
```

### 3. 서버 실행

```bash
uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```

## 📡 API 엔드포인트

### `GET /ping`
헬스체크 및 의존성 상태 확인

**응답 예시**
```json
{
  "status": "ok",
  "message": "pong",
  "deps": {
    "vision_enabled": true,
    "llm_enabled": true,
    "llm_key_source": "file:openai.key",
    "log_dir": "/app/logs",
    "dedup_enabled": true,
    "dedup_ttl_sec": 12.0,
    "dedup_sim_threshold": 0.985
  }
}
```

### `POST /predict`
스크린샷 분석 및 사이버불링 탐지

**요청**
- `file`: 이미지 파일 (multipart/form-data)
- `analId`: 분석 세션 ID (optional, query parameter)
- `crop_top`: 상단 크롭 비율 (default: 0.15)
- `crop_bottom`: 하단 크롭 비율 (default: 0.1)

**응답**
```json
{
  "flagged": true,
  "labels": ["BULLYING", "SEXUAL"]
}
```

**cURL 예시**
```bash
curl -X POST "http://localhost:8001/predict?analId=123" \
  -F "file=@screenshot.png" \
  -F "crop_top=0.15" \
  -F "crop_bottom=0.1"
```

## 📝 주요 구현 세부사항

### 1. Structured Output 강제

GPT-4o가 반드시 정해진 JSON 형식으로 응답하도록 스키마 강제:

```python
def _labels_envelope_schema() -> dict:
    return {
        "name": "cyberbullying_labels",
        "strict": True,  # 엄격 모드
        "schema": {
            "type": "object",
            "additionalProperties": False,  # 추가 필드 금지
            "required": ["items"],
            "properties": {
                "items": {
                    "type": "array",
                    "minItems": 1,
                    "maxItems": 5,
                    "items": {
                        "type": "object",
                        "required": ["label", "count"],
                        "properties": {
                            "label": {"type": "string", "enum": ALLOWED},
                            "count": {"type": "integer", "minimum": 1, "maximum": 5}
                        }
                    }
                }
            }
        }
    }
```

### 2. 비동기 콜백 처리

FastAPI의 `BackgroundTasks`를 활용해 응답 속도 향상:

```python
@app.post("/predict")
async def predict(
    file: UploadFile = File(...),
    background: BackgroundTasks = None,
    analId: Optional[int] = Query(default=None),
):
    # ... 분석 로직 ...
    
    if payload_items and analId is not None:
        # 백그라운드에서 콜백 전송 (응답 차단 안 함)
        post_callbacks(analId, payload_items, background)
    
    return {"flagged": bool(payload_items), "labels": unique_labels}
```

### 3. 지연 로딩 (Lazy Loading)

서버 시작 시간을 줄이고 메모리를 절약하기 위해 모델을 필요 시점에만 로드:

```python
@lru_cache  # 최초 1회만 로드
def get_unsmile():
    log.info("Loading UNSMILE pipeline...")
    return pipeline("text-classification", model="smilegate-ai/kor_unsmile")

@lru_cache
def get_llm_chain() -> Optional[ChatPromptTemplate]:
    # ... GPT-4o 체인 생성 ...
```

### 4. 로그 시스템

분석 결과를 JSON Lines 형식으로 저장:

**chat_log.json** (OCR 원문)
```json
{"user": "유저1", "text": "안녕하세요"}
{"user": "유저2", "text": "너 같은 쓰레기"}
```

**classified_chats.json** (탐지 결과)
```json
{
  "user": "유저2",
  "text": "너 같은 쓰레기",
  "score": "0.95",
  "classification": [{"label": "BULLYING", "count": 4}],
  "suppressed": false
}
```

## ⚙️ 환경 변수

| 변수명 | 기본값 | 설명 |
|--------|--------|------|
| `SPRING_CALLBACK_URL` | `http://127.0.0.1:8081/api/analysis/callback-model` | 라벨 저장 콜백 URL |
| `SPRING_CALLBACK_URL_2` | `http://127.0.0.1:8081/api/analysis/callback` | 분석 결과 저장 콜백 URL |
| `DTECT_LOG_DIR` | `./logs` | 로그 저장 경로 |
| `DTECT_DISABLE_VISION` | `0` | Vision API 비활성화 (1=비활성) |
| `DTECT_DISABLE_LLM` | `0` | LLM 비활성화 (1=비활성) |
| `DTECT_ENABLE_LOGS` | `1` | 로그 저장 활성화 (0=비활성) |
| `DTECT_DEDUP_ENABLED` | `1` | 중복 억제 활성화 |
| `DTECT_DEDUP_TTL_SEC` | `12` | 중복 판정 시간 윈도우 (초) |
| `DTECT_DEDUP_SIM_THRESHOLD` | `0.985` | 유사도 임계값 (0.0~1.0) |
| `DTECT_DEDUP_MAX_ENTRIES` | `400` | 세션별 최대 캐시 크기 |

## 🔍 성능 최적화

### 1. API 비용 절감
- **UNSMILE 사전 필터링**: GPT-4o 호출 전 유해 가능성 낮은 텍스트 제거
- **예상 절감률**: 약 70-80% (clean 텍스트 비율에 따라 상이)

### 2. 응답 속도
- **비동기 처리**: FastAPI의 async/await
- **백그라운드 태스크**: 콜백 전송을 응답과 분리
- **평균 응답 시간**: ~1-2초 (이미지 크기, 텍스트 양에 따라 변동)

### 3. 중복 처리 효율
- **메모리 기반 캐시**: deque 자료구조로 O(1) 삽입/삭제
- **TTL 자동 정리**: 만료된 기록 자동 제거
- **difflib 최적화**: 정규화된 텍스트로 비교 횟수 최소화

## 📊 모델 선택 이유

### UNSMILE
- **장점**: 한국어 특화, 빠른 추론 속도, 로컬 실행 가능
- **역할**: 1차 필터로 API 비용 절감 및 응답 속도 개선

### GPT-4o
- **장점**: 문맥 이해 우수, Structured Output 지원, 세밀한 분류
- **역할**: 유해 텍스트의 정확한 유형 분류 (6가지 라벨)

### Google Cloud Vision
- **장점**: 높은 OCR 정확도, 한글 지원 우수, 다양한 폰트/배경 처리
- **역할**: 게임 채팅 스크린샷에서 텍스트 추출

## 🚧 향후 개선 방향

- [ ] 실시간 스트리밍 연동 (WebSocket)
- [ ] 한국어 특화 LLM으로 교체 검토 (비용 절감)
- [ ] 탐지 임계값 동적 조정 (사용자 피드백 기반)
- [ ] 모니터링 대시보드 구축 (Grafana 연동)
- [ ] 멀티 언어 지원 (영어, 일본어)
- [ ] False Positive 분석 및 모델 재학습

## 📄 파일 설명

### main.py vs main_copy.py
- **main.py**: 프로덕션 버전 (실제 배포)
- **main_copy.py**: 개발/실험 버전 (tenacity 재시도 로직 포함)

### analyze_client.py
- 초기 테스트용 클라이언트
- 현재 미사용 (실시간 분석으로 전환)

### .gitignore
민감 정보 보호를 위한 제외 파일:
- API 키 파일 (`*.key`, `*credentials*.json`)
- 로그 디렉토리 (`logs/`)
- 모델 가중치 (`*.pth`, `*.bin`)
- 가상환경 (`venv/`, `.env`)

## 👤 Author

**정동인 (Dongin Jung)**
- Role: AI Engineer Intern
- Project: D-Tect (사이버불링 탐지 시스템)
- Tech Stack: FastAPI, Transformers, LangChain, Google Cloud Vision

## 📚 참고 자료

- [UNSMILE Model](https://huggingface.co/smilegate-ai/kor_unsmile)
- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Google Cloud Vision API](https://cloud.google.com/vision/docs)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

---

⚡ **Built with FastAPI for Real-time Cyberbullying Detection**
