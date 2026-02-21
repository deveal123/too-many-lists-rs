# IterMut

솔직하게 고백하건대, 사실 IterMut는 완전 돌아이(WILD)입니다. 물론 겉보기에는 Iter 구조와 판박이로 100% 동일(identical)해 보이지만요! 

네, 표면적인(Semantically) 구조는 똑같습니다. 그러나 공유 참조자(shared)와 가변 참조자(mutable)가 지닌 성질(nature) 차이 때문에, 결과적으로 Iter는 사소한(trivial) 아이들 장난인 반면, IterMut를 구현하는 일은 흡사 합법적 마법(Legit Wizard Magic)과 같습니다.

문제의 핵심 통찰력(key insight)은 다음과 같은 Iterator 구현부를 볼 때 드러납니다:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

이 녀석을 탈당(desugared) 상태로 풀어서 뜯어보면 이렇게 섬뜩한 모습이 됩니다:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

`next`의 선언 함수 서명(signature) 구조를 잘 보면, 입력(input)과 출력(output) 사이에 규칙이나 제약 조건(*no* constraint)이 단 하나도 없습니다! 이게 왜 중요(Why do we care)할까요? 즉 우리는 아무런 제약 없이 이 `next`를 원하는 만큼 계속 반복 호출(call over and over)할 수 있다는 뜻입니다!


```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

쩐다! (Cool!)

이 방식은 수천 수만 명끼리 무한정 나눠 가질 목적인(can have tons of them at once) 공유 참조자(shared references)에게는 *의심의 여지 없이 안전하고 합법(definitely fine)*입니다. 하지만 가변 참조자(mutable references)는 안타깝게도 공존할 수 없습니다(can't coexist). 애초에 그것들은 배타적 점유(exclusive)를 목적으로 할 때만 안전하기 때문이죠.

결과적으로 온전한 안전한 코드(safe code)만 사용해서 IterMut를 만들어 내는 일은 유달리 미치도록 험난(harder)합니다. 반전으로 놀라운 사실은(Surprisingly), 대부분의 구조에서 IterMut를 전혀 위험하지 않게 완벽히 안전한 곳(completely safely)에서 구현할 수 있다는 점입니다!

우선 Iter 코드를 가져와서 전부 `mut`라는 꼬리표를 달아 개조해 봅시다:

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

보아하니 두 개의 큰 오류 덩어리가 떨어졌네요. 허나 첫 번째 타격은 투명하고 아주 직관적(really clear)이며 해결 방법도 대놓고 알려 줍니다! 너덜거리는 공유 참조 따위를 함부로 극악한 가변 참조자로 업그레이드(upgrade) 시킬 순 없으니, `iter_mut` 함수 구문도 `&mut self`를 인자로 받아 넘겨야 합니다. 복사 붙여넣기로 인한 단순한 실수(silly copy-paste error)였네요. 

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_deref_mut() }
}
```

나머지 에러는요?

이런! 알고 보니 제가 저번 섹션에서 `iter` 쪽 코드를 야심 차게 짜놓으면서 굉장히 얼간이 같은 폭탄 실수를(accidentally made an error) 했는데, 운때가 기막히게 맞아떨어져 간발의 차이로 돌아갔던 구조였군요!

우린 이번 오류를 통해 위대한 Rust의 Copy 트레이트와 첫 밀접 조우를(first run in) 이뤘습니다. 이전에 소유권([ownership][ownership]) 파트에서, 어떤 것을 옭아매고 바깥으로 이동(move)시키면 그 원래 자리에 남은 형편없는 것는 절대 사용할 수 없다(can't use it anymore)고 배웠습니다. 대부분의 일반적인 타입에게 이는 소름 돋게 합리적입니다(perfect sense). Box 같은 친구를 보면 무의식에 갇힌 힙(heap)의 거대한 메모리 공간을 점유하며 포장되어 있는데, 절대로 이 할당 메모조각을 복제 노나가지고서 누군가 제멋대로 해방시켜라(free its memory)고 파괴 명령을 내리지 못하게 해야 하기 때문입니다.

하지만 또 다른 누군가에겐 이 논리가 완전 *쌉소리(garbage)*에 불과합니다. 아주 단순무식한 원시 정수형(Integers) 데이터에게는 그딴 소유권 운운하는 게 다 쓸모가 없습니다. 걔들은 그냥 의미 없는 깡 숫자라니까요! 그래서 얘들이 Copy 타입으로 마크된 것입니다. Copy 기능을 하사받은 친구들은 말 그대로 단순 비트 복제 방식을 거쳐도 정말 완벽하게 평안히 자가 적응할 수 있음이 학계에 보고되어 있습니다. 더 나아가, 거칠게 무심하게 강제 유출탈주시켜(move) 뜯어냈음에도 불구하고, 그 빈 껍데기 잔재 시신물(old value) *마저도* 정상적으로 마법처럼 여전히 쓸 수가 있습니다! 심지어는 아무 자리 대체물을 끼워넣어 보충해주지 않아도(without replacement!) 한 치의 오차 없이 다이렉트로 그 불쌍한 참조자를 소멸탈출 이동시켜버릴 수까지 있습니다.

Rust에서 세상 모든 숫자들(i32, u64, bool, f32, char 등...)은 뼛속까지 Copy 종자입니다. 심지어 여러분이 직접 만든 사용자 정의 타입조차 몽땅 알맹이가 Copy로 도배되었다면 자진해서 Copy 인증 도장을 받아 선포할 수 있습니다(declare any user-defined type to be Copy as well).

아까 그 얼렁뚱땅 Iter 코드가 기적처럼 통과했던 이유는 공유 참조자(shared references) 녀석들 마저도 Copy 파벌에 속해 있었기 때문입니다. 그래서 당연스레 그를 머금은 `Option<&>` 껍질 역시 *덩달아(also)* Copy 마법 도장이 찍혀있다는 소리입니다. 그렇기 때문에 예전에 `self.next.map`을 냅다 복제이탈 시전해갈겼을 때 아무 파괴적 파장없이 완벽하게 쿨하게(was fine) 끝맺어질 수 있었습니다. 그저 조용히 분신이 대신 갈기갈기 복사되어(just copied) 들어갔기 때문입니다. 하지만 이제 `&mut`는 그 Copy 무리가 아니기 때문에 저런 짓을 할 수 없습니다 (`&mut`을 복사시켜버렸다면 같은 메모리 장소를 가리키는 두 개의 가변 참조자가 동시에 생길 우려가 있어 절대 금지됩니다). 이런 짓거리 대신 이제 얌전히 `take` 메서드로 돌파하여 추출해 주어야만 합니다.


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.elem
    })
}
```

```text
> cargo build

```

어... 와우. 쩌네요(Holy shit!). IterMut 그냥 알아서 잘 돌아갑니다.

테스트 해보죠:


```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

예압. 잘 되네요.

미쳤다 진짜 세상에.

대체 뭐야.

사실 저게 제대로 돌아가게 맞음에도 불구하고 항상 무언가 쓸데없이 바보같은(stupid) 짓거리들이 기어들어와 가로막고 판을 엎어대지 않습니까?! 이참에 투명하게 짚고 넘어갑시다:

우리는 방금까지 단방향 연결 리스트를 인자로 쥐곤 그 내부의 속살 알맹이 파편 하나하나를 가변 참조자 형태로 반환하는 극강의 무적 코드를 구현해냈습니다. 이 모든 것이 정적으로 컴파일러에 감시받으며 완벽히 안전합니다. 이 녀석이 통과할 수 있게 해 준 두 가지 비밀은 이것입니다:

* 우린 `take`를 통해 `Option<&mut>` 소유권 독점 권리를 이양해 가져갔으므로 안전합니다.
* Rust는 저 구조체 구조가 서로 단절되어 겹치지 않고 완전 독단 점유하고 있단 사실을 제대로 파악하고(disjoint) 그 쪼가리를 향해 가변 참조자를 파편화 시전하는 걸 충분히 허용해 줍니다. 

게다가 똑똑하기도 이 녀석의 파생 변형을 거듭 우려먹어 단순한 트리의 가변 이터레이터조차 동일하게 제작 가능하며, 심지어 쌍방향 순회가 가능한 양방향 제련 이터레이터까지 변이를 뽑아 양끝 단에서 우수수 조각물을 퍼먹을 수 있습니다! 워후!

[ownership]: first-ownership.md
