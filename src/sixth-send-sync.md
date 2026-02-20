# Send, Sync, 그리고 컴파일 테스트 (Send, Sync, and Compile Tests)

자, 사실 우리가 신경 써야 할 트레이트(trait) 쌍이 하나 더 남아있긴 한데, 이 녀석들은 꽤 각별합니다. 이제 우리는 Rust의 신성 로마 제국(Holy Roman Empire)이라 불리는 그분들을 영접해야 합니다: 바로 [Send 와 Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html)라는 아름다운 이름의 '안전하지 않은 선택적 적용 내장 트레이트(Unsafe Opt-In Built-In Traits, OIBITs)'들이죠. (물론 현실은 선택적 배제(opt-out)에 내장 탈락(built-out)에 더 가깝긴 하지만, 이름에서 3개 중 1개나 맞췄으니 꽤 훌륭하군요!)

Copy와 마찬가지로, 이 트레이트들엔 연결된 실제 코드가 쥐뿔도 없습니다. 그저 여러분의 타입이 이러이러특정 속성을 지녔노라~ 하고 딱지만 붙여주는 마커(markers) 역할만 할 뿐이죠. Send는 당신의 타입이 다른 스레드로 안전하게 전송될 수 있다는 뜻이고, Sync는 당신의 타입이 여러 스레드 사이에서 안전하게 공유될 수 있다는 뜻입니다(`&Self: Send`).

LinkedList가 공변성(covariant)을 쟁취할 때 써먹었던 것과 똑같은 변명이 여기서도 통용됩니다: 일반적으로 뭐 존나 휘황찬란한 내부 가변성(interior mutability) 매직 꼼수기믹 따위를 안 쓰는 평범무난한 컬렉션들은 대부분 별 문젯거리 없이 안전하게 Send 와 Sync를 둘둘 말아 달아줘도 됩니다.

근데 아까 제가 이놈들이 *선택적 배제(opt out)* 방식이라고 했었죠. 그럼 질문, 우리 리스트는 지금 이미 Send랑 Sync가 적용되어 있는 걸까요? 그걸 대체 어떻게 확인하죠?

자, 우리 코드에 새로운 마법을 살짝 뿌려봅시다: 우리 타입들이 마땅히 가져야 할 그 속성을 지니지 못했다면 아예 컴파일 자체가 단칼에 거부당하게끔 만드는, 무작위로 만든 비공개 쓸데없는 쓰레기 코드(private garbage) 덩어리를 끼워 넣는 겁니다:

```rust ,ignore
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```text
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<a million more errors>
```

오 세상에 씨발, 대체 왜 이 모양이죠! 기껏 폼 잡고 아까 신성 로마 제국 개드립까지 거창하게 깔아뒀는데!

네 맞습니다, 제가 아까 원시 포인터엔 안전 가드 조항이 딱 하나뿐이라고 뻥쳤었죠: 사실 여기 하나 더 있습니다. `*const` 님이랑 `*mut` 놈 둘 다 안전제일주의 명분 하에 명시적으로 대놓고 Send 랑 Sync 자격을 스스로 박탈(opt out) 시켜버렸기 때문에, 우리는 찌질하게 굽신거리며 저거 다시 켜달라고 수동으로 *직접(actually)* 신청서 작성을 해야(opt back in) 합니다:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

여기서 우린 반드시 *unsafe impl* 이라고 떡하니 적어야 함을 주목하세요: 이놈들은 *안전하지 않은 트레이트(unsafe traits)* 입니다! (동시성 라이브러리들같이) 안전하지 않은 코드들은 오직 우리같이 선량한 사람들이 저 트레이트들을 한 치의 오차 없이 올바르게 찍어 발라놨을 거라는 굳건한 신뢰(rely on)만을 담보로 굴러갑니다! 실제 돌아가는 알맹이 코드가 1도 없기 때문에, 우리가 저 딱지를 붙임으로써 선언하는 보장성(guarantee)의 실체는 오직 "응, 우린 찐텐으로 안심하고 스레드 간에 쏴제끼고(Send) 돌려먹어도(Share) 되는 개안전한 놈들 맞당께요!" 라는 혓바닥 인증 선서뿐인 겁니다.

그러니까 행여라도 저런 딱지를 아무 데나 경박스럽게 허벌 창 내듯 찰싹찰싹 남발해대면 안 됩니다. 허나 전 엄연히 공인받은 프로페셔널(Certified Professional) 자격으로써 단언컨대: 옙 얘넨 암만 덕지덕지 발라도 개안전합니다 문제없어요. 여기서 IntoIter 놈한테는 굳이 Send 랑 Sync 를 떠먹여 주지 않았음을 주목하세요: 얜 뱃속에 고스란히 LinkedList 를 품고 있어서, 지 알아서 Send 랑 Sync 를 넙죽 자동 상속받아(auto-derives) 달아줍니다 &mdash; 거 보슈 진짜루 선택적 배제(opt out) 방식 맞다니까요! (참고로 여러분이 수동 배제(opt out)하고 싶으실 일은 없겠지만 할 때 쓰는 `impl !Send for MyType {}` 이딴 요단강 건넌 기괴한 문법 꼴아지를 보면 참으로 웃프기 짝이 없습니다.)

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

오케이 나이스!

...잠깐만요, 근데 저딴 딱지가 *붙으면 안 되는* 불순분자 새끼들한테까지 실수로 딱지가 붙어버리면 진짜 존나리 위험천만한 참사 아닙니까. 특히 저 IterMut 놈은 *절대로(definitely)* 공변성(covariant)의 탈을 쓰면 안 됩니다, 왜냐하면 걘 곧 걸어 다니는 짝퉁 `&mut T` 나 다름없는 놈이거든요. 아니 근데 그걸 대체 우리가 어떻게 검증하죠?

마법(Magic)으로요! 아니 사실대로 말하자면, rustdoc 나부랭이로요! 오케이 까놓고 말해서 굳이 rustdoc 안 쓰고 다른 방법 써도 되긴 하는데, 이게 제일 웃기고 골때리는 접근법이거든요. 보쇼, 만약에 문서 주석(doccomment)을 끄적거리고 그 안에 코드 블록 쪼가리를 우겨 넣으면, 이 멍청한 rustdoc은 그거 한 번 직접 컴파일해서 굴려보겠답시고 낑낑댑니다. 우리는 그 점을 악용해서 메인 본진 소스코드랑은 하등 상관없는 독립된 익명 "격리 프로그램(programs)"을 하나 뚝딱 밀반입 시킬 수 있는 겁니다:


```rust ,ignore
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```

오케이 지린다, 저놈이 무공변(invariant) 맞다는 걸 입증해 냈네요! 근데 씁, 이제 우리 테스트가 박살 났잖아요. 걱정 마쇼, 위대한 rustdoc 님께선 코드 울타리 펜스(fence) 꼬투리에다가 이건 원래 대가리 깨져서 터지는 게 정상이라는 `compile_fail` 주문을 달아놓는 걸 허락해 주시니까요!

(사실 냉정하게 따지고 보면 우리가 증명한 건 단지 얘가 "공변이 아니다(not covariant)" 라는 사실 하나뿐이긴 한데, 솔직히 사람이 자기 능지만으로 타입을 "실수로 사고 쳐서 부적절하게 반공변성(contravariant) 덩어리로 변이시키는" 경지에 도달했다면, 어... 대단하네요 박수 쳐 드려야죠 짝짝짝?)

```rust ,ignore
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

예이! 저는 무조건 처음엔 `compile_fail` 주문 없이 쌩으로 테스트를 한번 돌려보길 강력 권장하는 바입니다. 그래야 저 테스트 코드가 진짜루 내가 원했던 *바로 그 빌어먹을 이유 때문에(for the right reason)* 처발리고 장렬히 전사하는 게 맞는지 똑똑히 육안으로 확인할 수 있으니까요. 예를 하나 들자면, 저 위 예제 코드는 만약 실수로 `use` 문장 선언하는 걸 빼먹었을 경우에도 여지없이 장렬히 컴파일 에러를 뿜으며 자폭해 버리는데(그래서 결과적으로 `compile_fail` 버프 받고 통과처리 됨), 그건 우리가 애초에 간절히 바라던 찐 에러 원인이 아니란 말이죠! 뭐 개념적으로 따져보자면 컴파일러한테서 내가 바라는 "바로 그 특정 에러 구문" 외에는 안 받겠노라고 일일이 사전에 다 지령을 내려두고 깐깐하게 구는(require) 기능이 묘하게 구미가 당길 수도 있겠지만, 만일 그딴 기능을 진짜 도입했다간 차후 컴파일러가 에러 메시지를 좀 더 스무스하고 예쁘게 개선하고자 마음만 먹었다 하면 전 세계 모든 테스트 명세가 죄다 피바람 불며 동시다발적으로 깨져나가는(breaking change) 끔찍한 대학살 지옥도 나이트메어(nightmare)가 펼쳐지고야 말 겁니다. 우리는 당연히 컴파일러 님이 하루하루 리즈 갱신하며 에러 메시지를 예쁘장하게 뽑아주길(get better) 염원하기 때문에, 네 안 돼요 저딴 기능 함부로 바라시면 안 됩니다.

(아, 깜빡할 뻔했네, 사실 컴파일러가 내뱉는 수많은 에러 메들리 중에 우리가 정확히 어떤 에러 번호표 코드를 기다리고 있는지 저 `compile_fail` 옆 꼬투리에 슬쩍 꼽사리 껴서 명시할 순 있습니다. **근데 명심하쇼 이건 오직 nightly 빌드 뒷골목에서만 밀거래되는 위험한 꼼수 스킬이고, 아까 위에서 구구절절 피토하며 설명한 그 끔찍한 나이트메어 이유들 땜에 저런 불안정한 기능에 함부로 목숨 구걸하며 의존(rely on)하는 건 진짜 존나 병신 같은 오판(bad idea)입니다. nightly가 아니라면 그냥 개무시당하고 씹힙니다(silently ignored).**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...아 참, 그리고 여러분 방금 우리가 IterMut 놈을 스을쩍 무공변(invariant) 기형아로 개조시켰던 장면, 혹시 눈치채셨습니까? 제가 무슨 밑장빼기 타짜마냥 그냥 Iter 덩어리 째로 복붙해서 막판 말미 맨바닥에 휙 뭉뚱그려(dumped) 던져뒀기 때문에 아주 십중팔구 못 보고 넘어가기 딱 좋았을 겁니다. 바로 여기 맨 밑바닥 막줄 말이에요:

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

어디 그럼 한번 저 불순한 PhantomData 부적을 비틀어 빼버려 볼까요:

```text
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

하! 컴파일러 님이 우리 뒤통수를 든든하게 지켜봐 주시면서, 우리가 건방지게 수명(lifetime) 매개변수를 선언해 놓고 고의로 *유기 방치(not use)* 하는 꼴을 눈뜨고 못 보시겠다며 단칼에 철퇴를 내리치셨군요. 그럼 이번엔 어설프게 한눈팔고 *엉뚱한(wrong)* 예제 부적을 붙여놔 볼까요:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

빌드가 뚫렸습니다! 과연 우리의 든든한 테스트 나부랭이 새끼들이 이 참극 사태를 무사히 잡아챌 수 있을까요?

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

오예에이!!! 시스템이 미친 듯이 제 몫을 다하며 굴러갑니다! 이래서 제가 제구실 똑바로 하는 테스트 새끼들을 주렁주렁 달고 다니는 걸 끔찍이 사랑합니다, 그래야 다가올 파멸적인 실수 재앙들 앞에서 그나마 덜 벌벌 떨며 평온하게(horrified) 베개 베고 꿀잠 잘 수 있거든요!
