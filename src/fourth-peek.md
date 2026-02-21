# 훔쳐보기 (Peeking)

좋습니다, 우린 그 살 떨리는 `push` 와 `pop` 고개를 무사히 넘겼습니다. 솔직히 빈말 못 하겠네요, 그 과정 내내 심장이 꽤나 울컥 철렁 내려앉으며 감정 기복 쫄깃한 순간(emotional)들이 많았습니다. 컴파일-타임에서 검증되는 오류 없는 완벽함(Compile-time correctness)의 안도감은 참 짜릿한 마약선이죠.

이제 머리도 식힐 겸 아주 단순 유치한 꿀잡거리 하나로 뇌 좀 쓰다듬어(cool off) 줍시다: 그냥 만만한 `peek_front` 훔쳐 읽기 도구 하나만 뚝딱 연성 구현해 보죠. 옛날엔 이건 눈 감고 발가락으로도 굴리던 진짜 숨 쉬기보다 허접한 제일 만만따리 기초 꿀(really easy) 파츠였잖습니까. 당연히 지금도 눈 감고 떡 먹기 수준이겠죠, 앙?

안 그래요? (Right?)

사실 전, 걍 속 편히 저번 번주꺼 슥 복붙(copy-paste) 해오면 치트 패스될 거란 킹리적 갓심에 확신 찹니다!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

잠깐. 안돼, 이번엔 안 먹혀.

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // 야 너 차용 대출 패스권 끊어야 통과시켜준대!!!! (BORROW!!!!)
        &node.borrow().elem
    })
}
```

에-라이 젠장-하. (HAH.)

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

좋아 알았어 걍 당장 홧김에 이 망할 고철 컴퓨터 박스 불살라 부숴 화형(burning my computer) 시켜버릴랍니다.

진짜 억장이 무너지네요, 이 로직은 백 번 양보해서 예전 우리가 돌리던 그 낡아빠진 옛 가변 단방향 스택(singly-linked stack) 녀석의 숨구멍 로직 코드랑 진짜 100% 한 치 오차도 없는 똑같은(exactly the same) 완벽한 거울 쌍둥이 데칼코마니 판박이잖아요. 근데 도대체 왜 똑같은 걸 줬는데 반응이 다른 딴소릴 씨부리는 거죠. 왜냐고 진짜. (WHY.)

그 기막힌 빡침의 해답이야말로 이번 장막 챕터 전체 에피소드를 꿰뚫는 가슴 저미고 피맺힌 잔혹 동화 명언 철칙 교훈(moral) 본질 그 자체입니다: RefCell이라는 이 더러운 요망한 그릇 기생 포장지는 들이대는 순간 만병 만물 세계선의 필드 모든 것들을 절망 수치 통곡의 슬픔 구렁텅이(sadness) 구덩이로 끌고 들어가 추락시켜 버립니다. 지금 방금 이 찰나 직전 전까지만 해도 거슬리고 귀찮긴 했어도 얼추 꾹 참고 달고 넘어갈 만한(nuisance) 그냥 징그럽고 띠꺼운 각질 거머리 신세 같은 것로 참고 견딜만한 존재였죠. 하지만 이제부턴 이야기가 좀 크게 다를 겁니다, 이녀석은 우리 수명 깎아 먹으며 진짜 피 말려 죽일 최악의 꿈틀대는 악몽 군단(nightmare) 재앙으로 흑화 돌변 각성할 테니까요.

그래서 대관절 방금 장막 뒤편에서 뭔 난리이 터진(what's going on) 겁니까? 그 피 말리는 해답 진상 조사를 분석 추적하려면 불가피하게도 저 문제의 원흉인 빌어먹을 암막 꼼수 메서드 도구 `borrow` 녀석 호적 명부 증명서 등본 정의 체계(definition) 선언부 구문 족보의 근간으로 다시금 뒤로 거꾸로 파고(go back to) 올라가야 합니다:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

우리가 저기 앞쪽 옛날 옛적 레이아웃(layout) 구상 초반 극초기에 이런 잡썰을 언급한 전례가 있었습니다:

> 다만 이 깐깐한 심사가 정적 컴파일(statically) 검사망에서 미리 차단당하는 게 아니라, RefCell 녀석은 이 규율 위반 검열을 프로그램 런타임(runtime) 실행 직전 단계로 미뤄버립니다. 만약 당신이 겁도 없이 이 철칙을 위반해 이중 대출 짓거리를 벌이다 적발(break the rules)되는 날엔, 의례적인 RefCell 판사는 가차 없이 냅다 패닉(panic) 철퇴를 날려 모가지를 처형(crash)시켜 뻗게 만들어 버립니다. 
> 근데 이녀석은 대체 왜 쓸데없이 저런 기괴한 `Ref` 나 `RefMut` 따위의 껍데기 포장지(things)들을 토해내는 걸까요? 뭐, 까놓고 말하면 녀석들의 동작 습성은 대략 `Rc` 류의 수명 감시망 역할과 비스름한데 이건 대여 빌림용(for borrowing)이란 것만 다를 뿐입니다. 심지어 감시병인 이녀석이 스코프 밖으로 벗어나 죽어 없어지기(out of scope) 전까진 RefCell 본체 대여 권한 자물쇠를 지독하게 잡아끌어 감시 유지(keep borrowed)시켜주는 족쇄 역할까지 해냅니다. **어차피 이 골칫거리는 좀 이따 뒤에서 나중에 상대할 테니(We'll get to that later)** 넘어갑시다.

그 '좀 이따 나중(later)' 구역이 드디어 도래했습니다.

저 `Ref` 와 `RefMut`는 각각 `Deref` 와 `DerefMut` 트레이트를 구현(implement respectively)한 편리한 포장지들입니다. 따라서 대부분의 상황에서(for most intents and purposes) 이 녀석들은 완벽하게 일반적인 `&T` 와 `&mut T` 참조자처럼 행동합니다. 하지만 치명적인 차이점이 하나 있습니다. 이들이 반환하는 실제 알맹이 참초자(reference)의 수명(lifetime)은 본래의 `RefCell`과 연결되어 있는(connected to) 것이 아니라, 오직 `Ref` 포장지 자체의 수명(lifetime of the Ref)에만 한정되어 있다는(connected) 것입니다. 이것이 의미하는 바는, 우리가 저 참조자를 계속 붙잡고 유지(keep the reference around)하고 싶다면, 반드시 그 껍데기 포장지인 `Ref` 녀석도 생명이 다할 때까지 옆에 끼고 계속 함께 살려 둬야 한다는 뜻입니다.

사실 이는 올바른 작동(correctness)을 유지하기 위해 반드시 필요한(necessary) 조치입니다. `Ref` 포장지가 그 수명을 다해 파괴(dropped)되는 바로 그 순간, 이 녀석은 재빨리 원본 `RefCell`에게 "이제 대여가 끝났습니다요(tells the RefCell that it's not borrowed anymore)"라고 조용히 신호를 보내주기 때문입니다. 만약 우리가 어떻게든(managed to) 진짜 참조자는 손에 쥔 채로 은밀하게 `Ref`만 먼저 파괴시킬 수(Ref existed) 있었다면 어떻게 될까요? 우리는 여전히 그 무단 참조자를 자유자재로 굴려 먹을 텐데(kicking around), 시스템은 대여가 끝난 줄 알고 또 다른 누군가에게 마음껏 `RefMut` 대여권을 남발해 줄 수 있게(could get a RefMut) 될 겁니다. 이런 사태가 발생하면 그 완벽하다 자부하던 Rust의 거룩한 타입 시스템(Rust's type system)은 곧장 두 동강이 나버릴 겁니다(totally break in half).

자, 그럼 이제 우린 여기서 어떡해야 할까요(where does that leave us?) 우리는 단지 단순하게 내부의 참조자 하나만 깔끔하게 반환(only want to return a reference)받고 싶었을 뿐인데, 시스템은 우리에게 무거운 `Ref` 껍데기까지 통째로 책임지고 짊어지라며(keep this Ref thing around) 억지를 부립니다. 설상가상으로 우리의 `peek` 함수는 참조자를 반환(return the reference)하는 그 즉시 수명이 끝나버리도록(the function is over) 설계되어 있기 때문에, `Ref` 껍데기 역시 지역 변수로서 어김없이 수명이 다해 소멸(goes out of scope)될 운명입니다.

😖 (아이 씨바 아찔하네 으아악)

적어도 여태껏 제 한심한 고도성장 통찰력과 지식 밑천의 견문 옹졸한 수용량 데이터 구도 체제 통계 지식 시야 범위 추정치 기준(As far as I know) 바닥 도구망 한계 상으로는, 우린 젠장 지금 이 망각 늪지대 교착 웅덩이 암초 구역 한가운데서 도저히 배를 띄우고 나아갈 생존 확률이 완전 무동력 100% 교착 침몰 포위 오도 가도 고립무원 사각 영구 스탑 동결 사망 완전 정지(totally dead in the water) 뻗은 죽을 개노답 봉쇄 상태인 게 빼박 맞습니다. 이딴 구제 불능의 골칫덩이 RefCell 따위의 독극물 파편 형편없는 것 치명 오물 도구 장치 기믹들을 억지로 외부 시선으로부터 감쪽같이 완벽 위장 밀폐 철갑 차단 은폐 속임수(totally encapsulate) 장막 세탁 필터 캡슐 포장 차단막 밀봉 통제 덮기 기만 봉수 위장질로 깔끔 무결 은폐 축소 방관 통제하는 꼼수 야매 세탁 공작은 원천 봉쇄 1도 어림 반 푼어치 원천 불가(can't) 개씹불가인 게 팩트입니다.

하지만... 하지만 말입니다요 우리가 만약(what if) 애시당초 그 알량 자존심 개나 주고 처음부터 그 알맹이 깊숙한 심부 구멍의 내부 치부 속살 원장 정보 구현 부품 부속(implementation details) 구석 체계의 치부 노출 감춤 차단 은닉 밀폐 막음 세탁 은폐 기만 캡슐화 포장 공작에 대한 아집 환상 독선 목표 희망 수호 방어 고집 사수 미련 방벽 따위를 그냥 모조리 처절하게 시원히 깔끔 탈탈 철회 단념 백기 투항 상실 체념 굴복 내던져(give up on) 버린다면 어찌 될까요?
그러니까 다 까놓고 그냥 저 망할 껍데기 영수증 덩어리 증서 부적 `Refs` 자체를 냅다 통째로 토해 역류 반환 반출 배출(returns) 선물 세트로 던져 줘버리면(What if) 어떨까요?

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

뭭. (Blurp.) 또 저런 거지 같은 형편없는 것 부속 따위 좀비 쪼가리들을 억지로 억까 불법 반입 import 소환 밀입국 등재 명부(Gotta import some stuff) 신고 등록 이식 수입 조달 청구 절차 치러야 귀찮게 구는군요.


```rust ,ignore
use std::cell::{Ref, RefCell};
```


```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

흠... (Hmm...) 맞네요 니미랄 한 번에 뚫어주면 어디 덧나나 (that's right). 우리가 지금 삽질해 겨우 건져내 쥐고 있는 건 `Ref<Node<T>>`이고, 고객들이 애타게 원하는 건 오직 순결한 알맹이 `Ref<T>`인데 상황이 참 엇나갑니다. 사실 여기서 우리가 캡슐화 포장(encapsulation) 따위에 대한 미련과 고집을 완전히 버리고 포기(abandon all hope)한다면, 그냥 속 편하게 저 `Ref<Node<T>>` 자체를 반환해 버리는 것(just return that)도 한 가지 방법입니다. 반대로 더 심오하게 접근해서, 저 `Ref<Node<T>>`를 내부적으로 완전히 숨긴 채 새로운 가짜 타입 껍데기로 빙빙 감싸 포장해 버리고(wrap) 세상 밖에는 그저 순수한 `&T` 참조자만 보이게(expose access) 속이는, 상황을 더 복잡하게 만드는(make things even more complicated) 길도 있습니다.

근데 솔직히 저 두 가지 선택지 모두 *뭔가 좀(kinda)* 아쉽고 지질한(lame) 방식이네요.

그래서 그런 선택지 대신(Instead), 이번엔 좀 더 깊은 시스템 골짜기 심연(go deeper down)으로 잠수해 보는 신선하고 파멸적인 꿀잼 파티(fun)를 벌여봅시다. 이 장난질의 원천(source of fun)이 되어줄, 깊은 바닥에 웅크려 도사리는 끔찍하고 기괴한 비밀 장치 괴수(this beast)는 바로 이것입니다:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> 이미 차용된 데이터(borrowed data) 구성원(component) 중 일부에 대한 새로운 `Ref` 참조자를 생성합니다. (Make a new Ref for)

네 (Yes), 바로 이겁니다: 여러분이 그동안 `Option`을 다루면서 심심하면 써먹었던(map over an Option) 마법의 변환 주문, `map` 함수와 완벽하게 동일한 메커니즘을 `Ref` 포장지 위에서도 그대로 시전(map over a Ref)할 수 있다는 쇼킹한 사실이죠!

장담하건대(I'm sure) 진짜 어디선가 누군가(someone somewhere)는 이걸 보고 흥분에 휩싸여 "우와, 이게 바로 그 전설의 *모나드(monads)*구나!" 라며 온갖 학술적 용어로 지껄이며 열광하고(is really excited because) 있을 겁니다. 하지만 전 수학의 철학적 개념 같은 건 하나도 관심 없고 신경 쓰지 않으니(don't care about any of that) 그냥 넘어가겠습니다. (애초에 제대로 된 모나드(a proper monad) 구조상 `None` 같은 예외 처리가 부재(no None-like case)하는 이걸 모나드라고 부를 수 있을지도 의문(don't think)이긴 합니다. 뭐 제가 주제를 벗어났네요(but I digress).)

그냥 이건 엄청 멋집니다. (It's cool.) 적어도 저에겐 그것만이 유일하게 중요한 사실(that's all that matters to me)이니까요. *난 지금 당장 이게 너무 필요합니다. (I need this).*

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

오우예에에에에 야아아아스으으으 삐이익 (Awww yissss)

제대로 작동하는지 예전 스택 버전에서 가져온 코드를 적당히 수정하고 다듬어서(munging) 테스트해 봅시다. 이런 수고스러운 수정 작업이 필요한(We need to do some munging) 이유는, 우리가 무심코 반환하는 저 `Ref` 껍데기 타입 자체에는 동등성 비교 연산(implement comparisons) 기능이 애초에 지원되지 않기(don't) 때문에 이를 우회 처리(to deal with the fact)해야 하기 때문입니다.


```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

개조아 찢어버려따 젠장!!! (Great!)
