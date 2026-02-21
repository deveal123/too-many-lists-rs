# 더블 단일 연결 리스트 (The Double Singly-Linked List)

우리가 앞서 이중 연결 리스트(doubly-linked lists)를 짜면서 그토록 피똥을 싼 이유는, 이녀석이 소유권(ownership) 의미론적으로 아주 기가 막히게 꼬여(tangled) 있었기 때문입니다: 어떤 노드도 다른 노드를 엄격하게 완벽히 "소유"하지 못하거든요. 하지만 우리가 이따위로 고통받은 진짜 근본 원인은, 애초에 연결 리스트라는 게 필연적으로 *어때야만 한다*는 편견(preconceived notions)을 머릿속에 가득 채우고 있었기 때문입니다. 이름하여, 모든 링크가 무조건 같은 한 방향(same direction)으로만 뻗어나가야 한다는 고정관념 말이죠.

발상을 전환해서, 걍 우리 리스트를 반갈죽 낸 다음 두 덩이(halves)로 박살 내 봅시다: 한 녀석은 왼쪽으로, 다른 한 녀석은 오른쪽으로 뻗어나가는 겁니다:

```rust ,ignore
// lib.rs
// ...
pub mod silly1;     // NEW!
```

```rust ,ignore
// silly1.rs
use crate::second::List as Stack;

struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}
```

자, 이제 우린 그저 옹졸한 안전한 스택 따위가 아니라, 어엿한 범용 리스트(general purpose list)를 손에 쥐었습니다.
어느 쪽 스택이든 맘에 드는 곳에 `push`를 박아 넣으면 리스트를 왼쪽이나 오른쪽 양방향으로 길쭉하게 키워나갈 수 있습니다.
또한 한쪽 끝에서 값을 `pop`으로 뽑아다 반대쪽 머리에 씌워주는 짓거리를 반복하면, 리스트를 타고 우아하게 "걸어 다닐(walk)" 수도 있습니다. 덤으로 쓸데없는 메모리 할당(allocations) 낭비를 막기 위해, 이전에 짰던 안전한 Stack 코드를 돚거해와서 녀석의 숨겨진 은밀한(private) 내부 속살까지 다 까발려 써먹을 겁니다:

```rust ,ignore
pub struct Stack<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { head: None }
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
            let node = *node;
            self.head = node.next;
            node.elem
        })
    }

    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            &node.elem
        })
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|node| {
            &mut node.elem
        })
    }
}

impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

그리고 `push` 랑 `pop` 녀석만 살짝 마개조(rework) 해줍시다:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let new_node = Box::new(Node {
        elem: elem,
        next: None,
    });

    self.push_node(new_node);
}

fn push_node(&mut self, mut node: Box<Node<T>>) {
    node.next = self.head.take();
    self.head = Some(node);
}

pub fn pop(&mut self) -> Option<T> {
    self.pop_node().map(|node| {
        node.elem
    })
}

fn pop_node(&mut self) -> Option<Box<Node<T>>> {
    self.head.take().map(|mut node| {
        self.head = node.next.take();
        node
    })
}
```

이제 본격적으로 우리의 짬짜면 List를 조립해 봅시다:

```rust ,ignore
pub struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}

impl<T> List<T> {
    fn new() -> Self {
        List { left: Stack::new(), right: Stack::new() }
    }
}
```

그러면 뭐 평소 하던 흔해 빠진 짓거리들(usual stuff)도 다 할 수 있습니다:


```rust ,ignore
pub fn push_left(&mut self, elem: T) { self.left.push(elem) }
pub fn push_right(&mut self, elem: T) { self.right.push(elem) }
pub fn pop_left(&mut self) -> Option<T> { self.left.pop() }
pub fn pop_right(&mut self) -> Option<T> { self.right.pop() }
pub fn peek_left(&self) -> Option<&T> { self.left.peek() }
pub fn peek_right(&self) -> Option<&T> { self.right.peek() }
pub fn peek_left_mut(&mut self) -> Option<&mut T> { self.left.peek_mut() }
pub fn peek_right_mut(&mut self) -> Option<&mut T> { self.right.peek_mut() }
```

하지만 가장 흥미진진한 건, 드디어 리스트 위를 뚜벅뚜벅 걸어 다닐(walk around) 수 있다는 겁니다!


```rust ,ignore
pub fn go_left(&mut self) -> bool {
    self.left.pop_node().map(|node| {
        self.right.push_node(node);
    }).is_some()
}

pub fn go_right(&mut self) -> bool {
    self.right.pop_node().map(|node| {
        self.left.push_node(node);
    }).is_some()
}
```

여기서 `boolean` 녀석을 뱉어내는 건 걍 우리가 실제로 한 발짝 걷는 데(move) 성공했는지 편하게 눈치채려고 덤으로 달아둔 겁니다. 자, 이제 이 기염을 토하는 괴물(baby)을 테스트해 봅시다:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn walk_aboot() {
        let mut list = List::new();             // [_]

        list.push_left(0);                      // [0,_]
        list.push_right(1);                     // [0, _, 1]
        assert_eq!(list.peek_left(), Some(&0));
        assert_eq!(list.peek_right(), Some(&1));

        list.push_left(2);                      // [0, 2, _, 1]
        list.push_left(3);                      // [0, 2, 3, _, 1]
        list.push_right(4);                     // [0, 2, 3, _, 4, 1]

        while list.go_left() {}                 // [_, 0, 2, 3, 4, 1]

        assert_eq!(list.pop_left(), None);
        assert_eq!(list.pop_right(), Some(0));  // [_, 2, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(2));  // [_, 3, 4, 1]

        list.push_left(5);                      // [5, _, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(3));  // [5, _, 4, 1]
        assert_eq!(list.pop_left(), Some(5));   // [_, 4, 1]
        assert_eq!(list.pop_right(), Some(4));  // [_, 1]
        assert_eq!(list.pop_right(), Some(1));  // [_]

        assert_eq!(list.pop_right(), None);
        assert_eq!(list.pop_left(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 16 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fourth::test::into_iter ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::basics ... ok
test third::test::iter ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured
```

이 구현체는 이른바 *손가락(finger)* 자료 구조의 가장 극단적인(extreme) 예시 중 하나입니다. 우린 자료 구조 내부 어딘가에 가상의 손가락 하나를 푹 찔러넣고(maintain) 유지하며, 그 덕분에 손가락 위치에서부터 목표 지점까지 떨어진 거리(distance)에 비례하는 눈부신 속도로 연산을 처리할 수 있습니다.

우리가 가리키고 있는 손가락 주변에선 빛의 속도로(very fast) 리스트 내용을 갈아엎을 수 있지만, 만약 손가락에서 엄청나게 멀리 떨어진 외곽 지역을 건드리고 싶다면 걍 거기까지 뚜벅뚜벅 직접 걸어가야만(walk all the way) 합니다.
한쪽 스택에서 원소들을 뭉탱이로 뽑아다 반대쪽으로 무식하게 퍼다 나르면서 영구적으로(permanently) 이사를 가버릴 수도 있고, 아니면 `&mut` 가변 참조자를 들고 링크들을 타고 살금살금 임시로(temporarily) 기어가서 깽판을 칠 수도 있겠죠. 하지만 명심하세요, 불쌍한 `&mut` 녀석은 리스트를 거슬러 역주행(go back up) 할 순 없지만, 우리의 위대한 손가락은 앞뒤 막론하고 자유롭게 뚫고 다닐 수(can) 있다는 사실을요!

