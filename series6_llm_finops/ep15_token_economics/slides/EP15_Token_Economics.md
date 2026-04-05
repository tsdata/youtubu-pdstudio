---
marp: true
theme: default
paginate: true
header: "EP15. 에이전트 토큰 경제학"
footer: "고급 AI 채널 | Series 6 · LLM FinOps"
---

# EP15. 에이전트 토큰 경제학

## 에이전트 1회 호출에 $8? 비용 폭발을 막는 실전 전략

> Multi-turn 비용 분석 · 모델별 가격 비교 · Langfuse 비용 대시보드

난이도: ⭐⭐

---

## 목차

**비용 구조 이해 (섹션 1-5)**
1. 문제 제기: 에이전트 1회 호출에 $8?
2. 에이전트 vs 챗봇 비용 차이 (3~10x)
3. Multi-turn Loop 비용 분석
4. 입력/출력 토큰 비대칭
5. Context Accumulation 비용 시뮬레이션

**실전 관리 (섹션 6-10)**
6. 주요 모델별 비용 비교표
7. 비용 추적 방법론: 태깅 전략
8. Langfuse 비용 대시보드 구축
9. 비용 알림 시스템 설계
10. 비용 최적화 전략 정리 + Exercise

---

## 1. 문제 제기: 에이전트 1회 호출에 $8?

**실제 사례: 코드 리뷰 에이전트**

| 단계 | 설명 | 토큰 수 |
|------|------|---------|
| Turn 1 | 파일 읽기 요청 | 2,000 |
| Turn 2 | 코드 분석 | 8,000 |
| Turn 3 | 리뷰 생성 | 15,000 |
| Turn 4 | 수정 제안 | 22,000 |
| Turn 5 | 최종 정리 | 30,000 |
| **합계** | **입력+출력** | **~77,000** |

**Claude Sonnet 기준**: 입력 $3/M + 출력 $15/M = **약 $1.5~8/회**

> "챗봇은 $0.01인데, 같은 모델의 에이전트는 왜 $8인가?"

---

## 2. 에이전트 vs 챗봇 비용 차이

```mermaid
flowchart LR
    subgraph Chatbot ["💬 챗봇: 1회 호출"]
        direction TB
        U1("사용자 질문<br/>500 tokens") --> L1("LLM 응답<br/>300 tokens")
    end

    subgraph Agent ["🤖 에이전트: N회 루프"]
        direction TB
        U2("사용자 질문<br/>500 tokens") --> A1("Turn 1: 계획<br/>+1,000 tokens")
        A1 --> A2("Turn 2: 도구 호출<br/>+3,000 tokens")
        A2 --> A3("Turn 3: 분석<br/>+5,000 tokens")
        A3 --> A4("Turn 4: 종합<br/>+8,000 tokens")
        A4 --> A5("Turn 5: 최종 답변<br/>+12,000 tokens")
    end
```

| | 챗봇 | 에이전트 |
|---|------|---------|
| **호출 횟수** | 1회 | 5~15회 |
| **컨텍스트** | 고정 | 매 턴마다 누적 |
| **총 토큰** | ~800 | ~30,000~100,000 |
| **비용 배수** | 1x | **3~10x** |

**핵심 원인**: 에이전트는 매 턴마다 **이전 대화 전체**를 다시 보냄

---

## 3. Multi-turn Loop 비용 분석

```mermaid
sequenceDiagram
    autonumber
    actor U as 👤 사용자
    participant A as 🤖 에이전트
    participant LLM as 🧠 LLM API

    U->>A: 작업 요청
    
    A->>LLM: Turn 1: [시스템 + 질문] = 1,000 tokens
    LLM-->>A: 응답 500 tokens

    A->>LLM: Turn 2: [시스템 + 질문 + Turn1] = 2,500 tokens
    LLM-->>A: 응답 800 tokens

    A->>LLM: Turn 3: [시스템 + 질문 + Turn1 + Turn2] = 4,800 tokens
    LLM-->>A: 응답 1,200 tokens

    A->>LLM: Turn 4: [전체 누적] = 8,000 tokens
    LLM-->>A: 응답 1,500 tokens

    A-->>U: 최종 결과
    
    Note over A,LLM: 입력 토큰이 매 턴마다 누적 증가!
```

**각 턴의 입력 토큰 = 이전 모든 턴의 합 + 새 입력**

Turn N의 입력 토큰 수 = `base + sum(output_1 ... output_N-1)`

---

## 4. 입력/출력 토큰 비대칭

**왜 출력 토큰이 더 비싼가?**

| | 입력 토큰 | 출력 토큰 |
|---|----------|----------|
| **처리 방식** | 병렬 처리 (한 번에) | 순차 생성 (하나씩) |
| **GPU 활용** | 배치 최적화 가능 | auto-regressive |
| **지연 시간** | 짧음 | 길음 |
| **상대 가격** | 1x | **4~8x** |

```mermaid
pie title 에이전트 비용 구성 (일반적 비율)
    "입력 토큰 비용" : 35
    "출력 토큰 비용" : 65
```

> 토큰 수는 입력이 많지만, **비용은 출력이 더 크다**

**2026년 주요 모델 가격표**

| 모델 | 입력 ($/M) | 출력 ($/M) | 출력/입력 배율 |
|------|-----------|-----------|--------------|
| Claude Sonnet 4 | $3.00 | $15.00 | 5.0x |
| Claude Haiku 3.5 | $0.80 | $4.00 | 5.0x |
| GPT-4o | $2.50 | $10.00 | 4.0x |
| GPT-4o mini | $0.15 | $0.60 | 4.0x |
| Gemini 2.5 Pro | $1.25 | $10.00 | 8.0x |
| Gemini 2.5 Flash | $0.15 | $0.60 | 4.0x |

---

## 5. Context Accumulation 비용 시뮬레이션

**10턴 에이전트의 총 입력 토큰 계산**

| 턴 | 새 출력 | 누적 입력 | 누적 비용 ($3/M) |
|----|--------|----------|-----------------|
| 1 | 500 | 1,000 | $0.003 |
| 2 | 800 | 2,500 | $0.008 |
| 3 | 1,200 | 4,500 | $0.014 |
| 4 | 1,500 | 7,200 | $0.022 |
| 5 | 2,000 | 10,700 | $0.032 |
| 6 | 1,800 | 14,500 | $0.044 |
| 7 | 2,200 | 18,500 | $0.056 |
| 8 | 2,500 | 23,200 | $0.070 |
| 9 | 2,000 | 27,700 | $0.083 |
| 10 | 2,500 | 32,200 | $0.097 |
| **합계** | **17,000** | **누적 합 ~142K** | **$0.43 입력만** |

> 입력 토큰 총합이 **O(n^2)** 으로 증가한다!

---

## 5-1. O(n^2) 성장 시각화

```mermaid
flowchart TD
    subgraph Growth ["📈 컨텍스트 누적 성장"]
        direction TB
        T1["Turn 1<br/>■"]:::t1
        T2["Turn 2<br/>■■■"]:::t2
        T3["Turn 3<br/>■■■■■■"]:::t3
        T4["Turn 4<br/>■■■■■■■■■■"]:::t4
        T5["Turn 5<br/>■■■■■■■■■■■■■■■"]:::t5
    end

    classDef t1 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#000
    classDef t2 fill:#c8e6c9,stroke:#4caf50,stroke-width:2px,color:#000
    classDef t3 fill:#fff9c4,stroke:#fbc02d,stroke-width:2px,color:#000
    classDef t4 fill:#ffe0b2,stroke:#f57c00,stroke-width:2px,color:#000
    classDef t5 fill:#ffcdd2,stroke:#e53935,stroke-width:2px,color:#000
```

**수학적 분석**:
- 턴 당 평균 출력: `avg_output`
- 총 입력 토큰 = `n * base + avg_output * n * (n-1) / 2`
- **n이 클수록 입력 비용이 n^2에 비례하여 폭발**

---

## 6. 주요 모델별 비용 비교표

| 모델 | 입력 ($/M) | 출력 ($/M) | 컨텍스트 | 속도 | 추천 용도 |
|------|-----------|-----------|---------|------|----------|
| **Claude Sonnet 4** | $3.00 | $15.00 | 200K | 빠름 | 코드 생성, 분석 |
| **Claude Haiku 3.5** | $0.80 | $4.00 | 200K | 매우빠름 | 분류, 추출 |
| **Claude Opus 4** | $15.00 | $75.00 | 200K | 느림 | 고난이도 추론 |
| **GPT-4o** | $2.50 | $10.00 | 128K | 빠름 | 범용 |
| **GPT-4o mini** | $0.15 | $0.60 | 128K | 매우빠름 | 경량 작업 |
| **Gemini 2.5 Pro** | $1.25 | $10.00 | 1M | 보통 | 장문 분석 |
| **Gemini 2.5 Flash** | $0.15 | $0.60 | 1M | 매우빠름 | 경량 + 장문 |
| **Llama 3.3 70B** | $0.40 | $0.40 | 128K | 빠름 | 자체 호스팅 |

---

## 6-1. 모델 선택 의사결정 트리

```mermaid
flowchart TD
    START("🎯 작업 유형?"):::start --> Q1{"복잡도<br/>높음?"}
    Q1 -->|"예"| Q2{"예산<br/>충분?"}
    Q1 -->|"아니오"| Q3{"속도<br/>중요?"}
    
    Q2 -->|"예"| R1("Claude Opus 4<br/>$15/$75"):::premium
    Q2 -->|"아니오"| R2("Claude Sonnet 4<br/>$3/$15"):::standard
    
    Q3 -->|"예"| R3("GPT-4o mini / Haiku<br/>$0.15~0.80"):::budget
    Q3 -->|"아니오"| Q4{"장문 컨텍스트<br/>필요?"}
    
    Q4 -->|"예"| R4("Gemini 2.5 Flash<br/>$0.15/$0.60 · 1M"):::budget
    Q4 -->|"아니오"| R3

    classDef start fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef premium fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px,color:#000
    classDef standard fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
    classDef budget fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
```

---

## 7. 비용 추적 방법론: 태깅 전략

```mermaid
flowchart LR
    REQ("📨 LLM 요청"):::req --> TAG("🏷️ 태그 부착"):::tag
    TAG --> T1("model: sonnet-4")
    TAG --> T2("feature: code-review")
    TAG --> T3("team: backend")
    TAG --> T4("env: production")
    
    T1 & T2 & T3 & T4 --> LF("📊 Langfuse<br/>비용 집계"):::lf
    
    LF --> D1("모델별 비용"):::dash
    LF --> D2("기능별 비용"):::dash
    LF --> D3("팀별 비용"):::dash
    LF --> D4("일별 추세"):::dash

    classDef req fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef tag fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef lf fill:#c8e6c9,stroke:#388e3c,stroke-width:2px,color:#000
    classDef dash fill:#e1bee7,stroke:#8e24aa,stroke-width:2px,color:#000
```

**태깅 규칙**
| 태그 키 | 값 예시 | 용도 |
|---------|--------|------|
| `model` | sonnet-4, haiku-3.5 | 모델별 비용 추적 |
| `feature` | code-review, summarize | 기능별 비용 분석 |
| `team` | backend, data, infra | 팀별 비용 할당 |
| `env` | prod, staging, dev | 환경별 분리 |
| `priority` | high, medium, low | 비용 최적화 우선순위 |

---

## 8. Langfuse 비용 대시보드 구축

```mermaid
flowchart TD
    subgraph App ["🖥️ 애플리케이션"]
        LLM("LLM 호출"):::llm --> CB("CallbackHandler"):::cb
    end

    CB -->|"trace + cost"| LF("📡 Langfuse Server"):::lf

    subgraph Dashboard ["📊 대시보드"]
        LF --> M1("총 비용 추이"):::dash
        LF --> M2("모델별 비용 비교"):::dash
        LF --> M3("기능별 TOP 5"):::dash
        LF --> M4("이상치 탐지"):::dash
    end

    classDef llm fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef cb fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef lf fill:#c8e6c9,stroke:#388e3c,stroke-width:2px,color:#000
    classDef dash fill:#e1bee7,stroke:#8e24aa,stroke-width:2px,color:#000
```

**Langfuse CallbackHandler 기본 코드**

```python
from langfuse.callback import CallbackHandler

handler = CallbackHandler(
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    host="https://cloud.langfuse.com",
    tags=["feature:code-review", "team:backend"],
)

response = llm.invoke("코드를 리뷰해줘", config={"callbacks": [handler]})
```

---

## 9. 비용 알림 시스템 설계

```mermaid
stateDiagram-v2
    [*] --> Normal : 비용 모니터링 시작
    
    Normal --> Warning : 일일 비용 > $50
    Warning --> Critical : 일일 비용 > $100
    Critical --> Emergency : 일일 비용 > $200
    
    Warning --> Normal : 비용 감소
    Critical --> Warning : 비용 감소
    Emergency --> Critical : 비용 감소
    
    Normal --> Normal : 정상 범위
    
    note right of Warning
        Slack 알림 전송
        팀 리더에게 통보
    end note
    
    note right of Critical
        즉시 Slack + Email
        비용 원인 분석 시작
    end note
    
    note right of Emergency
        자동 트래픽 제한
        경영진 보고
    end note
```

**알림 임계값 설계**

| 레벨 | 임계값 | 액션 |
|------|-------|------|
| **Normal** | < $50/일 | 모니터링만 |
| **Warning** | $50~100/일 | Slack 알림 |
| **Critical** | $100~200/일 | Slack + Email + 원인 분석 |
| **Emergency** | > $200/일 | 자동 제한 + 경영진 보고 |

---

## 9-1. 비용 알림 구현 코드

```python
from dataclasses import dataclass
from enum import Enum

class AlertLevel(Enum):
    NORMAL = "normal"
    WARNING = "warning"
    CRITICAL = "critical"
    EMERGENCY = "emergency"

@dataclass
class CostAlert:
    level: AlertLevel
    daily_cost: float
    threshold: float
    message: str

def check_cost_alert(daily_cost: float) -> CostAlert:
    if daily_cost > 200:
        return CostAlert(AlertLevel.EMERGENCY, daily_cost, 200,
                         "비상! 자동 트래픽 제한 발동")
    elif daily_cost > 100:
        return CostAlert(AlertLevel.CRITICAL, daily_cost, 100,
                         "위험: 비용 원인 분석 필요")
    elif daily_cost > 50:
        return CostAlert(AlertLevel.WARNING, daily_cost, 50,
                         "경고: 비용 증가 감지")
    return CostAlert(AlertLevel.NORMAL, daily_cost, 50,
                     "정상 범위")
```

---

## 10. 비용 최적화 전략 정리

```mermaid
mindmap
  root((LLM FinOps<br/>최적화 전략))
    🔀 Model Routing
      작업별 최적 모델 분배
      복잡 → Sonnet, 단순 → Haiku
      비용 60-80% 절감 가능
    🗜️ Context Compression
      불필요한 컨텍스트 제거
      요약 후 재주입
      토큰 50%+ 절감
    💾 Semantic Cache
      동일/유사 질문 캐싱
      임베딩 유사도 기반
      반복 호출 90%+ 제거
    📊 비용 모니터링
      Langfuse 태깅
      일일 비용 대시보드
      임계값 알림
    ⚙️ 프롬프트 최적화
      간결한 시스템 프롬프트
      Few-shot 최소화
      출력 길이 제한
```

---

## 10-1. 최적화 전략 비교표

| 전략 | 구현 난이도 | 절감 효과 | 적용 시점 |
|------|-----------|----------|----------|
| **Model Routing** | 중간 | 60~80% | 즉시 적용 가능 |
| **Context Compression** | 중간 | 30~50% | 장문 에이전트에 효과적 |
| **Semantic Cache** | 높음 | 최대 90% | 반복 패턴이 있을 때 |
| **프롬프트 최적화** | 낮음 | 10~30% | 항상 먼저 적용 |
| **출력 길이 제한** | 낮음 | 20~40% | max_tokens 설정 |
| **배치 API** | 낮음 | 50% | 비실시간 작업 |

**ROI 순위**: 프롬프트 최적화 > Model Routing > 출력 제한 > 캐싱 > 압축

---

## 10-2. Model Routing 패턴

```mermaid
flowchart TD
    REQ("📨 사용자 요청"):::req --> CLS("🏷️ 복잡도 분류<br/>(Haiku로 분류)"):::cls
    
    CLS -->|"단순 질문"| H("Claude Haiku 3.5<br/>$0.80/$4.00"):::budget
    CLS -->|"중간 작업"| S("Claude Sonnet 4<br/>$3.00/$15.00"):::standard
    CLS -->|"고난이도"| O("Claude Opus 4<br/>$15.00/$75.00"):::premium
    
    H & S & O --> RES("📤 응답"):::res

    classDef req fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef cls fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef budget fill:#c8e6c9,stroke:#388e3c,stroke-width:2px,color:#000
    classDef standard fill:#bbdefb,stroke:#1976d2,stroke-width:2px,color:#000
    classDef premium fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px,color:#000
    classDef res fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#000
```

```python
def route_model(query: str) -> str:
    """복잡도에 따라 모델을 선택합니다."""
    # 1단계: 저비용 모델로 복잡도 판단
    complexity = classify_complexity(query)  # haiku로 분류
    
    if complexity == "simple":
        return "claude-haiku-3.5"    # $0.80/M
    elif complexity == "medium":
        return "claude-sonnet-4"     # $3.00/M
    else:
        return "claude-opus-4"       # $15.00/M
```

---

## 11. Exercise 1: 비용 시뮬레이터 구축

**목표**: 에이전트의 턴 수와 모델에 따른 비용을 계산하는 시뮬레이터 구현

**단계**:
1. 모델별 가격 딕셔너리 정의
2. 턴별 컨텍스트 누적 로직 구현
3. 총 비용 계산 함수 작성
4. matplotlib로 턴 수별 비용 그래프 시각화
5. 3개 모델을 비교하는 차트 생성

---

## 12. Exercise 2: Langfuse 비용 대시보드

**목표**: Langfuse로 LLM 호출 비용을 추적하고 알림 시스템 구현

**단계**:
1. Langfuse CallbackHandler 설정
2. 여러 모델로 LLM 호출 실행 (태그 포함)
3. 비용 데이터 집계 (모델별, 기능별)
4. 임계값 기반 알림 함수 구현
5. pandas DataFrame으로 비용 리포트 생성

**제출**: 비용 시뮬레이션 결과 + 대시보드 스크린샷 + 알림 로직

---

## 정리 & 마무리

**오늘 배운 것**

- 에이전트는 Multi-turn Loop 때문에 챗봇보다 **3~10x 비싼 비용** 발생
- 컨텍스트 누적으로 입력 토큰이 **O(n^2)** 증가하는 구조 이해
- 출력 토큰이 입력보다 **4~8배 비싼** 비대칭 가격 구조
- Langfuse **태깅 + 대시보드**로 비용을 모델/기능/팀별로 추적
- **임계값 기반 알림**으로 비용 폭발을 사전에 방지
- Model Routing, Context Compression, Caching으로 **60~90% 비용 절감** 가능

**다음 EP16**: Model Routing 심화 -- 작업별 최적 모델을 자동 선택하는 방법

> 전체 코드는 GitHub 레포에서, 심화 내용은 커뮤니티에서
