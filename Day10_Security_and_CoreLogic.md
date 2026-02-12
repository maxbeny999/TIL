# [SeSAC 도봉] 10일차 : 시스템의 보안과 데이터 로직의 본질

프레임워크가 제공하는 화려한 기능 뒤에는 항상 견고한 파이썬의 기초 로직이 숨어있습니다. 오늘은 FastAPI를 활용해 인증(Authentication)과 의존성 주입(Dependency Injection)이라는 고급 기술을 구현하는 한편, 복잡한 중첩 데이터를 파이썬 기본 문법만으로 능숙하게 다루는 훈련을 병행했습니다. "답도 안 나오던" 시기를 지나 이제는 실버 난이도의 문제를 스스로 해결해내는 성장을 확인하며, 개발자로서의 기초 체력이 단단해지고 있음을 느낀 하루였습니다.

---

## 1. 인증 심화와 의존성 (Dependencies)

`mysite4` 프로젝트를 통해 단순히 로그인을 구현하는 것을 넘어, 시스템이 사용자를 식별하고 보호하는 전체적인 흐름을 설계했습니다.

### 의존성 주입의 강력함 (Dependency Injection)
가장 인상 깊었던 점은 `Depends`를 활용한 의존성 주입 패턴입니다. 이전에는 모든 함수에서 토큰을 검사하는 코드를 반복해야 했을 것입니다. 하지만 `get_current_user`라는 의존성 함수를 만들고, 이를 라우터에 주입함으로써 **"비즈니스 로직은 사용자가 누구인지 검증하는 절차를 몰라도 된다"**는 책임의 분리를 실현했습니다.

이제 `current_user: User = Depends(get_current_user)` 한 줄이면, 프레임워크가 알아서 토큰을 추출하고, 서명을 검증하고, DB에서 유저를 찾아 제 손에 쥐여줍니다. 이것이 프레임워크를 사용하는 진짜 이유임을 깨달았습니다.

### 관계의 복잡성 해결 (1:N, N:M)
게시글(Post)과 태그(Tag)의 다대다 관계, 게시글과 댓글(Comment)의 일대다 관계를 모두 구현하며 ORM의 동작 원리를 재확인했습니다. 특히 `PostService`에서 게시글을 조회할 때 `selectinload`와 `joinedload`를 적절히 섞어 사용하여 N+1 문제를 방지하고, 태그 추가 시 `Association Proxy`나 연결 테이블(`PostTag`)을 직접 제어하는 로직을 통해 데이터 무결성을 보장하는 방법을 익혔습니다.

---

## 2. Python Core Logic : 데이터 핸들링의 본질

오후에는 `prac_day15.ipynb`를 통해 리스트와 딕셔너리가 중첩된 JSON 형태의 데이터를 다루는 훈련을 진행했습니다. 프레임워크 없이 순수 파이썬 문법만으로 데이터를 가공하며 논리적 사고력을 점검했습니다.

### 중첩 구조의 평탄화 (Flattening)
IoT 센서 로그 데이터에서 에러 메시지만 추출하는 과정은 2중 `for`문이 필요했습니다. 이를 리스트 컴프리헨션으로 `[err for d in data for err in d['errors']]`와 같이 간결하게 표현하고, `set`을 이용해 중복을 제거하는 패턴은 데이터 전처리의 핵심 기법임을 확인했습니다.

### 그룹핑(Grouping) 알고리즘
딕셔너리의 키가 존재하지 않을 때 리스트를 생성하고, 존재하면 추가(`append`)하는 로직은 알고리즘 문제에서 빈번하게 등장하는 패턴입니다. `active_devices` 필터링부터 그룹핑까지 직접 구현해보며, 예전에는 손도 못 대던 문제들이 이제는 머릿속에서 구조가 그려지는 경험을 했습니다. 아직 속도는 빠르지 않더라도, 실버 5 난이도의 문제를 스스로 풀어낼 수 있게 된 것은 분명한 성장의 증거입니다.

```python
# [FastAPI] 의존성 주입을 통한 인증 처리
# 라우터는 '검증 과정'을 모릅니다. 단지 '결과(user)'만 받습니다.
@router.post("", response_model=Post2DetailResponse)
def create_post(
    data: Post2Create,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),  # 핵심: 토큰 검증 로직의 캡슐화
):
    return post2_service.create_post(db, data, current_user)

# [Python Core] 복합적인 데이터 집계 로직 (Day 15 실습)
def analyze_sensor_data(data_list: list) -> dict:
    # 1. 중첩 리스트 평탄화 및 중복 제거
    error_list = list(set(err for d in data_list for err in d['errors']))
    
    # 2. 조건부 최대값 찾기 (Generator Expression 활용)
    max_temp = max(
        (d['value'] for d in data_list if 'temp' in d['device'] and not d['errors']), 
        default=None
    )
    
    return {"error_list": error_list, "max_temp": max_temp}