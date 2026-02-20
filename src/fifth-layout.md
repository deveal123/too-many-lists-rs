# 레이아웃 (Layout)

단방향 연결 큐(singly-linked queue)는 어떻게 생겼을까요? 이전에 단방향 스택(singly-linked stack)을 만들었을 때는 리스트의 한쪽 끝에 요소를 밀어 넣고(pushed), 같은 쪽 끝에서 요소를 꺼냈습니다(popped). 스택과 큐의 유일한 차이는 큐는 *반대쪽* 끝에서(the *other* end) 요소를 꺼낸다는 점입니다. 스택 구현체를 다시 살펴보죠:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

큐를 만들려면, `push`나 `pop` 중 하나를 리스트의 끝으로(to the end of the list) 이동시키면 됩니다. 단방향 리스트이므로 둘 중 *어떤 것*을 끝으로 옮기든 필요한 수고로움은 똑같습니다(with the same amount of effort).

`push`를 끝으로 옮긴다면, `None`까지 순회한 다음 그 자리에 새로운 요소를 포함한 Some을 넣으면 됩니다.

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

`pop`을 끝으로 옮긴다면, `None`의 *바로 앞(before)* 노드까지 순회한 다음 그걸 `take` 하면 됩니다:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

오늘 바로 이렇게 구현을 끝낼 수도 있겠지만(call it quits), 그러면 성능이 엄청나게 후질 겁니다(stink)! 두 연산 모두 리스트 *전체*를(entire list) 순회해야 하니까요. 올바른 인터페이스를 제공하니 큐가 맞다고 주장할(argue) 수도 있겠지만, 저는 성능 보장 역시 인터페이스의 일부라고 믿습니다. 엄밀한 점근적 표기법(asymptotic bounds)까지 따지진 않더라도 최소한 "빠름"과 "느림"의 차이는 확실히 구분해야죠. 큐는 push와 pop이 빠르다는(fast) 점을 보장해야 하지만, 리스트 전체를 걷는 무식한 행위는 확실히 *전혀* 빠르지 않습니다(definitely *not* fast).

여기서 핵심적인 관찰(key observation) 한 가지는 여태까지 *똑같은 짓(the same thing)* 을 매번 반복하며 막대한 낭비(wasting a ton of work)를 벌였다는 점입니다. 이 작업을 "캐시(cache)"해서 다시 재사용할 수 없을까요? 당연히 있죠! 리스트의 끝단을 가리키는 포인터를 저장해두고, 거기로 곧장 뛰어가면(jump straight to there) 됩니다!

그런데 이 방법은 오직 `push`와 `pop` 중 어느 한쪽의 방향 전환(inversion) 결과에만 들어맞습니다(works with this). `pop`을 전환하려면 꼬리("tail") 포인터를 역방향 뒤로 옮겨야만 하는데(move backwards), 우리 리스트는 단방향이라 효율적으로 뒤로 거슬러 갈 수 없습니다(can't do that efficiently). 반면 `push`쪽을 전환한다면, 꼬리가 아닌 머리("head") 포인터를 정방향 앞으로 이동시키기만 하면 되므로 아주 간단합니다(which is easy).

한번 그렇게 시도해 봅시다(Let's try that):

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // 꼬리에 추가할 때 다음 노드는 언제나 None입니다
            next: None,
        });

        // 기존 꼬리를 새 꼬리로 바꿔치기합니다
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // 기존 꼬리가 있었다면, 그 꼬리가 새 꼬리를 가리키게 합니다
                old_tail.next = Some(new_tail);
            }
            None => {
                // 기존 꼬리가 없었다면, 머리가 새 노드를 가리키게 합니다
                self.head = Some(new_tail);
            }
        }
    }
}
```

우리는 이제 이런 류의 구현에 꽤나 익숙해졌을 테니 진도를 좀 더 빠르게 빼보겠습니다(going a bit faster). 그렇다고 해서 바로 첫 방에 이 코드를 뚝딱 떠올릴(produce this code on the first try) 거라 기대하고 자책할 필요는 없습니다. 제가 기존에 저질렀던 시행착오 과정(trial-and-error)을 이번엔 그냥 편집해서 넘기고(skipping over) 있는 것뿐이니까요. 실제로 저는 이 코드를 쓰면서 여러분에게 보여주지 않는 엄청난 오타 삽질(ton of mistakes)을 저질렀지만, 허구한 날 `mut`를 빼먹었다거나 끝에 `;`를 안 찍는 바보 같은 실수를 수없이 재방송하는 건 이제 슬슬 교육적(instructive)이지 않기 때문입니다. 너무 걱정 마세요, 우린 충분히 차고 넘칠 만큼 *다른(other)* 무수한 여러 에러 메시지 폭탄을 더 보게 될 테니까요!

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

제기랄! (Shoot!)

> 이동된 값의 무단 재사용 위반(use of moved value): `new_tail`

Box는 Copy를 구현하지 않으므로, 우린 결코 한 번에 두 장소의 공간 위치에 동시에 할당(assign it to two locations)할 수 없습니다. 더 핵심적인 사실은, Box는 자신이 가리키는 대상을 완벽히 *소유(owns)* 하며, 메모리에서 버려지는(dropped) 순간 타겟 알맹이 대상을 전부 해제하려 든다는(try to free it) 겁니다. 만약 우리의 `push` 구현체가 통과해 컴파일되었다 하더라도, 우리는 리스트의 꼬리를 이중-해제(double-free) 시키며 터트릴 겁니다! 사실 지금 적힌 저 흐름만 놓고 보면(as written), 우리 코드는 매번 push를 때릴 때마다 old_tail을 가차 없이 파괴(free)해버리고 말 겁니다. 소름 돋네요! 🙀 (Yikes!)

좋습니다, 그럼 우리는 소유하지 않는 포인터(non-owning pointer)를 어떻게 만드는지 잘 알고 있습니다. 그건 단순히 참조자(reference)입니다!

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // 꼬리에 밀어넣을 때 다음은 언제나 None
            next: None,
        });

        // Box를 적절한 장소에 배치한 뒤, Node에 대한 참조자를 살짝 낚아채옵니다
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // 예전 꼬리가 있다면 그걸 새로운 꼬리로 갱신합니다
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // 그렇지 않다면 헤드 쪽을 같이 갱신합니다
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

이건 별로 트릭이나 꼼수(tricky)는 없습니다. 이전 코드 모델과 기본 아이디어는 완전 똑같지만(Same basic idea), 단지 실제 Box 상자 본체를 고정할 장소 안에다 잘 박아 안치한 다음 그 위치로부터 꼬리표 참조자만을 쪽 빼오는 암시적 반환 기술 특장점(implicit return goodness) 방식을 응용했을 뿐입니다.

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/fifth.rs:3:18
   |
 3 |     tail: Option<&mut Node<T>>, // NEW!
   |                  ^ expected lifetime parameter
```

아 맞다, 타입 선언 본체 안에 참조자를 구겨 넣을 때는 항상 꼬리표로 수명(lifetimes)을 달아줘야 했었죠. 흠... 이 거머리 참조자의 수명이 도대체 얼마로 매겨야 하는 걸까요(what's the lifetime)? 이거 곰곰이 생각해보니 옛날의 IterMut 녀석 달랠 때 상황과 완전 똑같지 않나요(IterMut, right?)? 옛날에 IterMut 한테 써먹었던 방식 그대로, 그냥 막무가내로 제네릭 수명 `'a` 딱지 하나(generic `'a`)를 달아줘 봅시다:

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // 꼬리 끝에 새로운 걸 매달 때 다음(next)은 늘 None입니다
            next: None,
        });

        // Box를 제대로 된 곳에 쑤신 뒤 참조자만 뜯어냅니다
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // 기존 꼬리가 있으면 새 꼬리 쪽을 보게 갱신
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // 없었다면 머리(head)도 새 걸 물게 갱신
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

```text
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_deref_mut()
   |                           ^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
...

note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>

```

와우, 정말 살 떨리게 자세하고 긴 에러 메시지(really detailed error message)입니다. 이런 병적인 오지랖 분석은 걱정스러운(concerning) 징후인데, 왜냐면 우리가 지금 뭔가 진짜 단단히 잘못 꼬여 망가진 짓거리(doing something really messed up)를 저질렀다는 사실을 강력히 시사하기 때문입니다. 여기서 이 재미난 대목 파트(interesting part) 하나만 짚어보죠:

> 그 수명은 당연히 impl 문 윗단에 선언 통과시켜둔 증명 딱지 수명 규약 `'a` 길이 기한 조건에 완벽하게 부합 일치해야만 한다(the lifetime must be valid for the lifetime `'a`)

우리는 이 `push` 메서드의 `self` 함수 몸통 구석에서 대출 참조자를 슬쩍 몰래 빌리고(borrowing from `self`) 있는데, 저기 컴파일러 님께서는 그 발급된 도주 참조자가 저 거대한 우주 통짜 수명 명줄 `'a`의 영겁 시간 내내 완벽 끝까지 뻗대어 유지 살아생존 장수(last as long as `'a`) 하기를 요구 명시하는 골때리는 모순 강요를 투척합니다. 그럼 까짓거 에라 모르겠다 이판사판으로, 컴파일러놈 얼굴 면전에다 대고 "야 내 숙주 `self` 몸통 녀석 자체가 진짜 우주급으로 완전 그 개긴 존엄 수명 기간 내내까지 살아 숨쉬어 버티거든!" 하고 대포허세 우기기 기만(tell it `self` *does* last that long..?)을 확 꽂아보면 어떤 꼴이 날까요..?

```rust ,ignore
    pub fn push(&'a mut self, elem: T) {
```

```text
cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

오, 헐퀴, 저 미친 농간이 통과(worked)하네요! 대박! (Great!)

여세가 꺾이기 전에 `pop` 녀석도 서둘러 뚝딱 끝장(just do `pop` too)을 바릅시다:

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    // 리스트의 거대한 현재 머리통 멱살을 잡아채 뜯어옵니다
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // 본체 `head` 가 다 발려 바닥났다면(out of `head`), 
        // 꼬리 포인터도 깔끔하게 완벽 무결 `None` 으로 청소 갱신(set the tail to `None`.)을 잊지 마세요.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

이 돌연변이 혼종의 성능 재판을 위한 야매 기초 뼈대 테스트(quick test)도 후다닥 박아보죠:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // 텅 빈 바구니 리스트가 정상 제정신 차리나 거동 확인(Check empty list behaves right)
        assert_eq!(list.pop(), None);

        // 리스트 안에 미친 듯 영양분 주입 불림 증식 탑재(Populate list)
        list.push(1);
        list.push(2);
        list.push(3);

        // 정상 무탈 쾌속 통상 방출 배출 제거(normal removal) 확인
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // 어디 좀 썩어 부패(corrupted) 하자가 나진 않았나 확인 사살차 재차 우겨넣어보기
        list.push(4);
        list.push(5);

        // 다시 정상 통상 추출 쾌속 방출 배출 제거(normal removal) 재점검
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // 아주 끝까지 바닥나 털려 쥐어짤 때까지 멸절 착즙 탈수 고갈(exhaustion) 생체 한계 파괴 검증
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);
    }
}
```

(이하 내용 뒤에 계속)

```text
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here
...
** WAY MORE LINES OF ERRORS (수많은 에러 줄 도배) **
...
error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

오 세상에. (Oh my goodness.)

우리에게 에러를 바가지로 쏟아낸 컴파일러는 잘못한 게 없습니다(The compiler's not wrong). 우린 방금 Rust의 가장 큰 죄악 중 하나를 저질렀거든요: 우리 *자신 내부(inside ourselves)*에 우리 자신을 가리키는 *참조자를 저장*해버린 겁니다.
어떻게 어떻게 Rust를 잘 구슬려서(managed to convince) 우리의 저 `push`와 `pop` 구현체가 말도 안 되는 짓임에도 불구하고 문법적으로는 말이 되게 통과시키긴 했습니다(저도 진짜 놀랐습니다).

이 짓거리가 *그럭저럭(sort of)* 컴파일은 되는 이유는, Rust에는 애초에 '자기 자신 안으로 향하는 포인터'라는 개념 자체(notion of a pointer into yourself at all)가 없기 때문입니다. 코드의 각 부분은 각개격파로 떼어 놓고 컴파일러가 심사했을 때 *기술적으로는* 모두 올바른 상태입니다. (우리는 push와 pop을 *단 한 번만* 호출할 수는 있습니다.) 하지만 그 직후, 우리가 창조해낸 저 모순 덩어리 존재(absurdity of what we created)가 활동을 개시하며 시스템의 모든 것을 *마비(locks up)* 시켜버립니다.

우리가 짠 이 코드가 어딘가에는(some use) 써먹을 데가 있을지도 모르겠지만, 적어도 제 생각(I'm concerned)에는 그저 문법적으로만 유효한(syntactically valid) *쓰레기 헛소리(gibberish)* 에 불과합니다. 우리는 지금 이 리스트가 `'a` 라는 수명을 품고 있고, `push` 와 `pop` 메서드가 바로 그 수명 내내 `self`를 쭉 빌려가(borrows *self*) 묶어둔다고 우기고 있습니다. 그건 진짜 이상한(weird) 상황이지만, Rust 컴파일러 입장에선 코드 각 구획을 쪼개서 개별적으로 심사할 땐(can look at each part .. individually) 딱히 당장 어긋나는 규칙을 발견하지 못합니다(doesn't see any rules being broken).

하지만 우리가 리스트를 실제로 *사용(use)* 하려고 시도하는 그 즉시, 컴파일러는 곧바로 철퇴를 내립니다. "그래, 네가 `self`를 `'a` 기간 전체 동안 가변으로 빌려갔지(borrowed `self` mutably for `'a`)? 그러니 넌 이제 그 `'a` 기간이 끝날 때까지 다신 `self`에 손도 못 대(can't use `self` anymore)." 거기에 그치지 않고 쌍싸대기(but *also*)를 더 날리죠. "근데 네 리스트 안에 `'a` 딱지 영수증이 들어있으니, 이 수명은 당연히 전체 리스트의 존재 수명 전체(entire list's existence) 기간 중 내내 강제로 유효하게 유지되어야만 해(must be valid for)."

이건 거의 완전한 모순(nearly a contradiction)인데, 딱 한 가지 돌파구(one solution)가 존재하긴 합니다: 당신이 `push`나 `pop`을 한 번이라도 호출하는 그 즉시(as soon as), 이 리스트가 스스로를 제자리에 얼려버려서 단단하게 고정 박제 핀(pinned) 상태로 만들어 버리는 겁니다. 리스트는 비유를 들자면 영원 속에 자기 꼬리를 삼켜버린(swallowed its own proverbial tail) 채, 더 이상 그 누구도 접근하지 못하는 승천한(ascended) 꿈의 세계(world of dreams)로 넘어가 버립니다.

> **해설자**: 이 책의 초판이 처음 씌어졌을(first written) 당시에만 해도 세상에 존재하지 않았던 개념이었지만(didn't exist), 추후 언어 업데이트를 통해 놀랍게도 Rust는 이 말뚝 박힌 핀(Pin)이라는 형이상학적 허상 개념을 실제로 꽤나 유용한 실전 도구로(something useful) 정식 규합 채택 공식 도입해 버렸습니다([공식 핀(Pin) 문서 참고][pin])!
> 이 핀 개념은 아마도 유구한 Rust 역사에서 그 *차용 검사기(the borrowchecker)* 도입 이래로, 이 언어 생태계 전반에 들이닥친 가장 거대하고 복잡한 핵기괴 도입 기능 스펙(most complex addition to the language)일 겁니다. 하지만 우린 우리 리스트가 그렇게 얼어버린 박제가 되길(pinned) *원하지 않습니다*!
>
> 핀(Pins)은 사실 비동기 async-await 나 퓨처(futures), 코루틴 등에선 무척 필수적이고 유용하게 쓰입니다. 컴파일러가 어떤 함수 내부의 모든 지역 변수들을 고스란히 싹 긁어모아 특정 구조체 덩어리 형태로 정지 보관시켜 둔 다음 퓨처가 재개될 때까지 어디 짱박아둬야 하기 때문이죠. 이때 지역 변수들끼리 서로를 교차 참조할 수 있고 우린 그걸 제대로 *동작하게* 만들어 줘야 하니, 이런 백그라운드 구조체들은 결국 내부적으로 자기 자신을 가리키는 내부 참조자(references to themselves)를 필연적으로 포함하게 됩니다!
>
> 그래서 `await`나 `yield`를 안전하게 하기 위해 Rust는 이 고정된 박제 상태(pinned)를 제대로 묘사하고 다룰 전용 규약 장치가 필요했습니다. 다행히 이 모든 복잡한 병크 짓거리들은 그저 암묵적인 컴파일러 자동 기계 장치(automatic compiler machinery) 뒤에 꼭꼭 숨겨져 있어서, 일반적인 평범한 상황에선 그 누구도 `Pin`이나 퓨처의 내부 심연을 직접 파헤쳐 고민할 필요가 없습니다. 단, tokio 같은 거대한 비동기 *런타임(runtimes)* 엔진 자체를 직접 설계하고 깎는 장인분들은 예외적으로 엄청 신경 써야 합니다.
>
> 우리는 이 책에서 비동기 런타임을 짤 게 아닙니다. 제 주변 지인들은 평소 `Pin`을 이용해 온갖 별의별 "기발하고" (맛이 간) 기묘한 마술 *트릭(tricks)* 들을 쓸 수 있다고들 입을 모으지만, 제가 봤을 땐 차라리 제 평생 그걸 모르고 사는 게(happier to just not know them) 정신 건강에 이로운 것 같습니다. 전 앞으로도 쭈욱 Pin 타입 따윈 이 세상에 애초에 없었으며 걘 절대 내 인생을 해칠 수 없다고 스스로 세뇌하며 살아갈 작정입니다.

우리의 저 `pop` 구현부를 뜯어보면, 내부에 자기 자신의 침입 참조자를 넣는 게 왜 정말 위험할 수 있는지 살짝 단서(hints)를 줍니다:

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

만약 우리가 저 초기화 `None` 클리닝 작업을 깜빡하고 안 했다면 어찌 될까요(forgot to do this)? 그럼 우리의 꼬리는 *이미 리스트 밖으로 떨어져 나가 버린(had been removed from the list)* 엉뚱한 폐기 노드 시체를 계속 붙잡은 채 가리키고 있게 됩니다. 그 노드 시체는 즉시 버려져 해제될 텐데, 그럼 필연적으로 빈 허공을 가리키는 댕글링 포인터 무덤이 되어 프로그램이 터지겠죠. 그리고 Rust의 존재 의의는 원래 우릴 이런 상황에서부터 멱살 잡아 구출해 보호하는 것이고요!

그리고 맞습니다, Rust는 실제로도 우린 그런 끔찍한 위험 구덩이 지옥에서 무사히 안전 방어 커버 보호를 해냈습니다. 단지 아주... 심하게 꼬인 지옥 빙빙 돌아가는(roundabout) 억지 땡벌 모순 방식을 써서 막았을 뿐입니다.

자 그럼 어떡해야 할까요? 여기서 또다시 그 이중 포장 끔찍 악몽 족쇄 지옥 `Rc<RefCell>>` 감옥으로 유턴하라고요?

부디. 살려. 제발.

아뇨, 대신 우리는 정상 궤도 궤도를 이탈해(off the rails) 완전 돌변해서 그 위험한 *원시 포인터(raw pointers)* 마개조 영역에 몸을 던질 겁니다.
우리의 새 레이아웃 꼬라지는 이제 이따위가 될 겁니다:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // 위험, 대량 살상 위험 구역 (DANGER DANGER)
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

자, 여기까지입니다. 여긴 더 이상 저 연약 쫄보 동적 참조자 안전 그물 렌탈 심사역(reference-counted-dynamic-borrow-checking) 나부랭이 찌끄러기 같은 헛소리 따윈 통용되지 않습니다! 레알 생. 존나게 하드. 무방비 프리패스 검문 제로. 진짜 생 포인터(Real. Hard. Unchecked. Pointers).

> **해설자:** 훗날 역사가 증명하건대, 사실 저 코딩 구현본 디자인조차도 여전히 끔찍할 만치 심각하게 뒤틀린 오류 시한폭탄 위험을 내포한 불완전 사생아였지. 하지만 아직은 이 오만방자 녀석이 그 피눈물의 쓰라린 교훈을 깨닫기엔 시기상조일세. 다음 진도 섹션에서 예전처럼 처절히 맨땅에 헤딩하며 하드 투쟁 모드로 깨달아갈 테지.

오케이 모두 C언어 생태계 야만인 마인드로 돌변합시다(Let's be C everyone). 매일매일 C처럼 살아갑시다.

전 기꺼이 그 고향에 입성할 겁니다(I'm home). 전 이미 각오 되었습니다(I'm ready).

안녕 `unsafe` 야.

> **해설자:** 와, 진짜로 여기서 이 저자 양반의 오만함 허세(hubris)가 정점을 찍었구만.

[pin]: https://doc.rust-lang.org/std/pin/index.html
