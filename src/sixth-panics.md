# Drop과 패닉 안전성 (Drop and Panic Safety)

자 여보세요, 아까 이런 주석이 달려있던 거 눈치채셨나요:

```rust
// 그리고 이제 더 이상 그 빌어먹을 `take` 같은 걸로 똥꼬쇼할 필요가 없어졌음을
// 주목하세요. 왜냐하면 모든 게 다 Copy 값들뿐이고 우리가 망치더라도 실행될
// 소멸자(dtors) 따윈 일절 없기 때문이죠... 맞죠? :) 그쵸오오오? :)))
```

이 말이 맞을까요?

죄송한데 지금 여러분이 무슨 책을 읽고 있는지 까먹으셨습니까? 당연히 헛소리죠! (어느 정도는요.)

아까 짰던 pop_front의 속살을 다시 한번 들여다봅시다:

```rust ,ignore
// 안의 값을 쑥 뽑아내고 마무리를 짓기 위해(Drop) Box 껍데기를 되살려냅니다
// (Box는 놀랍게도 이 짓거리를 마법처럼 완벽히 이해해 줍니다).
let boxed_node = Box::from_raw(node.as_ptr());
let result = boxed_node.elem;

// 다음 노드를 새로운 front 자리에 꽂습니다.
self.front = boxed_node.back;
if let Some(new) = self.front {
    // 뽑혀 나간 삭제된 노드를 향한 남겨진 미련(reference)을 깔끔하게 지워줍시다.
    (*new.as_ptr()).front = None;
} else {
    // 만일 front가 Null이 되어버렸다면, 이 리스트가 텅텅 비었다는 뜻입니다!
    debug_assert!(self.len == 1);
    self.back = None;
}

self.len -= 1;
result
// 여기서 Box는 알아서 내장 파괴(freed)수순을 밟게 되고, T가 없어졌다는 사실 역시 알아서 잘 압니다.
```

어디에 버그가 숨어있는지 보이십니까? 존나 소름 돋게도(Horrifyingly), 범인은 바로 이 녀석입니다:

```rust ,ignore
debug_assert!(self.len == 1);
```

*진짜로요(Really)*? 기껏 테스트할 때 쓰자고 달아놓은 무결성 검사(integrity check) 따위가 버그의 원흉이라고요?? 네 쒸발 맞습니다!!! 뭐 애초에 우리가 컬렉션을 완벽하게 짰다면 버그가 *아니어야만* 하겠지만, 이딴 사소한 검사 하나가 "어이쿠 우리가 실수로 len 업데이트하는 걸 까먹었네 헿" 같은 귀여운 휴먼 에러를 *악용 가능한 치명적 메모리 안전성 버그(An Exploitable Memory Safety Bug)*로 돌변시킬 수 있습니다! 왜냐고요? 얜 패닉(panic)을 터뜨릴 수 있거든요! 평소엔 패닉 따위 좆도 신경 안 쓰고 살아도 되지만, 당신이 일단 *진짜배기(really)* 안전하지 않은(unsafe) 코드의 세계에 발을 들이고 "불변성(invariants)" 따위와 목숨 건 줄타기를 시작하는 순간, 당신은 패닉에 대해 강박증 환자처럼 예민해져야만(hyper-vigilant) 합니다!

우린 여기서 반드시 [*예외 안전성(exception safety)*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html) (혹은 패닉 안전성(panic safety), 스택 풀기 안전성(unwind safety) 등등 여러 이름으로 불립니다스택 ...)에 대해 한 번 짚고 넘어가야 합니다.

자, 팩트만 까놓고 말해봅시다: 기본적으로, 패닉은 *스택 풀기(unwinding)*를 유발합니다. 스택 풀기라는 건 그냥 "모든 함수들이 하던 짓 다 내팽개치고 즉각적으로 반환(return)빵을 때린다"는 걸 폼나게 포장한 용어일 뿐입니다. 여러분은 아마 반문하겠죠. "아니 씨발, *모든 놈들(everyone)*이 다 단체로 반환빵 때리고 프로그램이 뒤지기 직전인 마당에 누굴 챙기고 자시고 할 정신이 대수야?", 네 틀렸습니다!

우리가 이걸 반드시 신경 써야 하는 데엔 두 가지 절박한 이유가 있습니다: 첫째, 함수가 반환될 땐 소멸자(destructors)들이 대거 출동하여 대학살극을 벌이고, 둘째, 이 스택 풀기(unwind) 현상마저도 중간에 *낚아채서 멈출 수(caught)* 있기 때문입니다. 두 경우 모두 프로그램은 패닉이 터진 후에도 어찌어찌 목숨을 연명하며 계속 굴러갈(keep running) 여지가 있으므로, 우리는 안전하지 않은(unsafe) 컬렉션들을 만질 때 패닉이 터질 위험이 조금이라도 도사리고 있다면, 그 순간마다 컬렉션 내부가 *대충 어떻게든* 논리적 일관성을 유지하는 온전한 상태(coherent state)로 꽉 잡혀 있도록 극도로 조심하고 대비해야만 합니다. 왜냐하면 모든 패닉 하나하나는 곧 암묵적인 조기 반환(early return) 지뢰나 다름없기 때문입니다!

아까 그 버그 라인에 도달하는 그 찰나의 순간에 우리 컬렉션이 도대체 어떤 막장 상태(state)에 놓이게 되는지 다시 곰곰이 상상해 봅시다:

우리의 boxed_node는 이미 스택 위로 끌려 올라왔고, 알맹이 값도 쏙 뽑힌 상태입니다. 만약 이 시점에서 곧바로 반환(return) 빔을 맞고 튕겨 나간다면, 저 Box는 제 딴엔 임무를 다한 줄 알고 그대로 버려지고(dropped) 노드 자체도 고스란히 메모리에서 허공으로 증발(freed)해 버릴 겁니다. 자 이제 눈치채셨습니까..? self.back 포인터 새끼는 아직도 저 이미 허공으로 날아가 버린 노드 시체 쪼가리를 미련하게 가리키고 있단 말입니다! 나중에 우리가 컬렉션 나머지 껍데기들을 덕지덕지 이어 붙여 완성하고 self.back을 요긴하게 써먹으려는 그 순간, 이 끔찍한 해골물은 여지없이 사용-후-메모리-접근(use-after-free) 참사로 폭발하고 맙니다! 으악 씨발!

재밌게도, 이 다음 줄 역시 비스무리한 문제점을 안고 있긴 하지만 훨씬 더 안전한 녀석입니다:

```rust ,ignore
self.len -= 1;
```

기본적으로 debug 빌드 상태의 Rust는 오버플로우나 언더플로우를 눈에 불을 켜고 감시하며 걸리는 즉시 인정사정없이 패닉을 터뜨립니다. 네 맞습니다, 사칙연산 하나하나조차도 모조리 패닉 안전성(panic-safety)을 위협하는 지뢰밭입니다! 그런데 얜 왜 *그나마 낫다(better)*고 하느냐? 이놈은 우리가 그동안 헤집어놨던 불변성(invariants) 퍼즐 조각들을 전부 제자리에 원상 복구 시켜 놓은 *후에야* 비로소 발생하기 때문입니다. 고로 메모리 안전성(memory-safety) 문제만큼은 절대 일으키지 않습니다... 물론 우리가 len 값이 제대로 들어맞으리란 썩은 동아줄 믿음을 애초에 내다 버리고 안 믿는다는 전제하에서 말이죠. 근데 애초에 언더플로우가 터졌다는 것 자체가 이미 어딘가 단단히 좆됐다는 뜻이니 어차피 우린 이래 죽으나 저래 죽으나 뒤진 목숨이었습니다! 반면에 아까 그 debug assert 놈은 사소한 생채기급 찐빠를 즉결 처형급(critical) 대참사로 눈덩이처럼 부풀려버리는 악질이므로 어떤 의미에선 *더 최악(worse)*입니다!

제가 방금 "불변성(invariants)"이란 용어를 몇 번 흘렸는데, 이게 바로 패닉 안전성(panic-safety) 논증에 있어서 가히 치트키만큼이나 존나게 유용한 개념이기 때문입니다! 기본적으로, 제3자인 외부 관찰자(outside observer)가 우리 컬렉션을 요리조리 뜯어볼 때, 그 코드가 죽이 되든 밥이 되든 우리가 하늘이 두 쪽 나도 *항상(always)* 사수해야 하는 절대적인 성질(property)들이 몇 가지 존재합니다. LinkedList의 경우를 예로 들자면, 리스트 안에서 살아 숨 쉬며 닿을 수 있는(reachable) 모든 노드들은 무조건 유효하게 할당(allocated)되어 있고 멀쩡히 초기화되어 있어야만 한다는 맹세 조항이죠.

물론 *우리들끼리의 내부(Inside)* 구현체 안방 구석에서만큼은, 나중에 *아무도 모르게* 쥐도 새도 모르게 다시 원상 복구 시켜서 덮어둔다는 조건 하에 *일시적으로(temporarily)* 저 불변성들을 망치질해 다 깨부수고 노는 유도리(flexibility)가 허용됩니다. 사실 이 짓거리야말로 Rust 컴파일러의 소유권(ownership)과 차용(borrowing) 시스템이 컬렉션을 다룰 때 뿜어내는 수많은 "핵심 킬러 기능(killer apps)" 중 단연 최고봉에 속합니다: 만약 어떤 연산이 `&mut Self`를 떡하니 요구한다면, 그 말인즉슨 우린 지금 이 컬렉션을 혼자 쥐락펴락할 수 있는 독점적 접근 권한(exclusive access)을 *100% 보장(guaranteed)* 받은 상태이며, 그 틈을 타 감시의 눈을 피해 잠시 불변성 거울을 와장창 깨부수더라도 아무도 그 파편을 몰래 훔쳐보며 딴지 걸 놈팽이가 없기 때문에 마음 놓고 깽판을 쳐도 절대적으로 안전하다는 뜻입니다.

아마 이를 가장 노골적이고 극단적으로 보여주는 대명사가 바로 저 [Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain) 녀석일 텐데, 이 미친놈은 아예 작정하고 Vec의 심장부 불변성을 와그작와그작 씹어먹으면서 Vec의 *맨 앞구르기(front)* 아니 심지어 *한가운데(middle)* 내장까지 허겁지겁 파먹고 뽑아내 버립니다. 그런데 이 짓거리가 *온전하게(sound)* 허용될 수 있는 이유는, 얘가 토해내는 Drain 반복자가 Vec 전체를 볼모로 잡은 `&mut` 권한을 꽉 움켜쥐고 있기 때문이며, 고로 이 문지기가 길을 막는 한 그 어떤 잡것들도 Vec 근처에 얼씬조차(gated) 못 하기 때문입니다! 그 누구도 저승사자 같은 Drain 반복자가 세상에서 사라지기 전까진 엉망진창 찢겨진 Vec을 관측할(observe) 수 없으며, 저승사자가 떠날 때쯤엔 소멸자가 쓱 출동하여 그 누구도 모르게 Vec 아가리를 원래대로 깔끔하게 스윽 봉합수술 해("repair") 버릴 테니, 이야 이거야말로 완벽하--

[시발 뭐긴 완벽하지 않습니다](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain). 안타깝게도, 여러분은 [당신의 통제권을 벗어난 영토에서 뛰노는 소멸자들이 반드시 100% 실행될 거란 망상을 믿어선(rely on) 안 되기 때문에](https://doc.rust-lang.org/std/mem/fn.forget.html), 저 막강한 Drain을 등골 빼먹을 때조차도 우리의 위대한 불변성을 어거지로 지켜내기 위해 약간의 웃픈 똥꼬쇼 몸부림(goofy way)을 더 추가해야만 합니다: 바로 [아예 시작부터 다짜고짜 Vec의 길이를 0으로 모가지를 쳐버리는 거죠](https://doc.rust-lang.org/std/mem/fn.forget.html). 그렇게 해두면 혹여나 어떤 씹새끼가 고의로 Drain 반복자를 메모리 누수(leaks)시켜 공중분해 해버리더라도, 적어도 그 새끼 손아귀에 불발탄처럼 덩그러니 남은 Vec은 *안전한(safe)* 깡통 상태일 겁니다... 물론 그 대가로 안에 옹기종기 모여있던 수많은 피 같은 데이터들도 다 같이 증발해 버렸겠지만요. 네가 내 데이터를 누수시켰어? 그럼 나도 네 뒷골수를 누수시켜주마! 눈에는 눈 이에는 이! 이게 진정한 참교육 정의(True justice) 아니겠습니까!

진짜 구질구질한 이런 개똥꼬쇼 없이 제대로 *진짜배기(actually)* 소멸자 보험을 낭랑하게 활용하는 예외 안전성 모범 사례가 궁금하시다면 [BinaryHeap::sift_up 사례 연구(BinaryHeap::sift_up case study)](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up) 코너를 구경하고 오십시오.

아무튼 간에, 꼬아놓은 우리 LinkedList 따위에 저런 휘황찬란한 기믹질(fancy stuff)까지 총선은 동원할 필요는 없고, 그저 우리가 도대체 어느 구멍에서 불변성 뚝배기를 깨고 있는지, 무엇을 맹신하고 무엇을 필수로 요구하여(trust/require) 정답으로 간주하는지 항상 두 눈 부릅뜨고 경계망을 펼치며(vigilant), 쓸데없이 복잡하고 골때리는 스파게티 똥구덩이 한가운데서 뜬금없는 패닉 스레드 붕괴(unwinds) 지뢰를 무지성으로 뿌려대지 않게 조심하기만 하면 됩니다.

지금 당장 우리 상황에선, 우리 코드를 좀 더 튼튼한 방탄갑옷 입히는 꼼수 전략으로 다음 두 가지 선택지가 있습니다:

* Option::take 같은 마법 주문들을 훨씬 더 공격적이고(aggressively) 무자비하게 난사해 댑니다. 얘네들은 그 자체로 되게 "원자적 트랜잭션(transactional)" 같은 특성을 띠어서 불변성 퍼즐을 웬만해선 흩뜨리지 않고 온전히 보호해(preserve) 주려는 성향이 짙기 때문입니다.

* 지저분한 debug_asserts 따위는 쿨하게 갖다 버리고 죽여버린 다음, 그딴 거 없이도 "무결성 검사(integrity check)" 용도의 기깔난 비빔밥 전용 테스트 코드 함수를 따로 이쁘게 잘 짜서 맹신하는 겁니다. 이놈들은 어차피 실전 유저 코드 영역에선 평생 1도 구경할 일이 없을 테니까요.

원론적으로 따지자면 전 당연히 전자(first option)를 신봉하는 파입니다만, 슬프게도 이놈의 이중 연결 리스트(doubly-linked list) 꼬라지엔 별로 찰지게 먹히질 않습니다. 왜냐면 모든 게 양방향으로 빌어먹을 더블-중복 인코딩(doubly-redundantly encoded) 되어 있어서 꼬여있기 때문이죠. Option::take 나부랭이로 여기서 발생한 똥을 치울 순 없고, 차라리 저 debug_assert 부적을 한 줄 밑으로 살포시 끌어내리는 편이 문제를 가려줄 방패막이가 되어주긴 할 겁니다. 하지만 진짜 까놓고 말해서, 왜 굳이 우리가 우리 스스로 발목 잡고 셀프 하드모드(harder)를 켜야 합니까? 그냥 쿨하게 저 병신 같은 debug_assert 놈들 싹 다 도려내 없애버리고, 혹여라도 패닉 지뢰가 터질 확률이 1%라도 묻은 코드는 모조리 우리 불변성 방패가 끄떡없이 튼튼하게 버텨주고 있다고 장담(known to hold)할 수 있는 방공호 입구나 끝자락 바깥쪽에 몰아붙여 배치해 버리면 그만인걸요!

(뭐 이런 관점에서 바라보자면 걔네들을 차라리 *사전조건(preconditions)* 이나 *사후조건(postconditions)* 따위로 둔갑시켜 합리화하는 게 조금 더 그럴싸하긴(accurate) 하겠지만, 그래도 여러분은 가능한 한 항상 이것들을 영원불멸한 불변성(invariants) 덩어리로 받들고 모시려고 처절히 발악하셔야(endeavour)만 합니다!)

자, 이리하여 영광스러운 우리의 풀버전 구현체 명세입니다:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: 이건 연결 리스트잖아, 대체 바라는 게 뭐야?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // 새로운 front를 옛날 front보다 앞에 끼워 넣습니다
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // 만약 front가 없다면, 이건 지금 빈 리스트라는 뜻이고 
                // back 역시 똑같이 맞춰줘야 합니다.
                self.back = Some(new);
            }
            // 이 구문들은 무조건 항상 실행됩니다!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // 뽑아낼 front 노드가 있을 때만 깔짝거려 주면 됩니다.
            self.front.map(|node| {
                // 안의 값을 쑥 뽑아내고 마무리를 짓기 위해(Drop) Box 껍데기를 되살려냅니다
                // (Box는 놀랍게도 이 짓거리를 마법처럼 완벽히 이해해 줍니다).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // 다음 노드를 새로운 front 자리에 꽂습니다.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // 뽑혀 나간 삭제된 노드를 향한 남겨진 미련(reference)을 깔끔하게 지워줍시다
                    (*new.as_ptr()).front = None;
                } else {
                    // 만일 front가 Null이 되어버렸다면, 이 리스트가 텅텅 비었다는 뜻입니다!
                    self.back = None;
                }

                self.len -= 1;
                result
                // 여기서 Box는 알아서 내장 파괴(freed)수순을 밟게 되고, T가 없어졌다는 사실 역시 알아서 잘 압니다.
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

이 코드에서 도대체 어디서 패닉이 터질 수 있을까요? 뭐 까놓고 말해 그걸 콕 집어낼 수 있으려면 당신은 꽤 상당한 부류의 고인물 Rust 전문가(Rust expert)여야 하겠지만, 너무나 다행스럽게도(thankfully), 저는 십고인물입니다!

제가 스캔해 본 바 이 허접한 코드 쪼가리 안에서 그나마 패닉 발작을 일으킬 일말의 *가능성(possibly)*이라도 있는 지뢰 구간은 (어떤 정신병자 새끼가 stdlib을 빌드할 때 미쳐가지고 debug_asserts를 송두리째 강제 활성화 시킨 채 컴파일해 버리는 식의 진짜 개지랄 맞은 억까 참사 상황은 빼고요, 애초에 그딴 짓거린 사람 새끼면 절대 해선 안 될 짓입니다) 오직 `Box::new` (메모리 고갈, out-of-memory 상황)와 저놈의 len 길이나 빼고 더하는 사칙연산(arithmetic)뿐입니다. 다행히도 이 모든 지뢰밭은 우리 메서드 함수들의 맨 처음이나 지극히 가장자리 끝부분에 치우쳐 몰아넣어져 있으니, 네 맞습니다, 우린 아주 좆나게 친절하고 착실하게 완벽한 안전(safe)을 도모하고 있는 겁니다!

...아깐 `Box::new` 나부랭이도 패닉 폭탄을 터뜨릴 수 있다는 말에 눈 깔창이 튀어나오시며 기겁하셨나요? 명심하세요, 패닉은 늘 그렇게 당신의 뒤통수를 아주 개찰지게 후려갈길(get you like that) 겁니다! 그러니까 애초에 저딴 자잘한 지뢰 파편 걱정 따위 1도 안 하고 발 뻗고 잘 수 있도록 당신의 아름다운 불변성(invariants)들을 온전히 지키는데 필사적으로 사활을(preserve) 거십쇼!
