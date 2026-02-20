# 뽑기 (Pop)

`push`와 마찬가지로 `pop` 역시 리스트를 가변 상태로 변경하길 뜻합니다. 하지만 `push`와 큰 차이가 있다면, 이번에는 우리가 모종의 *결과물*을 내보내(return) 원상태로 반환받기를 원한다는 점입니다. 하지만 이 단순한 `pop`에게는 한 가지 까다롭고 골치 아픈 코너 케이스가 도사리고 있습니다: 만약 리스트가 완전히 텅 비어있다면 도대체 우리 코드는 무얼 어떻게 처리해야 할까요? 이런 위태로운 상황을 우아하게 대변하기 위해 우리는 아주 믿음직스러운 단골손님 `Option` 타입을 전격 채용하기로 합니다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>`는 해당 값이 반드시 존재할 수도, 아닐 수도 있다는 불확실한 이중성을 명쾌하게 나타내는 열거형(enum)입니다. 이는 오로지 `Some(T)` 혹은 `None`의 둘 중 하나의 극단적인 상태만을 지닐 수 있습니다. 물론 우리가 이전에 Link 시절에 열과 성을 다했던 것처럼 아예 우리만의 기상천외한 열거형을 새로 선언해서 쓸 수도 있겠지만, 결국 우리 라이브러리를 쓰는 수많은 유저들이 대체 이 결과물의 정체가 무엇인지 직관적으로 쉽게 이해할 수 있기를 원합니다. 그 점에 돌이켜 보건대 Option은 워낙 흔해 빠지고 도처에 스탠다드로 깔린 범용적인 존재라 이 세계 *누구든지* 이것을 안방 보듯 훤히 깨차고 있습니다. 사실상 이건 너무 파멸적으로 기초적인 녀석이라 모든 파일들의 스코프의 빈틈 사이사이에 암묵적으로 스스로를 임포트(imported)시켜 두었을 지경이며 심지어 이의 양팔이 되어주는 강력한 자식 모듈들인 `Some`과 `None` 마저도 동등하게 빈공간 내에 퍼져있습니다 (덕분에 굳이 `Option::None`이라고 꼬치꼬치 따져 묻지 않아도 컴파일러가 알아서 번역 다 해줍니다).

`Option<T>`에 달린 저 뾰족한 괄호 조각들은 Option이라는 것이 사실은 엄청난 범용성을 띈 제네릭(generic) 기술을 통해 확장적으로 구현되어 있다는 것을 암시합니다. 저 T 자리에 우주상 그 *어떤(any)* 타입이든 입맛대로 밀어 넣어서 여러분만의 맞춤형 Option을 무한대 찍어낼 수 있다는 뜻이죠!

자 그러니 어.. 우리 앞엔 지금 아까 그 `Link` 녀석이 놓여있습니다. 이것의 내부 머리가 텅 빈 Empty 상태인지 아니면 계속 뒤로 줄지어 이어지는 More 상태 끈을 가졌는지 대체 어떻게 알아낼 수 있을까요? 그것은 바로 `match`를 통한 패턴 매칭(Pattern matching)입니다!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

우웁스, `pop`은 반드시 하나의 명확한 결과물을 도출해서 반환(return)해야만 하도록 우리가 약속했는데, 정작 본문에서는 우린 아직 아무 짓도 안 하고 있네요. 우린 여기서 그냥 이빨 꽉 깨물고 빈 `None`을 내뱉도록 할 *수도* 있겠지만, 지금 당장 이런 극초반 빌드업 상황에선 이것보다 그냥 `unimplemented!()` (아직 구현되지 않았음!) 매크로를 뿌려대서 우리가 이 함수 구현을 아직 미처 끝마치지 못했다는 더 크고 명확한 경고 시그널을 던져주는 편이 훨씬 나은 길잡이 선택일 겁니다.
`unimplemented!()`는 매크로(`!` 기호가 매크로 구문임을 암시합니다)로서 프로그램 실행 흐름이 이 지뢰를 밟는 순간 프로그램 전체에 거대한 고의적 패닉을 내뿜게 만들어버립니다 (\~통제 가능한 방식으로 시스템을 깔끔하게 추락시켜버립니다).

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

이렇게 무조건적으로 고의적인 패닉을 던져버리는 형태는 영원히 발산하는 함수(Diverging functions) 메커니즘을 따르는 대표적인 [예시][diverging] 중 하나입니다. 발산하는 함수 무리들은 최초의 호출자(caller) 위치로 생전에 결단코 두 번 영원히 회귀하지 않기 때문에, 어떠한 타입의 결과를 내어놓든 상관없이 최초에 기대되던 반환 값의 빈 공간을 무엇으로든 위장하여 은폐시킬 수 있는 뚫린 구멍에 마음대로 들이부어질 수 있습니다. 여기서는 `unimplemented!()`가 명백히 `Option<T>` 타입의 반환값을 뱉어내야 할 장소 공백에 떡하니 대못처럼 박혀 영구 대체되고 있습니다.

무심코 놓치기 쉬운 통찰이지만, 한 가지 더 여러분들이 눈여겨볼 점은 여태껏 우리 프로그램 코드상에 `return` 글씨를 단 한 차례도 안 썼다는 겁니다. 함수의 가장 마지막 줄 표현식(expression) 나부랭이는 모두 암묵적으로 해당 함수의 진짜 반환 값이 됩니다. 이 간결한 문법이 조금 더 칙칙한 코드들을 훨씬 콤팩트하고 무결하게 줄여 표현해 낼 수 있게 도와줍니다. 물론 타 C 언어 기반 친구들처럼 `return`을 명시해서 중간에 황급히 고향으로 돌아가는 조짐의 조기 반환 룰 또한 무지성으로 완전히 다 지원합니다.

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

야 Rust 참나 거, 좀 등 뒤에서 떨어져 줄래! 평소처럼 우리의 Rust 감찰관 컴파일러는 우리한테 사납게 분노를 노출하며 비명을 삼키고 있습니다. 참 엎드려 절할 정도로 다행스럽게도, 이번엔 분노 파동과 함께 엄청나게 친절하고 상세한 길잡이 힌트 풀이까지 한 방에 다 떠먹여 주었네요! 기본적으로 패턴 매칭 로직 덩어리는 자신이 한창 매칭한 분기 대상 요소의 알맹이를 그대로 통째로 뽑아들어서 새로운 하위 내부 분기점(branch) 안쪽 깊숙이 그대로 값으로 이동시켜(move) 버리려고 맹렬히 시도합니다. 그러나 다시 냉정하게 따져봐도 우린 여기서 `self` 요소를 소유권으로 확보해 값으로 통째 소유(own)한 적이 애초에 단 한 차례도 없기 때문에 당연하게도 이 짓거리를 마저 완수해낼 능력치조차 가지지 못합니다. 

```text
help: consider borrowing here: `&self.head`
(도움말: 여기에서 `&self.head` 형태로 참조 대여를 해보는 걸 한 번 고려해 보세요)
```

Rust가 귓등으로 속삭여준 친절한 가이드대로 우리가 짠 저 허술한 `match` 선언 구문부에 슬며시 참조자(reference) 하나만 몰래 얹어서 고쳐보도록 합시다. 🤷‍♀️ 자, 한 번 시도해 보겠습니다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

야호, 빌드가 다시 살아움직이고 컴파일이 돌기 시작했습니다! 이제 좀 더 깊이 머리를 굴려서 세부적인 논리 흐름을 스케치 볼 시간입니다. 우리는 무조건 결과로써 Option 덩어리를 하나 쪼물딱 만들어 도출해 내야 하니까, 일찍이 반환할 징검다리 목적지의 대기열 변수를 하나 빈 공간에 예쁘게 선언시켜 줍시다. 헤드가 완벽히 빈(Empty) 상태일 경우의 절망적인 시나리오라면 우린 그저 조용히 None을 되돌려서 끝맺으려야만 합니다. 그러나 무언가 잔뜩 연결된 무궁무진한 구간(More) 상태라면, 우리는 그 안에 소중히 들어있는 `Some(i32)` 내용물을 먼저 조심스럽게 꺼내되 반환용으로 고이 싸매주고 난 뒤, 이 리스트가 더 이어나갈 수 있도록 남은 꼬리를 한 칸 앞당겨서 리스트의 차기 헤드머리(head)로 마저 자리바꿈 시켜버려야만 합니다. 흠, 딱 이 수준 정도의 논리면 다 무사 통과될 것 같지 않나요?

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*머리를*

*책상에 거세게 후려칩니다.*

우리는 고작해야 우리가 굽실거리면서 빌려온 초라한 읽기 전용의 찌꺼기인 공유 참조자(shared reference)만을 달랑 가지고 있으면서 하늘 무서운 줄도 모르고 `node` 알맹이에 서린 주도권을 송두리째 강탈해서 멋대로 내용물을 빼돌리려 유출 시도를 하려는 엄청난 만용을 부리고 있는 중이었습니다.

지금 이 순간, 우리는 번뇌하는 키보드를 잠시 놓고 아주 잠시 거리를 한 발짝 물러서서 근본적으로 도대체 우리가 성취하려던 본래의 목적 로직이 진실로 무얼 하고 싶었는지 냉철하게 원인을 직시할 필요가 있겠습니다. 우리는 지금:

* 리스트가 완전히 텅텅 비어 메마른 공허한 상태인지 체크하길 소망합니다.
* 만약 비어있다면, 아주 가볍고 유쾌한 마음으로 텅 빈 None을 살며시 던져주고 탈출합니다.
* 하지만 만약 이것이 풍성하게 비어있지 *않다면*
    * 리스트의 제일 앞단 머리통(head)을 무시무시하게 적출 분리시킵니다.
    * 거기 쑤셔 보관돼 들어있던 노드 내부 깊숙한 지점의 실제 나만의 알짜배기 알맹이 요솟값(`elem`)을 빼냅니다.
    * 이미 가치가 단물 빠져 더 이상 쓸모가 없어진 뽑혀 나온 옛 머리의 다음 후계자 꼬리(`next`) 녀석 자리를, 리스트를 관철하는 어여쁜 새 리스트 머리통(head)으로 통째로 자리를 교체해 메꿔 줍니다.
    * 마침내 수백 번 고배를 마신 끝에 찾아낸 전설적인 알맹이를 따뜻하게 `Some(elem)` 캡슐로 단단히 포장하여 세상 밖으로 반환시킵니다.

위의 내용 속 가장 숨 막히는 인사이트는 여기서 우리가 이 요소들을 바깥 구멍으로 그 모양 그대로를 유지시켜 *유출 제거(remove)*하여 가져오길 몹시 원하고, 갈망하고 있으며, 이 말인즉 리스트에서 오직 헤드 요소를 찌질한 참조가 아니라 오로지 직빵 통짜 *값(by value)*으로 거칠게 탈취 취득하고 싶다는 탐욕의 의미가 내재되어 있다는 사실을 깨닫는 것입니다. 자명하게도, 우리가 얌전히 `&self.head`를 통해 얌전히 찔끔찔끔 동냥받았던 보잘것없는 공유 참조자 정도의 싸구려 접근증으로는 도저히 절대 이 위대한 위업의 금자탑을 이뤄낼 수 없습니다. 
게다가 우리가 현재 시점에서 가진 리스트 전체 본체(`self`) 전신 구석구석에 뻗쳐있는 사용 권한 한도조차도 "겨우" 가변 참조자에 나약하게 불과하므로, 기기괴괴하게 내부 요소들을 박살 내가며 밖으로 뽑아 올릴(move) 수 있는 유일무이한 최후의 마술 트릭은 옛 현인들의 교훈처럼 위치를 눈 깜짝할 새 *바꿔치기(replace it)* 해버리는 닌자 비기뿐입니다. 야, 참 좋겠습니다. 다시 한번 아무 짝에도 쓸데없는 텅 빈 깡통 허수아비 춤을 덩실덩실 벌여봐야 할 때입니다!

한 번 과감하게 저질러 보죠:


```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

세 상 에 나

*단 하나*의 하찮은 트집 잡기용 경고조차 없이 완벽하고 아름답게 컴파일 기적을 이뤄냈습니다!!!!!

근데 약간 솔직히 양심 고백을 하자면 제가 직접 나만의 커스텀 자제력 린트(lint) 알림 경고를 하나 좀 울리고 싶은데: 우리가 이 훌륭하게 뽑아낸 값을 임시 대기열로 보관하려고 굳이 `result`라는 변수 껍데기까지 아주 요란하게 장전시켜 놓긴 했지만, 곰곰이 생각해보면 사실 이렇게까지 빙 구불구불 돌아가며 복잡하게 꼬아서 할 이유나 필요가 전혀 절대로 없었거든요! 
하나의 함수가 마지막 종착지점에 위치한 평가된 결과물 표현식을 남기고 장렬히 산화하며 값을 내뱉듯, 중괄호가 둘러쳐 구획 지어진 모든 생명체 블록 단위들 조차도 하나하나 전부 동일하게 스스로의 제일 마지막 막다른 골목 뒤 단의 표현식 찌꺼기 결과 값을 바깥세상으로 내뱉을 줄 아는 신비한 기능을 소유하고 있습니다. 그래서 지루한 우리 일상의 개발 환경상에선, 항상 블록 끄트머리 종단에 닫기 세미콜론(;)을 딱 못 박아 두는 암묵적 억제 행위로써 그 구획의 결과물이 속 알맹이가 뻥 뚫혀서 껍데기뿐인 빈 구조체 튜플 조각인 `()`을 내어놓고 소리 소문 없이 평가 종료되게끔 일관되게 묵살시켜 왔을 뿐이었습니다. 심지어 여러분들이 무심코 지나친 이 빈 껍데기 부속이야말로 결국 `push`와도 같은 반환값이 당최 애초부터 구비되지 않은 허당 함수 녀석들조차 실제로 남모르게 몰래 살짝쿵 던져두고 가는 진짜 은밀한 반환값의 실체였습니다.

그러니 아까 그 혼돈스럽게만 비치던 긴 `pop` 함수를 실마리를 풀어 단 한 번 이렇게도 극적으로 줄여 표현해 볼 수 있습니다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

이전에 비하면 구차함도 없고 한층 훨씬 막혀있던 속이 뻥 뚫릴 정도로 더 콤팩트하고 간결하며 idiomatic Rust(러스트다운 문법)의 아우라가 가감 없이 뿜어져 나오는 세련됨이 물씬 풍겨오지 않나요! 더구나 저기 매칭 구간의 Link::Empty 쪽 분기점은 단 하나밖에 남지 않은 고독한 표현식 조각만을 반환하는 상태로 간소화되었기에 굳이 자신을 요란하게 치장하여 답답하게 감싸 두던 낡은 옷 중괄호 블록들을 완전히 미련 없이 싹 다 벗어던졌습니다. 아주 간단한 베이직 상황들에 극히 절묘하게 어울리는 이쁜 속기법(shorthand) 테크닉입니다.

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

조오오오습니다, 여전히 이쁜 자태로 미끄러지듯 잘 동작하며 성능을 뽐내네요!



[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
