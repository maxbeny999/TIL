# [SeSAC 도봉] 2월 5일 : 데이터의 제어와 계층적 아키텍처

파이썬의 기초 문법을 통해 데이터가 메모리 상에서 어떻게 조작되는지 복습하고, FastAPI를 활용하여 그 데이터를 웹상에서 효율적으로 전달하는 구조를 설계했습니다. 단순한 기능 구현을 넘어, 코드가 비대해지는 것을 방지하고 유지보수성을 확보하기 위해 왜 역할을 나누어야 하는지에 대한 필연성을 깨달은 하루였습니다.

---

## 1. 기초 문법의 재해석 : 논리의 원자(Atom)

오전에는 `prac_day10`과 `prac_day11` 과제를 통해 리스트, 딕셔너리, 그리고 제어문(for, if)의 조합을 집중적으로 점검했습니다.

### 복합 자료구조의 핸들링
단순히 문법을 아는 것과 데이터를 자유자재로 다루는 것은 별개의 영역임을 확인했습니다. 특히 딕셔너리가 요소로 들어있는 리스트(List of Dictionaries)를 순회하며 특정 조건에 맞는 데이터만 추출하거나 가공하는 로직은, 웹 백엔드에서 데이터베이스의 결과를 처리하는 과정과 정확히 일치했습니다. 결국 서비스의 복잡한 로직도 가장 작은 단위인 변수와 제어문의 논리적 결합으로 이루어짐을 확인했습니다.

---

## 2. FastAPI 아키텍처 : 책임의 분산과 위임

오후에는 `Product`, `TodoList`, 그리고 `Mysite4` 프로젝트를 실습하며, 하나의 파일(`main.py`)에 뭉쳐있던 코드를 역할에 따라 분리하는 리팩토링 과정을 경험했습니다.

### 계층형 아키텍처 (Layered Architecture)의 도입
코드가 길어질수록 가독성이 떨어지고 수정이 어려워지는 '스파게티 코드'의 위험성을 인지했습니다. 이를 해결하기 위해 `Mysite4` 실습에서는 전체 시스템을 다음과 같은 계층으로 분리했습니다.

1.  **Router (Controller):** 사용자의 요청(Request)을 가장 먼저 받아 입력값을 검증하고, 적절한 서비스로 작업을 지시하는 창구 역할을 합니다.
2.  **Service:** 실제 비즈니스 로직(데이터 가공, 계산, 예외 처리)을 수행하는 핵심 두뇌입니다.
3.  **Model/Schema:** 데이터가 어떤 형태여야 하는지 정의하는 설계도입니다.

이러한 분리를 통해 각 파일은 자신의 역할에만 집중할 수 있게 되었으며(SRP), 에러가 발생했을 때 어느 계층의 문제인지 즉각적으로 파악할 수 있는 구조적 이점을 얻었습니다.

### 데이터 흐름의 시각화
오늘 구현한 구조는 요청이 들어오면 라우터가 서비스를 호출하고, 서비스가 처리된 결과를 반환하는 단방향 흐름을 가집니다. 이를 도식화하여 머릿속의 추상적인 개념을 구체화했습니다.

```mermaid
sequenceDiagram
    participant Client as 사용자 (Client)
    participant Router as 라우터 (Router)
    participant Service as 서비스 (Service)
    participant DB as 데이터 (Repository/DB)

    Client->>Router: 1. 요청 (GET /todos)
    Note over Router: 입력 데이터 검증 (Schema)
    Router->>Service: 2. 비즈니스 로직 호출
    Service->>DB: 3. 데이터 조회 요청
    DB-->>Service: 4. 원본 데이터 반환
    Note over Service: 데이터 가공 및 필터링
    Service-->>Router: 5. 결과 반환
    Router-->>Client: 6. 응답 (Response)

## 라우터(APIRouter)의 확장성
include_router 메서드를 통해 메인 앱과 서브 기능을 연결하는 방식은 프레임워크가 제공하는 모듈화의 정수였습니다. product와 todos 같이 성격이 다른 기능들을 물리적으로 격리함으로써, 협업 시 발생할 수 있는 코드 충돌을 최소화하고 독립적인 개발이 가능한 환경을 구축했습니다.

# [구조적 분리의 예시]
# router는 오직 '요청 분배'에만 집중하고,
# 구체적인 로직은 service 계층으로 위임합니다.

from fastapi import APIRouter, Depends
from mysite4.services import post_service
from mysite4.schemas import PostCreate

router = APIRouter(prefix="/posts")

@router.post("/", status_code=201)
def create_post(post_data: PostCreate):
    # 라우터는 '어떻게' 저장하는지 모릅니다.
    # 단지 서비스에게 '저장해달라'고 요청할 뿐입니다.
    return post_service.create_new_post(post_data)