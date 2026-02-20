# IntoIter

Rust에서 컬렉션(Collections)을 순회 반복할 땐 *Iterator* 트레이트를 사용합니다. 이 녀석은 `Drop`보다 살짝 더 복잡합니다:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

여기서 새로 등장한 녀석이 바로 `type Item`입니다. 이는 Iterator 트레이트를 구현할 때 'Item'이라는 이름의 *연관 타입(associated type)*을 선언해야 한다고 명시하는 것입니다. 이 경우, Item은 당신이 `next`를 호출할 때 튀어나올 값의 타입을 가리킵니다.

Iterator가 `Option<Self::Item>`을 반환하는 이유는 인터페이스가 `has_next`와 `get_next` 개념을 통합(coalesces)했기 때문입니다. 다음 값을 가지고 있다면 `Some(value)`를 산출하고, 없다면 `None`을 산출합니다. 이렇게 하면 API가 일반적으로 더 인간공학적이고 안전하게 쓰이고 구현될 수 있으며, `has_next`와 `get_next` 사이의 중복된 검사와 로직을 피할 수 있습니다. 아주 멋지죠!

아쉽게도, Rust에는 (아직) `yield` 구문 같은 게 없기 때문에 로직을 우리가 직접 구현해야 합니다. 또한, 컬렉션이라면 의무적으로 다음 3가지 종류의 이터레이터를 모두 구현하기 위해 노력해야 합니다:

* IntoIter - `T` 
* IterMut - `&mut T` 
* Iter - `&T` 

사실 우리는 List의 인터페이스를 활용해 IntoIter를 구현할 모든 도구를 이미 갖추고 있습니다: 그냥 `pop`을 계속 반복 호출하면 됩니다. 따라서, IntoIter를 List를 감싸는 뉴타입 래퍼(newtype wrapper)로 구현해 보겠습니다:


```rust ,ignore
// 튜플 구조체(Tuple structs)는 구조체의 또 다른 형태이며,
// 다른 타입을 단순하게 감싸는 래퍼를 만들 때 유용합니다.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // 튜플 구조체의 필드는 숫자로 접근합니다
        self.0.pop()
    }
}
```

그리고 테스트를 하나 작성해 봅시다:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

훌륭하네요!
