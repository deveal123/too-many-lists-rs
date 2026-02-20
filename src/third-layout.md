# 레이아웃 (Layout)

좋습니다, 다시 레이아웃 설계도로 돌아가 봅시다.

영구 리스트(persistent list)에서 가장 중요한 점은 리스트의 꼬리 부분을 사실상 공짜로(for free) 우려먹고 조작할 수 있다는 점입니다:

예를 들어, 영구 리스트 환경에선 다음과 같은 작업 흐름이 흔하게 발생합니다:

```text
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

하지만 최종적으로 메모리 상태는 이렇게 형성되어야만 합니다:

```text
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

이 구조는 Box로는 절대 불가능합니다, 왜냐하면 `B`에 대한 소유권이 *공유(shared)*되고 있기 때문입니다. 만약 그렇다면 도대체 누가 저걸 해제(free)해야 할까요? 만일 제가 list2를 소멸(drop)시킨다면, B도 함께 해제되는 걸까요? 우리가 알던 Box의 상식이라면 당연히 그렇다고 단언했을 겁니다!

함수형 언어들 &mdash; 그리고 사실상 현존하는 대다수의 언어 &mdash; 들은 *가비지 컬렉터(garbage collection)*의 힘을 빌려 이 난관을 회피하고 살아남습니다. 기가 막힌 가비지 컬렉터의 마법 덕분에, B 녀석은 그 누구도 더 이상 자신을 바라보지 않는 고립무원이 되었을 때 비로소 얌전히 메모리에서 해제됩니다. 야호! (Hooray!)

하지만 Rust에는 이런 언어들이 기본으로 달고 나오는 그런 만능 가비지 컬렉터 따윈 코빼기도 없습니다. 이 언어들은 자원들이 실행 시간(runtime) 동안 동동 떠다니는 걸 일일이 추적 스캔하여 어질러진 기억 파편들을 긁어모아 쓰레기 소각 처리해주는 *추적식(tracing)* GC를 갖고 있습니다. 이와 반대로 오늘날 Rust가 쥐어줄 수 있는 GC라고는 *참조 횟수 카운터(reference counting)*뿐입니다. 참조 카운팅은 일종의 굉장히 멍청하고 원시적인 GC로 비유할 수 있습니다. 수많은 다중 흐름 작업 환경에서 추적식 수집기에 비해 심각하게 저조한 처리량(throughput) 효율을 보이며, 당신이 멍청하게 순환 참조 구조 사이클(cycles)이라도 구축해 버릴 경우 그 즉시 완전히 고장 나서 먹통이 되어 뻗어버립니다. 뭐 어쨌거나, 당장 우리 손에 쥐어진 건 이게 다입니다! 천만다행히도, 우리가 지금 파고들 이 작업에서는 순환 구조 궤도에 얻어걸릴 염려 따윈 죽었다 깨어나도 없습니다 (못 믿겠다면 직접 스스로 증명해 보셔도 좋습니다 &mdash; 전 절대 안 할 거지만요).

그럼 대체 참조 횟수 카운터 기반 GC를 어떻게 구현한담? 바로 `Rc`입니다! Rc는 기본적으로 Box와 뼈대가 같지만 마음껏 복사 및 복제(duplicate)할 수 있으며, 이 녀석이 머금은 기억 메모리 공간은 여기저기 파편화되어 나돌아다니던 모든 파생 Rc 꼬맹이들이 완전히 전멸해서 버려질(dropped) 때 비로소 해제될 것입니다. 불행하게도, 이토록 편안한 유연성 뒤엔 아주 뼈아픈 대가가(serious cost) 따릅니다: 우리는 이 녀석의 속살을 훔쳐볼 때 오직 공유 참조(shared reference)로 밖에 열지 못합니다. 이 말은 즉, 녀석 안에서 실물 데이터를 갈취해 빼돌릴 수도 없고 절대로 가변 조작(mutate)시킬 수도 없단 뜻입니다.

자 그럼 구조 모양새는 대강 어찌 빠지게 될까요? 음, 이전까진 이거였죠:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

그냥 단순하게 무지성으로 Box를 Rc로 죄다 바꾸면 되지 않겠습니까?

```rust ,ignore
// in third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```text
cargo build

error[E0412]: cannot find type `Rc` in this scope
  --> src/third.rs:5:23
   |
 5 | type Link<T> = Option<Rc<Node<T>>>;
   |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
   |
 1 | use std::rc::Rc;
   |
```

이런 젠장, 뼈 맞았네요 (sick burn). 우리가 가변 리스트를 짤 땐 여태 모든 걸 암묵적인 도구로 꿀 빨며 썼는데 반해, Rc 이놈은 얼마나 멍청한 찌끄래기인지 세상 모든 Rust 프로그래머의 기본 탑재 도구 목록에서조차 누락될 정도군요. 
*진짜 쭈구리 찐따 같네요 (What a loser).*

```rust ,ignore
use std::rc::Rc;
```

```text
cargo build

warning: field is never used: `head`
  --> src/third.rs:4:5
   |
 4 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

작동은 잘 되는 삘이군요 (Seems legit). Rust는 한결같이 놀랍도록 글 지어내기가 터무니없이 *가볍네요*. 분명 이제 대충 검색으로 찾아서 Box란 단어를 싹 다 Rc로 바꾸기만 하면 작업 끝 시마이 쳐버릴 겁니다!

...

안 돼. 안됩니다.
