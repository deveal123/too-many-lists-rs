# 대칭 구조의 형편없는 것들 (Symmetric Junk)

자, 이제 이 지긋지긋한 조합론적 좌우쌍 대칭(combinatoric symmetry) 패턴의 노동을 싹 다 한방에 해치워 끝내버립시다(get all that over with).

우리가 할 일이라곤 그저 기본적인 텍스트 복사-붙여넣기 글자 치환 작업뿐입니다:

```text
tail <-> head
next <-> prev
front -> back
```

오, 그리고 덤으로 훔쳐보기(peeking) 용 `_mut` 가변 참조 변종 메서드 버전들도 잊지 말고 미리미리 추가해 박아놔야 합니다.

```rust ,ignore
use std::cell::{Ref, RefCell, RefMut};

//..

pub fn push_back(&mut self, elem: T) {
    let new_tail = Node::new(elem);
    match self.tail.take() {
        Some(old_tail) => {
            old_tail.borrow_mut().next = Some(new_tail.clone());
            new_tail.borrow_mut().prev = Some(old_tail);
            self.tail = Some(new_tail);
        }
        None => {
            self.head = Some(new_tail.clone());
            self.tail = Some(new_tail);
        }
    }
}

pub fn pop_back(&mut self) -> Option<T> {
    self.tail.take().map(|old_tail| {
        match old_tail.borrow_mut().prev.take() {
            Some(new_tail) => {
                new_tail.borrow_mut().next.take();
                self.tail = Some(new_tail);
            }
            None => {
                self.head.take();
            }
        }
        Rc::try_unwrap(old_tail).ok().unwrap().into_inner().elem
    })
}

pub fn peek_back(&self) -> Option<Ref<T>> {
    self.tail.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}

pub fn peek_back_mut(&mut self) -> Option<RefMut<T>> {
    self.tail.as_ref().map(|node| {
        RefMut::map(node.borrow_mut(), |node| &mut node.elem)
    })
}

pub fn peek_front_mut(&mut self) -> Option<RefMut<T>> {
    self.head.as_ref().map(|node| {
        RefMut::map(node.borrow_mut(), |node| &mut node.elem)
    })
}
```

그리고 내친김에 우리들의 테스트 관문 검역소 코드도 이 대칭 구도에 맞게 대거 살을 왕창 찌워 도배(massively flesh out)해 보강해 줍시다:


```rust ,ignore
#[test]
fn basics() {
    let mut list = List::new();

    // 빈 깡통 리스트 녀석이 똑바로 정신 차리고 행동거지 군기 똑바로 처먹었는지 검수
    assert_eq!(list.pop_front(), None);

    // 머리에 알맹이 좀 구겨 넣어 바구니 살찌우기
    list.push_front(1);
    list.push_front(2);
    list.push_front(3);

    // 정상적으로 예의 바르게 배출해내는지 정방향 반환 검문
    assert_eq!(list.pop_front(), Some(3));
    assert_eq!(list.pop_front(), Some(2));

    // 혹시라도 이 요망한 녀석이 몰래 뒤로 슬쩍 고장 나(corrupted) 있진 않은지 압박 검증차 기습 추가 투입 강행 발동
    list.push_front(4);
    list.push_front(5);

    // 다시금 정상적으로 구역에 맞게 똑바로 예의 바르게 배출해내는지 2차 정방향 반환 검문
    assert_eq!(list.pop_front(), Some(5));
    assert_eq!(list.pop_front(), Some(4));

    // 마지막으로 단 한 방울의 국물도 남김없이 완전 씨가 말라 탈수 소진 탈곡 바닥 고갈(exhaustion) 상태의 나락 생체 한계 파괴 테스트
    assert_eq!(list.pop_front(), Some(1));
    assert_eq!(list.pop_front(), None);

    // ---- 이 밑으론 전부 대칭 꼬리(back) 버젼 -----

    // 뒤에서 쑤셔도 빈 깡통 행동거지 군기 똑바로 처먹었는지 검수 똑같이 점검
    assert_eq!(list.pop_back(), None);

    // 이번엔 꼬리 쪽을 쑤셔 알맹이 좀 구겨 넣어 바구니 살찌우기
    list.push_back(1);
    list.push_back(2);
    list.push_back(3);

    // 정상 배출 역방향 검문
    assert_eq!(list.pop_back(), Some(3));
    assert_eq!(list.pop_back(), Some(2));

    // 또 기습 모의 투입
    list.push_back(4);
    list.push_back(5);

    // 또 정상 배출 역방향 2차 점검 검문
    assert_eq!(list.pop_back(), Some(5));
    assert_eq!(list.pop_back(), Some(4));

    // 최후의 고갈 바닥 판별 생체 파괴 테스트 2차 종말 단죄
    assert_eq!(list.pop_back(), Some(1));
    assert_eq!(list.pop_back(), None);
}

#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    assert!(list.peek_back().is_none());
    assert!(list.peek_front_mut().is_none());
    assert!(list.peek_back_mut().is_none());

    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
    assert_eq!(&mut *list.peek_front_mut().unwrap(), &mut 3);
    assert_eq!(&*list.peek_back().unwrap(), &1);
    assert_eq!(&mut *list.peek_back_mut().unwrap(), &mut 1);
}
```

솔직히 구멍 숭숭 뚫려 아직 우리가 눈 뜬 장님처럼 검증 못 하고 미처 테스트 통과 안 시켜 넘긴 예외 케이스(cases we're not testing) 사각지대가 존재하냐고요? 아마 그럴 확률 100%(Probably) 일 겁니다. 이 양방향 대칭 조합의 복합 경우의 수 영역 넓이는 진심 여기서 문자 그대로 우주 차원급으로 폭발 급증 팽창(blown up) 해 있기 때문입니다. 그래도 뭐 우리 코드가 최소한 대놓고 바닥에 콧물 흘릴 만큼 *명백히 개노답 바보(obviously wrong)* 썩진 않았을 거라 위안해 봅니다.

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

개조아 (Nice). 역시 그저 대충 머리통 비우고 복붙(Copy-pasting) 찍어내기가 세상에서 통달한 제일 최고로 달달한 황금 프로그래밍 방식입니다.
