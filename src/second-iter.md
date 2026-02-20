# Iter

좋습니다, 내친김에 Iter까지 구현해 봅시다. 이번에는 List가 제공하는 기능에 기댈 수 없습니다. 우리가 직접 만들어야 합니다. 기본적인 로직은 다음에 산출할 요소 포인터를 현재 리스트의 노드에 유지하는 것입니다. 그 노드가 존재하지 않을 수도 있으므로(리스트가 비었거나 반복이 끝났을 경우), 참조자는 Option이 되어야 합니다. 요소를 하나 산출하면, 현재 노드의 `next` 노드로 이동합니다.

한번 해봅시다:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

오 세상에. 라이프타임(Lifetimes)이군요. 악명 높은 그걸 드디어 만났네요.

새로운 것을 시도해 보죠. 저 `error[E0106]` 이라는게 보이시나요? 이건 컴파일러 에러 코드입니다. `--explain`으로 이게 무슨 뜻인지 물어보겠습니다:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

음... 별로 명확하게 알려주지 않네요 (우리가 이런 걸 이해할 수 있다고 가정하는 것 같습니다). 하지만 구조체에 저 `'a` 같은 걸 추가해야 한다는 것 같습니다? 어디 한번 해보죠.

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

뭔가 패턴이 보이는 것도 같습니다... 그냥 가능한 모든 곳에 다 저 귀여운 걸 추가해 볼까요:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

오, 맙소사. 우리가 Rust를 망가뜨렸습니다.

도대체 이 `'a` 라이프타임이라는 게 뭔지 진짜로 알아봐야겠네요.

라이프타임은 많은 이들에게 공포의 대상이 되곤 합니다. 기존 프로그래밍 언어에서 친숙했던 것들의 문법을 바꿔버리거든요. 여태껏 우리는 라이프타임을 피해서 코딩해 왔지만, 사실 라이프타임은 우리 프로그램 전체에 뒤얽혀 있었습니다.

가비지 컬렉터(GC)가 있는 언어에서는 마법처럼 알아서 데이터를 유지해주기 때문에 라이프타임이 필요 없습니다. 하지만 Rust에서는 대부분의 데이터가 *수동으로* 관리되므로, 이를 위한 해결책이 필요합니다. C나 C++에서 스택의 데이터에 대한 포인터를 함부로 가져다 썼을 때 벌어지는 문제를 떠올려 보면 그 심각성을 알 수 있습니다:

* 범위를 벗어난(스코프가 끝난) 대상을 가리키는 포인터 유지하기
* 가변되어(mutated) 내용이 바뀐 대상을 가리키는 포인터 유지하기

라이프타임은 이 두 가지 문제를 모두 해결하며, 99%의 경우에는 완전히 투명하게 작동합니다.

그렇다면 라이프타임이 도대체 뭔가요?

아주 간단하게 말해서, 라이프타임은 프로그램 코드 내의 일정한 영역(region, ~block/scope)에 이름을 붙인 것에 불과합니다. 이게 전부입니다. 참조자에 라이프타임 태그가 붙었다는 건, 그 참조자가 해당 *전체* 영역 동안 유효해야 한다고 명시하는 것입니다. 요소들이 각각 참조자가 얼마나 오래 유효해야 하는지(must) 또한 언제까지 유효할 수 있는지(can)에 제약을 둡니다. 이 전체 라이프타임 시스템은 각각의 라이프타임의 유효 범위를 최소화하려고 시도하는 제약 조건 해결(constraint-solving) 시스템일 뿐입니다. 시스템이 모든 요구 조건을 만족시키는 라이프타임 쌍을 발견하면, 프로그램이 컴파일됩니다! 그렇지 못하면, 무언가가 충분히 길게 살아있지 못했다(didn't live long enough)는 에러를 뱉어내게 됩니다.

일반적으로 함수 본문 안에서는 라이프타임에 대해서 왈가왈부할 필요가 없고, 심지어 *그러고 싶지도* 않을 것입니다. 컴파일러가 알아서 최소한의 라이프타임을 계산하고 모든 제약을 찾을 만큼 충분한 정보를 가지고 있으니까요. 하지만 타입과 API 수준으로 올라가면, 컴파일러도 그 모든 정보를 속속들이 *알지 못합니다*. 컴파일러가 우리가 무슨 짓을 벌이려는지 파악할 수 있도록 여러 라이프타임 사이의 관계를 알려주어야 합니다.

원칙적으로는 이런 라이프타임 선언을 *전부 생략해도* 되게 만들 수 있겠지만, 그렇게 하면 온 프로그램 전체의 대여(borrows) 상태를 추적해야 하는 방대한 전역 분석 시스템이 되어버리며, 어디서 에러가 났는지도 알 수 없는 괴상한 에러를 토해낼 것입니다. 대신 Rust의 철학은, 모든 빌림 검사(borrow checking)가 각 함수 본문에서 개별 독립적으로 수행되도록 설계되었습니다. 덕분에 모든 에러는 매우 지역적(지엽적)으로 좁혀집니다 (만약 아니라면 당신의 타입 선언 서명이 너무 말도 안 된다는 뜻이겠죠).

"잠깐, 근데 우린 지금까지 함수 서명에 잘만 참조자들을 썼는데 괜찮았잖아요!"
맞습니다, 그건 특정 패턴들이 워낙 자주 쓰이다 보니 Rust가 알아서 라이프타임을 자동으로 정해주기 때문입니다. 이것이 바로 *라이프타임 생략(lifetime elision)*입니다.

특히 다음과 같은 경우에 말이죠:

```rust ,ignore
// 입력값 참조자가 단 한 개뿐이므로, 반환값 역시 그 입력값 파생일 수밖에 없음
fn foo(&A) -> &B; // 이는 내부적으론 이런 의미입니다:
fn foo<'a>(&'a A) -> &'a B;

// 입력값이 여러 개라면, 몽땅 서로 연관 없는 독립개체로 취급
fn foo(&A, &B, &C); // 축약 전 원래 의미:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// 타입에 속한 메서드라면, 출력 라이프타임은 모두 `self`로부터 파생된다고 취급
fn foo(&self, &B, &C) -> &D; // 축약 전 원래 의미:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

그렇다면 `fn foo<'a>(&'a A) -> &'a B`는 무슨 *의미*일까요? 실질적으로, 이것은 입력값이 반드시 최소한 출력값만큼은 길게 살아있어야 한다는 의미입니다. 따라서 출력 변수를 오랫동안 유지한다면, 입력이 유효해야 하는 영역 또한 길어질 것입니다. 출력 변수 사용을 멈추면 컴파일러는 이제 입력값이 무효화되어도 괜찮다는 것을 깨달을 것입니다.

이러한 시스템을 통해 Rust는 해제된 메모리를 사용하는 것(use after free)을 막고, 밖에서 쓰이고 있는 참조자가 있는데 값이 가변(mutated)되는 것을 완벽하게 막습니다. 제약 조건들만 다 맞아떨어지면 확신할 수 있습니다!

자. 그래서. Iter.

라이프타임이 전혀 없던 첫 상태로 롤백해봅시다:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

우리는 함수와 타입 서명에만 라이프타임을 붙이면 됩니다:

```rust ,ignore
// Iter는 *어떤* 라이프타임에 대해서 제네릭일 뿐, 특별히 신경 쓰지 않습니다
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// 여기선 별다른 라이프타임이 없습니다. List는 특정 라이프타임에 구속되지 않으니까요.
impl<T> List<T> {
    // 우리는 이 구역에서야 비로소 iter가 쥐어짜내는 그 *정확한* 대여(borrow) 시점에 국한되는 
    // 새로운 라이프타임 파이프라인을 선포합니다. 이제 &self는 Iter가 살아 숨쉬는 한
    // 끝까지 유효해야 합니다.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// 오 여긴 구체적 라이프타임을 붙여야만 합니다, 바깥 테두리 Iter에 맞춰 정의해야 하니까요
impl<'a, T> Iterator for Iter<'a, T> {
    // 여기도 타입을 구체적으로 찍어 명시하는 곳입니다
    type Item = &'a T;

    // 본문 내부 구문들은 전혀 바꿀 필요가 없습니다. 저 윗부분이 알아서 보호막을 칩니다.
    // 우리의 멋진 Self도 언제나 최고입니다.
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

좋습니다, 이번엔 진짜로 해결한 것 같군요.

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

오케이. 자. 라이프타임 에러 고쳤더니 이제 새로운 타입 에러가 터지네요.

우리는 `&Node`를 저장하려고 했는데 이 녀석이 `&Box<Node>`를 반환하고 있습니다. 그건 쉽죠. 그냥 참조자를 따기 전에 Box를 역참조하면 됩니다:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

우리가 멍청하게 `as_ref`를 빼먹어서 통째로 Box 자체를 `map` 내부로 꺼내며(move) 이동시켜 버렸고, 이는 소멸시켜(dropped) 버리겠다는 뜻이므로 우리가 들고나온 참조자는 매달린 포인터(dangling) 쓰레기가 될 것입니다:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref`가 한 겹의 간접 참조를 추가했기 때문에 결과적으로 쓸데없이 이 겹을 제거해야 합니다:


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
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

```text
cargo build

```

🎉 🎉 🎉

as_deref 함수와 as_deref_mut 함수는 Rust 1.40부터 안정화(stable)되었습니다. 그 이전 버전에서는 눈물을 흘리며 `map(|node| &**node)`와 `map(|node| &mut**node)` 처럼 기괴하게 작성해야 했습니다.
`&**`는 좀 별로라고 생각하실 텐데, 그 생각은 완전히 옳습니다. 시간이 갈수록 더 좋아지는 고급 와인처럼 Rust 역시는 진화했고, 이제 그런 더러운 짓을 반복할 필요가 없어졌습니다. 보통 Rust는 저런 역참조 캐스팅(deref coercion)을 꽤 훌륭하게 묵과해주고 알아서 내부적으로 별표 기호(\*)를 이리저리 끼워 맞춰 줍니다.

하지만 지금 상황에서는 우리가 단순한 `&T`가 아니라 `Option<&T>` 구조체의 함수 내부 로직 클로저 안과 맞물려있기 때문에 타입 강제 시스템이 이를 자동 추론해 풀어내기가 너무 어려웠습니다(too complicated). 그래서 우리가 수명줄을 잡아끌어 명시적으로(explicit) 표기해주어야 구제될 수 있었습니다. 참 다행으로도, 이런 피곤한 경우는 굉장히 드물게 일어납니다.

완벽한 구색을 갖추기 위해(completeness' sake), 이것과는 사뭇 다르지만 소위 *터보피쉬(turbofish)* 문법이라는 꼼수로 이를 벗어날 수도 있습니다:

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

map 녀석도 원래 제네릭(generic) 함수입니다:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

저 괴상망측하게 생긴 `::<>` 터보피쉬(turbofish)는 우리가 컴파일러에게 제네릭이 무슨 타입이어야 하는지 참견해서 알려줄 수 있는 힌트 창구입니다. `::<&Node<T>, _>`는 "반환할 때 당연히 `&Node<T>` 타입을 토해내라, 그 뒤에 나오는 딴 놈은 상관 안 하니까 알아서 해라"라고 말해주는 것이죠.

이 덕분에 컴파일러는 `&node` 부분에 자신이 그동안 꽁꽁 숨겨두었던 꼼수 역참조 강제 캐스팅(deref coercion) 스킬을 발동시키기로 결심하고, 우리는 \* 기호 도배질에서 해방될 수 있습니다!

하지만 이것도 크게 엄청난 개선이라 보기엔 힘드네요, 그저 나름대로 멋있는 역참조 강제 변환과 이따금 유용한 터보피쉬의 모습을 여러분에게 자랑할 핑계(veiled excuse)였을 뿐입니다. 😅

자 혹시 몰라 노-옵(no-op)으로 아무짝에도 쓸모없이 무효화되진 않았는지 한 번만 테스트해 줍시다:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

바로 이거죠.

마지막으로 흥미로운 사실 하나, 결국 여기까지 오면서 사실상 위 상황에서 우리는 눈 딱 감고 거룩한 수명 라이프타임 생략(lifetime elision) 꼼수를 뼛속까지 적용해버려도 무방하다는 고백을 드립니다:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

이 코드는 놀랍게도:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

이것과 똑같은 코드(equivalent)입니다!

적은 라이프타임 만세!

혹시 이게 라이프타임을 숨기고 있는게 싫으시다면 `_`을 써도 됩니다:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```
