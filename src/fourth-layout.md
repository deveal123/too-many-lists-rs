# 레이아웃 (Layout)

이번 설계의 알파이자 오메가 핵심 키워드는 바로 `RefCell` 타입입니다. 이 RefCell 구조를 지탱하는 심장(heart)부는 다음과 같은 두 메서드 쌍의 호출로 이루어져 있습니다:

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

저 `borrow` 와 `borrow_mut` 메서드 사이에 흐르는 룰은 무섭도록 `&` 및 `&mut` 의 철칙과 100% 한 치의 오차 없이 똑같습니다(exactly those of): 당신은 `borrow` 대출을 원하는 만큼 무제한 반복 복제해서 빼다 쓸 수 있지만, `borrow_mut` 대출 가변 조작을 원한다면 오직 철저한 **배타적 독점 접근권(exclusivity)**을 인정받아야만 인가됩니다.

다만 이 깐깐한 심사가 정적 컴파일(statically) 검사망에서 미리 차단당하는 게 아니라, RefCell 녀석은 이 규율 위반 검열을 프로그램 런타임(runtime) 실행 직전 즉석 단계로 미뤄버립니다. 만약 당신이 겁도 없이 이 철칙을 위반해 이중 대출 짓거리를 벌이다 현장에 적발(break the rules)되는 날엔, 의례적인 RefCell 판사는 가차 없이 예고도 없이 냅다 패닉(panic) 철퇴를 날려 당신의 프로그램 모가지를 그 자리에서 처형(crash)시켜 뻗게 만들어 버립니다. 
근데 이놈들은 대체 왜 원본 값도 아니고 기괴한 Ref 나 RefMut 따위의 허울갑 껍데기 포장지(things)들을 토해내는 걸까요? 뭐, 까놓고 말하면 녀석들의 동작 습성은 대략 `Rc` 류의 수명 카운터 감시망 역할과 얼추 비슷한데 그 용도가 대여 빌림용(for borrowing)이란 것만 다를 뿐입니다. 즉 감시병인 이놈들이 스코프 밖으로 벗어나 죽어 없어지기(out of scope) 전까진 징그럽게 RefCell 본체 대여 권한을 감시 유지(keep borrowed)시켜주는 족쇄 역할을 해냅니다. 어차피 이 더러운 골칫거리는 쫌 이따가 뒤에서 다시 상대할 테니(get to that later) 넘어갑시다.

어찌 됐건 Rc에 RefCell까지 장착했으니, 이로써 우리는 무려... 숨 막히게 말도 길어지고 도처에 어설픈 가변 구멍투성이가 판치면서 정작 순환 궤도는 제대로 소각 수거하지도 못하는(can't collect cycles) 희대의 반쪽짜리 하등형 쓰레기 가비지 컬렉팅 언어 흉내(pervasively mutable garbage collected language)나 내는 돌연변이 혼종 괴물로 재탄생하게 되었습니다! 으-야~호....(Y-yaaaaay...)

좋습니다, 우리는 *양방향-연결(doubly-linked)* 구조를 만들고 싶습니다. 그 말인 즉, 모든 개별 노드 알맹이들에겐 자신의 앞, 그리고 뒤를 찌르는(previous and next node) 위치 포인터 화살표가 두 개씩 장착되어 있어야만 한다는 뜻입니다. 나아가서, 리스트 몸통 본체 구조 자체 또한 리스트의 맨 첫머리 놈(first node)과 끄트머리 꼬리 놈(last node)을 동시에 가리키는 시선 두 개를 전부 갖추고 있어야 합니다. 이런 양각 구조가 완성되면, 우린 리스트의 끝단 *양방향(both ends)* 그 어디서든 빛의 속도로 쾌속 삽입(insertion)과 삭제(removal)를 때려 박을 수 있게 됩니다.

결과적으로 우리 입맛에 가장 비슷한 초안 구조의 청사진은 대각선으로 이런 모양새(something like)가 돼야 할 텐데요:

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
  --> src/fourth.rs:5:5
   |
 5 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
  --> src/fourth.rs:6:5
   |
 6 |     tail: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

이야, 컴파일이 뚫리네요 (built)! 안 쓰는 잉여 코드(dead code)라고 뜨는 무더기 경고 텍스트 폭탄이 안구 테러하긴 하지만, 어찌 됐든 빌드는 통과했습니다! 까짓것 대충 함 본격적으로 손대 써먹어 봅시다.
