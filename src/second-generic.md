# 모든 것을 제네릭으로 만들기 (Making it all Generic)

우리는 이미 앞서 Option과 Box를 만지작거리며 제네릭(Generics)을 혀끝으로 조금 맛보았습니다. 하지만 사실 우리는 그동안 어떠한 임의의 타입(arbitrary elements)을 감당해 내는 진짜 제네릭 타입을 단 하나도 직접 선언해 보진 못한 채 회피하며 살아왔습니다.

근데 사실 그거, 막상 해보면 엄청나게 쉽습니다. 당장 우리의 낡은 타입들에 제네릭 꺾쇠를 줄줄이 박아넣어봅시다:

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

모든 모서리를 조금 더 뾰족뾰족하게 만들어주기만 하면, 갑자기 여러분의 코드는 제네릭이 됩니다. 물론 우리가 *고작* 이것만 하고 만다면 컴파일러는 호락호락하지 않고 머리끝까지 극대노(Super Mad)해서 비명을 지르게 될 것입니다.


```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

터지는 문제는 무척 명백합니다: 우리는 여전히 이 `List`라는 것에 대해 이야기하고 있지만, 사실 그것들은 세상에 더는 존재하지 않습니다. Option이나 Box 녀석들이 그래왔듯, 이제부터 우리는 언제나 `List<Something>`이라는 형태로 이야기해야만 합니다.

하지만 이 모든 수많은 `impl` 구현체들에게 써줘야 하는 그 Something의 통일된 정체는 당최 뭘까요? List에서 그래왔듯, 우리는 당연히 우리의 구현들이 가능한 *모든* 종류의 T들과 호흡을 맞추며 동작하길 원합니다. 따라서 List의 껍데기와 마찬가지로 우리의 `impl` 블록들 역시 이렇게 뾰족하게 깎아 장식해 줍시다:


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...이게 끝입니다!


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

이제 우리의 모든 코드가 T라는 임의의 덩어리 값을 받아치며 제네릭의 마법 속에 완전히 동기화되었습니다. 젠장, Rust 정말 *쉽네요*. 이번 과정에서 심지어 코드 한 글자도 바뀌지 않고 구사일생한 자랑스러운 `new` 함수에게 특별히 박수를 보내고 싶습니다:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

복붙(copy-pasta) 코딩을 구원하고 무자비한 리팩토링의 험난한 폭풍을 막아낸 불멸의 수호자, 영광스러운 Self의 빛줄기 아래에서 편히 쉬십시오.
그리고 흥미로운 잡학지식으로, 우리가 실제로 list의 인스턴스를 생성할 때엔 일부러 `List<T>`라고 적어 넣어야 할 이유가 전혀 없습니다. Rust 컴파일러는 우리가 `List<T>`를 요구하는 함수로부터 값을 넘겨받아 반환하고 있다는 사실을 포착하고, 알아서 그 퍼즐 조각을 유추해 맞춰(inferred) 주기 때문입니다.

좋습니다, 자 그럼 한 번 완전히 새로운 *동작(behaviour)*을 작성하러 가봅시다!
