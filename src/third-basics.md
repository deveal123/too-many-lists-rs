# 기초 다지기 (Basics)

우리는 이미 지금까지 머리를 쥐어뜯으며 Rust의 수많은 밑바탕 기초(basics) 지식들을 뚫고 왔습니다, 그러니 아주 단순 유치한 겹치는 패턴 정도는 그냥 슉슉 해치워버릴 수 있습니다.

일단 생성자(constructor) 부분은 그냥 저번 걸 고대로 복사-붙여넣기 쓰겠습니다:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

`push`나 `pop` 같은 동작들은 성향상 이제 더 이상 어울리지 않습니다. 그 대신에 얼추 비슷하게 생겨 먹은 `prepend`(앞에 욱여넣기)와 `tail`(꼬리 잘라먹기) 메서드들을 세트로 쥐여주면 그만입니다.

자 `prepend` 부터 덤벼봅시다. 이건 리스트 구조 전체와 새로운 요소 알맹이를 하나 같이 밀어 넣으면, 그 결과로 완전히 딴판인 새로운 List를 뱉어냅니다. 지난 가변 리스트 시절 때와 흡사하게, 우리는 먼저 새 노드를 하나 예쁘게 빚어서 옛날 리스트 본체를 그 녀석의 꼬리인 `next` 구멍에다 이식하는 게 목표입니다. 유일하게 신기한 난관이 있다면, 우린 이제 그 어떤 데이터도 허락 없이 가변 훼손(mutate)시킬 권한이 박탈된 터라 도대체 어떻게 저 빌어먹을 옛 next 값을 뽑아다 조달하느냐(*get*)입니다.

다행히 우리 기도를 들어줄 구원자, Clone 트레이트님이 등판하실 차례입니다. Clone은 이 세상 거의 대다수 타입에 죄다 기본 탑재되어 구현되어 있으며, 오직 공유 참조자(shared reference) 하나만 쥐여줘도 안전하게 통제되어 완전 분리 독립적인(disjoint) "이거랑 판박이로 똑같이 생긴 녀석 하나 더요"라는 기적의 1+1 붕어빵 연성 능력을 제네릭하게 구사합니다. C++의 복사 생성자(copy constructor)랑 얼추 닮았지만, 막무가내로 암묵적 자동 돌발 호출이 걸리는 불상사는 결단코 절대 일어나지 않습니다.

특히 Rc 녀석에게 부착된 이 Clone은 본질적으로 그 내부의 참조 횟수 카운터(reference count) 눈금 하나만 스윽 올려주도록 얍삽하게 응용 설계된 물건입니다. 그래서 우린 Box처럼 노드를 적출하여 잘라다 통째로 다른 리스트 소속으로 이적시켜 버리는 무식한 이동 짓거리 대신, 그저 단순히 원래 리스트 맨 앞 머리에다가 눈치껏 클론 복제 광선을 쏴주기만 하면 만사형통입니다. 심지어 굳이 뚜껑을 열어 Option 안에 뭐가 든 지 패턴 검사 매칭해볼 귀찮음조차 없습니다, 똑똑한 Option 자체에 이미 우리가 바라는 바로 그 행위를 고스란히 복제 유도하는 Clone 구현이 달려있으니까요.

좋아요, 한번 시도해 봅시다:

```rust ,ignore
pub fn prepend(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

와우, Rust는 진짜 필드 구멍 하나하나 직접 뜯어 쓰는지 안 쓰는지 스토커 수준으로 매섭게 검사하네요. 심지어 코드 밖의 그 어떤 외부 소비층도 이 필드를 알 길조차 없어 접근이 불가능한 수준인데 말입니다! 그래도 뭐, 삐걱거리지 않는 걸 보아하니 아직까진 순항 중입니다.

`tail`은 방금 한 행동의 논리론적 완벽한 역순 대응(inverse) 작업입니다. 뭉텅이 리스트 녀석을 받아서 첫 번째 모가지 알맹이만 똑 잘라버리고 남은 꼬리 덩어리 전리품을 툭 반환합니다. 이 말이 뜻하는 바는 결국, 이 리스트에 달린 *두 번째* 요소 녀석(혹시나 그 자리에 뭔가가 생존해 있다면)의 머리를 찾아내 그걸 또 Clone 뜨겠다는 소리입니다. 한번 까보죠:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

흠 (Hrm), 우리가 뭔가 거하게 말아먹었네요. `map` 은 우리에게 딱 떨어지는 Y형 반환을 기대했지만, 우리가 넘긴 건 어이없게도 `Option<Y>` 덩어리였습니다. 정말 천만다행히도, 이런 바보짓은 이미 오래전부터 옵션 교도들 사이에 뻔히 예견된 일상적 단골 패턴이었기에 우린 그저 `and_then` 이라는 마법의 필터 망을 장착시켜 Option 자체로 돌려보내는 걸 은근히 무마시켜 통과할 수 있습니다.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

완벽해. (Great)

자 일단 `tail` 구동 장치를 터득했으니, 곧이어 맨 앞머리 모가지만 빼콤 내밀어 슬쩍 조회하는 `head` 장치도 겸사겸사 투입해놔야 밸런스가 맞겠습니다. 사실 이건 옛날 가변 시절의 그 지긋지긋한 `peek` 녀석이랑 완전 똑같은 녀석입니다:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```text
> cargo build

```

깔끔합니다 (Nice).

이렇게 대강 근본 동작들은 완비되었으니 곧바로 기능 검사장에 던져 넣어 테스트를 돌려봅시다:


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // 빈 꼬리가 제대로 돌아가는지 다시 찍어먹어 보기
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

개조아! (Perfect!)

참고로 Iter 또한 아까 만지던 그 가변 리스트 때 써먹던 녀석과 100% 모발까지 동일(identical)한 설계형입니다:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().prepend(1).prepend(2).prepend(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

누가 동적 타이핑이 더 쉽다고 헛소리 깐 걸까요?

(허접 찌질이들 ㅋ)

명심해 두시지만 우루루 우린 지금 환경에서 IntoIter 혹은 IterMut 기능 따위는 엎어 매쳐도 절대 구현 불가입니다(can't implement). 왜냐하면 우린 원소(element)들의 그 빌어먹을 더러운 공유 접근(shared access) 하나만 쥐고 있을 뿐이니까요.
