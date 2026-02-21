# 살 붙이기 (Building Up)

좋습니다, 그럼 일단 리스트 껍데기(building the list)부터 뼈대를 세우며 시작해 보죠. 이 망할 새로운 구조 모델 체계하에서도 `new` 메서드 덩어리 따위 굽는 거야 여전히 너무 허접하리만치 무식하게 기초적(pretty straight-forward)입니다. 그냥 무지성으로 전 필드 구멍이란 구멍에 모조리 None 이빨만 채워 넣으면 끝이니까요. 
덤으로 이 괴상망측해진 Node 노드 녀석 하나 소환해 연성해 내는 타자 구문 길이 자체가 꽤나 길고 살짝 지저분하게 거추장스러워(a bit unwieldy)졌으니, 이참에 아예 Node 전용 생성을 위한 커스텀 기계(Node constructor) 장치도 하나 옆에 슬쩍 조립해 끼워 넣어 분리시켜 둡시다:

```rust ,ignore
impl<T> Node<T> {
    fn new(elem: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            elem: elem,
            prev: None,
            next: None,
        }))
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

```text
> cargo build

**A BUNCH OF DEAD CODE WARNINGS BUT IT BUILT**
**안 쓰는 코드라고 무더기로 발작경고 뜨긴 했지만 어쨌든 빌드 통과 성공**
```

야호!

자 그럼 이번엔 리스트 주둥이 제일 앞단(front) 머리에다가 원소 하나 꾹 눌러 박기(pushing)를 시도해 봅시다. 애초에 양방향-연결 리스트(doubly-linked lists) 이녀석은 시스템 구조 자체가 징그럽게 복잡 다분하기 때문에, 우린 예전보다 훨씬 더 피를 토하는 막노동(fair bit more work)을 불사해야 할 겁니다. 옛날 단방향 리스트 시절엔 연산들이 거의 딸깍 한 줄(an easy one-liner) 복사붙여넣기면 다 마법처럼 통쳤었는데 반해, 양방향 녀석의 연산 설계 공정은 꽤나 머리통 쥐어짜는 복잡도(fairly complicated)를 과시합니다.

무엇보다 당장 이제 우린 특히 리스트 바구니가 완전히 텅 비어버린(empty lists) 극단적 변두리 경계(boundary cases) 시나리오 상황을 몹시 아주 특별히 배려 처리해 주어야 하는 번거로움에 봉착했습니다. 보통 어지간한 동작들은 기껏해야 `head` 머리 선이나 혹은 `tail` 꼬리 구멍선 둘 중 어느 한쪽 구역만 만졌을 뿐이지만, 이 리스트가 텅 빈 상태(empty list)를 오갈 때(transitioning to or from)만큼은, 양쪽 끝단 *두 짝(both)* 모두를 동시에 수정(edit)시켜 비틀어 줘야만 사달이 안 나기 때문입니다.

때문에 이런 지독한 설계 구상에서 우리 코드 조립이 과연 말이 되게(make sense) 짜였는지 안도하며 신뢰 검증해 볼 수 있는 아주 꿀팁 같은 관문 기준(invariant) 법칙이 하나 있는데 그건 이렇습니다: 세상의 모든 유효한 중간계 노드 피사체 녀석들은 반드시 자신의 주변에 나와 연결된 외부 포인터(pointers to it)를 2개씩 쌍으로 균형 있게 가져야만 정상이라는 점입니다.
리스트 중간에 처박힌 노드 녀석들은 앞선임자(predecessor)와 뒤후임자(successor) 양쪽 모두에게 찔려(pointed at by) 들어집니다. 반면 양옆 끄트머리에 매달린 끝단 노드 녀석들(nodes on the ends)은 오직 리스트 구조체 본체 자신만이 그들을 짚어 가리켜 보호해 줘야만(pointed to by the list itself) 등식이 타당해집니다.

어디 그러면 거하게 한번 부딪치고 깨져볼까요(take a crack at it):

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    // 새 노드는 +2개의 링크가 필요하며, 나머지 기존 녀석은 전부 +0 상태여야 합니다.
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            // 안 빈(non-empty) 리스트 상태이군요, 새로 온 녀석이랑 옛 머리(old_head)를 이어 붙입니다
            old_head.prev = Some(new_head.clone()); // +1 new_head
            new_head.next = Some(old_head);         // +1 old_head
            self.head = Some(new_head);             // +1 new_head, -1 old_head
            // 토탈 손익계산: +2 new_head 획득, +0 old_head 등락 없음 -- 완벽해, 패스(OK)!
        }
        None => {
            // 빈 리스트(empty list)이군요, 이참에 꼬리(tail) 쪽에도 세팅해 둡니다
            self.tail = Some(new_head.clone());     // +1 new_head
            self.head = Some(new_head);             // +1 new_head
            // 토탈 손익계산: +2 new_head 획득 -- 완벽해, 패스(OK)!
        }
    }
}
```

```text
cargo build

error[E0609]: no field `prev` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:39:26
   |
39 |                 old_head.prev = Some(new_head.clone()); // +1 new_head
   |                          ^^^^ unknown field

error[E0609]: no field `next` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:40:26
   |
40 |                 new_head.next = Some(old_head);         // +1 old_head
   |                          ^^^^ unknown field
```

아주 좋소. 시작하자마자 컴파일러 에러 알림. 훌륭한 출발이네요(Good start. Good start).

대체 왜 녀석들 필드에 있는 `prev` 랑 `next` 에 접근할 권한이 박탈당한(can't access) 걸까요? 분명 예전 `Rc<Node>` 구닥다리 장비일 때만 해도 아주 속 편히 뚫렸는데 말이죠(worked before). 합리적인 의심을 해보자면 빌어먹을 이식한 `RefCell` 포장지가 그 가운데 허리 사이를 딱 막고(getting in the way) 시야를 통제하는 게 원인 같습니다.

얌전히 공식 문서(docs)나 죽을 답을 구걸해봐야 할 소지가 다분합니다.

*구글 검색어창 타이핑: "rust refcell"*

*[첫 번째 링크 클릭](https://doc.rust-lang.org/std/cell/struct.RefCell.html)*

> 차용 규칙(borrow rules) 검사를 런타임(dynamically) 동적 단계에서 수행하는 가변 메모리 로케이션
>
> 상세 내용은 [모듈-레벨 공식 문서](https://doc.rust-lang.org/std/cell/index.html) 참조.

*링크 누름*

> 공유 자원 가변성이 탑재된 컨테이너 (Shareable mutable containers).
>
> `Cell<T>` 와 `RefCell<T>` 내부에 담긴 값(Values)은 공유 참조자(`&T`)를 통해서도 마음껏 변경(mutated)이 가능합니다. 이는 대부분의 일반적인 Rust 타입들이 오직 배타적인 가변 참조(`&mut T`)를 통해서만 변경을 허용하는 것과 매우 대조적(in contrast with)입니다. 우리는 이러한 셀 타입들의 특징을 가리켜 '내부 가변성(interior mutability)'을 가졌다고 부르며, 그 외 일반적인 타입들을 묶어 '계승된 가변성(inherited mutability)'을 따른다고 부릅니다.
>
> 셀 타입의 구현체로는 크게 `Cell<T>` 와 `RefCell<T>` 두 가지 종류(flavors)가 있습니다. `Cell<T>`의 경우 빠르고 간편하게 내부의 값을 덮어쓸(change the interior value) 수 있지만, 오직 `Copy` 트레이트를 구현한 타입에만 한정적으로 쓸(only compatible with types that implement Copy) 수 있습니다. 그 외의 나머지 타입들(other types)을 다루려 할 때는 반드시 `RefCell<T>`를 사용해야 하며, 이 구조체는 항상 변경(mutating) 작업 전에 먼저 동적으로 가변 대여 락(write lock)을 점유(acquiring a write lock)해야만 합니다.
>
> `RefCell<T>`는 언어 차원의 정적인 수명 주기(lifetimes) 검사 규칙을 런타임에 동적으로 수행하는 동적 차용술(dynamic borrowing) 기법을 사용합니다. 이를 통해 여러분은 내부 값에 한시적인(temporary) 가변 접근(mutable access)을 수행할 수 있습니다. 본래 Rust의 표준 차용 검사는 소스 코드가 컴파일될 무렵 철저하게 정적으로 추적(tracked statically)되지만, `RefCell<T>`의 차용 여부(Borrows for `RefCell<T>`s)는 프로그램이 동작하는 '런타임 실행 중(at runtime)'에만 감시되고 추적(tracked)됩니다. 이런 동적 감시 체계 때문에 만약 누군가 이미 `RefCell<T>` 내부의 값을 가변으로 차용(already mutably borrowed)하고 있는 도중에 또 다른 곳에서 중복 가변 차용을 시도(attempt to borrow)하게 되면 런타임 동적 오류를 발생시켜 스레드를 즉시 패닉(results in thread panic)시킵니다.
>
> # 그렇다면 내부 가변성은 언제 선택해야 하는가 (When to choose interior mutability)
>
> 보편적인 방식인 '계승된 가변성(inherited mutability)', 오직 가변 대여를 통해 독점된 소유권 하에서만 값을 변경하도록 엄격히 통제하는 이 모델은 Rust 초창기에 설계된 가장 핵심적인 언어 요소(one of the key language elements)입니다. 이는 파편화된 다중 포인터의 데이터 경합 현상(pointer aliasing)을 정적으로 원천 봉쇄(statically preventing)하여 런타임 크래쉬(crash bugs)를 강력하게 억제(reason strongly)하는 일등 공신입니다. 그 덕에 보통 계승된 가변성 모델이 항상 우선으로 권장(preferred)되며, 내부 가변성(interior mutability) 스킬은 어쩔 수 없이 사용해야 하는 최후의 보루(last resort)처럼 여겨지는 것이 이치에 맞습니다. 하지만, 때로는 이 편법 스킬이 아주 유용하게 쓰이거나 심지어 구조상 반드시 쓰일 수밖에 없는 숙명(must be used)인 영역도 존재합니다. 내부 가변성을 허용(enable mutation)해야만 설계가 온전히 굴러가는 전형적인 사례들은 다음과 같습니다:
>
> * 공유된 타입 안쪽에 슬쩍 가변의 뿌리(roots)를 밀반입해 넣기 (Introducing inherited mutability roots to shared types).
> * 표면적으론 불변(logically-immutable methods)이지만 실질 내부적으론 변경이 필요한 꼼수 뒷정리 로직(Implementation details).
> * 기본 `Clone` 구현의 규약을 깨고 내부적으로 훼손 변조(Mutating implementations of Clone)하는 시점.
>
> ## 공유된 타입 안쪽에 슬쩍 가변의 뿌리 들여오기 (Introducing inherited mutability roots to shared types)
>
> `Rc<T>` 나 `Arc<T>` 같은 공유 스마트 포인터 타입들(Shared smart pointer types)을 사용하면 데이터를 무수히 많은 패거리(mutiple parties)에 손쉽게 복제 배포(cloned and shared)할 수 있습니다. 그러나 이 스마트 포인터 내부의 캡슐 데이터(contained values)는 수많은 포인터의 중복 노출을 감안하여(multiply-aliased) 그 억압된 특성상 언제나 일반 공유 참조자로만 빌생(borrowed)되며 절대 가변형 참조 모드로 뚫려 나오지(not mutable references) 못합니다. 이 말은 곧 앞선 '셀(cells)' 장치들의 도움이 아예 없다면(Without cells), 저 튼튼한 공유 스마트 상자(shared boxes) 안의 데이터를 찢고 조작 가변(mutate data)하는 일 자체가 애초에 완전히 통제 절대 불가능(impossible at all!) 상태에 놓이게 된다는 걸 의미합니다.
>
> 따라서 우리는 `RefCell<T>` 트로이 목마를 덤으로 접합 기생시켜, 철갑 스마트 포인터 상자의 배를 가르고 무적 가변 파괴 투영권(reintroduce mutability)을 다시 살려내는 이종 교배 마법을 씁니다. 이런 식의 우회 설계 패턴은 아주 흔하게 쓰이는 교과서적인 통용 방책(very common)입니다:
>
> ```rust ,ignore
> use std::collections::HashMap;
> use std::cell::RefCell;
> use std::rc::Rc;
>
> fn main() {
>     let shared_map: Rc<RefCell<_>> = Rc::new(RefCell::new(HashMap::new()));
>     shared_map.borrow_mut().insert("africa", 92388);
>     shared_map.borrow_mut().insert("kyoto", 11837);
>     shared_map.borrow_mut().insert("piccadilly", 11826);
>     shared_map.borrow_mut().insert("marbles", 38);
> }
> ```
>
> 한 가지 유념하실 점은, 이 예제(this example) 코드는 다중 스레드 장갑차 `Arc<T>` 방패 대신 단일 스레드용 `Rc<T>` 녀석을 장착(Note that)했다는 점입니다. `RefCell<T>`는 기본적으로 한 개의 소박하고 통제된 단일 스레드 환경(for single-threaded scenarios)에만 알맞도록 만들어졌기 때문입니다. 진짜 무자비하게 난장판이 벌어지는 멀티 스레드 병렬 생태계(multi-threaded situation) 속에서 저런 스릴 넘치는 공유 가변 무대(shared mutability) 조작질이 필요하시다면 애초에 `Mutex<T>` 장비 사용을 먼저 고려(Consider using)하십시오.

이야, Rust 공식 가이드 문서 선생님들의 설명(docs) 클라스는 끝장나게 압권이네요 (continue to be incredibly awesome).

위 장문 수다 썰에서 정말 핵심 단물 고기 패티(meaty bit)라 꼽는 마법의 구문 라인은 이 한 줄이었습니다:

```rust ,ignore
shared_map.borrow_mut().insert("africa", 92388);
```

이 부분 중 굳이 하나 특정짓자면, 저 요망한 `borrow_mut` 함수 액션 짓거리 말이죠. 보아하니 RefCell 알맹이를 까잡수려면 아주 싹수없고 번거롭게도 매번 대놓고 우리가 내 손가락 명시적 콜 호출로(explicitly) `borrow_mut` 수동 버튼을 꾹 밟아 가변 대출 신청 차용 점거(borrow) 패스를 접수 인지시켜 주어야 루틴 기어를 돌려주는 방식인가 봅니다. 그저 만병통치 점 `.` 암묵적 연산자 툴키 요정 녀석팽이가 우릴 대신해 알아서 쓱싹 껍질 벗겨 요술 마법 부리듯 투명하게 떠먹여 주진(not going to do it for us) 못한다는 이 소리군요. 진짜 희한하네요 (Weird). 아무튼 대충 알겠으니 시도(try) 해봅시다:

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            old_head.borrow_mut().prev = Some(new_head.clone());
            new_head.borrow_mut().next = Some(old_head);
            self.head = Some(new_head);
        }
        None => {
            self.tail = Some(new_head.clone());
            self.head = Some(new_head);
        }
    }
}
```


```text
> cargo build

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

오, 빌드 성공(built)! 문서 선생님의 예지력이 또 이겼군요(Docs win again).
