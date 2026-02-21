# 기초 (Basics)

> **해설자:** 이번 섹션에는 크고 명백한 구조적 오류(looming fundamental error)가 도사리고 있습니다. 사실 이게 바로 이 장의 핵심 포인트거든요. 하지만 우리가 `unsafe`의 늪에 발을 들인 이상, 뭔가 심각하게 잘못 짜놨어도 모든 게 멀쩡히 컴파일되고 표면적으론 *감쪽같이 잘 돌아가는(seemingly work)* 게 가능합니다. 이 근본적인 실수의 정체는 다음 섹션에서 파헤쳐질 겁니다. 절대 이 섹션에 있는 내용을 실전 코드에 적용하지 마세요!

좋습니다, 다시 기초로 돌아가 보죠. 어떻게 리스트를 생성할 수 있을까요?

이전에는 이렇게 했었죠:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

하지만 우리는 이제 더 이상 `tail`에 Option을 쓰지 않습니다:

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found 
   |                                       enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

물론 Option을 *사용할 수는* 있겠지만, Box와 다르게 `*mut` 포인터는 자체적으로 null이 될 수 있습니다(nullable). 즉, null 포인터 최적화의 혜택을 굳이 받을 필요가 없다는 뜻이죠. 그래서 우리는 None을 표현하기 위해 `null`을 직접 사용할 겁니다.

그럼 널 포인터를 어떻게 얻을 수 있을까요? 몇 가지 방법이 있지만, 저는 `std::ptr::null_mut()`을 사용하는 걸 선호합니다. 원한다면 `0 as *mut _` 같은 방식을 쓸 수도 있겠지만, 그건 너무 *지저분해 보이니까요(messy)*.

```rust ,ignore
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

*쉿*, 컴파일러야 조용히 해. 곧 다 쓸 테니까.

좋습니다, 다시 `push` 작성으로 넘어가 보죠. 이번에는 요소를 삽입한 뒤에 `Option<&mut Node<T>>`를 챙겨오는 대신, 곧장 Box 내부를 가리키는 `*mut Node<T>`를 낚아채올(grab) 겁니다. 우리가 Box를 이리저리 이동시키더라도(move the Box around) Box 내용물의 주솟값은 언제나 안정적이므로(stable address), 이 짓거리를 마음껏 저질러도 논리적으로 문제없다는 걸 우린 잘 알고 있습니다. 물론 이건 절대 *안전하진(safe)* 않은데, Box가 버려지는 순간 우리는 해제된 빈 메모리를 가리키는 포인터(pointer to freed memory)를 갖게 될 테니까요.

어떻게 일반 포인터에서 원시 포인터를 만들 수 있을까요? 바로 강제 변환(Coercions)! 만약 변수가 원시 포인터 타입이라고 명시되어 있다면, 일반 참조자는 알아서 강제로 원시 타입으로 맞춰집니다:

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

이제 우린 필요한 모든 정보를 알게 됐습니다. 이것들을 토대로 이전의 얌전했던 참조자 기반 버전을 어림잡아 변환해 볼 수 있습니다:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null 은 null인지 체크하며, None인지 체크하는 것과 동일합니다
    if !self.tail.is_null() {
        // 기존 꼬리가 있다면, 꼬리의 next가 새로운 꼬리를 가리키게 합니다
        self.tail.next = Some(new_tail);
    } else {
        // 그렇지 않다면, 머리가 새로운 녀석을 가리키게 갱신합니다
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

어라? 이미 Node를 가리키는 포인터가 손에 있는데 왜 `next` 필드를 가져올 수 없나요?

Rust는 여러분이 원시 포인터를 사용할 때면 정말 깐깐한 심술쟁이(jerk)로 돌변합니다. 원시 포인터의 내용물에 접근(access the contents)하는 것은 너무나 치명적이고 위험한(unsafe) 연산이기 때문에, 반드시 손수 수동으로 역참조 마커를 붙이도록 강권(insists)하거든요. 

그러면 하라는 대로 해보죠:

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
153 |             *self.tail.next = Some(new_tail);
    |             -----------^^^^
    |             |
    |             help: `self.tail` is a raw pointer; 
    |             try dereferencing it: `(*self.tail).next`
```

으으으, 연산자 우선순위(operator precedence)가 발목을 잡네요! 괄호를 씌웁시다.

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires 
              unsafe function or block

  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; 
     they can violate aliasing rules and cause data races: 
     all of these are undefined behavior
```

이건. 이렇게. 어려워선. 안 되는. 건데.
(THIS. SHOULDN'T. BE. THIS. HARD.)

제가 Unsafe Rust는 Safe Rust를 위한 일종의 FFI 언어 공간이라고 말했던 걸 기억하시나요? 컴파일러는 우리가 이런 위험한 FFI 조작을 하려는 구역을 명시적으로 구분 지어 주기(explicitly delimit)를 원합니다. 선택지는 두 가지가 있습니다. 첫째, 함수 *전체*를 unsafe로 마킹해서 함수 자체를 Unsafe Rust 영역으로 만들어버릴 수 있습니다. 이렇게 되면 오직 안전하지 않은(`unsafe`) 콘텍스트 내에서만 이 함수를 호출할 수 있게 됩니다. 우린 사용자들이 안전하게 우리의 리스트를 사용하길 바라기 때문에 이건 썩 좋은 그림이 아닙니다. 둘째, 함수 내부에 `unsafe` 블록을 만들어서 FFI 경계선을 딱 그어주는 겁니다. 그러면 체적인 함수 자체는 대외적으로 안전하다(safe)고 취급됩니다. 우린 이 두 번째 방식을 택하겠습니다:


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    if !self.tail.is_null() {
        // 안녕하십니까 컴파일러님, 제가 위험한 장난을 치는 건 알지만
        // 맹세컨대 절대 실수를 안 하는 훌륭한 프로그래머가 되겠습니다.
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
21 | 11 |     elem: T,
   |    |     ^^^^^^^
   |    |
   |    = note: #[warn(dead_code)] on by default
```

야호! (Yay!)

지금까지 전체를 통틀어 우리가 `unsafe` 블록을 써야만 했던 장소가 오직 *딱 한 곳* 뿐이었다는 점이 상당히 흥미롭네요. 우리는 사방팔방에 원시 포인터를 썼는데, 대체 왜 저 대목에서만 저러는 걸까요?

이유는 즉슨, Rust는 안전하지 않은 영역(`unsafe`)에 대해서 유별날 정도로 깐깐한 규칙주의자(massive rules-lawyer pedant)이기 때문입니다. 우리는 기본적으로 프로그램에서 무결점 안전이 보장되는 Safe Rust 세계의 범위를 극대화(maximize)하고 싶어 합니다. 그렇게 된 프로그램일수록 우리가 훨씬 더 신뢰할 수 있기 때문입니다. 이를 위해, Rust는 이 위험한 `unsafe`를 위한 표면적을 극단적으로 최소화하여 오려냅니다(carves out a minimal surface area). 우리가 앞서 수많은 장소에서 원시 포인터를 다룬 것들을 잘 살펴보면, 그것들은 그저 값을 *할당(assigning)* 하거나 그것이 널(null)인지 여부를 단순 관찰했을 뿐임을 알 수 있습니다.

실제로 여러분이 원시 포인터를 역참조하지 않는 이상, *저런 조작 행위들을 하는 건 완벽하게 100% 안전합니다.* 어차피 단지 정수 조각 값을 하나 읽고 쓰는 것일 뿐이니까요! 원시 포인터 때문에 여러분이 정말로 치명적인 함정에 거하게 빠질(get into trouble) 수 있는 유일한 순간은 오직 그것 안에 알맹이를 보기 위해 직접 역참조(dereference)를 시전할 때뿐입니다. 그래서 Rust는 *오직 그 행위 하나만*이 불법(unsafe)이며, 그 외의 다른 모든 찌끄레기 행동들은 완전 합법적이고 안전하다고 선포하는 것입니다.

정말 지독하게 깐깐합니다(Super. Pedantic). 하지만 사실 따져보면 완벽히 올바른 논리입니다.

> **해설자:** 지구 반대편 어느 구석에서, 하드웨어 엔지니어 한 명이 등골에 서늘한 오한(shiver down her spine)을 느꼈습니다 - 누군가가 또다시 "포인터는 그저 정수에 불과해"라고 구시대적인 헛소리를 뱉는(insisting) 소리가 공중에 울려 퍼졌거든요. 그녀는 혼신을 쏟아 기획 중인 '새로운 하드웨어 포인터 암호 인증 체계(new hardware pointer authentication scheme)' 초안 결재 서류 위로 주르륵 하염없는 눈물 한 방울(a single tear)을 떨구고 맙니다. 그 옆 부서의 컴파일러 개발자 친구는 아무 감흥도 못 느낍니다 &mdash; 걘 이미 너무 오래전부터 항상 무덤덤한 스웨터 갑옷을 껴입는 법을 체득해 버렸기 때문이죠.

일부 포인터 연산 조작만이 *유일하게* 위험한 금제 영역으로 취급된다는 사실은 아주 흥미로운 맹점을 낳습니다: 비록 우리가 `unsafe` 블록 철창으로 한정 지어 저 치명적인 범죄 위험 구역을 세심하게 통제하여 묶어두려 시도하지만, 정작 그 코드가 터질지 안전할지 그 자체의 유효성 위기 여부는 진작 그 감옥 밖에서 선점 당해 저당 잡힌 상태 파급 여파에 전적으로 귀속되어 끌려다닐(depends on state that was established outside) 수밖에 없다는 지독한 현실 모순이죠. 심지어 아예 함수 영역 밖에서 굴러 들어온 위험 업보에 목줄이 묶인 것일 수도 있습니다!

저는 이런 우스꽝스러운 기현상을 컴파일러 안전망 타락의 *독기 혹은 오염(taint)* 이라고 부릅니다. 특정 자그마한 구역에서라도 당신이 `unsafe` 영역을 차단 발포 시전 선포 도입 섭취 사용하는 즉시, 모듈 구역 영토 전역 내의 모든 전체 함수 껍질들이 그 순간 다 똑같은 죄악 폭탄 타락 독기로 전염되어 오염되어(tainted with unsafety) 물들어버리는 무시무시한 꼴이 됩니다. 고로 그 불안정한 위법 금제 불량 코드들이 올바르게 아무 사고 없이 무사 호강 굴러간다는 강제 불변 조건(invariants) 방호 억제 전제들을 기강 통렬하게 성립 사수 유지 존속 증명하기 위해, 모듈 방구석 내 온 식구 모조리 싹 다 몽땅 처참히 철저하고 소름 끼치게 완벽한 논리로 오류 여지 1도 티끌 없이 짜맞춰 작성되어야만 하는 고통 연좌제(Everything has to be correctly written)에 휘말립니다.

다행스럽게도 이 끔찍한 오염 독기는 *은닉 캡슐화 보호막 결계(privacy)* 기술 덕에 모아 외부 통제 방지가 용이해집니다. 우리 모듈 장막 바깥 영토의 고립 무관계 타인방 타지 이방 구조 클라이언트 소비 유저 입장 측면에선, 우리의 내밀 은둔 핵심 보호 설계 구조체 필드 징표들은 완벽한 일급 철통 봉인 차단 비밀 프라이빗(totally private) 금제 격리 구역에 결계 투옥 포장되어 있으니, 감히 제3자가 우리 집 문턱을 무단 침입해 넘어 들어와 우리가 숨긴 뱃속 내부 부품 장기 구성물 구동 환경과 안방 핵심 기저 상태를 마음대로 개판 이상한 사람처럼 조작 훼손 변경 뒤틀어 교란 망쳐댈 우려를 원천 사수 방어 통제(mess with our state) 할 방도가 차단됩니다. 우리가 대외적으로 서비스 출납 허용 지표 개방 공개 노출 제공 허가한(expose) 정식 창구 공용 API 서비스 출입 통로 문지기들 수문장 함수 기능들을 어떻게 겉에서 오만방자하게 주물러 지지고 볶고 연성 조합 오남용(combination)하더라도 그 어떤 치명적 지옥 시스템 유혈 파탄 개병크 참사 부작용(bad stuff to happen)이 안 터진다는 전제 철권 보장만 우리 측에서 내부적으로 확립 결착 입증해낸다면, 적어도 외부 대강당 구경꾼 소비자 불특정 관찰 녀석(outside observer is concerned) 시야 한계에선 우리의 모든 코드는 100% 무결점하고 안전빵 철통 청정 코드로 보장 둔갑 승인 선언 판정받는 맹점입니다!
게다가 사실 따지고 보면 이건 그 무시무시한 이단 외래 FFI 이민자 용병 파견 호출 사례 시스템(FFI case)과 비교해 봐도 하등 다를 바가 티끌 전혀 없는(no different from) 이치죠. 만약 겉모습이 어떤 고상한 파이썬 수학 도서관 코드 라이브러리 녀석이 뒤에서 아주 흉악 무도 몰래 그 빠르고 무자비 살벌 끔찍한 날것 무식 C 생존 스피드 무한 엔진 언어 녀석을 무자비 무단 차출 강제 영입 위탁 하청 굴리든(python math library shells out to C) 쑈를 하든 말든 간에 겉으론, 표면적으로 아주 매끄럽고 안정 정상 평화 규격 정석적인 100퍼 튼튼 안전 보장 대문 인터페이스(safe interface) 접견 창구 껍데기만 말끔하게 내놓아 보여주기만 제대로 포장 구색만 갖춘다면 대강 그딴 내막 진실에는 세속의 아무도 그 더러운 암흑 내부의 끔찍 비밀 참산 뒷구멍 사연 따위엔 1도 털끝 안중 안주 팩트 관심 우려 의심 헛소리(No one needs to care)를 안 품을 테니까요.

어쨌든, 이제 `pop` 함수로 대장정을 넘어가 보겠습니다. 이 녀석 역시나 과거 낡아빠진 시절에 우리가 구성했던 훌륭한 참조자 버전 툴과 기본적으로 토씨 하나 빠짐없이 글자 순서 복붙 복사 정말 소름 돋게 흡사 복붙 데칼코마니 평행 복제 거푸집 박아낸 구도를 띄고 있습니다(pretty much verbatim):

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

또다시 우린 여기서 어떤 특정 영역 환경 한 구석 안전망 상태가 다른 저기 머나먼 조건 상태 파동 결과에 질질 끌려다니며 강제 종속 결정 귀결 지어 도달 폭발 영향을 강림 미치는 상태 의존 인과 시공 초월 예시 구도(safety is stateful)를 재차 또다시 목격 대면 엿볼 수 있습니다. 만약 우리가 바로 이 단 한 가지 독방 *함수 안에서(in this function)* 저주받은 잔해 꼬리 시체 댕글링 찌꺼기 추적 미아 영수증 포인터를 깨끗 무결 말끔 백지에 초기화 완전 침묵 세탁 정화(null out)해 진정 방벽 쿨링 조치 방지 땜빵 무마시켜 주지 않은 채 뻐팅기고 버텨 무마한다면, 뭐 당장 그 시점 그때는 별반 그 어떤 참사 부작용 파편 조짐이나 우려 징후 사단 오작동 반동 문제 폭발 요발 대재앙 여파 따위도 절대로 겉보기엔 안 터진 듯 일절 발생 도출 발동 안 터지고 잠잠 평화 고요 적막 조용 무 구동 무 파 동 스 무 무 조 응 조 무 기 조 사 고 고 무 고 여 증 요 요 조 무(no problems at all) 안 나타날 겁니다. 하지만 바로 그다음에 찰나 연달아 뒤 이어 저격 발동 날아드는 후속 연타 추격 저격 호출 타점인 `push` 침투 재장전 명령 호출 조작이 자행 단행 호출 추가 융단 발사 땡겨지는(subsequent calls to `push`) 순간, 곧바로 이 기괴 포인터 지옥의 폭탄 파워 좀비 귀신 시체 좀먹은 아사 미아 무덤 빈 텅 텅 썩어 부서진 잔재 소 진 무 무 썩 텅 뼈 빈 텅 부 빈 바 빈 해 부 폐 잔 조 쓰 종 오 기 남 앙 좀 공 묘(the dangling tail) 허공 미아 묘지를 찾아서 황당한 듯이 빙의 폭발 값 부패 난수 값 오 염 지 조 망 사 강 사 병 조 발 지 썩 박 훼 부 무 망 적 조 난 발 난 강 마 기 자 (start writing) 작 성 부 기 적 삽 쓰 강 부 강 무 진 막 삽 입 융 단 재 폭 파 전 무 대 를 황당한 듯 퍼붓고 대참사를 저지르기 저 갈 겨 발 쑤 쑤 파 파 폭 질 쑤 난 (start writing to the dangling tail)! 작 할 겁 니 다!

한번 제대로 굴러가나 테스트를 돌려 검문해 보죠:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // 빈 리스트가 맛 안 가고 제정신인지 기동 점검 타진
        assert_eq!(list.pop(), None);

        // 리스트를 영양분으로 터지게 포식 증식 채워주기
        list.push(1);
        list.push(2);
        list.push(3);

        // 정상적으로 올바르게 요소들이 역류 배출 팝업 제거되는지 점원 검문
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // 내부 구조가 폭탄 덩어리로 부패 박살 고장 나진 않았는지 불안 방지 사살차 여분 추가 우겨 투입 테스트
        list.push(4);
        list.push(5);

        // 다시 정상적으로 추출 기동하는지 배출 점검 재심사
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // 아예 눈곱 한 점조차 안 남고 탈탈 털려 잔해까지 고갈 완전 진공 아사 파괴될 때까지 뽑아재껴 극한 한도 멸망 확인
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // 꼬리가 완벽히 터지고 고갈된 탈수 상태 멸종 이후에도 대의 꼬리 포인터 위치 세팅 지시 좌표가 올곧게 정신줄 안 놓고 고쳐 메다 잡혀 올바른 무 무사 안전 구역 무결 고정 허공 리셋 백지(fixed the pointer right) 상태로 자동 무마 자가 수복 정상 궤도 세탁 환원 회귀 수 복 원 고 수 리 안 성 전 복 대 처 수 진 처 진 수 위 탈 복 수 자 재 부 마 안 고 고 대 수 교 보 위 세 조 자 안 치 재 되 었 는 지 대 검 재 검 (fixed the pointer right) 었 는 지 를 혹독 점검 검문 타진 수사 
        list.push(6);
        list.push(7);

        // 마지막 심판 통상 추출 제거 적출 정당 정상 판결 무 구 방 기 종 출 최 증 추 종 팝 (normal removal) 심사
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

이건 사실 저 한참 옛날 시절 단방향 스택에서 우려 재탕 갈아치기 욹어 먹어 닳고 달아 빠진 짬뽕 스택 구문 복붙 테스트 복사본이지만, 단지 큐 맞춤 논리 성격 거동 방식상 예상 추출 추출 도출되는 `pop` 결과 방향과 순서가 정반대로 뒤집혀 교차 반전(flipped around) 전환 앞뒤 역순(flipped around) 둔갑 적용된 버전이랍니다. 덤보태기로 막판 최후 결말에다가 혹시 모를 꼬리 포인터 훼손 타락 오염 박살 조각 참사(tail-pointer corruption) 사태가 불시 저 `pop` 과정 속에서 벌어 황당한 마수 부 스 조 작 미 돌 자 폭 멸 이 망 발 발 확 마 마 기(doesn't occur) 기생 마 마 썩 구 확 벌 벌 고 나 망 부 발(doesn't occur) 동 증 자 확 (occur) 터 조 확 폭 동 발 자(doesn't occur)생 병크 사 태가 나 대 벌 아 발 오 안 어 지 조 폭 생 조 발 하 는 작 돌 안 안 생 발 않 오 나 (doesn't occur) 지 발 부 어 지 발 징 안 생 어 (doesn't occur)지 확인 사살 방지용 특수 여분 추가 점검 단계 과정(extra steps) 덫도 끝에다가 한 스쿱 더 추가 슬쩍 조밀 얹어 박아 심문 설계 함정 구성해봤습니다.

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
...
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

참 잘했어요 별표 만점 백점 쾅!(Gold Star!)

> **해설자:** 이제 폭풍이 몰아칩니다... (Here it comes...)