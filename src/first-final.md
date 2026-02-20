# 최종 코드 (The Final Code)

휴, 6000단어가 넘는 피 튀기는 고생 끝에 우리가 마침내 겨우 짜낸, 실제로 동작하는 전체 코드의 영광스러운 자태가 여기 있습니다:

```rust
use std::mem;

pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, Link::Empty),
        });

        self.head = Link::More(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => None,
            Link::More(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);

        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
        }
    }
}

#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // 텅 빈 리스트가 올바르게 작동하는지 한 번 찍어먹어 봅니다.
        assert_eq!(list.pop(), None);

        // 값들을 왕창 밀어 넣습니다.
        list.push(1);
        list.push(2);
        list.push(3);

        // 올바른 정상 제거가 이뤄지는지 검증합니다.
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // 쓸데없이 리스트가 망가졌는지 확인 차 위에 몇 번 더 집어던져 봅니다.
        list.push(4);
        list.push(5);

        // 다시 한번 정상 제거 메커니즘 확인.
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // 모조리 영혼까지 털려 더 이상 뽑힐 게 없는지 확인합니다.
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

크으 세상에. 전체 80줄 남짓한 코드인데다 절반 이상이 검증용 테스트 코드 쪼가리네요! 뭐어, 애초에 제가 이 첫 번째 과정은 엄청나게 긴 여정이 될 거라고 미리 당부해 드렸지 않습니까! 
