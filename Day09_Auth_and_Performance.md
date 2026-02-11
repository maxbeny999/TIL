# [SeSAC 도봉] 9일차 : 경험이 이론을 만났을 때 (인증과 성능)

과거 프로젝트에서 인증 기능을 맡았을 때는 구글링으로 찾은 코드를 끼워 맞추며 기능 구현에만 급급했었습니다. 소위 '바이브 코딩'으로 완성은 했지만, 내부 원리를 명확히 설명하기 어려웠던 부채감이 있었습니다. 오늘 FastAPI의 인증(Authentication) 로직과 데이터베이스 성능 이슈(N+1)를 깊이 있게 학습하며, 그동안 막연하게 사용했던 코드들이 비로소 머릿속에서 논리적인 체계를 갖추게 되었습니다.

---

## 1. 인증(Authentication) : 마법이 아닌 논리의 흐름

`mysite4` 프로젝트에 JWT(JSON Web Token) 기반의 로그인 시스템을 구축하며, 보안은 마법 같은 암호화가 아니라 철저한 검증 절차의 연속임을 확인했습니다.

### 해싱과 검증의 분리
비밀번호를 데이터베이스에 그대로 저장하는 것이 왜 위험한지, 그리고 `bcrypt` 라이브러리가 어떻게 단방향 해싱을 수행하는지 코드로 확인했습니다. 과거에는 단순히 복사해서 썼던 `hashpw`와 `checkpw` 함수가, 실제로는 난수(Salt)를 섞어 해시를 생성하고 이를 저장된 값과 대조하는 정교한 수학적 과정임을 이해했습니다.

### 세션의 관리와 토큰
로그인 성공 시 서버가 세션을 기억하는 대신, 사용자에게 출입증(Access Token)을 발급하고 유효기간을 설정하는 Stateless 구조의 이점을 배웠습니다. `AuthService` 클래스에서 비밀번호 검증부터 토큰 생성(`jwt.encode`)까지의 책임을 분리하여 구현함으로써, 유지보수하기 좋은 인증 로직이 무엇인지 깨달았습니다.

---

## 2. 성능 최적화 : N+1 문제의 발견과 해결

`nplusone` 실습을 통해 ORM(Object-Relational Mapping)이 주는 편리함 이면에 숨겨진 성능 저하의 위험성을 목격했습니다.

### 보이지 않는 쿼리의 공포
게시글 목록을 조회할 때, 코드상으로는 단 한 번의 `select(Post)`를 호출했을 뿐입니다. 하지만 각 게시글의 작성자(`post.user`)나 댓글(`post.comments`)에 접근하는 순간, 게시글의 개수만큼 추가적인 SELECT 쿼리가 발생하는 현상을 확인했습니다. 이는 사용자가 적을 땐 드러나지 않다가 서비스 규모가 커지면 서버를 마비시키는 치명적인 기술 부채임을 절감했습니다.

### Eager Loading 전략
이를 해결하기 위해 데이터를 가져올 때 연관된 데이터도 즉시 로딩하는 전략을 학습했습니다.
* **joinedload:** N:1 관계에서 SQL의 `JOIN`을 사용하여 한 번에 데이터를 가져옵니다.
* **selectinload:** 1:N 관계에서 `IN` 절을 사용하여 메인 쿼리 1번, 연관 데이터 쿼리 1번으로 나누어 효율적으로 가져옵니다.

상황에 맞는 로딩 전략을 선택하는 것이 백엔드 개발자의 중요한 역량임을 알게 되었습니다.

---

## 3. 개인 학습 : 복습과 실전 압축

밀린 진도를 따라잡기 위해 개인 프로젝트 `Netfliz` 구축과 실전 압축(Blitz) 문제 풀이를 병행했습니다.

### 구조의 체화 (Netfliz)
수업 시간에 배운 내용을 단순히 따라 치는 것을 넘어, 바닥부터 다시 `Netfliz`라는 미니 프로젝트를 설계하며 디렉토리 구조(Router-Service-Repository)를 손에 익혔습니다. 의존성 주입(`Depends`)과 Pydantic 스키마 정의 과정을 반복하며 프레임워크의 흐름을 완전히 내 것으로 만들었습니다.

### 알고리즘과 파이썬 심화 (Blitz)
시간 관계상 효율적인 학습을 위해 핵심 문법과 알고리즘을 압축적으로 훈련했습니다.
* **자료구조:** 스택(Stack)을 활용해 괄호의 짝(`()`)을 맞추는 문제(백준 9012)를 해결하며 LIFO 구조의 유용성을 복습했습니다.
* **Pythonic Code:** `*args`, `**kwargs`를 활용한 동적 쿼리 생성기와, `lambda`를 이용한 다중 조건 정렬(중요도 순 정렬 후 시간순 정렬)을 구현하며 파이썬 고유의 간결하고 강력한 문법을 재확인했습니다.

```python
# [Auth Service] : 명시적인 책임의 분리 (로그인 로직)
def login(self, db: Session, data: UserLogin) -> str:
    # 1. 사용자 존재 여부 확인 (Repository 계층 활용)
    user = user_repository.find_by_email(db, data.email)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # 2. 비밀번호 검증 (해시 비교)
    if not self._verify_password(data.password, user.password):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # 3. 토큰 발급 (비즈니스 로직)
    return self._create_access_token(user.id)

# [Algorithm] : 람다를 활용한 다중 조건 정렬 (Day 14 Blitz)
# 우선순위: High(3) > Medium(2) > Low(1), 이후 시간 내림차순
priority_map = {"High": 3, "Medium": 2, "Low": 1}
sorted_logs = sorted(
    logs, 
    key=lambda x: (priority_map[x['priority']], x['time']), 
    reverse=True
)