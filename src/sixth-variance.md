# 가변성(Variance)과 무령데이터(PhantomData)

이 문제를 당장 뒤로 미루고 나중에 수습하려 들면 아주 골치가 아파질 테니, 우리는 지금 바로 하드코어 레이아웃(Hardcore Layout)의 늪에 뛰어들어 보겠습니다.

안전하지 않은(unsafe) Rust 컬렉션들을 빚어낼 때 우리가 맞닥뜨려야 하는 다섯 명의 끔찍한 기사(five terrible horsemen)들이 있습니다:

1. [가변성 (Variance)](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
2. [Drop 검사 (Drop Check)](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
3. [NonNull 최적화 (NonNull Optimizations)](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
4. [isize::MAX 할당 규칙 (The isize::MAX Allocation Rule)](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
5. [크기가 0인 타입 (Zero-Sized Types)](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

다행스럽게도(Mercifully), 마지막 두 녀석은 우리에게 아무런 문제도 일으키지 않을 겁니다.

세 번째 녀석은 우리가 억지로 *문젯거리로 만들 수는* 있겠지만, 얻는 이득보다 잃는 게 더 많을 겁니다 -- 애초에 연결 리스트(LinkedList)를 선택한 시점에서 당신은 이미 메모리 효율성 따위를 따지는 전장에서는 100배쯤 일찌감치 백기를 든 셈이니까요.

두 번째 녀석의 경우엔 제가 예전엔 엄청 중요한 거고 std에서도 빡세게 굴린다고 핏대 세워 주장하곤 했었습니다만, 기본값(defaults)으로도 충분히 안전하며, 이걸 건드리는 방식들은 죄다 불안정(unstable)한데다, 이 기본값의 한계를 눈치채려면 진짜 *어마어마하게 억지로 노력해야(try so very hard)* 하므로, 그냥 속 편하게 신경 끄셔도 좋습니다.

자, 그럼 우리에겐 오직 가변성(Variance)만이 남게 됩니다. 까놓고 말해서, 사실 이 녀석마저도 그냥 은근슬쩍 뒤로 뭉개버려도(punt on) 무방하긴 합니다만, 그래도 명색이 '컬렉션 장인(Collections Person)'을 자처하는 제 알량한 자존심이 허락하지 않으므로, 우린 오늘 바로 저 가변성 짓거리(Do The Variance Thing)를 해치우겠습니다.

놀랍게도: Rust에는 하위 타이핑(subtyping) 개념이 있습니다. 구체적으로 말하자면, `&'big T`는 `&'small T`의 *하위 타입(subtype)*입니다. 왜냐고요? 프로그램의 어떤 특정 영역 동안만 살아있으면 되는 참조자가 필요한 코드에, 그것보다 *더 오래(longer)* 살아남는 참조자를 던져줘 봤자 대체로 아무런 문제가 없기 때문이죠. 너무 당연한 말 같지 않습니까? 직관적으로도 그냥 맞는 말이죠?

이게 왜 중요할까요? 자, 똑같은(same type) 두 개의 값을 인자로 받는 어떤 코드를 상상해 봅시다:

```rust ,ignore
fn take_two<T>(_val1: T, _val2: T) { }
```

진짜 더럽게 재미없는 평범한 코드네요. 당연히 `T=&u32`인 경우에도 잘 돌아가리라 기대할 겁니다, 맞죠?

```rust
fn two_refs<'big: 'small, 'small>(
    big: &'big u32, 
    small: &'small u32,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

네, 아주 나이스하게 컴파일됩니다!

그럼 이제부터 장난을 좀 쳐서, 저걸 어디 한 번, 음, 뭘로 할까, `std::cell::Cell`로 요리조리 감싸 볼까요:

```rust ,compilefail
use std::cell::Cell;

fn two_refs<'big: 'small, 'small>(
    // 참고: 요 두 줄이 바뀌었습니다.
    big: Cell<&'big u32>, 
    small: Cell<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
error[E0623]: lifetime mismatch
 --> src/main.rs:7:19
  |
4 |     big: Cell<&'big u32>, 
  |               ---------
5 |     small: Cell<&'small u32>,
  |                 ----------- these two types are declared with different lifetimes...
6 | ) {
7 |     take_two(big, small);
  |                   ^^^^^ ...but data from `small` flows into `big` here
```

엥??? 우린 수명(lifetimes)을 털끝 하나 안 건드렸는데, 컴파일러녀석이 대체 왜 갑자기 급발진을 하죠!?

아하, 그 수명 "하위 타이핑(subtyping)"이란 녀석이 무지하게 단순 무식한 녀석이라서 참조자들을 다른 껍데기로 감싸버리면 지레 겁먹고 나자빠지는 모양이네요. 보세요, `Vec`으로 감싸도 똑같이 박살 날걸요:

```rust
fn two_refs<'big: 'small, 'small>(
    big: Vec<&'big u32>, 
    small: Vec<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/playground`
```

거 보슈, 이것도 컴파일 안 되잖-- 잠깐 뭬야??? `Vec`에 무슨 마법이라도 걸린겨??????

뭐, 걸리긴 걸렸죠. 아니 안 걸렸나. 사실 마법은 항상 우리 내면 속에 깃들어 있었고, 그 마법의 이름은 바로 ✨*가변성(Variance)*✨입니다.

그 피 튀기고 잔혹한 세부 내용들을 전부 파헤치고 싶다면 [노미콘(nomicon)의 하위 타이핑(subtyping) 챕터](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)를 정독하시길 바랍니다만, 기본적으로 하위 타이핑이 *항상* 안전무결한 것만은 아닙니다. 특히 가변 참조자(mutable references)들이 얽혀들기 시작하면 여지없이 불안전해지는데, 왜냐하면 여러분이 `mem::swap` 같은 걸 섞어 쓰다가 어느 순간 갑자기 '앗차 댕글링 포인터(dangling pointers)!' 하고 터뜨려 먹을 수 있기 때문입니다.

이처럼 "가변 참조자 비스무리한" 특징을 띤 녀석은 *무공변성(불변성, invariant)*을 가집니다. 쉽게 말해 이녀석은 하위 타이핑이란 마법이 자신들의 제네릭 매개변수 위로 기어오르는 꼴을 싹 다 원천봉쇄(block) 하겠다는 뜻입니다. 고로 안전을 도모하기 위해 `&mut T`는 T에 대해 무공변(invariant)이며, `Cell<T>` 또한 T에 대해 무공변(invariant)인데, 이는 `&Cell<T>`가 내부 가변성(interior mutability) 덕분에 본질적으로 `&mut T`와 똑같은 패거리나 다름없기 때문입니다.

무공변(invariant)이 아닌 거의 모든 떨거지들은 *공변성(covariant)*을 지닙니다. 이건 그저 하위 타이핑 마법이 얘들 몸통을 스무스하게 "통과하여(passes through)" 평소처럼 정상 작동을 이어간다는 걸 의미할 뿐입니다 (사실 하위 타이핑의 진행 방향을 거꾸로 뒤집어버리는 반공변성(contravariant) 타입이라는 듣도 보도 못한 이상한 녀석도 있긴 한데, 현실에선 워낙 희귀종인데다 아무도 안 좋아하는 왕따들이니까 두 번 다시 언급하진 않겠습니다).

컬렉션들은 대개 자신들이 머금은 데이터들을 가리키는 가변 포인터를 품고 있기에, 여러분은 십중팔구 얘들도 죄다 무공변성(invariant)이겠거니 지레 짐작할 겁니다. 하지만 놀랍게도(in fact), 굳이 그럴 필요가 없습니다! Rust의 이기적인 소유권 시스템(ownership system)의 가호 덕분에, `Vec<T>`는 의미론상(semantically) 온전한 `T` 일개 값 덩어리와 100% 동치(equivalent)로 취급받으며, 이는 곧 녀석이 공변성(covariant)의 탈을 뒤집어써도 하등 안전에 흠집이 안 간다는 뜻이 됩니다!

불행하게도, 저 아래 우리 구조체의 정의 모습는 여지없는 무공변(invariant) 판정입니다:

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

아니 근데 도대체 Rust 컴파일러 같은 것 따위가 무슨 권한으로 이 만물들의 무/공변성을 지 맘대로 척척 판결하고 결재 내리는 걸까요? 글쎄요, 1.0 릴리스 이전의 호랑이 담배 피우던 좋았던 옛(good-old) 시절에는, 그냥 사람들이 자기가 원하는 대로 알아서 가변성을 수동 지정하게끔 놔두고 놀았는데... 결과는 한마디로 완전 개판 5분 전 대참사 호러쇼(train-wreck) 였습니다! 하위 타이핑 구조랑 가변성 공식들이 워낙에 머리를 터뜨리는 헬 난이도라, 심지어 코어 개발자들끼리 멱살 잡고 아주 기본적인 기초 용어 뜻 가지고 치고박고 싸우는 해프닝까지 벌어졌으니까요! 그래서 우린 우아하게 "알아서 예제 보고 눈치껏 배워먹기(variance by example)" 방식 쪽으로 방향을 틀었습니다: 요즘 컴파일러 님께서는 단지 여러분이 적어 둔 필드 구성품들을 쓰윽 넘겨짚고 스캔해서 그 뱃속 부품들의 가변성 DNA를 그대로 스포이드로 쭉 빨아다가 몽땅 복사할(copies) 뿐입니다. 만약 그 내부 장기들끼리 의견이 충돌해 파업하고 쌈박질을 시작한다면 무조건 무공변성(invariance)이 몰표로 대빵 먹고 이깁니다. 왜냐고요? 그편이 제일 튼튼하고 안전빵(safe)이니까.

그럼 대체 우리 타입 정의 코드 어디에 컴파일러 님 심혈을 거슬리게 한 빌런이 숨어 있단 겁니까? 네, 바로 저 `*mut` 녀석입니다!

Rust 동네에서 무법자 원시 포인터(raw pointers)들은 사실 사람들이 무슨 황당한 짓거릴 하든 방임하고 냅두려 노력하시지만, 저들에게도 단 하나 지켜야 할 최후의 보루 맹세 같은 안전장치 조항(safety feature)이 남아있습니다: 안타깝게도 대부분의 나사 빠진 개발자 인간들은 Rust 안에서 가변성이니 하위 타이핑이니 하는 그딴 개념 따위가 존재하는지조차 1도 상상 못 할뿐더러, 혹여나 잘못 찍어서 *부적절한(incorrectly)* 공변성 지뢰 코드를 밟는 날엔 그 대참사가 상상을 초월할 만큼 끔찍하게 피 튀길 게 뻔하므로, 아예 처음부터 싹을 자르기 위해 안전빵으로 `*mut T`는 닥치고 무공변(invariant)을 때려 박았습니다. 왜냐? 십중팔구 저 악마는 나중에 분명 필연적으로 `&mut T`와 맞짱뜨는 용도 "그대로(as)" 남용 조작될 게 뻔할 뻔자(good chance) 거든요.

솔직히 말해서 저는 이것 때문에 *진짜 정말 화나는(Exactly)* 울화통이 터질 노릇이고 짜증이 치솟아 힘든 상황입니다, 왜냐면 전 제 인생의 황금기를 Rust에서 컬렉션들 박박 짜내고 닦아재끼는 데 통째로 헌납한 인간이었으니까요. 그래서 제가 직접 빡쳐서 [std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html)을 하사해 빚어냈을 때, 이 앙증맞은 마법 주문서를 한 귀퉁이에 쓱 밀어 넣어 추가했습니다:

> 멍청한 `*mut T` 녀석과는 차원이 다르게, 갓 `NonNull<T>`님께서는 영광스럽게도 `T`에 대해 공변(covariant)이시도록 선택받으셨습니다. 덕분에 여러분은 굽신거리며 공변 타입(covariant types)을 건축하려 할 때 떳떳하게 `NonNull<T>` 님을 영접하여 쓸 수 있습니다. 허나 만약 행여나 절대 공변이 되어선 안 될 저주받은 금기의 구역(type)에서 이 능력을 멋모르고 함부로 휘둘렀다가는 어마어마한 불안정성(unsoundness) 대폭발 참사 리스크의 파멸을 초래할 것입니다.

하지만 잠깐만요 이보쇼, 이거 결국 껍데기 인터페이스는 다 `*mut T`를 중심으로 비비 꼬여(built around) 처발려 있잖아요, 대체 사기가 뭡니까! 진짜 이거 무슨 신비한 자연계 우주 마법(magic)이라도 됩니까?! 어디 싹 다 까서 좀 봅시다:

```rust
pub struct NonNull<T> {
    pointer: *const T,
}


impl<T> NonNull<T> {
    pub unsafe fn new_unchecked(ptr: *mut T) -> Self {
        // SAFETY: 애초에 호출한 녀석(caller)이 `ptr`가 하늘이 두 쪽 나도 절대 null이 아님을 목숨 걸고 보장(guarantee)해야만 합니다.
        unsafe { NonNull { pointer: ptr as *const T } }
    }
}
```

땡. 여긴 그딴 거저먹는 마법(NO MAGIC) 따윈 1도 찾아볼 수 없습니다! NonNull 얍삽이 녀석는 그저 `*const T`가 이미 친절하게 공변성(covariant) 특권을 누리고 있다는 꼼수 꿀 빠는 사실을 아주 야비하게 악용하여 자기 뱃속 깊숙이 꼬불쳐 저장(stores)해 두기만 한 채, 밖에서 API 입출구 경계벽 사이를 왔다 갔다 넘나들 때마다 그녀석을 대충 포장지만 교묘하게 `*mut T`로 앞뒤로 스리슬쩍 야바위 속여가며 탈바꿈 변장(casting back and forth)시켜, 마치 지가 처음부터 버젓이 `*mut T`를 소유하고 보관(storing) 중인 것처럼 밖을 향해 뻔뻔하게 "위장 코스프레 사기극(look like)"을 칠 뿐이란 겁니다. 이게 그 빌어먹을 마술 트릭의 밑천 민낯 전부올시다! 이게 바로 Rust 동네 좀 노는 컬렉션 녀석들이 그동안 공변성을 합법 꼼수 쟁취해 꿀 빨던 방식이었습니다! 그리고 이건 진짜 추잡스럽게 토악질 나게 미칠 듯 비참하고 끔찍하죠(miserable)! 그래서 제가 굽어살펴 불쌍한 늬들을 위해 착하디착한 선녀 포인터 타입성님(Good Pointer Type)을 하사하여 대신 이 더러운 짓거릴 전담해 주시도록 배려한 겁니다! 절하십쇼! 환영합니다! 부디 마음껏 그 잘난 서브타이핑 발등 찍기 권총 난사 플레이(subtyping footgun)를 즐기시길!

고로 여러분의 모든 번뇌와 십자가 고민거리를 타파해 줄 궁극의 만병통치약 해법은 바로 저 찬란한 NonNull님을 영접하시고, 만약 또 쓸데없이 널값(nullable) 변덕이 생겨 널 포인터를 주워 담고 싶어지면 얌전히 `Option<NonNull<T>>` 따위로 둘둘 싸매 구겨 넣는 겁니다. 자 그럼, 우리가 진짜 이 짜증 나고 피곤한 생쇼 짓을 굳이 해야 할까요..?

눼엥! 아주 끔찍한(sucks) 정말 화나는, 우린 지금 *상용 실전 등급 캡짱 연결 리스트(production grade linked lists)*를 장인정신으로 빚어내는 중이기 때문에, 눈물을 머금고 힘겹게 몸에 좋다는 맛없는 녹황색 채소를 씹어 넘기며 온갖 하드코어 변태 고행 난이도 길(hard way)을 걸어갈 작정입니다 (사실 그냥 속 편하게 천박한 생짜 `*const T` 같은 것를 들고 매번 눈물의 캐스팅(cast) 노가다를 처바르고 땜빵칠 수도 있는데, 전 순수하게 이 짓거리가 어디까지 우리 멘탈을 갉아먹고 빡치게 고통(painful) 줄 지 참으로 너무나 궁금하거든요... 오로지 불순한 순수 인체공학적 변태 과학적 탐구 목적(for Ergonomics Science) 측면에서 말이죠).


그래서 이 짓거리를 반영한 우리의 최종 타입 정의 본체 공개는 이러합니다:

```rust
use std::ptr::NonNull;

// !!!이번엔 진짜 바뀜!!!
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

...아 잠깐 스톱 빠꾸, 마침내 끝으로 마지막 하나만 더. 혹여나 당신이 부득불 굳이 원시 포인터 바보 짓거리(raw pointer stuff)들과 멱살잡이 뒹굴 일이 생긴다면, 항상 반드시 무조건(Any time) 당신의 나약한 포인터들을 악귀들로부터 배후수비 가드 쳐 줄 든든한 등신불 허수아비 유령 수호신(Ghost) 하나를 방패막이 토템으로 박아두셔야만 합니다:

```rust ,ignore
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    /// 우리는 실상은 안 그렇지만 의미론상(semantically) T 녀석 값들을 꿀꺽 삼켜서 값-복사 전달 방식(by-value)으로 보관 저장하는 듯 사기를 칠 겁니다.
    _boo: PhantomData<T>,
}
```

사실 까놓고 말해서 작금의 우리 상황 케이스에선 굳이 진짜배기로 [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) 부적을 덕지덕지 치덕치덕 붙여야만 *합당할지(actually)*는 저도 글쎄올시다입니다만, 여하튼 간에 일단 NonNull님을 주워와 써먹든(아니면 뭐 그냥 일반 깡통 뼈다귀 원시 포인터들을 아무 생각 없이 주워 먹든 간에) 일단 이걸 머리 빠지게 무조건(always) 쑤셔 박아 대충 안전 도모(safe) 위장을 꾸려 놓고 컴파일러 님과 지나가는 행인 코드 사냥꾼 외지인들에게 "나 지금 이러이러한 정말 깊은 뜻(think)을 품고 이 바보 짓거릴 벌이고 있는 중임"을 명명백백 공개 포고(make it clear) 박아 버리는 게 최소한 백만 배 나을 겁니다.

PhantomData는 우리가 컴파일러에게 "자 님아 보삼, 비록 젠장 내 정말 길고 장황한 타입 뱃속 내부 창고 안엔 현실 물리적으로 정말  하나도 없긴 하지만, 젠장 각종 구구절절 기구한 말 못 할 억까 구차한 알빠노 사연들(간접 조작 참조 꼼수, 타입 껍데기 박박 긁개 세탁 위장 은폐술 지우개 따위) 덕분에 대충 관념상으론(conceptually) 존재하고 상주한다고 박박 우겨 대충 넘어갑시다" 하고 가오 잡게끔 시위용 허수아비 가라 모델 "예제(example)" 표본 필드를 강제로 뇌물 먹여 던져주는(give) 치트키 마커 방법론 중 하나입니다. 지금 이 망치질 삽질 공사판 케이스에선, 우리가 대놓고 노골적으로 부득불 NonNull 같은 것를 주워 쓰는 핑계 알리바이가 바로 "우리 타입은 마치 T 라는 값 어치 덩어리를 고스란히 저장 머금고(stored) 있는 양 '시치미 뚝 떼고 행동하(behaves as if)'려는 중이다" 라고 가짜 사기극 거짓 명분 어필(claiming)을 날리고픈 목적 때문이므로, 우린 기꺼이 바지사장 얼굴마담 스파이 유령 부적(PhantomData) 하나를 떡밥으로 깔창 투척해서 그 앙큼한 거짓 선동 기만전술 사실을 눈치껏 대외 표출 자수 까발려(make that explicit) 인지시키려는 의도입니다.

표준 라이브러리(stdlib) 녀석은 사실 저 짓거릴 구사해야 할 또 다른 음침한 꿍꿍이 사연(other reasons)들이 차고 넘치는데, 왜냐면 걔넨 저주받은 마가 낀 [Drop 검사 오버라이드 회피 술법(Drop Check overrides)](https://doc.rust-lang.org/nightly/nomicon/dropck.html) 다이렉트 치트키 패스 뒷구멍에 접속할 만반의 권력을 쥐고 있기 때문이죠. 하지만 빌어먹을 그 저주의 그녀석 기능 자체가 역사적으로 몇 번이고 정말 다 갈아엎어져 리모델링 뜯어고쳐지는(reworked) 대참사 격변을 너무 많이 겪은지라, 솔직히 이 PhantomData가 그 기능에 아직까지 *실제(is a thing)* 유효한 치트키 제물로서 빌어먹을 숨은 역할을 지니고 있는지조차 까먹어 버렸습니다. 허나 젠장 전 내 알바 아니니 영겁의 지옥 끝까지(for all eternity) 맹목적으로 무지성 숭배 카고 컬트식 기도 메타 우상화 카고-컬트 복붙 충성을 대대손손 이어갈 겁니다, 왜냐하면 그 존엄 Drop 검사 마술(Drop Check Magic) 주문의 트라우마 환각이 아직까지 제 머리 망막 속에 노릇노릇 화상 낙인 흉터로 불타올라 각인(burned) 남겨져 있단 말입니다!

(저기 찐따 Node 녀석는 그 자체 몸통에 문자 그대로 T 를 다이렉트로 처먹어 소유(stores)하고 자빠져 있기 때문에, 저런 사기극 미신 변태 주술 짓거리(this) 따위를 벌일 필요조차 없습니다, 앗싸 만세 야호!)

...오케이 농담 아니고 이젠 진짜 밑 닦기 뼈대 기초공사 바보 레이아웃(layout) 공정 놀이 종결 마쳤습니다! 이제부터 찐 실제 진짜배기 찐따 기능 쌩기초(basic functionality) 삽질 작업 노가다 지옥으로 다 같이 돌진 갑시다!
