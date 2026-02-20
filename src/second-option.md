# Option 활용 (Using Option)

특히 눈썰미가 예리하신 독자분들이라면 이미 직감하셨겠지만, 우리는 사실 지금까지 정말 말도 안 되게 끔찍한 짝퉁 버전의 Option을 쓸모없이 재발명해서 사용하고 있었습니다:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link는 그저 `Option<Box<Node>>`의 완벽한 동의어나 다름없습니다. 물론, 귀찮게 사방팔방 `Option<Box<Node>>`를 매번 길게 따라 적지 않아도 된다는 이점은 나름 괜찮습니다. 게다가 `pop` 메서드와는 다르게 이 추악한 속살을 굳이 외부 세계에 노출시키지도 않으니, 뭐 아마 이대로 두어도 크게 문제는 없을지도 모릅니다. 
하지만 Option에는 우리가 여태 손수 노가다로 구현해 왔던 눈물겨운 작업들을 알아서 단박에 처리해 주는 *정말 끝내주게 훌륭한* 내장 메서드 도구들이 잔뜩 담겨 있습니다. 자 이제 멍청한 재발명 짓거리는 그만 *때려치우고*, 코드를 전부 Option 구문으로 갈아 끼워버립시다. 
일단 가장 멍청하고 원초적인 방법으로 모든 단어만 Some과 None으로 일대일 맞교환하는 단순 이름 교체로 시작합니다:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// 야호 타입 앨리어스 만세! (yay type aliases!)
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

이 정도만 해도 조금 낫긴 하지만, 진정한 차이는 Option이 기본 내장한 최정예 메서드들을 사용할 때 나타납니다.

첫 번째 타자, `mem::replace(&mut option, None)`이라는 구문은 너무나도 전 우주적인 공통 분모이자 믿을 수 없으리만치 징그럽게 도처에 쓰이는 관용구(idiom)이기에, Option 설계자들조차 그냥 이 마법을 통째로 묶어 `take`라는 공식 내장 메서드로 박아버렸습니다!

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

두 번째 구원 타자, `match option { None => None, Some(x) => Some(y) }` 짓거리 역시 징그럽게 흔한 관용구라서, 결국 `map`이라는 이름이 붙게 지어 되었습니다. `map`은 `Some(x)` 안테 짱박혀있는 불쌍한 `x`를 건네받아 실행하는 외부 함수(function)를 인자로 받아 그 결과물 `y`를 `Some(y)`에 담아 내놓습니다. 물론 우리가 정식 `fn` 함수를 바깥에 선언해서 `map`에게 넘겨줄 수도 있겠지만, 우리는 이 짓거리보다는 코드를 *인라인(inline)*으로 적어 넣는 것을 훨씬 선호합니다.

이 방법을 가능하게 해주는 것이 바로 *클로저(closure)*입니다. 클로저는 익명 함수(anonymous functions)의 일종인데 하나의 엄청난 초능력을 가지고 있습니다: 바로 클로저 *바깥 세상*의 로컬 변수들을 스스럼없이 들이마시고 참조할 수 있다는 점입니다! 이 덕택에 온갖 종류의 조건부 논리들을 자유자재로 짜놓는데 아주 미치도록 유용합니다. 어쨌든 우리 코드에서 현재 `match`를 끙끙대며 쓰는 곳이라곤 오직 `pop` 소굴 단 한 군데뿐이니 냅다 재작성해 봅시다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

아아, 훨씬 낫네요. 혹시나 우리가 뭔가를 박살내지는 않았는지 테스트를 돌려봅시다:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

훌륭합니다! 그럼 이제 한 발 더 나아가 진정으로 코드의 내적 *능률(behaviour)*을 개조시켜줄 다음 스테이지로 나아갑시다.
