# 레이아웃과 기초 2: 진짜 날것(Raw)으로 돌아가기 (Layout and Basics 2: Getting Raw)

> 이전 세 섹션들의 요약(TL;DR): `&`, `&mut`, `Box` 같은 안전한 포인터(safe pointers)들과 `*mut`, `*const` 같은 안전하지 않은 원시 포인터(unsafe pointers)들을 무작위로 뒤섞어 쓰면 미정의 동작(Undefined Behaviour)이라는 재앙을 초래합니다. 왜냐하면 안전한 포인터들이 원시 포인터들로는 감당할 수 없는 추가적인 깐깐한 제약 조건들을 도입하기 때문입니다.

오 맙소사, 저는 또다시 연결 리스트를 짜야 합니다. 알겠습니다. 알겠어요. 괜찮습니다.

이번 섹션은 정말 빠르게 치고 나갈 겁니다. 우리는 이미 첫 시도에서 전체적인 구조 설계를 논의했고, 안전한 포인터와 원시 포인터를 섞어 쓴 것만 빼면 우리가 했던 모든 일들은 본질적으로 올바른(basically correct) 것이었으니까요.


# 레이아웃 (Layout)

자, 이제 새로운 레이아웃에서는 오직 원시 포인터 체계만(only use raw pointers) 고집할 것이며, 모든 것이 완벽해져 다신 어리석은 실수를 저지르지 않게 될 겁니다.

여기 망가졌던 우리의 과거 설계도가 있습니다:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // 순진하고 친절한 녀석 (INNOCENT AND KIND)
}

type Link<T> = Option<Box<Node<T>>>; // 진짜 악의 축 (THE REAL EVIL)

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

그리고 이게 새로 바뀐 우리의 레이아웃입니다:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // 훨씬 낫네요 (MUCH BETTER)

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

명심하세요: 원시 포인터를 사용할 때 `Option`은 별로 유용하지 않으므로 더 이상 쓰지 않습니다. 먼 훗날 다른 섹션에서 세련된 `NonNull` 타입에 대해 배울 테니 지금은 걱정하지 마세요.



# 기초 (Basics)

`List::new` 구문은 기본적으로 동일합니다.

```rust ,ignore
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

`push` 구문도 거의 똑같-


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(
```

잠시만요, 우리 이제 `Box` 안 쓰기로 했잖아요. 그럼 Box 없이 메모리 할당(allocate memory)은 어떻게 하나요?

음, 사실 `std::alloc::alloc`으로 할 수도 있긴 한데, 그건 파 하나 썰자고 일본도(katana)를 부엌에 들여오는 꼴입니다. 작동이야 하겠지만 너무 과하고 부담스럽습니다.

우린 Box를 가지되, *가지지 않으려* 합니다. 완전히 미쳤지만 *아주 조금은* 가능성 있어 보이는(wild but *maybe* viable) 방법으로는 이렇게 하는 게 있습니다:

```
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

이 방법의 의도는 Box를 생성해서 우리의 노드 안에 저장(store them)해 둡니다. 그러고 나서 원시 포인터를 하나 뽑아낸 다음, 그 노드 사용이 다 끝나고 파괴(destroy it)하고 싶어질 때까지 오직 그 원시 포인터만 줄창 사용하는 겁니다. 마지막에 이르러서야 `real_next`에서 Box를 `take`로 꺼내서 드롭(drop)시키는 거죠. 제 생각엔 이러면 꽤나 간소화된 Stacked Borrows 모델에도 잘 부합할 것 *같습니다만*? 

만약 직접 저걸 만들어보고 싶다면 "행복한" 시간 되시길 바랍니다. 하지만 진짜 끔찍해 보이지 않나요? 여긴 Rc와 RefCell 챕터가 아닙니다. 우린 그따위 오물 장난질(game)은 더 안 할 겁니다. 아주 깨끗하고 심플한 것을 만들 테니까요.

대신, 우리는 아주 멋진 [Box::into_raw][] 함수를 사용할 겁니다:

> ```rust ,ignore
>   pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> Box를 소비하여, 감싸진 원시 포인터를 반환합니다.
>
> 이 포인터는 적절히 정렬되어 있으며(properly aligned) null이 아닙니다.
>
> 이 함수를 호출한 후에는, 이전에 Box가 책임지던 메모리 관리에 대한 책임이 호출자(caller)에게 넘어갑니다. 특히, 호출자는 T를 올바르게 파괴(properly destroy)하고 Box의 메모리 레이아웃을 고려하여 메모리를 해제해야 합니다. 이를 수행하는 가장 쉬운 방법은 `Box::from_raw` 함수를 사용하여 원시 포인터를 다시 Box로 변환한 뒤, Box의 소멸자가 뒤처리를 수행(perform the cleanup)하도록 하는 것입니다.
>
> 참고: 이것은 연관 함수(associated function)이므로 `b.into_raw()` 대신 `Box::into_raw(b)` 형식으로 호출해야 합니다. 이는 내부 타입의 메서드와의 이름 충돌을 피하기 위함입니다.
>
> **Examples**
>
> 원시 포인터를 `Box::from_raw`로 다시 Box로 변환하여 자동 정리를 수행하는 예:
>
> ```
>  let x = Box::new(String::from("Hello"));
>  let ptr = Box::into_raw(x);
>  let x = unsafe { Box::from_raw(ptr) };
> ```

정말 훌륭하네요, 말 그대로 우리의 사용 사례를 위해 설계된(*literally* designed for our use case) 완벽한 물건입니다. 안전한 것에서 시작해서(start with safe stuff), 원시 포인터로 변환하고(turn into into raw pointers), 마지막(Drop 하길 원할 때)에만 다시 안전한 것으로 변환(convert back to safe stuff)한다는 우리의 철칙 규칙과도 매우 잘 맞습니다.

기본적으로 이건 우리가 아까 궁상떨며 구상했던 기괴한 `real_next` 꼼수와 완전히 똑같은(exactly like) 방식입니다. 어차피 원시 포인터나 똑같은 주솟값을 갖는 Box를 굳이 번거롭게 저장해 가며 고생할 필요 없이 바로 직행하는 셈이죠.

또한 우리는 이제 모든 곳에 원시 포인터를 무더기로 사용할 것이므로, `unsafe` 블록을 좁게 유지해야겠다는 오지랖 따윈 그냥 무시합시다: 이젠 모든 것이 안전하지 않습니다(it's all unsafe now). (물론 언제나 그래왔습니다만, 가끔 스스로에게 거짓말을 하는 것도 기분 좋잖아요.)


```rust ,ignore
pub fn push(&mut self, elem: T) {
    unsafe {
        // 즉시 Box를 원시 포인터로 변환합니다
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```

오, 그저 원시 포인터에만 집중했을 뿐인데 코드가 훨씬 더 깔끔해 보이지 않나요(looking a lot cleaner)!

자 다음은 `pop` 입니다. 이 녀석 역시 과거의 모습과 상당히 비슷합니다. 다만 할당을 정리하기 위해(to clean up the allocation) `Box::from_raw`를 써야 한다는 점을 기억해야 합니다:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    unsafe {
        if self.head.is_null() {
            None
        } else {
            // 무덤에서 부활하라 (RISE FROM THE GRAVE)
            let head = Box::from_raw(self.head);
            self.head = head.next;

            if self.head.is_null() {
                self.tail = ptr::null_mut();
            }

            Some(head.elem)
        }
    }
}
```

안타깝게도 귀여웠던 `take`와 `map` 기능은 이제 죽었습니다(are dead). 우리는 손수 직접 체크하고 `null` 세팅을 해주어야 합니다.

그리고 여기까지 온 김에(while we're here), 소멸자(destructor)도 달아줍시다. 이번에는 굉장히 귀엽고 단순하게(cute and simple) 구현하기 위해, 그냥 내내 계속해서 `pop`을 호출(repeatedly popping)하는 방식으로 짜보겠습니다:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```

자, 이제 진실의 시간(moment of truth)입니다:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

좋습니다. 그럼 Miri 도 이에 동의할까요(miri agree)?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

이야아아아아!!!!! (EYYYY!!!!!)

존나게 잘 작동하는구만! (IT FRIGGIN WORKED!)

뭐 아마도요! (PROBABLY!)

미정의 동작(UNDEFINED BEHAVIOUR)을 잡아내지 못했다고 해서 이 안에 진짜 아무 문제가 숨어있지 않다는 완벽한 증명이 될 순 없겠지만, 솔직히 연결 리스트 관련 농담 책(JOKE BOOK)을 쓰면서 제가 얼마나 더 엄밀(RIGOROUS)해져야 할지 한계가 있으니, 저는 그냥 이 결과를 100% 기계 검증된 완벽한 증명서(100% MACHINE VERIFIED PROOF)라고 선언해 버리렵니다. 딴지 거는 새끼들은 다 엿 먹으라죠!(SUCK MY COQ!)

∴ 증명 완료 (QED) □

[Box::into_raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw
