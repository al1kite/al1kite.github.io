---
layout: post
title: "n8n + LangChain + Qdrant로 주식 시장 분석 파이프라인 구축하기"
comments: true
excerpt: "RAG가 무엇인지, 왜 개발자들이 보통 코드로 파이프라인을 짜는데 나는 n8n이라는 워크플로 도구를 선택했는지, 그리고 Ollama + Qdrant + LangChain으로 완전한 온프레미스 RAG 시스템을 설계한 전 과정을 코드와 함께 정리합니다."
date: 2025-08-15
categories: [Doodle, Retrospect]
tags: [RAG, LangChain, n8n, Qdrant, Ollama, VectorDB, NLP, Retrospect]
---

# On-Premise RAG 시스템 설계기

## 들어가며

데보션 RAG 스터디 2회차 발표를 들으며 RAG와 Agent의 개념을 정리했습니다. 그때 스터디 과제로 각자 관심 주제로 프로그램을 하나 만들어보자는 이야기가 나왔고, 저는 이전부터 진행하던 **코스콤 Apex 공모전** 프로젝트를 RAG 기반으로 본격 재설계하기로 했습니다.

Apex는 한국 주식 시장 뉴스·커뮤니티·공시 데이터를 수집해 감정 분석과 시장 요약을 제공하는 서비스입니다. 핵심 질문은 단순했습니다. **"최신 뉴스와 커뮤니티 글을 기반으로, 특정 종목에 대해 신뢰할 수 있는 분석을 LLM이 할 수 있을까?"** 이를 위해 RAG가 필요했고, 데이터가 금융 도메인이다 보니 클라우드 API 의존 없이 **완전한 온프레미스** 환경을 목표로 했습니다.

**결과적으로 이 프로젝트는 코스콤 공모전 1차 심사를 통과했습니다.** 다만 최종 발표 일정이 다른 회사 면접과 겹치면서 발표에 참여하지 못한 것이 가장 큰 아쉬움으로 남았습니다. 그래도 이 과정에서 쌓은 RAG 시스템 설계 경험과 온프레미스 환경에서의 LLM 운영 노하우는 값진 자산이 되었습니다.

이 글에서는 RAG의 개념부터 시작해, 데이터 수집 파이프라인에 n8n을 선택한 이유, 전체 아키텍처 설계, 그리고 LangChain 체인 구현까지 실제 코드와 함께 심도 있게 다뤄보겠습니다.

---

## RAG란 무엇인가

### 기본 개념

**RAG(Retrieval-Augmented Generation)**는 LLM이 답변을 생성하기 전에, 외부 저장소에서 관련 정보를 **검색(Retrieve)**해 프롬프트에 함께 넣어주는 방식입니다.

```
사용자 질문
    ↓
[1] 벡터 DB에서 유사한 문서 검색 (Retrieve)
    ↓
[2] 검색된 문서 + 원래 질문을 LLM에 전달 (Augment)
    ↓
[3] LLM이 근거 기반 답변 생성 (Generate)
```

복잡해 보이지만, 본질은 **"검색 + 생성의 조합"**입니다. Google에서 검색한 결과를 읽고 답변을 정리하는 행위를 LLM이 자동으로 하는 것과 같습니다.

### 왜 RAG가 필요한가

LLM 단독 사용의 한계는 명확합니다.

| 문제 | 설명 | RAG의 해결 |
|------|------|-----------|
| **할루시네이션** | 학습 데이터에 없는 내용을 그럴듯하게 지어냄 | 실제 문서를 근거로 답변 |
| **지식 단절** | 학습 시점 이후 정보를 모름 | 실시간 수집 데이터 반영 |
| **도메인 한계** | 범용 모델은 특정 도메인 지식이 부족 | 도메인 특화 문서 제공 |
| **출처 부재** | 답변의 근거를 알 수 없음 | 검색된 문서 URL 제공 가능 |

주식 시장 분석처럼 **정확성과 최신성이 동시에 요구되는 도메인**에서 RAG는 거의 필수적입니다. "SK하이닉스 최근 이슈"를 물었을 때, 6개월 전 학습 데이터가 아니라 오늘 수집된 뉴스를 기반으로 답변해야 하니까요.

### RAG의 핵심 구성요소

RAG 시스템을 구축하려면 크게 세 가지가 필요합니다.

**1. 임베딩(Embedding) 모델** — 텍스트를 고차원 벡터로 변환합니다. 의미가 비슷한 문장은 벡터 공간에서 가까운 위치에 놓이게 됩니다.

**2. 벡터 데이터베이스(Vector DB)** — 임베딩된 벡터를 저장하고, 유사도 검색을 수행합니다. "이 질문과 가장 비슷한 문서 5개를 찾아줘"라는 요청을 밀리초 단위로 처리합니다.

**3. LLM(Large Language Model)** — 검색된 문서와 질문을 받아 최종 답변을 생성합니다.

---

## 왜 n8n인가: 개발자가 워크플로 도구를 선택한 이유

### 일반적인 개발자의 접근

보통 개발자가 RAG 파이프라인을 구축한다면 이렇게 합니다.

```python
# 일반적인 Python 기반 수집 코드
def ingest_pipeline(query: str, days: int):
    news = fetch_naver_news(query, days)
    cafe = fetch_naver_cafe(query, days)
    dart = fetch_dart_filings(query, days)

    documents = normalize(news + cafe + dart)
    segments = chunk_documents(documents)

    for seg in segments:
        vector = embed(seg.text)
        qdrant.upsert(id=hash(seg), vector=vector, payload=seg.metadata)
```

LangChain의 `DocumentLoader` → `TextSplitter` → `VectorStore` 체인을 Python으로 작성하는 것이 정석입니다. **코드로 짜면 디버깅이 쉽고, 버전 관리가 되고, 테스트를 붙일 수 있습니다.** 그래서 대부분의 RAG 튜토리얼과 프로덕션 사례가 이 방식입니다.

### 그런데 왜 n8n을 선택했는가

솔직히 말하면, 처음부터 n8n을 쓰려고 했던 건 아닙니다. 스터디에서 RAG 개발 도구로 n8n, Langflow, Flowise 등이 소개되었고, 과제 삼아 한번 써보자는 가벼운 마음이었습니다. 그런데 실제로 적용해보니 **데이터 수집(Ingestion) 파이프라인**에는 n8n이 꽤 합리적인 선택이었습니다.

이유를 정리하면 다음과 같습니다.

#### 1. 수집 파이프라인은 "글루 코드"의 집합이다

뉴스 API 호출, 카페 API 호출, DART API 호출 — 이 세 가지는 각각 인증 방식이 다르고, 응답 포맷이 다르고, 에러 처리 방식이 다릅니다. Python으로 짜면 `requests` 호출 + JSON 파싱 + 에러 핸들링의 반복입니다. 비즈니스 로직이라고 할 것도 없는 **글루 코드(glue code)**입니다.

n8n은 이런 HTTP 요청-응답 처리를 노드 단위로 시각화하기 때문에, 어떤 API에서 에러가 났는지 워크플로 UI에서 바로 확인됩니다.

#### 2. 병렬 수집이 선언적이다

```
Build Date Window
    ├→ Fetch Naver News   (병렬)
    ├→ Fetch Naver Cafe   (병렬)
    └→ Fetch DART List    (병렬)
        ↓
    Normalize Merge
```

n8n에서는 하나의 노드 출력을 여러 노드에 연결하면 **자동으로 병렬 실행**됩니다. Python에서 `asyncio.gather()`나 `ThreadPoolExecutor`로 구현할 수 있지만, 시각적으로 "이 세 API가 동시에 호출된다"는 것이 즉각 드러나는 건 n8n의 장점입니다.

#### 3. RAG의 "두뇌"는 코드로, "손발"은 워크플로로

제가 n8n을 쓴 건 **데이터 수집(Ingestion)** 파이프라인뿐입니다. RAG의 핵심인 **검색 + 생성(Retrieval + Generation)** 로직은 전부 Python/LangChain으로 작성했습니다. 이렇게 역할을 분리한 이유가 있습니다.

| 역할 | 특성 | 도구 |
|------|------|------|
| **데이터 수집** | 외부 API 호출, 포맷 정규화, 스케줄링 | n8n (워크플로) |
| **임베딩 + 저장** | 벡터 변환, DB upsert | n8n 또는 Shell Script |
| **검색 + 생성** | 유사도 검색, 프롬프트 구성, LLM 추론 | FastAPI + LangChain (코드) |

수집은 "정해진 순서대로 API를 호출하고 결과를 합치는" 정형 작업이라 워크플로 도구가 잘 맞고, 검색과 생성은 LLM 체인 구성, 프롬프트 엔지니어링, 세션 관리 등 **코드로 섬세하게 제어**해야 하는 영역이라 LangChain이 적합합니다.

#### 4. 비개발자와의 협업 가능성

공모전 팀에 비개발자 팀원이 있었습니다. "뉴스 수집 소스를 추가하고 싶다"거나 "DART 대신 다른 공시 API를 써보자"라는 요청이 올 때, n8n 워크플로에서 노드 하나를 추가하는 것과 Python 코드를 수정하는 것의 진입 장벽은 다릅니다.

### n8n의 한계도 분명하다

물론 한계도 있습니다.

- **복잡한 로직**에는 부적합: n8n의 Function 노드에서 JavaScript를 쓸 수 있지만, 테스트·타입 체크·리팩토링이 어렵습니다.
- **버전 관리**: 워크플로 JSON을 Git으로 관리할 수 있지만, diff가 읽기 어렵습니다.
- **디버깅**: 실행 로그가 UI에서 제공되지만, breakpoint나 step-through는 불가합니다.

이런 이유로 저는 수집 파이프라인의 **대안(fallback)**으로 동일한 로직의 Shell Script(`ingest_run.sh`)도 만들어두었습니다. n8n이 내려가도 CLI로 수집을 돌릴 수 있도록요.

---

## 전체 아키텍처

### 시스템 구성도

[![jemog-eobs-eum-(4).png](https://i.postimg.cc/WbBFnnzs/jemog-eobs-eum-(4).png)](https://postimg.cc/fJvRWYYP)
### 기술 스택 선정 이유

| 컴포넌트 | 선택 | 이유 |
|---------|------|------|
| **Vector DB** | Qdrant | HNSW 기반 고속 검색, 페이로드 필터링 지원, Docker 이미지 제공 |
| **LLM** | Exaone 3.5 7.8B (Ollama) | 한국어 성능 우수, 로컬 추론 가능, API 비용 제로 |
| **Embedding** | BGE-m3 (Ollama) | 다국어 지원, 1024차원, 한국어 금융 텍스트에 적합 |
| **프레임워크** | FastAPI + LangChain | Runnable 체인 조합, LangServe 통합, 비동기 지원 |
| **워크플로** | n8n | 데이터 수집 오케스트레이션, 시각적 파이프라인 관리 |
| **오케스트레이션** | Docker Compose | 5개 서비스 원커맨드 배포 |

### Docker Compose 구성

```yaml
version: "3.9"

services:
  qdrant:
    image: qdrant/qdrant:latest
    environment:
      QDRANT__SERVICE__API_KEY: "${QDRANT_API_KEY}"
    volumes:
      - ./volumes/qdrant:/qdrant/storage
    ports:
      - "${QDRANT_PORT:-6333}:6333"

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: "${POSTGRES_USER}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
      POSTGRES_DB: "${POSTGRES_DB}"

  n8n:
    image: n8nio/n8n:latest
    depends_on: [postgres]
    environment:
      DB_TYPE: "postgresdb"
      NAVER_CLIENT_ID: "${NAVER_CLIENT_ID}"
      NAVER_CLIENT_SECRET: "${NAVER_CLIENT_SECRET}"
      DART_API_KEY: "${DART_API_KEY}"
      QDRANT_API_KEY: "${QDRANT_API_KEY}"
      OLLAMA_URL: "http://ollama:11434"
    ports:
      - "${N8N_PORT}:5678"

  ollama:
    image: ollama/ollama:latest
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    volumes:
      - ./volumes/ollama:/root/.ollama

  apex-llm-service:
    build:
      context: ./apex-llm-service
    depends_on: [qdrant, ollama]
    environment:
      QDRANT_URL: "http://qdrant:6333"
      QDRANT_COLLECTION: "stock_segments"
      OLLAMA_URL: "http://host.docker.internal:11434"
    ports:
      - "8000:8000"
    command: uvicorn main:app --host 0.0.0.0 --port 8000
```

`docker-compose up -d` 한 줄로 전체 스택이 올라갑니다. 모든 서비스가 Docker 네트워크 안에서 통신하므로, 외부 API 키를 제외하면 인터넷 연결 없이도 RAG 추론이 가능합니다.

---

## 데이터 수집 파이프라인 (n8n Workflow)

### 워크플로 전체 흐름

n8n으로 구현한 수집 워크플로 `ingest_docs.workflow.json`의 흐름입니다.

[![jemog-eobs-eum-(5).png](https://i.postimg.cc/B67dr3hZ/jemog-eobs-eum-(5).png)](https://postimg.cc/hzxymkWN)

### n8n 워크플로 JSON 구조 이해하기

n8n 워크플로는 JSON 파일로 저장됩니다. 기본 구조는 다음과 같습니다.

```json
{
  "name": "Ingest Stock Documents",
  "nodes": [
    {
      "id": "node-1",
      "type": "n8n-nodes-base.webhook",
      "name": "Webhook Trigger",
      "parameters": { "path": "ingest-docs", "method": "POST" },
      "position": [100, 200]
    },
    {
      "id": "node-2",
      "type": "n8n-nodes-base.function",
      "name": "Build Date Window",
      "parameters": { "code": "..." },
      "position": [300, 200]
    }
  ],
  "connections": {
    "Webhook Trigger": { "main": [[{ "node": "Build Date Window", "type": "main", "index": 0 }]] }
  }
}
```

- **nodes**: 각 노드의 타입, 이름, 파라미터, UI 위치
- **connections**: 노드 간 연결 관계 (어떤 노드의 출력이 어떤 노드의 입력으로 가는지)

이 JSON을 n8n UI에서 import하면 시각적 워크플로가 복원됩니다.

### 핵심 노드 상세

#### 1. Webhook Trigger: HTTP 엔드포인트 설정

```json
{
  "type": "n8n-nodes-base.webhook",
  "name": "Webhook Trigger",
  "parameters": {
    "path": "ingest-docs",
    "method": "POST",
    "responseMode": "whenLastNodeFinished",
    "responseData": "lastNode",
    "authentication": "headerAuth",
    "headerAuth": {
      "name": "X-API-Key",
      "value": "={{$env.N8N_WEBHOOK_API_KEY}}"
    }
  }
}
```

**설정 설명:**
- **`path`**: `/webhook/ingest-docs` 경로로 호출 가능
- **`responseMode: whenLastNodeFinished`**: 워크플로 전체가 끝날 때까지 기다렸다가 응답. "lastNodeExecuted"로 하면 다음 노드 실행 즉시 응답.
- **`authentication: headerAuth`**: API 키 기반 인증. 외부에서 무단 호출 방지.

**호출 예시:**
```bash
curl -X POST http://localhost:5678/webhook/ingest-docs \
  -H "X-API-Key: your-secret-key" \
  -H "Content-Type: application/json" \
  -d '{"query": "SK하이닉스", "days": 7}'
```

#### 2. Build Date Window: 날짜 범위 계산

```json
{
  "type": "n8n-nodes-base.function",
  "name": "Build Date Window",
  "parameters": {
    "functionCode": "const query = items[0].json.query;\nconst days = items[0].json.days || 7;\nconst endDate = new Date();\nconst startDate = new Date();\nstartDate.setDate(endDate.getDate() - days);\n\nreturn items.map(it => ({\n  json: {\n    query,\n    start: startDate.toISOString().slice(0, 10),\n    end: endDate.toISOString().slice(0, 10),\n    days\n  }\n}));"
  }
}
```

**왜 필요한가:**
- 네이버 API와 DART API는 모두 날짜 범위를 요구합니다.
- Webhook에서 받은 `days` 파라미터를 "YYYY-MM-DD" 형식의 `start`, `end`로 변환.

**출력 예시:**
```json
{
  "query": "SK하이닉스",
  "start": "2025-08-08",
  "end": "2025-08-15",
  "days": 7
}
```

#### 3. HTTP Request 노드: 네이버 뉴스 API 호출

```json
{
  "type": "n8n-nodes-base.httpRequest",
  "name": "Fetch Naver News",
  "parameters": {
    "url": "https://openapi.naver.com/v1/search/news.json",
    "method": "GET",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [
        { "name": "query", "value": "={{$json.query}}" },
        { "name": "display", "value": "100" },
        { "name": "sort", "value": "date" }
      ]
    },
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "X-Naver-Client-Id", "value": "={{$env.NAVER_CLIENT_ID}}" },
        { "name": "X-Naver-Client-Secret", "value": "={{$env.NAVER_CLIENT_SECRET}}" }
      ]
    },
    "options": {
      "timeout": 30000,
      "retry": {
        "enabled": true,
        "maxRetries": 3,
        "retryInterval": 1000
      }
    }
  }
}
```

**핵심 포인트:**

1. **`={{$json.query}}`**: n8n 표현식. 이전 노드의 JSON 데이터에서 `query` 필드를 참조.
2. **`={{$env.NAVER_CLIENT_ID}}`**: 환경변수 참조. Docker Compose의 `environment`에서 주입.
3. **`retry` 설정**: API 호출 실패 시 1초 간격으로 3회 재시도. 네이버 API는 간헐적으로 500 에러를 반환하므로 필수.
4. **`timeout: 30000`**: 30초 타임아웃. 기본값(10초)은 너무 짧아서 대량 데이터 조회 시 실패.

**응답 구조:**
```json
{
  "lastBuildDate": "...",
  "total": 1234,
  "start": 1,
  "display": 100,
  "items": [
    {
      "title": "SK하이닉스, HBM3E 본격 양산...",
      "originallink": "https://...",
      "link": "https://news.naver.com/...",
      "description": "SK하이닉스가 차세대 고대역폭메모리...",
      "pubDate": "Tue, 15 Aug 2025 10:30:00 +0900"
    }
  ]
}
```

#### 4. Normalize Merge: 3개 소스의 포맷 통합

네이버 뉴스, 네이버 카페, DART는 각각 응답 구조가 다릅니다. 이 노드에서 통일된 형태로 정규화합니다.

```javascript
// n8n Function 노드 - Normalize Merge
function sanitize(s) {
  return (s || "")
    .replace(/<[^>]+>/g, " ")    // HTML 태그 제거
    .replace(/\s+/g, " ")        // 연속 공백 압축
    .trim();
}

const merged = [];

// 뉴스
if (items[0]?.json?.body?.items) {
  for (const it of items[0].json.body.items) {
    merged.push({
      source: "news",
      title: sanitize(it.title),
      text: sanitize(it.description),
      url: it.originallink || it.link,
      date: new Date().toISOString().slice(0, 10)
    });
  }
}

// 카페
if (items[1]?.json?.body?.items) {
  for (const it of items[1].json.body.items) {
    merged.push({
      source: "cafe",
      title: sanitize(it.title),
      text: sanitize(it.description),
      url: it.link,
      date: new Date().toISOString().slice(0, 10)
    });
  }
}

// DART 공시 목록
if (items[2]?.json?.body?.list) {
  for (const it of items[2].json.body.list) {
    merged.push({
      source: "dart",
      title: sanitize(it.report_nm),
      text: `${it.corp_name} ${it.report_nm}`,
      url: `https://dart.fss.or.kr/dsaf001/main.do?rcpNo=${it.rcept_no}`,
      date: (it.rcept_dt || "").replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3')
    });
  }
}

return merged.map(m => ({ json: m }));
```

#### Split → Segments: 문맥을 보존하는 텍스트 분할

RAG에서 청킹(chunking) 전략은 검색 품질에 직접적인 영향을 줍니다. 너무 크면 노이즈가 섞이고, 너무 작으면 문맥이 끊깁니다.

```javascript
// n8n Function 노드 - Split → Segments
function splitText(t) {
  const chunks = [];
  const paras = t.split(/\n+/).map(s => s.trim()).filter(Boolean);

  for (const p of paras) {
    if (p.length <= 800) {
      chunks.push(p);          // 800자 이하면 그대로
    } else {
      // 문장 단위로 분리 후 800자까지 묶기
      const sents = p.split(/(?<=[.!?])\s+/);
      let buf = "";
      for (const s of sents) {
        if ((buf + s).length > 800) {
          if (buf) { chunks.push(buf); buf = ""; }
        }
        buf += (buf ? " " : "") + s;
      }
      if (buf) chunks.push(buf);
    }
  }
  return chunks;
}
```

**설계 포인트:**
- **문단 우선 분할**: `\n\n`으로 먼저 나눠서 자연스러운 단락 경계를 존중합니다.
- **문장 경계 보존**: 800자를 넘을 때만 문장 단위(`(?<=[.!?])\s+`)로 재분할합니다. 문장 중간에서 끊기지 않습니다.
- **800자 상한**: BGE-m3 임베딩 모델이 문단 길이 텍스트에서 좋은 성능을 보이는 점을 고려했습니다.

#### Build Qdrant Point: 중복 방지를 위한 해시 기반 ID

```javascript
const crypto = require('crypto');

return items.map(it => {
  const j = it.json;
  // 회사명 간이 추출
  const company = (j.title || "").match(/삼성전자|SK하이닉스|네이버|카카오|현대차/)
    ? RegExp.lastMatch : "";

  // 콘텐츠 해시 = 고유 ID → 동일 내용 재수집 시 덮어쓰기
  const id = crypto.createHash('sha256')
    .update(`${company}|${j.date}|${j.source}|${j.text}`)
    .digest('hex');

  return {
    json: {
      point: {
        id,
        vector: it.json.embedding,
        payload: {
          company, date: j.date, source: j.source,
          url: j.url, title: j.title, text: j.text,
          content_hash: id
        }
      }
    }
  };
});
```

`SHA-256(회사명|날짜|소스|텍스트)` 조합으로 point ID를 만들기 때문에, **같은 뉴스를 여러 번 수집해도 Qdrant의 upsert가 자동으로 중복을 처리**합니다.

---

## Shell Script 기반 Fallback 시스템

### 왜 n8n과 별도로 Shell Script를 만들었는가

n8n 워크플로가 있는데 왜 Shell Script를 또 만들었을까요? 이유는 세 가지입니다.

#### 1. 의존성 분리 (Dependency Isolation)

n8n은 훌륭하지만, **추가 의존성**입니다.

- n8n 컨테이너가 내려가면 수집 파이프라인 전체가 멈춥니다.
- n8n 워크플로 JSON이 손상되거나 삭제되면 복구가 어렵습니다.
- PostgreSQL (n8n 워크플로 DB)에 문제가 생기면 워크플로 실행 불가.

Shell Script는 **curl, jq, python만 있으면 돌아갑니다**. Docker Compose 전체가 내려가도, Qdrant와 Ollama만 살아있으면 수집이 가능합니다.

```bash
# n8n 없이 CLI로 직접 수집
./scripts/ingest_run.sh "SK하이닉스" 7
```

#### 2. 디버깅과 커스터마이징

n8n 워크플로는 UI에서 편집하지만, **세밀한 제어**가 어렵습니다.

- 각 API 응답을 파일로 저장하고 싶다면? Shell에서는 `tee` 한 줄이면 끝.
- 특정 조건에서만 임베딩을 스킵하고 싶다면? Shell의 `if` 문이 더 직관적.
- 병렬 처리 수를 동적으로 조절하고 싶다면? `xargs -P`로 간단히 제어.

```bash
# 응답을 파일로 저장하며 디버깅
curl ... | tee /tmp/naver_news_response.json | jq '.items[]'
```

#### 3. 프로덕션 배포 시나리오

프로덕션 환경에서는 **Cron이나 Airflow**로 스케줄링하는 경우가 많습니다.

```cron
# 매일 오전 6시에 주요 종목 수집
0 6 * * * /app/scripts/ingest_run.sh "삼성전자,SK하이닉스,네이버" 1
```

n8n의 워크플로 스케줄러도 있지만, **기존 인프라(Cron/Airflow)에 통합**하려면 Shell Script가 더 적합합니다.

### ingest_run.sh 전체 코드

```bash
#!/usr/bin/env bash
set -euo pipefail  # 에러 발생 시 즉시 종료, 미정의 변수 사용 금지

# ===== 환경변수 로드 =====
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

# .env 파일에서 환경변수 읽기
if [[ -f "${PROJECT_ROOT}/.env" ]]; then
  export $(grep -v '^#' "${PROJECT_ROOT}/.env" | xargs)
fi

# ===== 설정 =====
NAVER_CLIENT_ID="${NAVER_CLIENT_ID:?NAVER_CLIENT_ID not set}"
NAVER_CLIENT_SECRET="${NAVER_CLIENT_SECRET:?NAVER_CLIENT_SECRET not set}"
DART_API_KEY="${DART_API_KEY:?DART_API_KEY not set}"

OLLAMA_BASE_URL="${OLLAMA_BASE_URL:-http://localhost:11434}"
QDRANT_BASE_URL="${QDRANT_BASE_URL:-http://localhost:6333}"
QDRANT_API_KEY="${QDRANT_API_KEY:-}"
QDRANT_COLLECTION="${QDRANT_COLLECTION:-stock_segments}"

EMBED_MODEL="${EMBED_MODEL:-bge-m3}"
SENTIMENT_DIR="${PROJECT_ROOT}/sentiment"
MODEL_PATH="${SENTIMENT_DIR}/sentiment_model.npz"

# ===== 인자 파싱 =====
QUERY="${1:?Usage: $0 <query> <days>}"
DAYS="${2:-7}"

# ===== 날짜 계산 (macOS / Linux 호환) =====
if date -v-1d >/dev/null 2>&1; then
  # macOS
  START_DATE=$(date -v-"${DAYS}"d +%Y-%m-%d)
else
  # Linux
  START_DATE=$(date -d "${DAYS} days ago" +%Y-%m-%d)
fi
END_DATE=$(date +%Y-%m-%d)

echo "[INFO] 수집 시작: query=${QUERY}, range=${START_DATE}~${END_DATE}"

# ===== 1. 네이버 뉴스 API 호출 =====
echo "[INFO] 네이버 뉴스 수집 중..."
NEWS_JSON=$(curl -sS -G "https://openapi.naver.com/v1/search/news.json" \
  --data-urlencode "query=${QUERY}" \
  --data-urlencode "display=100" \
  --data-urlencode "sort=date" \
  -H "X-Naver-Client-Id: ${NAVER_CLIENT_ID}" \
  -H "X-Naver-Client-Secret: ${NAVER_CLIENT_SECRET}" \
  | jq -c '.items[]')

# ===== 2. 네이버 카페 API 호출 =====
echo "[INFO] 네이버 카페 수집 중..."
CAFE_JSON=$(curl -sS -G "https://openapi.naver.com/v1/search/cafearticle.json" \
  --data-urlencode "query=${QUERY}" \
  --data-urlencode "display=100" \
  --data-urlencode "sort=date" \
  -H "X-Naver-Client-Id: ${NAVER_CLIENT_ID}" \
  -H "X-Naver-Client-Secret: ${NAVER_CLIENT_SECRET}" \
  | jq -c '.items[]')

# ===== 3. DART 공시 API 호출 =====
echo "[INFO] DART 공시 수집 중..."
DART_JSON=$(curl -sS -G "https://opendart.fss.or.kr/api/list.json" \
  --data-urlencode "crtfc_key=${DART_API_KEY}" \
  --data-urlencode "corp_name=${QUERY}" \
  --data-urlencode "bgn_de=${START_DATE//-/}" \
  --data-urlencode "end_de=${END_DATE//-/}" \
  --data-urlencode "page_count=100" \
  | jq -c '.list[]? // empty')

# ===== 4. 정규화: 통일된 포맷으로 변환 =====
echo "[INFO] 데이터 정규화 중..."
NORMALIZED_JSON=$(
  {
    # 뉴스 정규화
    printf '%s\n' "$NEWS_JSON" | jq -c '{
      source: "news",
      title: .title | gsub("<[^>]+>"; "") | gsub("\\s+"; " ") | rtrimstr(" "),
      text: .description | gsub("<[^>]+>"; "") | gsub("\\s+"; " "),
      url: .originallink // .link,
      date: (.pubDate | sub(" \\+0900$"; "") | strptime("%a, %d %b %Y %H:%M:%S") | strftime("%Y-%m-%d"))
    }'

    # 카페 정규화
    printf '%s\n' "$CAFE_JSON" | jq -c '{
      source: "cafe",
      title: .title | gsub("<[^>]+>"; "") | gsub("\\s+"; " "),
      text: .description | gsub("<[^>]+>"; "") | gsub("\\s+"; " "),
      url: .link,
      date: (now | strftime("%Y-%m-%d"))
    }'

    # DART 정규화
    printf '%s\n' "$DART_JSON" | jq -c '{
      source: "dart",
      title: .report_nm,
      text: (.corp_name + " " + .report_nm),
      url: ("https://dart.fss.or.kr/dsaf001/main.do?rcpNo=" + .rcept_no),
      date: (.rcept_dt | tostring | sub("([0-9]{4})([0-9]{2})([0-9]{2})"; "\\1-\\2-\\3"))
    }'
  } | jq -s '.'
)

# ===== 5. 청킹 (Chunking) =====
echo "[INFO] 텍스트 분할 중..."
SEGMENTS_JSON=$(printf '%s' "$NORMALIZED_JSON" | python3 -c "
import sys, json, re

def split_text(text, max_len=800):
    chunks = []
    paras = [p.strip() for p in re.split(r'\n+', text) if p.strip()]

    for p in paras:
        if len(p) <= max_len:
            chunks.append(p)
        else:
            sents = re.split(r'(?<=[.!?])\\s+', p)
            buf = ''
            for s in sents:
                if len(buf + s) > max_len:
                    if buf: chunks.append(buf)
                    buf = s
                else:
                    buf += (' ' if buf else '') + s
            if buf: chunks.append(buf)
    return chunks

docs = json.load(sys.stdin)
segments = []

for doc in docs:
    chunks = split_text(doc['title'] + '\\n\\n' + doc['text'])
    for chunk in chunks:
        segments.append({**doc, 'text': chunk})

print(json.dumps(segments))
")

SEGMENT_COUNT=$(printf '%s' "$SEGMENTS_JSON" | jq 'length')
echo "[INFO] 총 ${SEGMENT_COUNT}개 세그먼트 생성"

# ===== 6. 임베딩 + 감정 분석 + Qdrant 업서트 =====
echo "[INFO] 임베딩 및 Qdrant 업서트 중..."

PROCESSED=0
printf '%s' "$SEGMENTS_JSON" | jq -c '.[]' | while IFS= read -r row; do
  text="$(printf '%s' "$row" | jq -r '.text')"
  source="$(printf '%s' "$row" | jq -r '.source')"
  title="$(printf '%s' "$row" | jq -r '.title')"
  url="$(printf '%s' "$row" | jq -r '.url')"
  date="$(printf '%s' "$row" | jq -r '.date')"

  # 회사명 간이 추출
  company=$(printf '%s' "$title" | grep -oE '삼성전자|SK하이닉스|네이버|카카오|현대차' | head -1 || echo "Unknown")

  # 콘텐츠 해시 (중복 방지 ID)
  content_hash=$(printf '%s|%s|%s|%s' "$company" "$date" "$source" "$text" | shasum -a 256 | awk '{print $1}')

  # 1) Ollama 임베딩
  vector=$(curl -sS -X POST "${OLLAMA_BASE_URL}/api/embeddings" \
    -H "Content-Type: application/json" \
    -d "$(jq -nc --arg model "${EMBED_MODEL}" --arg prompt "$text" '{model:$model, prompt:$prompt}')" \
    | jq '.embedding')

  # 2) 감정 분석
  if [[ -f "$MODEL_PATH" ]]; then
    sentiment_json=$(printf '%s\n' "$vector" | python3 "${SENTIMENT_DIR}/infer_sentiment_vector.py" "$MODEL_PATH")
    sentiment_label=$(printf '%s' "$sentiment_json" | jq -r '.label')
    sentiment_confidence=$(printf '%s' "$sentiment_json" | jq -r '.confidence')
  else
    sentiment_label="neutral"
    sentiment_confidence="0.5"
  fi

  # 3) 시장 관련도
  market_relevance=$(printf '%s\n' "$text" | python3 "${SENTIMENT_DIR}/market_relevance.py" | jq -r '.relevance')

  # 4) Qdrant 업서트
  curl -sS -X PUT "${QDRANT_BASE_URL}/collections/${QDRANT_COLLECTION}/points" \
    -H "Content-Type: application/json" \
    -H "api-key: ${QDRANT_API_KEY}" \
    -d "$(jq -nc \
      --arg id "$content_hash" \
      --argjson vector "$vector" \
      --arg company "$company" \
      --arg source "$source" \
      --arg title "$title" \
      --arg text "$text" \
      --arg url "$url" \
      --arg date "$date" \
      --arg sentiment_label "$sentiment_label" \
      --arg sentiment_confidence "$sentiment_confidence" \
      --arg market_relevance "$market_relevance" \
      '{points: [{
        id: $id,
        vector: $vector,
        payload: {
          company: $company, source: $source, title: $title, text: $text, url: $url, date: $date,
          sentiment_label: $sentiment_label, sentiment_confidence: ($sentiment_confidence | tonumber),
          market_relevance: ($market_relevance | tonumber), content_hash: $id
        }
      }]}'
    )" >/dev/null

  PROCESSED=$((PROCESSED + 1))
  echo "[PROGRESS] ${PROCESSED}/${SEGMENT_COUNT} 완료"
done

echo "[SUCCESS] 수집 완료: ${SEGMENT_COUNT}개 세그먼트 처리"
```

### Shell Script의 핵심 설계 포인트

#### 1. `set -euo pipefail`: Strict Mode

```bash
set -e  # 명령어 실패 시 즉시 종료
set -u  # 미정의 변수 사용 시 에러
set -o pipefail  # 파이프 중간에 실패해도 에러 코드 전파
```

이 세 줄이 없으면, curl 실패 → jq가 빈 데이터 처리 → 에러 없이 종료되는 사일런트 실패가 발생합니다.

#### 2. jq로 JSON 파이프라인 구축

```bash
# JSON 파싱, 변환, 병합을 모두 jq로 처리
NEWS_JSON | jq -c '{source: "news", title: .title | gsub("<[^>]+>"; ""), ...}'
```

Python의 `json` 모듈처럼 쓸 수 있지만, **별도 스크립트 없이 inline**으로 처리됩니다.

#### 3. Python과의 협업

복잡한 로직(청킹, 감정 분석)은 Python으로 위임하고, Shell은 오케스트레이션만 담당합니다.

```bash
# Shell → Python → Shell 파이프라인
SEGMENTS_JSON=$(printf '%s' "$NORMALIZED_JSON" | python3 -c "...")
```

이 방식의 장점은, Python 코드를 별도 파일로 분리할 수도 있고, `-c`로 inline할 수도 있다는 점입니다.

#### 4. macOS / Linux 호환성

```bash
if date -v-1d >/dev/null 2>&1; then
  START_DATE=$(date -v-"${DAYS}"d +%Y-%m-%d)  # macOS
else
  START_DATE=$(date -d "${DAYS} days ago" +%Y-%m-%d)  # Linux
fi
```

macOS의 `date`는 BSD 버전, Linux는 GNU 버전이라 옵션이 다릅니다. 양쪽 모두 지원하도록 분기 처리했습니다.

---

## 기술 선택의 배경과 트레이드오프

### LLM: Ollama + Exaone 3.5 vs OpenAI API

| 요소 | Ollama + Exaone 3.5 (선택) | OpenAI GPT-4o |
|------|-------------------------|---------------|
| **비용** | 초기 하드웨어 비용만 (GPU 권장) | 호출당 과금 (~$0.03/1K 토큰) |
| **지연시간** | 로컬: 2-5초 (GPU), 10-20초 (CPU) | API: 3-8초 + 네트워크 |
| **한국어 성능** | ★★★★☆ (Exaone은 한국어 특화) | ★★★★★ (거의 완벽) |
| **JSON 출력 안정성** | ★★★☆☆ (파서 필요) | ★★★★★ (거의 완벽) |
| **프라이버시** | 완전 온프레미스 | 데이터 외부 전송 |
| **스케일** | GPU 메모리 제한 (동시 1-2 요청) | API 속도 제한까지 무제한 |

**왜 Exaone 3.5를 선택했는가:**

1. **제로 비용**: 공모전 프로젝트라 예산이 없었습니다. GPT-4로 하루 100회 요약 생성 시 월 $90+ 발생.
2. **금융 데이터 프라이버시**: 실제 뉴스·공시 텍스트를 OpenAI 서버로 보내는 것이 데이터 보호 관점에서 꺼려졌습니다.
3. **한국어 특화**: Exaone은 LG AI연구원이 한국어 코퍼스로 집중 학습한 모델. "실적이 컨센서스를 상회"같은 금융 표현도 잘 이해합니다.

**Trade-off와 대응:**

- **낮은 JSON 출력 품질** → `strict_json_parser()`로 복구
- **느린 추론 속도** → `num_predict=256`으로 토큰 수 제한, CPU 대신 GPU 사용 (NVIDIA GTX 1660 이상 권장)
- **동시 요청 제한** → Ollama는 기본적으로 단일 요청 처리. 멀티 GPU나 vLLM으로 배치 추론 가능하지만, 이 프로젝트 규모에서는 필요 없음.

**실제 성능 측정:**

```bash
# MacBook Pro M2 (CPU 모드)
- 요약 생성: 평균 18.5초
- 피드백 대화: 평균 12.3초

# Ubuntu 22.04 + NVIDIA RTX 3060 (GPU 모드)
- 요약 생성: 평균 3.2초
- 피드백 대화: 평균 2.1초
```

CPU로도 충분히 쓸 만하지만, GPU가 있으면 체감 품질이 크게 개선됩니다.

### 임베딩 모델: BGE-m3 vs OpenAI text-embedding-3-large

| 요소 | BGE-m3 (선택) | OpenAI text-embedding-3-large |
|------|--------------|------------------------------|
| **벡터 차원** | 1024 (설정 가능) | 3072 (large) / 1536 (small) |
| **한국어 지원** | ★★★★☆ (다국어 모델) | ★★★★☆ |
| **비용** | 무료 (로컬) | $0.00013 / 1K 토큰 |
| **속도** | CPU: ~500ms/doc, GPU: ~50ms/doc | API: ~300ms/doc + 네트워크 |
| **오프라인** | 가능 | 불가능 (API 필수) |

**왜 BGE-m3인가:**

1. **다국어 성능**: BGE-m3는 MTEB (Massive Text Embedding Benchmark)에서 한국어 검색 태스크 1위권.
2. **1024차원**: OpenAI의 3072차원보다 작아서 Qdrant 인덱싱이 빠르고, 메모리도 절약.
3. **Ollama 지원**: `ollama pull bge-m3` 한 줄로 설치. LangChain에서도 즉시 사용 가능.

**대안 후보들:**

- **OpenAI text-embedding-3-small**: 가장 무난한 선택. 한국어도 잘 되고, 비용도 저렴 ($0.00002/1K). 하지만 온프레미스 제약으로 탈락.
- **multilingual-e5-large**: Hugging Face에서 직접 로드 가능. BGE-m3와 비슷한 성능이지만, Ollama 통합이 없어 LangChain 설정이 복잡.
- **KoBERT**: 한국어 특화지만, 문장 임베딩보다 토큰 분류에 강점. RAG에는 부적합.

### 벡터 DB: Qdrant vs Pinecone vs Weaviate vs Chroma

| 요소 | Qdrant (선택) | Pinecone | Weaviate | Chroma |
|------|--------------|----------|----------|--------|
| **배포 방식** | Self-hosted + Cloud | Cloud only | Self-hosted + Cloud | Self-hosted |
| **무료 티어** | 무제한 (로컬) | 1GB (Cloud) | 무제한 (로컬) | 무제한 (로컬) |
| **성능** | HNSW, 10M 벡터에서 sub-100ms | HNSW, 비슷함 | HNSW, 비슷함 | 소규모에 적합 |
| **페이로드 필터링** | ★★★★★ | ★★★★☆ | ★★★★★ | ★★★☆☆ |
| **LangChain 통합** | `langchain-qdrant` | `langchain-pinecone` | `langchain-weaviate` | `langchain-chroma` |
| **Docker 이미지** | 공식 제공 | N/A | 공식 제공 | 공식 제공 |

**왜 Qdrant인가:**

1. **페이로드 필터링 성능**: Qdrant는 벡터 검색과 메타데이터 필터링을 **동시에** 수행합니다.

   ```python
   # "SK하이닉스" + "최근 7일" + "뉴스 소스만" 필터링
   vectorstore.similarity_search(
       "SK하이닉스",
       k=5,
       filter={"must": [
           {"key": "company", "match": {"value": "SK하이닉스"}},
           {"key": "date", "range": {"gte": "2025-08-08"}},
           {"key": "source", "match": {"value": "news"}}
       ]}
   )
   ```

   Chroma는 이런 복잡한 필터가 느리고, Pinecone은 Cloud만 가능해서 온프레미스 제약에 맞지 않습니다.

2. **HNSW 인덱스**: Hierarchical Navigable Small World 알고리즘으로, 수백만 벡터에서도 밀리초 단위 검색. 이 프로젝트는 수천 개 벡터라 오버스펙이지만, 확장성을 고려했습니다.

3. **gRPC + HTTP API**: Python SDK뿐 아니라 REST API도 제공. Shell Script에서 `curl`로 바로 업서트 가능.

4. **스냅샷 & 백업**: Qdrant는 컬렉션을 `.snapshot` 파일로 저장 가능. Git에 커밋하거나 S3에 백업 가능.

**Weaviate와의 비교:**

Weaviate도 강력한 후보였습니다. GraphQL 쿼리, 자동 스키마 추론 등 기능이 많습니다. 하지만:

- LangChain 통합이 Qdrant보다 덜 성숙 (2025년 기준)
- 설정이 복잡 (모듈 시스템, 벡터라이저 설정)
- 이 프로젝트에는 Qdrant의 단순함이 더 적합

### Docker Compose vs Kubernetes

| 요소 | Docker Compose (선택) | Kubernetes |
|------|-----------------------|-----------|
| **복잡도** | YAML 50줄 | YAML 200줄+ (Deployment, Service, ConfigMap, ...) |
| **로컬 개발** | `docker-compose up` | Minikube/Kind 설정 필요 |
| **프로덕션** | 단일 서버 배포 | 멀티 노드 클러스터 |
| **오토 스케일링** | 없음 | HPA (Horizontal Pod Autoscaler) |

**왜 Docker Compose인가:**

1. **프로젝트 규모**: 5개 컨테이너, 동시 접속 10명 이하 → Kubernetes는 오버킬.
2. **로컬 개발 우선**: 공모전 데모 환경은 MacBook Pro 1대. Kubernetes 없이도 충분.
3. **학습 곡선**: 팀원 중 Docker는 알아도 K8s는 모르는 사람이 많았음.

**프로덕션 배포 시나리오:**

- **1-10 TPS**: Docker Compose + Nginx 리버스 프록시
- **10-100 TPS**: Docker Swarm (Compose 문법 거의 그대로 사용 가능)
- **100+ TPS**: Kubernetes + Prometheus + Grafana (모니터링 필수)

이 프로젝트는 1단계에 해당하므로, Docker Compose가 최적입니다.

---

## 감정 분석 시스템

### 왜 직접 만들었는가

범용 감정 분석 API(ex. Google NLP, AWS Comprehend)를 쓸 수도 있었지만, 두 가지 이유로 직접 구축했습니다.

1. **온프레미스 제약**: 클라우드 API 호출 없이 로컬에서 돌아가야 합니다.
2. **도메인 특화**: "실적이 시장 기대를 상회"는 긍정, "조정 압력이 강화"는 부정 — 금융 도메인의 감정 판단은 일반 감정 분석과 다릅니다.

### 학습 파이프라인

임베딩 벡터를 입력으로 받는 **로지스틱 회귀** 분류기를 직접 구현했습니다.

```python
# sentiment/train_sentiment.py

def train_logistic_regression(X, y, lr=0.1, epochs=200):
    """배치 경사하강 로지스틱 회귀"""
    N, D = X.shape
    w = np.zeros(D, dtype=np.float32)
    b = 0.0

    for epoch in range(1, epochs + 1):
        z = X @ w + b
        p = sigmoid(z)

        # Binary cross-entropy loss
        eps = 1e-7
        loss = -np.mean(y * np.log(p + eps) + (1 - y) * np.log(1 - p + eps))

        # Gradient descent
        diff = (p - y)
        w -= lr * (X.T @ diff) / N
        b -= lr * float(np.mean(diff))

    return w, b
```

**학습 데이터**: 한국 주식 시장 관련 문장 200여 건을 pos/neg으로 직접 라벨링했습니다.

```csv
text,label
실적이 시장 기대를 크게 상회하며 주가가 강세를 보이고 있다,pos
AI 서버용 메모리 수요가 급증하면서 관련 업체들의 실적 개선이 기대된다,pos
반도체 업황 부진이 지속되며 조정 압력이 강화되고 있다,neg
```

**전체 과정:**
1. CSV에서 텍스트 + 라벨 로드
2. Ollama BGE-m3로 각 텍스트를 1024차원 벡터로 임베딩
3. Z-score 정규화 (평균 0, 표준편차 1)
4. 로지스틱 회귀 학습 (200 epoch, learning rate 0.1)
5. 가중치(w), 편향(b), 정규화 파라미터(mean, std)를 `.npz`로 저장

### 추론

```python
# sentiment/infer_sentiment_vector.py
def infer(vec, model_path):
    data = np.load(model_path)
    w, b, mean, std = data["w"], float(data["b"]), data["mean"], data["std"]

    x_norm = (vec - mean) / std        # 정규화
    z = float(x_norm @ w + b)          # 선형 결합
    prob = 1.0 / (1.0 + np.exp(-z))    # 시그모이드
    label = "pos" if prob >= 0.5 else "neg"

    return {"label": label, "confidence": prob}
```

벡터를 직접 입력받기 때문에, 임베딩 과정에서 이미 추출된 벡터를 재사용할 수 있어 **추론 시 임베딩 API를 다시 호출하지 않아도 됩니다**.

### 시장 관련도 점수

뉴스 중에는 "SK하이닉스 채용박람회 개최"처럼 주가와 무관한 PR성 기사도 많습니다. 이를 필터링하기 위해 키워드 기반 시장 관련도 점수를 계산합니다.

```python
# sentiment/market_relevance.py

# 시장/투자 키워드 (가산)
MARKET_PAT = re.compile(r"(주가|증시|코스피|실적|영업이익|매수|매도|HBM|DRAM|...)")

# 노이즈 키워드 (감산)
NOISE_PAT = re.compile(r"(채용박람회|사회공헌|캠퍼스|봉사활동|스포츠 후원|...)")

def score_market_relevance(text: str) -> float:
    m_hits = len(MARKET_PAT.findall(text.lower()))
    n_hits = len(NOISE_PAT.findall(text.lower()))

    if m_hits == 0:
        return 0.0

    raw = m_hits / float(m_hits + n_hits + 1)
    return max(0.2, min(1.0, raw))  # 0.2 ~ 1.0 클램핑
```

이렇게 계산된 `sentiment_label`, `sentiment_confidence`, `market_relevance`는 Qdrant 페이로드에 함께 저장되어, 나중에 검색 시 **감정이나 관련도로 필터링**할 수 있게 됩니다.

---

## FastAPI 서비스 상세 구현

### 왜 FastAPI인가

Python 웹 프레임워크 선택지는 Django, Flask, FastAPI 등이 있습니다. 이 프로젝트에서 FastAPI를 선택한 이유는 다음과 같습니다.

| 요구사항 | FastAPI의 장점 |
|---------|--------------|
| **LangChain 통합** | LangServe가 FastAPI 기반으로 설계됨. `add_routes()`로 체인을 즉시 API화 |
| **비동기 처리** | `async/await` 네이티브 지원. Ollama API 호출 시 non-blocking |
| **타입 안정성** | Pydantic 모델로 입출력 검증. 런타임 에러를 사전에 방지 |
| **자동 문서화** | OpenAPI/Swagger 자동 생성. `/docs`로 API 테스트 가능 |
| **성능** | Starlette 기반. 동기 Flask보다 2-3배 빠른 처리 속도 |

특히 **LangServe와의 통합**이 결정적이었습니다. LangChain 체인을 Flask로 서빙하려면 수동으로 엔드포인트를 구현해야 하지만, FastAPI + LangServe는 체인을 `add_routes()`로 등록하면 `/invoke`, `/batch`, `/stream` 엔드포인트가 자동 생성됩니다.

### 프로젝트 구조

```
apex-llm-service/
├── main.py                  # FastAPI 앱 진입점
├── config.py                # 공통 설정 (Qdrant, Ollama 클라이언트)
├── models.py                # Pydantic 입출력 모델
├── chains/
│   ├── summary_chain.py     # 시장 요약 체인
│   ├── feedback_chain.py    # 투자 피드백 체인
│   └── simulation_chain.py  # 시나리오 시뮬레이션 체인
├── prompts/
│   ├── summary_prompt.py    # Summary 프롬프트 템플릿
│   ├── feedback_prompt.py   # Feedback 프롬프트 템플릿
│   └── simulation_prompt.py # Simulation 프롬프트 템플릿
├── utils/
│   ├── json_parser.py       # LLM 출력 JSON 파싱 유틸
│   └── session_store.py     # 대화 세션 관리
├── Dockerfile
├── requirements.txt
└── .env
```

**설계 원칙:**
- **체인별 파일 분리**: 각 체인이 독립적으로 테스트 가능하도록 모듈화
- **프롬프트 외부화**: 프롬프트를 별도 파일로 관리해 코드 수정 없이 튜닝 가능
- **유틸 재사용**: JSON 파싱, 세션 관리 등 공통 로직은 utils로 추출

### main.py 전체 코드

```python
# main.py
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from langserve import add_routes
import logging

from chains.summary_chain import build_summary_chain
from chains.feedback_chain import build_feedback_chain, get_history
from chains.simulation_chain import build_simulation_chain
from models import SummaryInput, SummaryOutput, FeedbackInput, SimulationInput

# 로깅 설정
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s'
)
logger = logging.getLogger(__name__)

# FastAPI 앱 생성
app = FastAPI(
    title="Apex LLM Service",
    description="RAG-based stock market analysis API",
    version="1.0.0",
    docs_url="/docs",           # Swagger UI
    redoc_url="/redoc",         # ReDoc
)

# CORS 설정 (프론트엔드에서 호출 가능하도록)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5173"],  # React/Vue 개발 서버
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 체인 초기화 (앱 시작 시 한 번만)
summary_chain = build_summary_chain()
feedback_chain = build_feedback_chain()
simulation_chain = build_simulation_chain()

# ===== Summary Chain (LangServe 자동 라우팅) =====
add_routes(
    app,
    summary_chain,
    path="/internal/summary",
    input_type=SummaryInput,
    output_type=SummaryOutput,
    config_keys=["metadata", "tags"],
)
logger.info("Summary chain registered at /internal/summary")

# ===== Feedback Chain (수동 엔드포인트 - 세션 관리 필요) =====
@app.post("/internal/feedback/invoke")
async def invoke_feedback(request: Request):
    """투자 피드백 대화 API (세션 기반)"""
    try:
        body = await request.json()
        input_data = body.get("input", {})

        # 세션 ID 추출 (없으면 "default")
        session_id = body.get("config", {}).get("configurable", {}).get("session_id", "default")

        # 세션에서 이전 대화 복원
        history = get_history(session_id)
        chat_history = [
            ("human", m.content) if m.type == "human" else ("ai", m.content)
            for m in history.messages
        ]

        # 체인 실행
        result = feedback_chain.invoke({
            "question": input_data.get("question", ""),
            "company": input_data.get("company", "Unknown"),
            "sentiment_level": input_data.get("sentiment_level", "중"),
            "chat_history": chat_history,
        })

        # 대화 저장
        history.add_user_message(input_data.get("question", ""))
        history.add_ai_message(result.get("answer", ""))

        logger.info(f"Feedback invoked for session={session_id}, company={input_data.get('company')}")
        return {"output": result}

    except Exception as e:
        logger.error(f"Feedback chain error: {e}", exc_info=True)
        return {"error": str(e)}, 500

# ===== Simulation Chain (수동 엔드포인트 - 전처리 필요) =====
@app.post("/internal/simulation/invoke")
async def invoke_simulation(request: Request):
    """시나리오 시뮬레이션 API"""
    try:
        body = await request.json()
        input_data = body.get("input", {})
        session_id = body.get("config", {}).get("configurable", {}).get("session_id", "default")

        # 세션에서 이전 대화 복원
        history = get_history(session_id)
        chat_history = [
            ("human", m.content) if m.type == "human" else ("ai", m.content)
            for m in history.messages
        ]

        # 체인 실행 (내부에서 벡터 DB 검색 수행)
        result = simulation_chain.invoke({
            "question": input_data.get("question", ""),
            "company": input_data.get("company", "Unknown"),
            "chat_history": chat_history,
        })

        # 대화 저장
        history.add_user_message(input_data.get("question", ""))
        history.add_ai_message(result.get("answer", ""))

        logger.info(f"Simulation invoked for session={session_id}, company={input_data.get('company')}")
        return {"output": result}

    except Exception as e:
        logger.error(f"Simulation chain error: {e}", exc_info=True)
        return {"error": str(e)}, 500

# ===== Health Check =====
@app.get("/health")
async def health_check():
    """컨테이너 헬스체크용 엔드포인트"""
    return {"status": "ok", "service": "apex-llm-service"}

# ===== Root =====
@app.get("/")
async def root():
    return {
        "message": "Apex LLM Service",
        "docs": "/docs",
        "endpoints": {
            "summary": "/internal/summary/invoke",
            "feedback": "/internal/feedback/invoke",
            "simulation": "/internal/simulation/invoke"
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, log_level="info")
```

**핵심 포인트:**

1. **Summary는 LangServe, Feedback/Simulation은 수동**: Summary는 상태 없는(stateless) 체인이라 LangServe로 자동 등록했지만, Feedback과 Simulation은 세션 관리가 필요해서 수동으로 엔드포인트를 구현했습니다.

2. **세션 ID 기반 대화 관리**: `config.configurable.session_id`로 사용자별 대화 히스토리를 분리합니다. 같은 session_id로 요청하면 이전 대화 맥락을 유지합니다.

3. **에러 핸들링**: try-except로 체인 실행 에러를 잡아서 500 에러 대신 구조화된 JSON 응답을 반환합니다. 프로덕션에서는 Sentry 등으로 에러를 추적해야 합니다.

4. **CORS 설정**: 프론트엔드가 다른 포트(3000, 5173)에서 돌아가므로 CORS를 허용했습니다. 프로덕션에서는 실제 도메인만 허용하도록 수정해야 합니다.

### config.py: 공통 설정 관리

```python
# config.py
import os
from qdrant_client import QdrantClient
from langchain_qdrant import QdrantVectorStore
from langchain_ollama import OllamaEmbeddings, OllamaLLM

# 환경변수 로드
QDRANT_URL = os.getenv("QDRANT_URL", "http://localhost:6333")
QDRANT_API_KEY = os.getenv("QDRANT_API_KEY", "")
QDRANT_COLLECTION = os.getenv("QDRANT_COLLECTION", "stock_segments")

OLLAMA_URL = os.getenv("OLLAMA_URL", "http://localhost:11434")
EMBED_MODEL = os.getenv("EMBED_MODEL", "bge-m3")
LLM_MODEL = os.getenv("LLM_MODEL", "exaone3.5:7.8b")

def get_vectorstore() -> QdrantVectorStore:
    """Qdrant 벡터 스토어 인스턴스 생성 (싱글톤 패턴 권장)"""
    embeddings = OllamaEmbeddings(
        model=EMBED_MODEL,
        base_url=OLLAMA_URL,
    )

    client = QdrantClient(
        url=QDRANT_URL,
        api_key=QDRANT_API_KEY,
        timeout=30,  # API 타임아웃 30초
    )

    return QdrantVectorStore(
        client=client,
        collection_name=QDRANT_COLLECTION,
        embedding=embeddings,
    )

def get_llm() -> OllamaLLM:
    """Ollama LLM 인스턴스 생성"""
    return OllamaLLM(
        model=LLM_MODEL,
        base_url=OLLAMA_URL,
        num_predict=256,       # 최대 생성 토큰 수 (금융 분석은 간결함이 중요)
        temperature=0.4,       # 낮은 온도 → 일관되고 보수적인 출력
        top_p=0.9,             # 누적 확률 상위 90%만 샘플링
        repeat_penalty=1.1,    # 반복 억제 (같은 문구 반복 방지)
        num_ctx=4096,          # 컨텍스트 윈도우 (문서 5개 + 프롬프트)
    )
```

**설계 결정:**

- **`num_predict=256`**: 금융 분석은 장황한 설명보다 핵심만 간결하게. GPT-4처럼 길게 설명하지 않도록 제한.
- **`temperature=0.4`**: 창의성보다 일관성. 같은 입력에 대해 비슷한 출력이 나오도록.
- **`repeat_penalty=1.1`**: "주가가 상승했습니다. 주가가 올랐습니다. 주가는..."처럼 반복되는 문구를 억제.
- **`num_ctx=4096`**: 검색된 문서 5개(각 800자) + 프롬프트를 모두 담기 위해 충분한 컨텍스트 확보.

### Dockerfile: 컨테이너 이미지 빌드

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 시스템 패키지 설치 (gcc는 일부 Python 패키지 빌드에 필요)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Python 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 앱 코드 복사
COPY . .

# 포트 노출
EXPOSE 8000

# 헬스체크 (Docker Compose에서 사용)
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Uvicorn으로 FastAPI 실행
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

**왜 workers=1인가:**
- Ollama와 Qdrant는 별도 컨테이너라 상태 공유 문제 없음.
- 하지만 **세션 저장소가 인메모리 딕셔너리**라서, 멀티 워커로 띄우면 각 워커마다 다른 세션 저장소를 가지게 됨.
- 프로덕션에서는 Redis나 PostgreSQL로 세션을 영속화하고, workers를 CPU 코어 수만큼 늘려야 함.

### requirements.txt

```txt
fastapi==0.115.0
uvicorn[standard]==0.32.0
langchain==0.3.7
langchain-community==0.3.7
langchain-ollama==0.2.0
langchain-qdrant==0.2.0
langserve==0.3.0
qdrant-client==1.12.0
pydantic==2.9.2
pydantic-settings==2.6.0
python-dotenv==1.0.1
requests==2.32.3
```

**버전 고정 이유:**
- LangChain 생태계는 빠르게 변화하므로, 마이너 버전까지 고정해서 예상치 못한 breaking change를 방지.
- `uvicorn[standard]`는 WebSocket, HTTP/2 지원을 포함. LangServe의 `/stream` 엔드포인트에 필요.

### LangChain 체인 구현 상세

LangChain의 핵심은 **Runnable** 인터페이스입니다. 모든 체인 컴포넌트가 `invoke()`, `batch()`, `stream()` 메서드를 구현하므로, 파이프 연산자(`|`)로 조합할 수 있습니다.

```python
# Runnable 인터페이스의 개념
chain = component1 | component2 | component3

# 실행
result = chain.invoke(input)

# 위는 다음과 동일:
temp1 = component1.invoke(input)
temp2 = component2.invoke(temp1)
result = component3.invoke(temp2)
```

이 설계 덕분에 복잡한 RAG 파이프라인을 **선언적(declarative)**으로 구성할 수 있습니다.

### Chain 1: Summary Chain — 시장 요약 생성

가장 핵심적인 RAG 체인입니다. 종목명과 감정 지표를 받아 관련 문서를 검색하고, JSON 형태의 시장 요약을 생성합니다.

```python
# chains/summary_chain.py
from langchain_core.runnables import RunnableMap, RunnableLambda
from langchain_core.output_parsers import StrOutputParser
from config import get_llm, get_vectorstore
from prompts.summary_prompt import summary_prompt
from utils.json_parser import strict_json_parser

def build_summary_chain():
    """
    Summary Chain: 종목명 + 감정 지표 → JSON 형태의 시장 요약

    입력 스키마:
    {
        "company": "SK하이닉스",
        "sentiment_level": "상",
        "positive": 65,
        "neutral": 20,
        "negative": 15
    }

    출력 스키마:
    {
        "sentiment_based": "감정 분석 기반 요약 (조언형)",
        "market_summary": ["시장 분석 1", "시장 분석 2", "시장 분석 3"]
    }
    """
    llm = get_llm()
    vectorstore = get_vectorstore()

    # 벡터 유사도 검색: 종목명과 가장 유사한 문서 5개 추출
    retriever = vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={
            "k": 5,                    # 상위 5개 문서
            "score_threshold": 0.3,    # 유사도 0.3 이상만 (너무 낮으면 노이즈)
        }
    )

    def retrieve_and_format(x):
        """
        종목명으로 문서 검색 → 텍스트 포맷팅

        왜 RunnableLambda로 감쌌는가:
        - retriever.invoke()는 Document 객체 리스트를 반환
        - 이를 LLM이 읽을 수 있는 텍스트로 변환해야 함
        - 각 문서를 "\n\n"로 구분해서 하나의 컨텍스트 문자열로 병합
        """
        docs = retriever.invoke(x["company"])

        if not docs:
            return f"{x['company']}에 대한 최근 뉴스나 공시 데이터가 부족합니다."

        # 각 문서를 "출처 | 제목 | 본문" 형태로 포맷팅
        context_parts = []
        for doc in docs:
            source = doc.metadata.get("source", "unknown")
            title = doc.metadata.get("title", "")
            text = doc.page_content

            context_parts.append(f"[{source}] {title}\n{text}")

        return "\n\n---\n\n".join(context_parts)

    get_context = RunnableLambda(retrieve_and_format)

    # 체인 구성: RunnableMap → Prompt → LLM → JSON Parser
    chain = (
        RunnableMap({
            # RAG의 "R" (Retrieval): 벡터 DB에서 관련 문서 검색
            "context":         get_context,

            # 나머지 입력값은 그대로 전달
            "company":         lambda x: x["company"],
            "sentiment_level": lambda x: x["sentiment_level"],
            "positive":        lambda x: x.get("positive", 0),
            "neutral":         lambda x: x.get("neutral", 0),
            "negative":        lambda x: x.get("negative", 0),
        })
        | summary_prompt          # Prompt Template에 값 주입
        | llm                     # LLM 추론 (RAG의 "G" - Generation)
        | strict_json_parser()    # JSON 파싱 + 실패 시 폴백
    ).with_config(
        run_name="summary_chain",
        tags=["rag", "summary", "stock_analysis"]
    )

    return chain
```

**왜 RunnableMap을 사용하는가:**

RunnableMap은 여러 Runnable을 병렬로 실행하고, 결과를 딕셔너리로 묶어줍니다. 위 코드에서 `get_context`만 실제 비동기 작업(Qdrant API 호출)이고, 나머지는 단순 값 추출이지만, LangChain은 내부적으로 최적화해서 병렬 실행합니다.

```python
# RunnableMap의 동작 방식
input = {"company": "SK하이닉스", "sentiment_level": "상", ...}

# 각 키-값 쌍을 병렬 실행:
# - "context": get_context.invoke(input) → Qdrant 검색
# - "company": lambda x: x["company"] → "SK하이닉스"
# - "sentiment_level": lambda x: x["sentiment_level"] → "상"
# ...

# 결과를 딕셔너리로 병합:
output = {
    "context": "검색된 문서들...",
    "company": "SK하이닉스",
    "sentiment_level": "상",
    ...
}
```

**검색 품질 최적화:**

- **`score_threshold=0.3`**: 벡터 유사도가 0.3 미만인 문서는 제외. 너무 낮으면 "삼성전자"를 검색했는데 "삼성생명" 문서가 섞일 수 있음.
- **`k=5`**: 너무 많으면 프롬프트가 길어져서 LLM이 핵심을 놓치고, 너무 적으면 정보가 부족. 5개가 적당한 균형점.
- **문서 포맷팅**: 단순히 텍스트만 합치지 않고 `[출처] 제목\n본문` 형태로 구조화. LLM이 어느 소스에서 온 정보인지 구분 가능.

#### 프롬프트 설계: Few-Shot + 제약 조건

```python
# prompts/summary_prompt.py
from langchain_core.prompts import PromptTemplate

summary_prompt = PromptTemplate.from_template("""
너는 한국어 금융 리포트 작성을 전문으로 하는 AI 분석가이다.
**절대 규칙: JSON 형식으로만 출력하고, 설명이나 불필요한 문장은 쓰지 않는다.**

입력 정보:
- 회사명: {company}
- 감정 과열도: {sentiment_level} (상/중/하)
- 긍정 비율: {positive}%
- 중립 비율: {neutral}%
- 부정 비율: {negative}%
- 관련 문서(context):
{context}

출력 형식 (반드시 이 JSON 구조를 따를 것):
{{
  "sentiment_based": "감정 분석 기반 요약 (2-3문장)",
  "market_summary": [
    "시장 분석 포인트 1",
    "시장 분석 포인트 2",
    "시장 분석 포인트 3"
  ]
}}

작성 가이드:
1. "sentiment_based": 감정 지표 수치를 언급하며 투자 조언 형태로 작성.
   - 감정 과열도 해석 기준:
     * 상: 과열된 시장 심리 → 단기 변동성 증가 가능, 신중한 접근 필요
     * 중: 균형잡힌 심리 → 안정적 흐름 예상
     * 하: 위축된 심리 → 저평가 기회 탐색 가능성

2. "market_summary": context에 나온 실제 뉴스/공시/커뮤니티 내용을 3가지로 요약.
   - 각 항목은 1-2문장으로 간결하게.
   - 출처(뉴스/공시/카페)를 명시하지 말고, 내용만 요약.
   - "~한 것으로 나타났다", "~라는 의견이 있다" 등 객관적 표현 사용.

3. 주의사항:
   - context에 없는 내용을 지어내지 않는다.
   - 숫자나 날짜를 언급할 때는 context의 정확한 수치를 사용한다.
   - "투자 권유"나 "매수/매도 추천"은 절대 하지 않는다.

예시 (회사명="SK하이닉스", sentiment_level="상", positive=70, neutral=20, negative=10):
{{
  "sentiment_based": "긍정 감정이 70%로 높고 과열도가 '상' 수준입니다. 단기 급등 후 조정 가능성이 있으므로 진입 시점을 신중히 판단하시기 바랍니다.",
  "market_summary": [
    "HBM3E 양산 본격화로 AI 서버 시장 점유율 확대 전망이 나오고 있습니다",
    "2분기 영업이익이 시장 예상치를 상회하며 실적 개선세가 확인되었습니다",
    "FOMC 금리 인하 기대감으로 반도체 업종 전반에 자금 유입이 이어지고 있습니다"
  ]
}}

이제 위 형식에 맞춰 JSON을 생성하라. JSON 외에는 어떤 텍스트도 출력하지 마라.
""")
```

**프롬프트 엔지니어링 포인트:**

1. **Few-Shot 예시**: Exaone 3.5 같은 7B 모델은 추상적인 지시만으로는 JSON 구조를 잘 따르지 않습니다. 실제 예시를 보여주면 출력 품질이 크게 개선됩니다.

2. **"절대 규칙" 강조**: "오직 JSON만", "어떤 텍스트도 출력하지 마라" 같은 강한 표현을 반복해서, 모델이 "제 생각에는...", "분석 결과입니다:" 같은 부가 설명을 붙이지 않도록 유도합니다.

3. **할루시네이션 방지**: "context에 없는 내용을 지어내지 않는다", "정확한 수치를 사용한다"를 명시. 로컬 모델은 GPT-4보다 할루시네이션 경향이 강해서 이런 제약이 필수입니다.

4. **투자 권유 금지**: 금융 서비스에서 법적 문제를 피하기 위해, "매수/매도 추천" 대신 "정보 제공" 수준으로 제한합니다.

#### JSON 파싱 방어 코드: 불완전한 출력 복구

로컬 7.8B 모델은 종종 다음과 같은 불완전한 JSON을 출력합니다.

```json
분석 결과는 다음과 같습니다:
{
  "sentiment_based": "긍정 비율 70%...",
  "market_summary": ["HBM3E 양산
```

닫는 괄호가 없거나, 앞뒤에 설명이 붙거나, 배열이 중간에 끊깁니다. 이를 복구하는 파서를 만들었습니다.

```python
# utils/json_parser.py
import re
import json
from langchain_core.runnables import RunnableLambda

def strict_json_parser():
    """
    LLM 출력에서 JSON을 추출하고 파싱.
    실패 시 폴백 로직으로 최대한 복구 시도.
    """
    def try_parse(result: str):
        # Step 1: JSON 블록 추출 (앞뒤 설명 제거)
        m = re.search(r"\{[\s\S]*\}", result)
        if not m:
            # JSON 블록이 아예 없으면 폴백
            return fallback_response(result)

        json_str = m.group(0)

        # Step 2: 괄호 불균형 복구
        open_braces = json_str.count("{")
        close_braces = json_str.count("}")
        open_brackets = json_str.count("[")
        close_brackets = json_str.count("]")

        if open_braces > close_braces:
            json_str += "}" * (open_braces - close_braces)

        if open_brackets > close_brackets:
            json_str += "]" * (open_brackets - close_brackets)

        # Step 3: JSON 파싱 시도
        try:
            return json.loads(json_str)
        except json.JSONDecodeError as e:
            # 파싱 실패 시 정규식 기반 복구
            return fallback_with_regex(json_str, result)

    return RunnableLambda(try_parse)


def fallback_with_regex(json_str: str, original: str):
    """
    JSON 파싱 실패 시 정규식으로 필드 추출
    """
    result = {}

    # "sentiment_based" 필드 추출
    m = re.search(r'"sentiment_based"\s*:\s*"([^"]*)"', json_str)
    if m:
        result["sentiment_based"] = m.group(1)
    else:
        result["sentiment_based"] = "감정 분석 데이터가 부족합니다."

    # "market_summary" 배열 추출
    m = re.search(r'"market_summary"\s*:\s*\[(.*?)\]', json_str, re.DOTALL)
    if m:
        items_str = m.group(1)
        # 배열 항목 파싱: "항목1", "항목2", ...
        items = re.findall(r'"([^"]*)"', items_str)
        result["market_summary"] = items if items else ["시장 데이터 수집 중입니다."]
    else:
        result["market_summary"] = ["시장 데이터 수집 중입니다."]

    return result


def fallback_response(original: str):
    """
    JSON 블록이 아예 없을 때 기본 응답
    """
    return {
        "sentiment_based": "현재 분석 데이터가 부족합니다.",
        "market_summary": ["최신 뉴스 및 공시를 확인해주세요."],
        "_raw_output": original  # 디버깅용으로 원본 출력 보존
    }
```

**복구 전략:**

1. **정규식으로 JSON 블록 추출**: 모델이 "분석 결과입니다:" 같은 서문을 붙여도, `\{...\}` 패턴으로 JSON만 잘라냅니다.

2. **괄호 자동 보충**: `{` 개수와 `}` 개수를 세서, 부족한 만큼 끝에 추가. "문자열 중간에 `{`가 있으면?"이라는 엣지 케이스는 있지만, 실전에서는 95% 이상 잘 동작합니다.

3. **정규식 기반 필드 추출**: `json.loads()` 실패 시, 정규식으로 `"sentiment_based": "..."`와 `"market_summary": [...]` 부분을 직접 파싱합니다. 완벽한 JSON은 아니어도 **사용 가능한 데이터를 최대한 추출**하는 것이 목표입니다.

4. **폴백 응답**: 그래도 실패하면 "데이터 부족" 메시지를 반환. 빈 응답보다는 사용자에게 상황을 알리는 것이 낫습니다.

**실전 효과:**

- Before (파서 없음): 10회 호출 중 3-4회 JSON 파싱 에러
- After (파서 적용): 10회 호출 중 0-1회 폴백 응답 (성공률 90%+)

로컬 모델을 프로덕션에 쓸 때는 이런 방어 로직이 필수입니다. GPT-4나 Claude는 JSON 출력이 거의 완벽하지만, Exaone/Llama 같은 오픈 모델은 여전히 불안정합니다.

### Chain 2: Feedback Chain — 투자 코칭 대화

사용자와 대화하며 투자 피드백을 제공하는 체인입니다. **세션 기반 대화 히스토리**를 관리합니다.

```python
# chains/feedback_chain.py

# 세션별 대화 히스토리 저장소 (인메모리)
_store: dict[str, ChatMessageHistory] = {}

def get_history(session_id: str) -> ChatMessageHistory:
    if session_id not in _store:
        _store[session_id] = ChatMessageHistory()
    return _store[session_id]

def build_feedback_chain():
    llm = get_llm()

    chain = (
        RunnableMap({
            "company":         lambda x: x["company"],
            "sentiment_level": lambda x: x["sentiment_level"],
            "input":           lambda x: x["question"],
            "chat_history":    lambda x: x.get("chat_history", []),
        })
        | feedback_prompt     # ChatPromptTemplate (system + history + human)
        | llm
        | StrOutputParser()
        | RunnableLambda(lambda x: {"answer": x})
    )

    return chain
```

#### 프롬프트: 공감 + 논리의 균형

```python
feedback_prompt = ChatPromptTemplate.from_messages([
    ("system", """
    너는 사용자의 감정과 투자 판단을 돕는 한국어 투자 피드백 AI 코치이다.

    참고 정보:
    - 회사명: {company}
    - 감정 과열도: {sentiment_level}
    - 직전 대화 맥락: {chat_history}

    가이드라인:
    1) 과열 '상' → 신중 강조
    2) '중' → 균형·추세 지속 가능성
    3) '하' → 위축 속 기회 가능성
    4) 공감+차분한 조언, 자기언급 금지
    5) JSON/코드/마크다운 금지
    """),
    ("human", "{input}")
])
```

#### 세션 관리

```python
# main.py - Feedback 엔드포인트
@app.post("/internal/feedback/invoke")
async def invoke_feedback(request: Request):
    body = await request.json()
    session_id = body.get("config", {}).get("configurable", {}).get("session_id", "default")

    # 세션에서 이전 대화 복원
    history = get_history(session_id)
    chat_history = [
        ("human", m.content) if m.type == "human" else ("ai", m.content)
        for m in history.messages
    ]

    result = feedback_chain.invoke({
        "question": question,
        "company": company,
        "sentiment_level": sentiment_level,
        "chat_history": chat_history,
    })

    # 대화 저장
    history.add_user_message(question)
    history.add_ai_message(str(result))

    return {"result": result}
```

현재는 인메모리 딕셔너리로 세션을 관리하고 있어, **컨테이너 재시작 시 대화 히스토리가 사라집니다.** 프로덕션에서는 PostgreSQL이나 Redis로 영속화해야 하는 부분입니다.

### Chain 3: Simulation Chain — 시나리오 시뮬레이션

"FOMC 금리 인상 발표 후 SK하이닉스에 어떤 영향이 있을까?"와 같은 시나리오 질문을 처리합니다.

```python
# chains/simulation_chain.py

def retrieve_market_data(company: str, top_k: int = 3):
    """벡터 DB에서 최신 시장 데이터 검색"""
    try:
        vectorstore = get_vectorstore()
        results = vectorstore.similarity_search(company, k=top_k)
        return [r.page_content for r in results]
    except Exception:
        return [f"{company} 관련 최신 시장 데이터가 부족합니다."]

def build_simulation_chain():
    llm = get_llm()

    def preprocess(x):
        company = x.get("company", "Unknown")
        market_summary = retrieve_market_data(company)
        market_summary_text = " / ".join(market_summary[:2])
        return {
            "input": f"{x['question']}\n\n[참고: 회사={company}, 시장 요약={market_summary_text}]",
            "company": company,
            "market_summary": market_summary_text,
            "chat_history": x.get("chat_history", []),
        }

    chain = (
        RunnableLambda(preprocess)     # 질문에 시장 데이터 주입
        | RunnableMap({...})
        | simulation_prompt
        | llm
        | StrOutputParser()
        | RunnableLambda(lambda x: {"answer": x})
    )

    return chain
```

Simulation Chain은 Feedback Chain과 달리, **체인 내부에서 직접 벡터 DB를 검색**합니다(`preprocess` 함수). 사용자의 가상 시나리오에 실제 시장 데이터를 결합해 현실적인 시뮬레이션을 생성합니다.

### API 엔드포인트 구조

```python
# main.py
app = FastAPI()

# Summary: LangServe 라우터 (자동 OpenAPI 문서 생성)
add_routes(app, summary_chain,
    path="/internal/summary",
    input_type=SummaryInput,
    output_type=SummaryOutput)

# Feedback: 수동 invoke (세션 관리 필요)
@app.post("/internal/feedback/invoke")
async def invoke_feedback(request: Request): ...

# Simulation: 수동 invoke (세션 관리 + 전처리 필요)
@app.post("/internal/simulation/invoke")
async def invoke_simulation(request: Request): ...
```

Summary Chain은 LangServe의 `add_routes`로 등록해 자동으로 `/internal/summary/invoke`, `/internal/summary/stream` 등의 엔드포인트가 생성됩니다. 반면 Feedback과 Simulation은 세션 관리 로직이 필요해 수동으로 엔드포인트를 구현했습니다.

---

## 데이터 적재 처리: Qdrant 컬렉션 설계

### Qdrant 컬렉션 초기화

벡터 DB에 데이터를 넣기 전에, 컬렉션(테이블)을 생성해야 합니다.

```python
# scripts/init_qdrant_collection.py
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PayloadSchemaType

client = QdrantClient(url="http://localhost:6333", api_key="your-api-key")

# 컬렉션 생성
client.create_collection(
    collection_name="stock_segments",
    vectors_config=VectorParams(
        size=1024,              # BGE-m3 임베딩 차원
        distance=Distance.COSINE,  # 코사인 유사도
        hnsw_config={
            "m": 16,            # HNSW 그래프의 연결 수 (기본값 16)
            "ef_construct": 100  # 인덱스 구축 시 탐색 깊이 (높을수록 정확하지만 느림)
        }
    ),
    # 페이로드 스키마 정의 (선택사항, 하지만 필터링 성능 향상)
    payload_schema={
        "company": PayloadSchemaType.KEYWORD,        # 정확한 일치 검색 (인덱스)
        "source": PayloadSchemaType.KEYWORD,         # news / cafe / dart
        "date": PayloadSchemaType.KEYWORD,           # YYYY-MM-DD 형식
        "sentiment_label": PayloadSchemaType.KEYWORD,  # pos / neg / neutral
        "market_relevance": PayloadSchemaType.FLOAT,  # 0.0 ~ 1.0
        "sentiment_confidence": PayloadSchemaType.FLOAT,
    }
)

print("컬렉션 'stock_segments' 생성 완료")
```

**설정 설명:**

#### 1. `size=1024`: 벡터 차원

BGE-m3는 1024차원 벡터를 생성합니다. 모든 벡터가 이 차원과 일치해야 합니다. 차원이 다르면 업서트 시 에러.

#### 2. `distance=Distance.COSINE`: 유사도 메트릭

| 메트릭 | 범위 | 사용 사례 |
|--------|------|----------|
| COSINE | -1 ~ 1 (1에 가까울수록 유사) | 텍스트 임베딩 (방향만 중요, 크기 무관) |
| EUCLIDEAN | 0 ~ ∞ (0에 가까울수록 유사) | 이미지 임베딩 (절대 거리 중요) |
| DOT | -∞ ~ ∞ (높을수록 유사) | 추천 시스템 (내적 값 자체가 의미) |

텍스트 임베딩은 **COSINE**이 표준입니다. "삼성전자 HBM 양산"과 "삼성전자 HBM3E 출하" 같은 문장은 방향(의미)이 비슷하지만, 길이(단어 수)는 다를 수 있습니다. 코사인은 길이를 정규화하므로 의미적 유사성을 잘 포착합니다.

#### 3. `hnsw_config`: 인덱스 파라미터

Qdrant는 **HNSW (Hierarchical Navigable Small World)** 알고리즘으로 벡터를 인덱싱합니다.

- **`m=16`**: 각 벡터가 평균 16개의 다른 벡터와 연결됩니다. 높이면 검색 정확도 증가, but 메모리 사용량 증가. 16은 대부분 케이스에 적합.
- **`ef_construct=100`**: 인덱스 구축 시 탐색 깊이. 높이면 인덱스 품질 증가, but 초기 적재 속도 느려짐. 100은 균형잡힌 값.

**참고:** 검색 시에는 `ef` 파라미터로 정확도-속도 트레이드오프 조절 가능.

```python
# 검색 시 ef 조절
vectorstore.similarity_search(
    "SK하이닉스",
    k=5,
    search_params={"hnsw_ef": 128}  # 기본값 16 → 128로 증가 (더 정확하지만 느림)
)
```

#### 4. `payload_schema`: 메타데이터 인덱싱

Qdrant는 기본적으로 모든 페이로드를 JSON으로 저장하지만, **명시적으로 스키마를 정의하면 인덱스를 생성**합니다.

- `KEYWORD`: 정확한 일치 검색. "삼성전자"로 필터링하면 "삼성"은 안 나옴.
- `FLOAT` / `INTEGER`: 범위 검색 가능. `market_relevance > 0.7` 같은 필터.

**스키마 정의 전 vs 후 성능 비교:**

```python
# 10,000개 벡터에서 필터링 검색
# Before (스키마 없음): 450ms
# After (스키마 정의): 120ms
```

### 중복 제거 메커니즘: 콘텐츠 해시 기반 ID

같은 뉴스를 여러 번 수집하면 벡터 DB가 중복 데이터로 가득 찹니다. 이를 방지하기 위해 **콘텐츠 해시를 point ID로 사용**합니다.

```python
import hashlib

def generate_point_id(company: str, date: str, source: str, text: str) -> str:
    """
    회사명 + 날짜 + 소스 + 텍스트의 SHA-256 해시 → 고유 ID

    예시:
      company="SK하이닉스", date="2025-08-15", source="news",
      text="SK하이닉스가 HBM3E 양산을 시작..."
      → "a3f8d92e..." (64자 hex)

    같은 뉴스를 다시 수집하면 동일한 ID 생성 → Qdrant의 upsert가 덮어쓰기
    """
    content = f"{company}|{date}|{source}|{text}"
    return hashlib.sha256(content.encode('utf-8')).hexdigest()
```

**Qdrant의 upsert 동작:**

```python
# Point ID가 이미 존재하면 덮어쓰기, 없으면 새로 삽입
qdrant_client.upsert(
    collection_name="stock_segments",
    points=[
        {
            "id": "a3f8d92e...",  # 중복 시 기존 벡터 업데이트
            "vector": [...],
            "payload": {...}
        }
    ]
)
```

**주의사항:**

- 제목이나 본문에 오타가 하나라도 있으면 다른 해시가 생성됩니다 → 미세한 중복 발생 가능.
- 해결책: 정규화 단계에서 공백 압축, 소문자 변환 등을 통일. 또는 Fuzzy 해싱 (SimHash) 사용.

### 페이로드 설계: 검색과 필터링을 동시에

Qdrant의 페이로드는 RDB의 컬럼과 비슷합니다. 어떤 필드를 넣을지 신중히 설계해야 합니다.

```json
{
  "id": "a3f8d92e...",
  "vector": [0.123, -0.456, ...],  // 1024차원
  "payload": {
    "company": "SK하이닉스",
    "source": "news",
    "title": "SK하이닉스, HBM3E 본격 양산... AI 서버 시장 공략",
    "text": "SK하이닉스가 차세대 고대역폭메모리(HBM3E) 양산을 시작했다...",
    "url": "https://news.example.com/...",
    "date": "2025-08-15",
    "sentiment_label": "pos",
    "sentiment_confidence": 0.87,
    "market_relevance": 0.92,
    "content_hash": "a3f8d92e..."
  }
}
```

**필드별 용도:**

| 필드 | 타입 | 용도 |
|------|------|------|
| `company` | KEYWORD | 필터링: "SK하이닉스" 관련만 검색 |
| `source` | KEYWORD | 필터링: 뉴스/카페/공시 중 선택 |
| `date` | KEYWORD | 필터링: 최근 7일 데이터만 검색 |
| `title` | TEXT | 표시: 검색 결과 제목 |
| `text` | TEXT | 검색 대상: 벡터로 변환된 원본 텍스트 |
| `url` | TEXT | 출처: 사용자에게 원문 링크 제공 |
| `sentiment_label` | KEYWORD | 필터링: 긍정 뉴스만 / 부정 뉴스만 |
| `sentiment_confidence` | FLOAT | 정렬: 감정 확신도 높은 순 |
| `market_relevance` | FLOAT | 필터링: 시장 관련도 0.7 이상만 |
| `content_hash` | KEYWORD | 디버깅: 중복 확인 |

**복합 필터링 예시:**

```python
# "SK하이닉스" + "최근 3일" + "긍정 뉴스" + "시장 관련도 0.7 이상"
from datetime import datetime, timedelta

cutoff_date = (datetime.now() - timedelta(days=3)).strftime("%Y-%m-%d")

results = vectorstore.similarity_search(
    "SK하이닉스 실적 전망",
    k=10,
    filter={
        "must": [
            {"key": "company", "match": {"value": "SK하이닉스"}},
            {"key": "date", "range": {"gte": cutoff_date}},
            {"key": "sentiment_label", "match": {"value": "pos"}},
            {"key": "market_relevance", "range": {"gte": 0.7}}
        ]
    }
)
```

이렇게 벡터 유사도와 메타데이터 필터를 결합하면, **"의미적으로 유사하면서 조건도 만족하는"** 문서만 정확히 추출할 수 있습니다.

### 벡터 검색 성능 최적화

#### 1. 적절한 `k` 값 선택

```python
# k=5: 대부분의 RAG 시나리오에 적합
# k=10: 문서 다양성이 중요할 때
# k=20+: LLM 컨텍스트가 긴 모델(GPT-4 128K)에서만 유효
```

`k`가 클수록 검색은 느려지고, LLM 추론도 느려집니다. Exaone 3.5 (4K 컨텍스트)에서는 k=5가 적절.

#### 2. Quantization (양자화)

Qdrant는 벡터를 압축해서 저장할 수 있습니다.

```python
# Scalar Quantization: float32 → uint8 (4배 메모리 절약)
client.update_collection(
    collection_name="stock_segments",
    quantization_config=ScalarQuantization(
        type=ScalarType.INT8,
        quantile=0.99  # 99% 데이터 범위를 커버하도록 스케일링
    )
)
```

- **Before**: 1024 float32 = 4KB/벡터
- **After**: 1024 uint8 = 1KB/벡터

10만 벡터 기준 400MB → 100MB 절약. 검색 속도도 약간 향상 (메모리 대역폭 감소).

#### 3. 벡터 + 페이로드 분리 저장

매우 큰 텍스트를 페이로드에 넣으면 검색 시 I/O 오버헤드가 큽니다.

```python
# BAD: 전체 원문을 payload에 저장
payload = {"text": "매우 긴 뉴스 본문 5000자..."}

# GOOD: 요약만 payload에, 원문은 별도 DB(PostgreSQL 등)에 저장
payload = {
    "text_preview": "SK하이닉스가 HBM3E...",  # 처음 200자만
    "external_doc_id": 12345  # PostgreSQL의 PK
}
```

Qdrant는 벡터 검색용으로만 쓰고, 전체 원문은 RDB에서 가져오는 하이브리드 아키텍처도 고려할 수 있습니다.

---

## End-to-End 데이터 흐름

한 번의 분석 요청이 시스템을 어떻게 관통하는지 정리합니다.

### Step 1: 데이터 수집

```bash
curl -X POST http://localhost:5678/webhook/ingest-docs \
  -H "Content-Type: application/json" \
  -d '{"query": "SK하이닉스", "days": 7}'
```

→ n8n이 네이버 뉴스/카페/DART에서 데이터를 수집하고, 청킹 → 임베딩 → Qdrant 저장까지 자동 처리합니다.

### Step 2: 시장 요약 요청

```bash
curl -X POST http://localhost:8000/internal/summary/invoke \
  -d '{
    "input": {
      "company": "SK하이닉스",
      "sentiment_level": "상",
      "positive": 65, "neutral": 20, "negative": 15
    }
  }'
```

→ Qdrant에서 "SK하이닉스" 관련 문서 5개를 검색하고, 감정 지표와 함께 LLM에 전달해 요약을 생성합니다.

### Step 3: 투자 피드백 대화

```bash
curl -X POST http://localhost:8000/internal/feedback/invoke \
  -d '{
    "config": {"configurable": {"session_id": "user-123"}},
    "input": {
      "question": "지금 사도 괜찮을까요?",
      "company": "SK하이닉스",
      "sentiment_level": "상"
    }
  }'
```

→ 감정 과열도가 "상"이므로 신중함을 강조하면서, 이전 대화 맥락을 고려한 피드백을 생성합니다.

### Step 4: 시나리오 시뮬레이션

```bash
curl -X POST http://localhost:8000/internal/simulation/invoke \
  -d '{
    "config": {"configurable": {"session_id": "user-123"}},
    "input": {
      "question": "FOMC 금리 인상 발표 후 시나리오는?",
      "company": "SK하이닉스"
    }
  }'
```

→ 벡터 DB에서 최신 시장 데이터를 검색해 주입하고, 단기 반응/자금 흐름/뉴스 헤드라인 예시/투자자 심리까지 포함한 시나리오를 생성합니다.

---

## 돌아보며: 개선할 점과 실전 문제 해결

### 실제 운영 중 발견한 문제들

#### 문제 1: 세션 영속화 부재 → 대화 휘발

**증상:**
- 사용자가 "지금 사도 괜찮을까요?" → "그렇다면 목표가는 얼마로?" 연속 대화 중
- FastAPI 컨테이너 재시작 → 이전 대화 맥락 완전 소실
- 사용자: "왜 갑자기 기억을 못해?"

**원인:**
```python
# utils/session_store.py
_store: dict[str, ChatMessageHistory] = {}  # 인메모리 딕셔너리
```

컨테이너 재시작 = 프로세스 종료 = 메모리 소실

**해결안 1: Redis 기반 영속화**

```python
# utils/session_store.py (Redis 버전)
import redis
import json
from langchain.memory import ChatMessageHistory

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_history(session_id: str) -> ChatMessageHistory:
    key = f"session:{session_id}"
    data = redis_client.get(key)

    if data:
        messages = json.loads(data)
        history = ChatMessageHistory()
        for msg in messages:
            if msg["type"] == "human":
                history.add_user_message(msg["content"])
            else:
                history.add_ai_message(msg["content"])
        return history
    else:
        return ChatMessageHistory()

def save_history(session_id: str, history: ChatMessageHistory):
    key = f"session:{session_id}"
    messages = [{"type": m.type, "content": m.content} for m in history.messages]
    redis_client.setex(key, 3600 * 24, json.dumps(messages))  # 24시간 TTL
```

**해결안 2: PostgreSQL 기반 영속화 (LangChain 네이티브)**

```python
from langchain_postgres import PostgresChatMessageHistory

def get_history(session_id: str):
    return PostgresChatMessageHistory(
        session_id=session_id,
        connection_string="postgresql://user:pass@localhost/apex_db"
    )
```

**프로덕션 권장:** Redis (빠름, 간단) 또는 PostgreSQL (장기 보관, 분석 가능)

---

#### 문제 2: 감정 분류 모델의 낮은 정확도

**증상:**
- "실적이 예상을 상회하며 긍정적" → 모델 예측: **neg (부정)**
- "조정 압력이 강화되고 있다" → 모델 예측: **pos (긍정)**

**원인:**

학습 데이터 200건이 너무 적고, 로지스틱 회귀는 표현력이 제한적입니다.

```python
# sentiment/train_sentiment.py
# 200건 학습 데이터 → Train/Val split 160/40
# 40개 검증 세트에서 정확도: ~72% (낮음)
```

**해결안 1: Few-Shot LLM 분류**

```python
# sentiment/llm_sentiment.py
from langchain_ollama import OllamaLLM

def classify_sentiment_llm(text: str) -> dict:
    llm = OllamaLLM(model="exaone3.5:7.8b", temperature=0.1)

    prompt = f"""다음 주식 시장 관련 문장의 감정을 분류하라.

긍정 예시: "실적이 예상을 상회하며 주가 강세", "HBM 수요 급증으로 수혜"
부정 예시: "조정 압력 강화", "실적 악화 우려 확산"

문장: {text}

출력 형식 (JSON만):
{{"label": "pos" 또는 "neg", "confidence": 0.0~1.0}}
"""

    result = llm.invoke(prompt)
    # JSON 파싱 (strict_json_parser 재사용)
    ...
```

**해결안 2: FinBERT 같은 금융 특화 모델 파인튜닝**

- FinBERT (Financial BERT): 금융 뉴스로 사전학습된 BERT 모델
- 한국어 버전은 없지만, KoBERT + 금융 코퍼스로 직접 파인튜닝 가능
- 1000-2000건 라벨링 데이터 확보 시 80-85% 정확도 기대

**현재 선택:** Few-Shot LLM으로 임시 대응 중. 정확도 ~78%로 개선되었지만, 추론 속도가 느림 (500ms → 1.5s).

---

#### 문제 3: Re-ranking 부재로 인한 노이즈 문서

**증상:**
- 검색어: "SK하이닉스 HBM 실적"
- 검색 결과 1위: "SK하이닉스, 직원 복지 프로그램 확대" (벡터 유사도: 0.68)
- 검색 결과 2위: "SK하이닉스, HBM3E 양산으로 영업이익 증가" (벡터 유사도: 0.65)

**원인:**

벡터 유사도만으로는 **의미적 유사성**만 측정합니다. "SK하이닉스"가 문서에 많이 나오면 유사도가 높지만, **실제 질문과의 관련성**은 낮을 수 있습니다.

**해결안: Cross-Encoder Re-ranking**

```python
# utils/reranker.py
from sentence_transformers import CrossEncoder

# Cross-Encoder 모델 로드 (한번만 초기화)
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-12-v2')

def rerank_documents(query: str, docs: list, top_k: int = 5):
    """
    1단계: 벡터 검색으로 후보 20개 추출
    2단계: Cross-Encoder로 재정렬해서 상위 5개 반환
    """
    # 쿼리-문서 쌍 생성
    pairs = [[query, doc.page_content] for doc in docs]

    # Cross-Encoder 점수 계산 (높을수록 관련도 높음)
    scores = reranker.predict(pairs)

    # 점수 기준 정렬
    ranked = sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)

    return [doc for doc, score in ranked[:top_k]]
```

**LangChain 체인에 통합:**

```python
def build_summary_chain_with_reranking():
    vectorstore = get_vectorstore()

    # 1단계: 벡터 검색 (k=20, 후보 많이 뽑기)
    retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

    # 2단계: Re-ranking
    def retrieve_and_rerank(x):
        docs = retriever.invoke(x["company"])
        reranked = rerank_documents(x["company"], docs, top_k=5)
        return "\n\n---\n\n".join([doc.page_content for doc in reranked])

    get_context = RunnableLambda(retrieve_and_rerank)

    # 나머지 체인은 동일
    ...
```

**성능 개선:**
- Before Re-ranking: 5개 중 2-3개가 노이즈 문서
- After Re-ranking: 5개 중 4-5개가 실제 관련 문서

---

#### 문제 4: 단일 임베딩 모델 의존 → 특정 쿼리에서 낮은 리콜

**증상:**
- 검색어: "반도체 업황 전망" → BGE-m3로 검색 → 관련 문서 누락
- 같은 검색어 → OpenAI Embedding으로 검색 → 문서 잘 나옴

**원인:**

임베딩 모델마다 강점이 다릅니다. BGE-m3는 다국어에 강하지만, 특정 도메인(금융 전문 용어)에서는 OpenAI Embedding이나 금융 특화 모델이 더 나을 수 있습니다.

**해결안: Hybrid Search (벡터 + 키워드)**

```python
# Qdrant는 벡터 검색과 전문(Full-Text) 검색을 동시 지원
from qdrant_client.models import SearchRequest, FieldCondition, MatchText

# 벡터 검색 (의미적 유사성)
vector_results = vectorstore.similarity_search("반도체 업황 전망", k=10)

# 키워드 검색 (정확한 단어 매칭)
keyword_results = qdrant_client.scroll(
    collection_name="stock_segments",
    scroll_filter=FieldCondition(
        key="text",
        match=MatchText(text="반도체 업황")
    ),
    limit=10
)

# 두 결과 병합 (중복 제거)
combined = merge_and_deduplicate(vector_results, keyword_results)
```

이렇게 하면 BGE-m3가 놓친 문서도 키워드 매칭으로 보완할 수 있습니다.

---

### 개선 로드맵

#### Short-term (1-2주)

1. **Redis 세션 영속화** (우선순위: 높음)
   - 예상 작업: 3시간
   - 효과: 대화 맥락 보존, 사용자 경험 개선

2. **Re-ranking 추가** (우선순위: 중간)
   - 예상 작업: 5시간
   - 효과: 검색 품질 20-30% 향상

3. **에러 모니터링** (Sentry 통합)
   - 예상 작업: 2시간
   - 효과: 프로덕션 이슈 추적 가능

#### Mid-term (1-2개월)

1. **감정 분석 모델 개선**
   - FinBERT 파인튜닝 or Few-Shot LLM
   - 라벨링 데이터 1000건 확보 필요
   - 예상 작업: 2주

2. **Multi-company 배치 수집**
   - 현재: `./ingest_run.sh "SK하이닉스" 7` (하나씩)
   - 개선: `./ingest_run.sh "삼성전자,SK하이닉스,네이버" 7` (병렬)
   - 예상 작업: 1주

3. **Temporal 분석 기능**
   - 시간별 감정 추이 그래프
   - Qdrant 쿼리: `group_by("date")` + 감정 집계
   - 예상 작업: 1주

#### Long-term (3-6개월)

1. **Kubernetes 마이그레이션**
   - Helm Chart 작성
   - HPA (Horizontal Pod Autoscaler) 설정
   - 목표: 100 TPS 처리 가능

2. **Multi-modal RAG**
   - 주식 차트 이미지도 벡터화 (CLIP 모델)
   - "SK하이닉스 주가 차트" 이미지 검색 지원

3. **Agent 기반 분석**
   - LangGraph로 "뉴스 요약 → 감정 분석 → 시나리오 생성" 자동 파이프라인
   - 사용자 질문 없이도 매일 아침 시장 리포트 자동 생성

---

### 아쉬운 점: 코스콤 공모전 최종 발표를 못 간 아쉬움

이 프로젝트는 **코스콤 Apex 공모전 1차 심사를 통과**했습니다. 온프레미스 RAG 시스템이라는 기술적 시도와 금융 도메인 특화 설계가 인정받은 것 같아 기뻤습니다.

하지만 **최종 발표 일정이 다른 회사 면접과 겹치면서 발표에 참여하지 못했습니다.** 당시 취업 준비 중이었고, 면접 기회를 포기할 수 없는 상황이었습니다. 팀원들과 함께 준비했던 발표 자료와 데모 시연을 직접 보여드리지 못한 것이 가장 아쉬웠습니다.

결과적으로 이 시스템은 공모전 데모 환경에서 잘 돌아갔지만, **실제 서비스로 배포하지는 못했습니다.** 이유는:

1. **법적 문제**: 금융 정보 제공 서비스는 금융위원회 인가 필요. 학생 프로젝트로는 불가능.
2. **API 라이선스**: 네이버 API 개발자 계정은 일 25,000회 제한. 사용자 100명만 되어도 초과.
3. **인프라 비용**: Ollama GPU 인스턴스 (AWS g4dn.xlarge) = 월 $150+. 무료 서비스로 감당 불가.

그럼에도 불구하고, 이 경험은 제게 큰 자산이 되었습니다. **"아이디어를 실제 동작하는 시스템으로 만들어보기"**, **"로컬 LLM으로 실용적인 서비스 설계하기"**, **"워크플로 도구와 코드의 역할 분리"** — 이런 시도들이 이후 실무에서 RAG 시스템을 설계할 때 큰 도움이 되었습니다.

**만약 다시 한다면:**
- 법적 이슈 없는 도메인 선택 (예: 개발자 커뮤니티 Q&A RAG)
- 공개 API 대신 크롤링 or 자체 데이터 수집
- 초기에는 Ollama CPU 모드로 시작 (느리지만 무료)
- **그리고 무엇보다, 공모전 일정을 미리 확인하고 면접 일정과 겹치지 않도록 조율했을 것입니다!**

---

## 마무리

RAG는 결국 "검색 + 생성"이라는 단순한 조합이지만, 실제로 시스템을 구축하면 **수집 → 정제 → 청킹 → 임베딩 → 저장 → 검색 → 프롬프트 구성 → 생성 → 파싱**이라는 긴 파이프라인을 관리해야 합니다.

이 프로젝트에서 가장 중요했던 설계 결정은 **"수집은 워크플로(n8n)로, 분석은 코드(LangChain)로"** 역할을 분리한 것이었습니다. 각 도구의 강점을 살리면서, 한쪽이 실패해도 다른 쪽이 독립적으로 동작할 수 있는 구조를 만들었습니다.

완벽한 시스템은 아닙니다. 200건짜리 감정 분류 모델, 인메모리 세션, 단일 임베딩 모델 — 개선할 점이 많습니다. 하지만 **"작게 시작하고, 동작하는 것부터 만들자"**는 원칙으로 실제로 돌아가는 온프레미스 RAG 시스템을 완성할 수 있었고, 코스콤 공모전 1차 심사도 통과할 수 있었습니다.

비록 최종 발표는 면접 일정과 겹쳐 참여하지 못했지만, 이 과정에서 얻은 경험은 값졌습니다. **로컬 LLM의 가능성과 한계**, **워크플로 도구의 실전 활용**, **온프레미스 벡터 DB 운영**, **프롬프트 엔지니어링의 중요성** — 이 모든 것이 이후 실무에서 RAG 시스템을 설계하는 데 큰 밑거름이 되었습니다.

이 글이 비슷한 프로젝트를 시작하려는 분들에게 참고가 되었으면 합니다. 그리고 공모전에 도전하시는 분들께는 꼭 **발표 일정을 미리 확인하세요!** 😅
