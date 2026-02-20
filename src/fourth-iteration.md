# 반복 (Iteration)

이 골칫덩이(bad-boy)에 대한 반복(iterating) 기능을 한 번 뚫어봅시다.

## IntoIter

언제나 그랬듯, `IntoIter`는 가장 쉬울 겁니다. 그저 스택을 감싸고 `pop`만 호출하면 되니까요:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop_front()
    }
}
```

하지만 아주 흥미로운 새로운 발전이 하나 생겼습니다. 이전에는 리스트에 단방향의 "자연스러운(natural)" 순서밖에 없었지만, Deque는 본질적으로 양방향(bi-directional)입니다. 앞에서 뒤로 가는 방식이 굳이 특별할 게 있나요? 누군가가 반대 방향으로 순회하고 싶다면 어떻게 해야 하죠?

Rust는 사실 이에 대한 해답을 이미 가지고 있습니다: 바로 `DoubleEndedIterator`입니다. `DoubleEndedIterator`는 `Iterator`를 상속받으며(즉, 모든 DoubleEndedIterator는 Iterator이기도 합니다) 한 가지 새로운 메서드인 `next_back`을 추가로 요구합니다. 이 메서드는 `next`와 완전히 동일한 시그니처를 지니지만, 리스트의 반대쪽 끝에서부터 요소들을 반환하도록 설계되어 있습니다. `DoubleEndedIterator`의 의미론(semantics)은 우리에게 아주 편리합니다: 반복자 자체가 사실상 또 하나의 deque가 되는 셈이죠. 여러분은 양쪽 끝이 만나 수렴(converge)하여 반복자가 비어버릴 때까지 양옆에서 요소들을 차례대로 소모할 수 있습니다.

일반적인 `Iterator`의 `next`처럼, `DoubleEndedIterator`를 사용하는 소비자들도 실제로 `next_back`을 직접 호출하는 데에는 크게 관심(care about)이 없습니다. 이 인터페이스가 제공하는 최고의 장점은 따로 있는데, 바로 `rev` 메서드를 사용할 수 있게 해준다는 것입니다. `rev`는 원래의 반복자를 감싸 요소들을 역순으로 반환하는 새로운 반복자를 생성합니다. 이 녀석의 동작 논리는 아주 명쾌합니다: 뒤집힌 반복자에게 `next`를 호출하는 것은 결국 내부적으로 `next_back`을 호출하는 것일 뿐입니다.

어쨌든 우리는 이미 완벽한 deque를 가지고 있으므로, 이 API를 제공하는 건 아주 쉽습니다:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

테스트를 돌려보죠:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

좋네요. (Nice.)

## Iter

Iter 녀석은 조금 덜 호락호락할 겁니다. 우리는 다시 그 무시무시한 `Ref` 녀석들을 또 상대해야 하거든요! 저 골치 아픈 Refs 포장지 탓에, 우리는 예전 우리가 하던 방식처럼 `&Node` 들을 그대로 편하게 보관(store)할 수가 없습니다. 대신, 이번에는 `Ref<Node>` 자체를 보관해 봅시다:

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

지금까진 아주 순조롭네요(So far so good). `next`를 구현하는 건 아마 털이 곤두서게 복잡해지겠지만(hairy), 기본 뼈대 논리는 이전 스택의 IterMut와 비슷할 거라 생각합니다. 거기에 끔찍한 RefCell의 광기(madness)가 살짝 곁들여진 수준이겠죠:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

`node_ref`의 생존 수명이 턱없이 부족합니다(doesn't live long enough). 일반적인 참조자들과 달리, Rust는 우리가 저런 식으로 `Refs`를 마음대로 쪼개는 것을 허락하지 않습니다. `head.borrow()`에서 우리가 얻어낸 `Ref`는 오직 `node_ref`만큼만 살아있을 수 있는데, 정작 우리는 `Ref::map`을 호출하면서 그 `node_ref`를 통째로 소모해 파기해버리니까요(trashing that).

우리가 원하는 그 기능의 함수는 다행히 존재(exists)하며, 그 이름은 *[map_split][]* 입니다:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

흐익. 한번 시도나 해보죠... (Woof. Let's give it a try...)

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

으윽. (Ergh.) 수명(lifetimes)을 올바르게 맞추려면 또다시 `Ref::Map`을 호출해야 하는 늪에 빠졌네요. 그런데 `Ref::Map`은 `Ref`를 반환하고 우리에게 필요한 건 `Option<Ref>`입니다. 하지만 그러려면 `Ref`를 거쳐서 우리 Option을 맵핑해야 하는데...

**(초점 없는 눈빛으로 한참 동안 허공을 응시함)**

??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

아. 맞네요. (Oh. Right.) RefCell이 한 개가 아닙니다. 우리가 리스트 안으로 더 깊이 순회해 들어갈수록, 우리는 각 RefCell 아래에 층층이 더 깊게 중첩(nested)되게 됩니다. 이걸 해결하려면, 지금 우리가 붙잡고 있는 아직 반납되지 않은 모든 대출(outstanding loans)들을 관리하기 위해 `Refs`들의 스택(stack of Refs) 같은 걸 따로 수동으로 관리하고 유지해야만 할 겁니다. 왜냐하면 우리가 어떤 요소를 다 보고 손을 뗄 때마다, 그 요소 이전에 있던 모든 RefCell들의 대여 카운트(borrow-count)를 전부 일일이 일일히 마이너스 시켜줘야 하니까요.................

제 생각에 여기서 우리가 할 수 있는 건 더 이상 아무것도 없습니다(nothing we can do here). 이건 완벽한 막다른 길(dead end)입니다. 자, 이제 이 지긋지긋한 RefCells 지옥에서 빠져나가 봅시다(try getting out of).

우리의 `Rc`들은 어떨까요? 애초에 우리가 참조자들을 저장해야 한다고(needed to store references) 누가 그러던가요?
그냥 그 `Rc` 자체를 통째로 Clone 해서 리스트 한가운데를 가리키는 소유권 있는 멋진 핸들(owning handle)을 얻으면 안 되는 이유라도 있습니까?

```rust ,ignore
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```

어... 잠깐 우린 이제 뭘 반환하죠? `&T`? `Ref<T>`?

아뇨, 그딴 건 다 안 먹힙니다... 우리 `Iter`에는 이제 수명(lifetime) 개념 자체가 아예 없어졌으니까요! 저 `&T`든 `Ref<T>`든 간에 우리가 `next` 함수 안에 들어가기 전에 어떻게든 수명을 미리 선언(declare some lifetime)하도록 요구합니다. 그런데 우리가 `Rc` 껍질에서 뭘 어떻게든 끄집어낸다고 한들, 그 결과물은 우리 구조체 Iterator 자체를 빌리게(borrowing the Iterator) 되는 꼴이 되어버립니다... 뇌가... 두뇌가 아파온다... 으아아아아아아아

어쩌면 우리가... 저 Rc를 맵핑해서... `Rc<T>`를 얻어내는 게 가능할지도 몰라요? 그런 기능이 있나요? Rc 공식 문서를 뒤져봐도 그런 기능은 없는 것 같습니다. (사실 어떤 용자가 그런 정신 나간 짓을 가능하게 해주는 [크레이트][own-ref]를 따로 만들긴 했습니다.)

하지만 잠깐, 우리가 진짜로 **그걸** 해낸다고 해도 이젠 더 크고 거대한 문제(even bigger problem)가 우리를 덮칩니다: 바로 악명 높은 **반복자 무효화(iterator invalidation)** 사태 말입니다. 여태껏 우리는 반복자 무효화라는 재앙으로부터 완벽하게 면역(totally immune)이었습니다, 왜냐하면 우리 `Iter` 자체가 리스트 전체를 대여(borrowed)하여 완전히 불변(immutable) 상태로 잠가버렸기 때문입니다. 하지만 만약 우리 `Iter`가 Rc들을 뱉어내는 식이라면, 리스트를 전혀 대여하지 않을 겁니다(wouldn't borrow the list)! 이 말인즉슨 누군가가 리스트 내부를 가리키는 내부 포인터를 쥔 상태에서 리스트에다 대고 `push`와 `pop`을 호출하기 시작할 수 있다는 뜻입니다!

오 맙소사, 그럼 도대체 무슨 끔찍한 일이 벌어질까요?! (Oh lord, what will that do?!)

뭐, 놀랍게도 집어넣는 것(`push`ing) 자체는 사실 괜찮습니다(actually fine). 우리는 그저 리스트 내부의 어느 특정 하위 구간(sub-range)만을 들여다보고 있을 뿐이고, 리스트는 우리 시야 밖에서 넘어서 알아서 덩치를 키워(grow beyond our sights) 갈 테니까요. 뭐 별거 아닙니다(No biggie).

하지만 저 `pop` 녀석은 이야기가 다릅니다(another story). 만약 그들이 우리가 들여다보는 범위 바깥쪽에서 요소를 팝(pop)해서 빼낸다면야, 뭐 그건 우리 눈에 보이지 않는 바깥 동네 노드들이니 아무 일도 안 일어날(nothing will happen) 겁니다. 하지만 만약 그들이 정확히 당장 우리가 가리키며 들여다보고 있는 바로 그 노드(pointing at)를 팝해서 뽑아내려 한다면... 모든 게 펑펑 다 터져버릴 겁니다(everything will blow up)! 특히 그쪽에서 팝업된 결과를 가지고 `try_unwrap`을 부숴 내용물을 벗겨내려(unwrap) 시도할 때 에러가 나면서 실제로 실패(actually fail)할 것이고, 결국 프로그램 전체가 깔끔하게 패닉(panic)으로 뻗어버리게 될 겁니다.

사실 이거 꽤나 멋진 일이긴 합니다(pretty cool). 우리는 수많은 내부 소유 포인터(interior owning pointers)들을 리스트 안으로 마음껏 집어넣고는, 뻔뻔하게 동시에 리스트를 막 변조 수정(mutate it at the same time)할 수 있으며, 이 모든 짓거리가 아주 기가 막히게도 그들이 우리가 가리키는 바로 그 딱 지정된 노드를 파괴하려고 시도하기 전까진 *진짜 멀쩡하게 잘 돌아갑니다(just work)*. 게다가 그렇게 선을 넘더라도 이 시스템은 절대 잡다한 댕글링 포인터(dangling pointers) 같은 더러운 찌꺼기 버그 오류 참사 따위를 세상에 남기지 않습니다, 그냥 곧바로 아주 투명하고 확실하게 프로그램이 확정적으로 패닉(deterministically panic) 뻗어서 깔끔하게 방어해 주니까요!

하지만 이 골치 아픈 Rc 맵핑(mapping Rcs) 고생길 조작질 위에 얹어서 추가로 저 더러운 '반복자 무효화 격변 방어 관리 대처'(deal with iterator invalidation) 과제 더미까지 감당하며 꾸역꾸역 씨름해야 한다는 것은... 정말 그저 엄청나게 끔찍합니다(bad). 결국 진정코 마침내 `Rc<RefCell>`은 우리를 완전히 철저하게 배신하여 실패(failed us)로 이끌어줬습니다. 아주 흥미로운(Interestingly) 대조 역전 모순점 하나를 꼽자면, 우린 저번 불변 영구 스택(persistent stack) 때 겪었던 것의 정반대 극단 경험(inversion)을 했다는 사실입니다. 그때 그 불변 스택에선 데이터의 영구 소유권 원장을 탈환 확보해 얻어보려고(reclaim ownership) 영겁을 피눈물 흘리며 투쟁(struggled) 했어도, 단기 대출 참조자(references) 지분 정도야 365일 평생(all day every day) 무제한 펑펑 뿌려대고 퍼줄 수 있었습니다; 허나 정반대로 지금 우리 이 리스트(our list) 상황에서는 소유권 독취 흡수 확보(gaining ownership) 정도는 아주 코 파듯 식은 죽 먹기로 쉽게 해낼 수 있으면서, 정작 참조자 거울 토큰 하나 단기 대출(loan our references) 내주고 쥐여주는 일 하나만으로 이런 지옥도를 맛보며 진흙탕 똥꼬쇼 고투(struggled to)를 치러야 하니까요. 

뭐, 솔직 까놓고 객관적으로 털어놓자면(to be fair), 우리가 지금껏 헤맸던 이 개고생 난장 분투 폭발 삽질(our struggles)의 근본 99.9% 지분 원인은, 전부 이 흉측 복잡한 구현 세부 비밀 속살(implementation details) 전부를 철통같이 감추고 숨겨 깔끔 번지르르 허울 좋은 훌륭한 API 도구면상 간판 포장 인터페이스(have a decent API) 껍데기를 어떻게든 가져보려고 아등바등 아집을 발악(revolved around) 부렸기 때문입니다. 처음부터 속 편하게 막무가내로 노드(Nodes) 부품 나부랭이 덩어리들을 동네 천지 사방팔방에 마구 뿌리고 넘기고 던지며(just pass around) 쓰고자 마음먹었다면 이런저런 제약 따윈 개무시하고 전부 훌륭하게 완벽 구동 작동 성사 클리어(do everything fine) 시킬 수 있었을 텐데 말이죠.

씨발, 심지어 우린 미친 척하고 동시에 접속하는 다수 가변 다중 참조 반복기 군단 IterMuts 들을 떼거지로 발급 찍어내버리고(could make), 이놈들이 한 지점에 충돌해서 동시에 같은 요소를 수정 가변 접근하지(not be mutable accessing the same element) 못하도록 지키는 동적 런타임 제어 보호망 방벽 검열 조종 체계(were runtime checked)까지 죄다 달아 배포할 수도 있었어요!

진심 팩트로 평가하자면(Really), 애초에 이딴 기괴한 디자인 설계 체계(this design) 구조는 바깥의 깡패 외부 클라이언트(consumers of the API) 소비자들에게 공용으로 노출 개방 조달 판매(makes it out to) 배포되는 그런 표면의 도구 용도가 아니라, 그저 온전히 시스템 내부에서 혼자 조립 뚝딱 은닉되어 굴러가는 흑막 밀실 내부 데이터 구조(internal data structure) 전용 부속 부품 용도에나 기가 막히게 맞아떨어집니다(more appropriate). 내부 가변성 체계(Interior mutability) 기믹은 완벽하고 튼튼한 안전 철벽 요새 응용 애플리케이션 서비스(safe applications) 거대 성곽 하나를 끝장나게 지어 조합 성형(writing)해 내는 데엔 정말 빛과 소금 같은 최강의 도구(is great for)이지만, 수많은 불특정 객원 제3자들이 이리저리 끼워 맞춰 굴려 쓸 안전 방어 범용 도서관 조립 부품 라이브러리(safe libraries) 용도로 공개 살포 배포 전시하기엔 진짜 너무나 개오바 낙제 쓰레기(Not so much) 안티 패턴 수준 결함투성이 구조인 게 뼈저린 현실입니다.

어쨌든간에 저의 이 Iter와 IterMut 새끼들에 대한 헛발질 여정 뺑뺑이는 여기까지 할 거며 여기서 깨끗이 미련 버리고 항복 단념(me giving up on) 포기를 선언합니다. 까놓고 어떻게든 꾸역꾸역 억지수작 봉합해서 해낼 순 있었겠지만(could do them), *어휴 진짜 토나와 극혐이야 역겨운 쓰레기들 (ugh)*.

[own-ref]: https://crates.io/crates/owning_ref
[map-split]: https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split
