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
> `Cell<T>` 와 `RefCell<T>` 타입 수납장 안에 담긴 값(Values) 알맹이들은 모두 공유 참조자(through shared references - 즉 `&T`) 신분으로도 심지어 내부 조작 훼손(mutated)이 가능합니다; 반면 대다수 전형적인 Rust 타입류 녀석을 건드려 변형시키려면, 오직 오리지널 한 몸뚱이 단 1인 전용 혼자 독식 소유 접근권인 유니크 가변 차용(`&mut T`) 포인터여야만 철저히 강제됩니다. 그래서 저 파격적인 반전 무기인 `Cell<T>` 와 `RefCell<T>` 녀석들을 극찬해 유독 '내부 가변성(interior mutability)' 강령 능력을 품었다 수식하며, 그 외 나머지 평범 그 자체 정석 타입들을 싸잡아 '계승/세습형 가변성(exhibit 'inherited mutability')' 굴레 교도들이라 부르며 멸칭 대조(in contrast with)시킵니다.
>
> 셀 장비 타입들은 2가지 버전(flavors) 체제로 출시됩니다: `Cell<T>` 그리고 `RefCell<T>`. `Cell<T>` 녀석은 값어치가 싸고 편하게 `get` 이랑 `set` 이란 함수버튼 하나만 깔짝대도 즉석에서 무식하게 직방으로 알맹이 내용물을 덮어씌워 갈아채 치웁니다(change the interior value). 단 이 기똥찬 `Cell<T>`의 한계가 명확한데 오직 복제 가능 Copy 체질을 물려받은 타입 녀석 한정(only compatible with types that implement Copy) 전용이라는 구속입니다. 이걸 못 견디는 그 밖 다른 나머지 타계층 시민 유형들(other types)을 다루려면 하는 수 없이 고급 파생버전 `RefCell<T>`를 꺼내써야 하는데, 이건 조작 훼손 칼침(mutating) 꽂기 직전 구문에 필히 나홀로 점유락 자물쇠(write lock)를 한 번 더 결재 밟고 요구하는(acquiring a write lock) 과정을 거쳐야 합니다.
>
> `RefCell<T>`는 Rust 특성인 수명 주기(lifetimes) 고무줄 철학을 집어삼킨 뒤 자체적인 '동적 대차 빌림술(dynamic borrowing)' 엔진 편법을 발명해 냅니다, 이 편법 과정하에서 여러분은 내부 단전 안쪽 알맹이에 다가갈 한시적(temporary) 배타 독점(exclusive) 가변 권한(mutable access) 사용 패스를 나름 선점 주장(claim) 외쳐 발급받을 자격을 누립니다. 본디 Rust 순정 오리지널 참조자 대출 락 감시망은 죄다 안전한 컴파일 타임 밖에서 완벽하게 문서 분석 추적(entirely tracked statically) 처리되지만 반면 사설 `RefCell<T>` 녀석 발급표 빚 대출(Borrows for `RefCell<T>`s) 딱지들은 오직 독단 룰로 돌아가는 어리둥절 '런타임 실행 즉석 현장 타임(at runtime)'에서만 동네방네 감시 통제(tracked)됩니다. 바로 이 감시대장 장부 자체가 미치도록 런타임 야매 동적 파동(dynamic)을 띄기 땜시, 만약 여러분이 넋을 빼놓고 이미 누가 앞서 가변 선점(already mutably borrowed) 대출 먹은 상태의 슬롯에 다시 나도 똑같은 가변 찌르기(attempt to borrow) 삽질을 재시도 들이밀 경우, 시스템상 어어 하고 통과 진입 돌파(possible to attempt) 병크가 열려버리는 현상; 이 터집니다. 그렇게 대충돌 스파크 박치기가 마찰되는 그 즉시 현장에 상주하던 사설통제 심사관은 즉결 처형으로 스레드 패닉 파괴 붕괴 폭발(results in thread panic) 버튼을 눌러버립니다.
>
> # 그렇다면 언제 이 요망한 내부 가변성 무기 사용을 택설해야 하는가 (When to choose interior mutability)
>
> 보편적 진리의 유일신 강령인 전형적 계승/세습 가변성(inherited mutability), 즉 무조건 절대 배타 독점 1인 독식 권리장전 가변 대출 허가증 창구에서 승인을 얻는 행위만 허락되는 그 꽉 막힌 철칙이야말로, 초창기 태곳적 Rust 언어가 설계되었을 때 온 세계 수많은 파편 동시다발 포인터 지옥(pointer aliasing) 복합 참조자 붕괴 겹침 사태를 완벽하게 분할 척결 억압(reason strongly)시킴으로써 프로그램 돌연사 사망 오류 크래쉬(crash bugs) 펑크사고를 원천부터 차단 봉쇄하는(statically preventing) 근간 핵심 요소 중 알파이자 오메가(one of the key language elements)이자 절대 신성 마법 전제 대의입니다. 고로 이 계승/가변성 철칙 규율은 항상 제 1지망 최우선 선호 선택지 모범 답안으로 추앙 유도되지만(preferred), 저 반대편 도박 뒷구멍 패스권 무기인 내부 가변성(interior mutability) 채용 스킬 트리는 정말 진짜 말 그대로 최후의 비상 수단 벼랑 끝(last resort) 선택지 정도로만 치부해 취급 대면하는 게 이치에 맞습니다. 하지만, 세상사가 원래 저런 편법 셀 기술 조직들이 원래는 안 돼요 거절 차단 금지 당했을 법(disallowed)한 기막힌 조건 환경에서조차 마법처럼 길을 뚫어 가변 수정 파동의 기적을 여과 없이 열어주기에(enable mutation), 살다 보면 때론 이 내부 가변성이 딱 맞춤(appropriate)으로 제격이거나 아니 아예 필수로 써먹을 수밖에 없는 숙명(must be used) 그 자체의 상황들도 여럿 맞게 됩니다. 이를테면:
>
> * 도저히 답이 안 나오는 불변 공유 참조 자산들(shared types) 집단 한가운데 심장에 그럴싸한 야매 가변 시드 뿌리 연무 계승 공작(Introducing inherited mutability roots) 은닉 씨앗 투척.
> * 겉표면은 엄숙한 순결 '안 건드립니다 논리적-불변성(logically-immutable methods)' 딱지를 달아놓고선 실질 뒤에선 쪼금 가변 치워야 할 사정 숨긴 뒷처리 로직 구현 실세(Implementation details).
> * Clone 구현체 구성을 필히 찢고 개조 수술 변조(Mutating implementations of Clone)해야 할 슬픈 운명 시점.
>
> ## 공유된 타입 안쪽에 슬쩍 가변의 뿌리(roots)를 밀반입해 이식하기 (Introducing inherited mutability roots to shared types)
>
> 우리가 그토록 사랑해 마지않는 `Rc<T>` 랑 `Arc<T>` 형제 자매 같은 스마트 포인터 타입들(Shared smart pointer types) 부류 집단은, 여럿 당사자 패거리 녀석끼리(mutiple parties) 마음껏 씹고 뜯고 복제 분신(cloned and shared) 시전 퍼나르는 기가 막힌 방패 용기(containers)를 제공해 줍니다. 헌데 이 포장지 안쪽 용접 내장 깊숙이 박힌 내용 알맹이 값 덩어리(contained values)의 운명을 고심 감정해 보건대, 이 녀석들은 세상 천지 사방 무더기 다발 다면 포인터 별칭들에게 중복 노출 공유(multiply-aliased) 될 운명을 전제하고 깔기 때문에, 무조건 영원히 무기력한 단순 '공유 참조자' 계급 신분으로만 엮여 파생 대여(borrowed)만 성립될 뿐, 절대 그 위에 쌍도끼 도륙살의 변조 가변 전용 패스 `&mut` 권력 인가권 대출(not mutable references) 위임 따위는 원천 봉쇄 박탈 상태입니다. 고로 앞서 설명한 이 치트 무기 셀(cells) 부류 세트 도구들이 만약 아예 없었더라면(Without cells), 그 튼튼한 무적 다중 방어막 공유 철갑 박스(shared boxes) 장막 내부 안에 잠긴 알맹이 데이터에 가변 칼침(mutate data) 조작 수술의 은혜를 투사하는 발상 자체가 완전 정말  불가능 영원 봉인(impossible at all!) 처방 불가 낙인이 자명할 뿐입니다!
>
> 하지만 그래서 우린 찌질한 기행 꼼수를 씁니다, 이 철갑 스마트 포인터 타입 형편없는 것 구조물 안쪽 단전 후미진 틈바구니 알맹이 핵심 코어 영역을 고의로 은밀히 파내어 `RefCell<T>` 암막 치트 장치 모듈 하나를 박아놓고 접합 기생 장식시켜 버림으로써, 무적 가변 파괴 투영권 부활 스파크의 숨통(reintroduce mutability) 이종 접합 이식을 쿨하게 재가동 연성해 내는 방식의 설계 편법 야매 길루트야말로 너무나도 일상적이고 흔해 빠지게(very common) 통용되는 바이블 국밥 잡학 편법 도구 그 자체입니다:
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
> 콕 찝어 꼭 유념해 주실 건, ಈ 잡코딩 예제 형편없는 것 장난감 도구 놀음 세트(this example)에는 어디 감히 다중 난입 전용 철갑 `Arc<T>` 방패 구조 대신, 초식 전용 찌질이 `Rc<T>` 녀석을 장비했는지 주시(Note that) 명심하셔야 한단 점입니다. 애초 구제불능의 `RefCell<T>`s 사설 기계 시스템 자체 태생 한계 부품 수급 쪼가리가 무조건 고작해야 평화로운 한 줄기 흐름의 소박한 싱글 채널 노숙 전용 환경 제어 단일 스레드 무대 전용 칩셋(for single-threaded scenarios) 도구에 불과 한정 규정이기 때문입니다. 진짜 피 튀기고 다중 아수라 폭발 멀티 깽판 터지는 난전 병렬 헬파이어 전장 한복판 상황(multi-threaded situation)에서 저따구 분산 공유 무지성 난장 가변 투척의 폭력 가혹 쾌감 공유(shared mutability) 조작 니즈 스릴 욕구 뽕이 밀려 올라오신다면 얌전히 제발 부탁이니 `Mutex<T>` (뮤텍스 괴물형 자물쇠)님을 영접 장착 고려 참배 모셔 두시길(Consider using) 진심으로 싹싹 빕니다.

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
