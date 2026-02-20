# 푸쉬 (Push)

자 이제 리스트 위에 새로운 값을 집어넣어(push) 봅시다. `push` 연산은 리스트 상태를 *가변(mutates)*시키므로 매개변수로 `&mut self`를 가져와야 할 것입니다. 또한 방금 추가하려는 i32 타입의 요소(element)도 하나 가져와야 합니다:

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

제일 먼저 해야 할 일은 우리가 받은 요소를 보관할 새로운 노드를 한 개 만들어내는 것입니다:

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

새 술은 새 부대에, 다음(`next`)엔 뭐가 들어가야 할까요? 네, 당연히 예전에 있던 기존 리스트 전체가 들어가야 합니다! 과연 우리가... 그냥 대충 그렇게 들이부으면 될까요?

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

아~~니요오. Rust는 우리에게 아주 정확한 조언을 해주고 있습니다만, 도대체 저게 정확히 무슨 뜻인지도 그리고 무얼 어떡하라는 건지도 명확하진 않습니다:

> cannot move out of borrowed content (빌려온 참조 내용물에서 밖으로 이동시킬 수 없습니다)

우리는 여기서 `self.head` 값을 바깥의 `next`로 이동(move out)시키려 하고 있지만, Rust는 우리가 감히 그런 짓을 벌이길 원치 않습니다. 왜냐하면 우리가 대여 기간(borrow)을 마치고 그것의 온전한 소유자에게 "다시 반납해 주어야 할" 때, 정작 `self`는 일부 데이터가 증발한 반쪽짜리 비정상 상태로 텅 비게 되어버릴 것이기 때문입니다. 앞서 말했듯이 이건 여러분이 `&mut`를 다루며 *절대 해서는 안 되는 단 하나의 행동*입니다: 그건 소유자에게 지독하게 무례한 행동이고, 우리의 Rust씨는 언제나 매우 깍듯한 신사니까요 (물론 그딴 예절보다도 믿을 수 없을 만큼 끔찍하게 위험한 메모리 동작이기 때문이지만, 결단코 *저런* 이유만은 아닐 겁니다).

그렇다면 우리가 무언가 다른 걸 대신 그 빈자리에 밀어 넣어두면 어떨까요? 이를테면 지금 우리가 한창 만들고 있는 바로 저 새로운 노드 말입니다:


```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

어림도 없군요. 원칙적으로 말하자면 이건 마땅히 Rust가 허용해 줄 수도 있었을법한 논리 구조지만 애석하게도 단칼에 거부당했습니다 (수많은 원인이 있지만 가장 무거운 죄목은 [예외 안전성(exception safety)][exception safety] 때문이었습니다). 우리는 Rust가 눈치채지 못하게 아주 재빠르게 헤드를 가로채 올 수 있는 어떤 마법 같은 우회 수단이 필요합니다. 우리는 여기서 악명 높은 Rust 마스터 해커 '인디아나 존스' 씨에게 조언을 청해보기로 하겠습니다:

![Indy Prepares to mem::replace](img/indy.gif)

아 그래요, 우리의 존스 박사님께서 `mem::replace` 기술을 제안해 주셨습니다. 이 믿을 수 없을 정도로 유용한 함수는 우리가 빌려온 참조자 위치에 냅다 *다른 빈 값(another value)*을 대신 채워 넣음으로써 무사히 탐내던 값을 몰래 훔쳐 올 수(steal) 있게 해줍니다. 일단 `std::mem`을 파일 최상단에 끌어와서 지역 스코프(local scope) 내에 맴돌게 해둡시다:

```rust ,ignore
use std::mem;
```

그리고 이것을 적절히 사용해 보겠습니다:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

여기서 우리는 `self.head`를 리스트의 새로운 헤드로 교체하기에 앞서 잠시 임시로 빈 허수아비인 `Link::Empty`와 위치를 `교환(replace)`해주었습니다. 거짓말은 하지 않겠습니다: 이렇게까지 눈속임하며 우회해야 한다는 건 꽤나 유감스럽고 슬픈 일이긴 합니다. 애석하지만 (지금은 당장) 이렇게 해야만 합니다.

하지만 헤이, 어쨌든 `push` 연산부의 작성은 모든 과정이 온전히 끝났습니다! 필시 정상 동작하겠죠. 하지만 솔직하게 말하면 우린 이 녀석이 정말 잘 도는지 확인해 볼 테스트 코드를 짜야만 합니다. 지금 당장 그것을 검증해 볼 수 있는 가장 쉬운 수단은 십중팔구 `pop` 역시 마저 작성해서 이 둘을 맞부딪혀 올바른 결과가 도출되는지 살펴보는 방법일 것입니다.





[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
