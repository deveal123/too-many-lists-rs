# 엿보기 (Peek)

지난 번에 우리가 구현조차 하지 않고 거들떠보지도 않고 대차게 무시했던 기능 하나가 바로 '엿보기(peeking)' 동작입니다. 이번엔 이 녀석을 직접 한번 만들어 봅시다. 우리가 이 기능에 바라야 할 전부라곤 리스트의 맨 앞(head) 부분에 알맹이(element)가 존재한다면 그 요소 내부로의 포인터 참조자(reference)를 슬쩍 반환해 주기만 하면 될 뿐입니다. 말로 들으니 꽤 쉬워 보이네요, 한 번 직접 들이대 볼까요:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*깊은 한숨 (Sigh)*. 자 이제 또 뭐 어떡하라는 거니, Rust야?

원인은 Map 메서드가 `self`를 안면몰수하고 그 본체를 *값(by value)*으로 꿀꺽 집어삼켜 버리기 때문입니다. 이것은 Option을 자신이 원래 박혀있던 그곳에서 통째로 뜯어내 밖으로 이동(move out)시켜버리는 행위입니다. 저번 `pop` 시절엔 어차피 우리가 무식하게 냅다 `take`로 몽땅 적출해버렸기 때문에 문제가 되지 않았지만, 단지 구멍 틈새로 살짝 엿보고 싶은(peek) 지금 상황에서 당연히 피사체는 제자리에 그대로 보존된 채 얌전히 남겨져 있어야만 합니다. 이 꼬인 문제를 푸는 *정상적인(correct)* 방법은 Option이 친히 구비해 둔 `as_ref`라는 특수 메서드를 등판시키는 것입니다. 이 녀석의 정의는 다음과 같습니다:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

이 녀석은 `Option<T>`를 강제로 한 단계 강등(demotes)시켜, 그 내부 내용물에 대한 참조자를 슬그머니 품은 한 겹 덮인 `Option<&T>`으로 둔갑시켜 줍니다. 우린 물론 이 귀찮은 작업을 구식 명시적 match 구문으로 손수 대체할 수도 있겠지만 *으웩 저리 치워 안 해*라고 외칠 겁니다. 뭐, 이 말인 즉 우리가 이 두꺼운 겹 추가 간접 껍질을 뚫고 들어가기 위해 매번 한 차례씩 여분의 이중 역참조(dereference) 과정을 수고스럽게 치러야 한다는 것을 의미하지만, 신께 영광스럽게도 마법의 `.` 점 연산자(operator)가 알아서 이런 수고를 모조리 대리 수행해 줍니다.


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

딱 들이맞았네요 (Nailed it).

우리는 `as_mut`를 활용해서 똑 닮은 이 녀석의 가변성 버전(*mutable* version) 엿보기 메서드도 개조해 낼 수 있습니다:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

너무 엄청 쉽구먼 (EZ)

이걸 테스트하러 검증장을 만들어두는 것도 잊지 마세요:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

꽤 훌륭하군요. 하지만 까놓고 솔직히 말해서 우린 정작 저 `peek_mut` 가 결과로 토해 낸 반환값을 진짜로 요리조리 칼로 썰어 가변 조작(mutate)시킬 권능이 확실히 부여되었는지 여부를 단 한 번도 몸소 테스트 조차 안 해보고 건너뛰지 않았나요? 만약 가변(mutable) 수식어가 붙은 참조자가 돌아왔는데도 그 누구도 찔러가며 가변시키질 않는다면, 우린 이걸 정말 '가변성'을 검증했다고 떳떳이 이빨을 깔 수 있을까요? 자 그러니, 돌려받은 이 끔찍한 `Option<&mut T>` 변수를 재차 `map`의 도마 위에 버무려서 아주 심오하고 대단한 값(42)을 안쪽에 박아 쑤셔 넣어 봅시다!:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

컴파일러가 울분을 토하며 저깟 `value` 따위는 절대 불변(immutable)이라고 불평을 터뜨리고 있는데, 솔직히 방금 직전에 두 눈 시퍼렇게 뜨고 확실히 `&mut value` 라고 떡하니 몹시 명백하게 박아 적어넣어주지 않았답니까; 도대체 이거 머리가 어떻게 된 거 아닙니까? 
진실은 이렇습니다. 클로저(closure)의 아가리에 건네는 인자에다 대고 저런 요망한 방식으로 휘갈겨 적어놨다고 해서, 여전히 그게 진짜로 `value`가 가변 참조자(mutable reference)라는 거룩한 상태를 구체적으로 선언(specify)해 주진 못한다는 것입니다! 그 대신, 이것은 그저 클로저로 꾸역꾸역 들어쳐 박힐 인자를 상대로 단지 어떤 기묘한 패턴(pattern) 형태의 그물이 맞물려 포개어지는지 대조할 뿐입니다. 즉슨 `|&mut value|` 의 진정한 속내 뜻은 조롱 삼아 일컫자면 "인자로 받은 놈이 가변 참조자인 건 잘 알겠고요, 우린 그냥 그 가리키는 원본 화살표를 타고 넘어가 종착지의 날 것 치즈 값만을 쪽 빨아당겨서 내 변수 `value` 통 안에 복사해(copy) 쑤셔 넣어 주시렵니까, 제발요."라는 황당한 지시어였던 셈입니다. 

만일 우리가 그 껍질을 벗어던지고 단호하게 그냥 `|value|` 라고 퉁명스레 던져버린다면, 그제서야 비로소 `value` 의 진정한 타입은 묵직한 오리지널 포인터 창날인 `&mut i32` 그대로 인수인계될 것이며, 우린 드디어 소원대로 마음껏 앞단 헤드의 면상 자체를 직접 조작해버려 가변시켜(mutate) 버릴 수 있게 됩니다:

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

크으 훨씬 더 낫네요!
