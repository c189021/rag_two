# 가상Tech 사내 업무 가이드 RAG

사내 업무 가이드 PDF(`data/가상Tech_업무가이드.pdf`)를 근거로 질문에 답하는 RAG(Retrieval-Augmented Generation) 챗봇 실습 프로젝트입니다.
LangChain + OpenAI + FAISS로 구성했으며, 답변 끝에 근거 페이지를 `(p.N)` 형식으로 함께 표시합니다.

## 파이프라인

```
PDF → Loader → TextSplitter → Embedding → VectorStore(FAISS)
                                               ↓
질문 → Retriever(top-k) → Prompt → LLM → OutputParser → 답변 (p.N)
```

| 단계 | 사용 기술 |
|------|-----------|
| Loader | `PyPDFLoader` (페이지 단위 Document) |
| TextSplitter | `RecursiveCharacterTextSplitter` (chunk 800 / overlap 100) |
| Embedding | OpenAI `text-embedding-3-small` (1536차원) |
| VectorStore | FAISS (로컬 저장/재사용) |
| Retriever | 유사도 검색, k=5 |
| Prompt | `ChatPromptTemplate` (문서 근거 답변, 근거 없으면 "문서에서 확인되지 않습니다.") |
| LLM | `gpt-4o-mini` (temperature=0) |
| Chain | LCEL (`retriever \| prompt \| llm \| parser`) |

## 프로젝트 구조

```
.
├── data/
│   ├── 가상Tech_업무가이드.pdf   # 원본 문서
│   └── faiss_index/              # 저장된 FAISS 인덱스 (재실행 시 임베딩 비용 절감)
├── rag_01_0928.ipynb             # 1차 실습: 기본 RAG 체인 (chunk 500 / overlap 50)
├── rag_02_0928.ipynb             # 2차 실습: 개선 버전 (출처 표기, 프롬프트 강화, 인덱스 캐싱)
├── src/rag_two/                   # 패키지
├── pyproject.toml                # uv 프로젝트 설정
├── requirements.txt              # 전체 의존성 (uv export)
└── requirements_rag_02.txt       # rag_02 실행용 최소 의존성
```

## 노트북 비교

| | rag_01 | rag_02 |
|---|---|---|
| chunk_size / overlap | 500 / 50 | 800 / 100 (표가 끊기지 않도록 확대) |
| 검색 | 기본 retriever (k=4) | k=5, 검색 결과 페이지 확인 함수 |
| 프롬프트 | 간단한 시스템 메시지 | 근거 기반 답변, 항목별 답변, 규정 혼동 방지, 출처 페이지 표기 |
| 인덱스 | 매번 새로 생성 | 로컬 저장 후 재사용 |
| 평가 | 질문 테스트 | 질문 10개 일괄 실행 (`chain.batch`) |

## 실행 방법

### 1. 환경 준비

Python 3.12 이상이 필요합니다. [uv](https://docs.astral.sh/uv/) 사용을 권장합니다.

```bash
uv sync
# 또는
pip install -r requirements_rag_02.txt
```

### 2. 환경변수 설정

프로젝트 루트에 `.env` 파일을 만들고 키를 입력합니다.

```env
OPENAI_API_KEY=sk-...

# (선택) LangSmith 추적
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=...
LANGSMITH_PROJECT=rag_two
```

> ⚠️ `.env`는 API 키가 들어 있으므로 **절대 커밋하지 마세요.**

### 3. 노트북 실행

VS Code 또는 Jupyter에서 `rag_02_0928.ipynb`를 위에서부터 순서대로 실행합니다.
`data/faiss_index/`가 이미 있으면 임베딩 없이 저장된 인덱스를 불러옵니다.
PDF나 청크 설정을 바꿨다면 `data/faiss_index/` 폴더를 삭제한 뒤 다시 실행해 인덱스를 재생성하세요.

## 테스트 질문 예시

- 연차를 사용하려면 최소 며칠 전에 신청해야 하나요?
- 서울 출장 시 숙박비는 최대 얼마까지 지원되나요?
- 출장을 다녀온 후 비용 정산은 언제까지 완료해야 하나요?
- 회사 노트북을 분실했을 경우 어떤 절차로 신고해야 하나요?
- 제주도로 2박 3일 출장을 가는 경우 숙박비 지원 한도와 출장비 정산 기한을 각각 알려주세요. *(복합 질문)*

## 참고 사항

- FAISS 인덱스 로드 시 `allow_dangerous_deserialization=True`를 사용합니다. 직접 생성한 신뢰할 수 있는 인덱스만 불러오세요.
- 문서에 근거가 없는 질문에는 추측하지 않고 "문서에서 확인되지 않습니다."라고 답하도록 프롬프트로 제한했습니다.
