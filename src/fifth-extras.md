# 기타 잡동사니 (Extra Junk)

이제 `push`와 `pop`을 다 작성했으니, 놀랍게도(weirdly) 나머지 부분은 스택 경우와 완전히 똑같습니다. 리스트 길이를 바꾸는 연산만이 `tail` 포인터를 건드릴 필요가 있기 때문이죠.

하지만 물론 이제 모든 게 안전하지 않은 포인터(unsafe pointers)가 되었으니 코드를 포인터를 사용하도록 고쳐야 합니다! 그리고 어차피 코드를 다 건드려야 할 바엔 겸사겸사 우리가 놓친 게 없는지 쭉 돌아보는 것도 좋겠습니다.

어쨌든, 스택 구현에서 복붙부터 시작해 봅시다:

```rust ,ignore
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

`IntoIter`는 괜찮아 보입니다만, `Iter`와 `IterMut`는 우리 타입 안에 더 이상 안전한 포인터(safe pointers)를 두지 않겠다던 단순 원칙을 정면으로 위반하고 있습니다. 일단 안전빵으로(safe) 얘들을 원시 포인터(raw pointers)로 바꿔주죠:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

좋아 보입니다!

```text
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

안 좋아 보이네요(Doesn't look good)! 얘들이 떠드는 이 [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)가 대체 뭡니까?

> `T`를 소유한 "것처럼 행동하는(act like)" 것들을 표시하기 위해 사용되는 크기가 0인 타입(Zero-sized type)입니다.
>
> 어떤 타입 안에 `PhantomData<T>` 필드를 추가하면, 설사 실제론 그렇지 않더라도 이 타입이 마치 타입 `T`의 값을 저장하고 있는 것처럼 행동한다고 컴파일러한테 알려주게 됩니다. 이 정보는 특정한 안전 기능 속성(safety properties)들을 계산할 때 쓰입니다.
>
> `PhantomData<T>`를 사용하는 방법에 대한 더 깊은 설명은 [노미콘(the Nomicon)](https://doc.rust-lang.org/nightly/nomicon/)을 참조하시기 바랍니다.

어이 너무 성급하게 굴지 마십쇼, 우린 지금 *제가* 쓴 책을 읽고 있는 중이거든요. 웬 엄청 구리구리한(huge *nerd*) 녀석이 갈겨썼을 게 뻔한 저 딴 책 말고요! 내기하건대 쟤네가 데이터 구조 책 같은 걸 쓴다고 치면 기껏 나오는 게 기껏해야 정말 지루한 배열 스택(Array Stack) 따위나 만들고 앉아있지 이 연결 리스트(Linked List) 같은 건 절대 *안(not)* 만들 겁니다.

> 사용하지 않는 수명 매개변수 (Unused lifetime parameters)
>
> 아마도 `PhantomData`의 가장 흔한 사용 사례는 보통 안전하지 않은 코드(unsafe code)에서 수명 매개변수(lifetime parameter)를 쓰지 않은 채 남겨둔 구조체일 것입니다.

아하, 그러니까 우리 타입에 떡하니 수명을 이름표로 붙여놨으면서 실제론 하나도 가져다 쓰지 않았다는 소리군요. 그렇다면 `PhantomData` 길을 갈 수도 *있(could)*겠지만, 전 이걸 진짜로 *진짜 심각하게(really)* 필요하게 될 다음 챕터의 이중 연결 리스트(doubly-linked list)를 위해 킵(save)해 두고 싶습니다.

우리는 지금 꽤 흥미로운 상황에 놓여 있습니다. 사실 우리한테 `PhantomData` 따윈 없어도 됩니다. *라고 전 생각합니다.* 일단 구라핑을 치고(claim that) 나중에 문제가 없다고 우겨본 다음, 제일 마지막에 Miri가 쌍욕을 날리면 그때서야 쿨하게 인정하고(concede the point) `PhantomData`를 쓰는 짓을 해볼까 합니다.

자 우리가 지금부터 진짜 할 작정인 짓은, 도로 저 반복자(Iterator) 타입 안에 떡하니 안전 참조자들을 밀어 넣고 여전히 안전 참조자를 사용할 수 있다는 사실을 감사히 여기는 겁니다. 저는 이게 구조적으로 완벽하게 안전(sound)하다고 주장하는데요, 왜냐하면 여러분이 반복자를 사용할 때는 어떤 식으로든 나름의 적절한 중첩 구조(proper nesting) 규칙이 지켜지기 때문입니다: 여러분은 우선 반복자를 하나 창조하고, 한참 동안 거기에 매달린 안전 참조자(safe references)들을 지지고 볶아 써먹다가, 마침내 반복자를 형편없는 것통에 폐기 처분하니까요. 

오직 그 반복자 녀석이 완전히 소멸된 다음에야 비로소 당신은 당신의 리스트 그 본체 덩어리에 돌진해서 꼬리표(tail pointer)나 그 안쪽의 Box들 구석구석을 훼집고 다녀야 하는 `push`나 `pop` 짓거리를 마음 놓고 지를 수 있습니다. 뭐 그 중간 순환 반복 공정 과정 와중에(during the iteration), 우리가 한 움큼 원시 포인터들을 역참조로 빙빙 돌려 갈아대긴 *할 건데(are)*, 이 때문에 약간 뒤죽박죽 쓰까스까 섞임(mixing) 오염 짬뽕이 일어날 순 있지만, 이것도 까놓고 보면 그냥 그 무법 원시 포인터들에서 새로 정제 파생된 재차용(reborrows) 짬뽕 국물 쯤으로 가볍게 치부해 버릴 수 있으니까 말이죠.

뭐 사실 *저조차도* 이 논리에 100퍼센트 무한 신뢰 완벽 설득 맹신 확신 납득(convinced)하진 않지만, 그래도 그냥 일단 던져보고 뭐가 튀어나오나 구경이나(see) 해보고 싶을 뿐입니다!

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

만약 우리가 저 잘난 참조자(references)들을 창고에 차곡차곡 쌓아(storing) 둘 심산이라면, 애초에 원시 포인터 따위를 참조자 같은 것로 감싼 옵션 덩어리(options-of-references)로 개조 업그레이드(upgrade) 시키는 조치를 해줘야 합니다. 우리가 손수 일일이 널(null) 포인터 검사를 갈겨줄 수도 *있겠지만*, 이 대목만큼은 그 지저분하고 천박한 [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1)랑 [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut) 따위의 메서드에 기대도 무방할 엄청나게 극소하게 드물디드문 희박한 예외 조항(narrow cases) 중 하나라고 *전 생각합니다(think)*.

전 *보통은(usually)* 이딴 쥐약 같은 메서드 근방 몇 미터 내에 얼씬도 하지 말고 코로나(the plague) 피하듯 극혐 기피하라고 열혈 설파해왔습니다, 왜냐면 이것들이 하는 짓거리 자체가 상식을 파괴하는 뒷골 서늘한 역대급 기괴 엽기 짓(surprising and nasty stuff) 투성이인데다, 애초에 내가 세운 최강 절대 존엄 가이드 원칙 "쉬운 규칙: 참조자(references) 따위 근처에도 가지 마!"를 정면으로 배신 뒷통수 까재끼면서 도로 우회 불법 수입해오는(reintroducing) 쳐죽일 짓이기 때문이죠!

이 메서드 녀석 설명서 끄트머리에 지뢰 폭탄만 한 주의 무시무시 경고 협박 살벌한 조항(warnings)들이 수두룩 빽빽하게 붙어있는데, 그중 단연 압권 흥미 대박인 구절은 이겁니다:

> 도출되어 나온 반환 수명(lifetime) `'a`는 그저 임의로 무단 맘대로 꼽사리 지정된(arbitrarily chosen) 허상에 불과하며 원래 원본 데이터 본체의 실제 찐 생명 주기 연한을 무조건 절대 대변 보장 보증 투영 반영(reflect)하진 않으므로, 여러분은 스스로 독박을 쓰고 목숨 걸고 Rust의 살벌한 앨리어싱 규칙을 칼같이 지켜야 합니다(enforce). 특히, 저 임의 발급된 허상 수명 연장(duration of this lifetime)의 효력이 살아 숨 쉬는 그 절체절명 순간 동안에는, 이 포인터가 가리키고 꽂혀있는 그 기억장치 본체 메모리 과녁 자리에 행여라도 지구상 그 어떤 또 다른 외부 포인터 찌꺼기 외부 접근 침투로 무단 접속 접근 간섭 접근 기입 난입(accessed (read or written)) 따위가 일체 조금도 기어들지 않도록 철통같이 절대 방어해야(must not get accessed)만 합니다.

허참 세상에나 맙소사, 이보쇼 우리 아까 전에 무려 25페이지만큼이나 질리도록 입에 단내나게 박 터지게 물고 뜯고 떠들었던 바로 그 화두 주제(the thing we talked about) 아닙니까! 저는 아까 미리 제가 제 입으로 우린 여기서 이 예의 바른 참조자를 갈겨 쓰는 행동이 *어찌 됐건 백 퍼 무조건 킹왕짱 안전 무결점 결백 무사 통과 될 겁니다(definitely gonna be fine)* 하고 먼저 선빵 밑밥 선수 치며 공언(asserted) 해놨었죠, 고로 앨리어싱(aliasing) 관련 심판 협박 트러블 논란 따위도 당연히 이로써 완벽 종결 파훼(aliasing solved!) 해결 처리 성사 타파 끝! 나머지 남은 개뼈다귀 같은 악당 지뢰(evil part) 파트는 대충 함수 선언부 생김새 모습(signature) 였습니다:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

저 수명 생태 주기 따위가 함수 입력 입력값 조달원(input)이랑 단 1도 상종이나 종속 연결 고리 얽힘 연관 개연(isn't attached)이 없는 거 보입니까, 왜냐하면 애초에 저기 `self`가 통째로 복사 이관 다이렉트 값 전달(by-value)이기 때문이에요? 뭐 그렇습니다 이게 바로 우리가 흔히 부르길 "무소불위 영생 족쇄 없는 생명(unbounded lifetime)"이라 떨며 지칭하는 재앙 악몽 폭탄(nasty stuff) 덩어리란 겁니다. 이 빌어먹을 생존 주기 타이머 녀석은 우리가 요구하면 달나라로 무한 영생 확산 시공간 제약 초월 확장 영구 보존 존버 뻗치기 무한 연장 존장('static'까지!)도 기꺼이 척척 코스프레 위장 놀이(pretend to be)를 기꺼이 수용해 줍니다! 이 황당한 영원불멸 수명 덩어리 녀석를 다루기 위해(deal with that) 우리가 흔히 단골로 즐겨 써먹는 전통 통제 억제 견제 기법(way)이라 함은, 그냥 이 불사신을 어떻게든 안전 철창 우리 벽 바리케이드 통제 제한 경계 그물(bounded)망 내부로 꽁꽁 잡아 밀어 구겨 처박아버리는 수밖에 없습니다. 이게 대체 무슨 소리냐면, 대충 "그냥 함수에서 이 포인터를 정말 지극 정성 최대한 신속 속성 재빨리 밖으로 뱉어 토해 패스 리턴(return this)해버리란 소리입니다. 그럼 그 함수 모습 틀(signature)이 알아서 저 괴물을 쌈 싸 먹고 목줄 쥐어 통제(limits it)해줄 테니까"요.

하 젠장 이쯤 되니 나 진짜 정말 오싹 지저스 초조 식은땀(nervous) 범벅이지만 언제나 그렇듯 노빠꾸 풀악셀(keep pushing through) 닥돌 강행 ㄱㄱ! 일단 저기다 스택(stack) 동네에서 스리슬쩍 소매치기 훔쳐 약탈 긴빠이(steal) 쳐온 반복자(iterator) 구현체들을 갈아 끼워 넣어보죠:

```rust ,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

운명의 진실 게임 타임마...(Moment of truth time...)

```text
cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

예스(YES)!!! 봤냐 **해설자** 녀석야! 가끔은 나도 완벽할 때가 있다고!

> **해설자**: 하지만 애초에 이 책의 취지 자체가 일부러 실수를 저지르며 독자들에게 가르침을 주는 거 아니었습니까.

아 젠장 그래 어떨 때는 그 교훈이란 게 사실 내가 맞았고 니들 젠장 전원 다 내가 안전하지 않은 코드(UNSAFE CODE)에 대해 지껄일 땐 닥치고 아가리 여물고 내 말을 쳐 들어야 한다는 걸 깨닫는 거라고! 왜냐고? 내가 그동안 이 빌어먹을 반복자 구현체의 안전성(SOUNDNESS OF ITERATOR IMPLEMENTATIONS) 하나 때문에 내 인생 정말 귀한 시간 다 꼬라박으면서 머리 쥐어뜯으며 밤새 고민했으니까?! 오케이?! 오케이.

어쨌든 여기 `peek`이랑 `peek_mut` 대령이오.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref()
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut()
    }
}
```

이건 테스트도 안 돌릴 겁니다. 왜냐, 이제 난 실수 따위 절대 안 하니까.

> **해설자**: `cargo build`

```text
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`

```

좋습니다 젠장(FINE).

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

안타깝지만 전 앞으로도 실수를 밥 먹듯 *만들어 댈(continue)* 팔자인 것 같네요. 그러니 한 치의 빈틈도 허용하지 않도록 "미리 먹이(miri food)"라는 이름의 아주 살벌한 테스트를 하나 던져 주기로 합시다. 이 녀석의 목적은 우리 API들을 온갖 해괴망측한 순서로 마구마구 갈아 넣고 뒤죽박죽 비벼대면서(mixes up), Miri 님이 친히 우리 실수 쪼가리들을 색출해 낼 수 있게 상다리를 휘어지게 차려주는 역할입니다.

```rust ,ignore
#[test]
fn miri_food() {
    let mut list = List::new();

    list.push(1);
    list.push(2);
    list.push(3);

    assert!(list.pop() == Some(1));
    list.push(4);
    assert!(list.pop() == Some(2));
    list.push(5);

    assert!(list.peek() == Some(&3));
    list.push(6);
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&30));
    assert!(list.pop() == Some(30));

    for elem in list.iter_mut() {
        *elem *= 100;
    }

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&400));
    assert_eq!(iter.next(), Some(&500));
    assert_eq!(iter.next(), Some(&600));
    assert_eq!(iter.next(), None);
    assert_eq!(iter.next(), None);

    assert!(list.pop() == Some(400));
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&5000));
    list.push(7);

    // 그냥 바닥에 내팽개쳐서 소멸자(dtor)가 진짜 열일하는지 구경합시다
}
```


```text
cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

완벽해(Perfect).
