# 스택 할당 연결 리스트 (The Stack-Allocated Linked List)

이 책은 대체로 *힙(heap) 할당* 연결 리스트에 집중하고 있습니다. 왜냐면 그게 현실에서 제일 만만하고 실용적이니까요. 하지만 우리가 무조건 힙 할당만 써야 한다는 *법칙*은 없습니다. 힙 할당은 동적으로 메모리를 찍어내기 참 편하다는 장점이 있죠. 반면 스택 할당은 이런 쪽에선 영 불친절합니다 &mdash; C 언어의 `alloca` 같은 놈들은 범세계적으로 '개저주받은 쓰레기 문제아(Very Cursed And Problematic)' 취급을 받고 있죠.

그러니 우린 스택에 메모리를 할당하는 훨씬 쉬운 꼼수(easy way)를 쓸 겁니다: 바로 함수를 호출해서 여유 공간 낭낭한(more space) 새 스택 프레임을 공짜로 타내는 겁니다! 이건 우리의 문제를 해결하는 존나 병신 같고 멍청한(silly) 해결책이긴 하지만, 의외로 찐으로 실용적이고 쓸만하기도 합니다. 솔직히 여러분도 평소에 이게 연결 리스트인지 눈치채지도 못한 채 숨 쉬듯이 맨날 쓰고 있는 기법입니다!

재귀적으로 뭔가 삽질(recursively)을 할 때마다, 당신은 걍 현재 단계의 상태(state)를 가리키는 포인터를 다음 단계로 쑥 밀어 넣어버리면 그만입니다. 만약 그 포인터 자체가 상태의 *일부분(part)* 이 된다면? 축하합니다, 방금 당신은 스택 영역에 둥지를 튼 어엿한 연결 리스트를 하나 창조하신 겁니다!

물론 다들 아시겠지만 지금 우린 이 책의 *멍청한 짓거리(silly)* 파트를 진행 중이니, 이 짓거리도 아주 멍청한 꼬라지로(silly way) 해볼 겁니다: 연결 리스트 놈을 메인 주인공(star)으로 만들어 버리고, 유저의 불쌍한 코드를 전부 콜백 지옥(swamp of callbacks)의 늪에 강제로 쓸어 넣어버리는 거죠. 누구나 달콤한 중첩 콜백 지옥(nested callbacks)을 사랑하잖아요!

우리의 List 타입은 걍 다른 Node를 가리키는 참조자 하나 달랑 쥐고 있는 Node가 될 겁니다:

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

그리고 이 녀석이 쓸 수 있는 연산은 오직 `push` 하나뿐입니다. 이 함수는 옛날 리스트 잔해(old list), 현재 노드에 박아 넣을 상태값(state), 그리고 콜백 함수 하나를 집어삼킵니다. 새로 탄생한 리스트는 콜백 함수의 뱃속에서 빚어질 겁니다. 덤으로 콜백 녀석이 무슨 값이든 자유롭게 뱉어낼(return any value) 수 있게 허락해 주고, `push`가 임무를 다 마치면 그 값을 릴레이 하듯 던져주게 만들 겁니다:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&List<'a, T>) -> U,
    ) -> U {
        let list = List { data, prev };
        callback(&list)
    }
}
```

이게 답니다! 우린 이걸 대충 이렇게 써먹을 수 있습니다:

```rust ,ignore
List::push(None, 3, |list| {
    println!("{}", list.data);
    List::push(Some(list), 5, |list| {
        println!("{}", list.data);
        List::push(Some(list), 13, |list| {
            println!("{}", list.data);
        })
    })
})
```

눈물 나게 파멸적이네요. 😿

유저들은 이미 `while-let` 구문을 써서 `prev` 값들을 징검다리 밟듯 밟고 지나가며(walk over) 이 리스트를 뒤적거릴(traverse) 수 있습니다만, 걍 재미 삼아(just for fun) 우리가 늘 하던 대로 반복자(iterator) 하나 깎아봅시다:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev;
            &node.data
        })
    }
}
```

어디 구운 맛 좀 볼까요(test it out):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.iter().copied().sum::<i32>(), 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
                })
            })
        })
    }
}
```

```text
> cargo test

running 18 tests
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::basics ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test second::test::iter_mut ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test second::test::peek ... ok
test third::test::iter ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

자 이쯤 되면 아마 머릿속에 이런 개같은 호기심(wonder)이 피어오를 겁니다. "어이, 노드 속에 처박힌 데이터 좀 가변(mutate)으로 주무르면 안 됨?". 어쩌면요(Maybe)! 이참에 불변 참조 나부랭이(shared ones) 대신 상남자답게 가변 참조(mutable references)를 써먹도록 리스트를 뜯어고쳐 봅시다:


```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a mut List<'a, T>>,
}

pub struct Iter<'a, T> {
    next: Option<&'a List<'a, T>>,
}

impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a mut List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&mut List<'a, T>) -> U,
    ) -> U {
        let mut list = List { data, prev };
        callback(&mut list)
    }

    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev.as_ref().map(|prev| &**prev);
            &node.data
        })
    }
}

```


```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:47:32
   |
46 |  List::push(Some(list), 13, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
47 |      assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:45:28
   |
44 |  List::push(Some(list), 5, |list| {
   |                             ----
   |                             |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |      assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here


<ad infinitum>
```

아이고 맙소사(Whelp). 이 새끼가 우리 반복자(iterator)를 좆같아하는 눈치네요. 우리가 반복자 코딩을 조져놨을 수도 있겠죠(messed that up)? 걍 테스트를 살짝 가볍게 덜어내서(simplify) 팩트 체크 좀 들어가 봅시다:


```rust ,ignore
#[test]
fn elegance() {
    List::push(None, 3, |list| {
        assert_eq!(list.data, 3);
        List::push(Some(list), 5, |list| {
            assert_eq!(list.data, 5);
            List::push(Some(list), 13, |list| {
                assert_eq!(list.data, 13);
            })
        })
    })
}
```

```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:46:17
   |
44 |   List::push(Some(list), 5, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |       assert_eq!(list.data, 5);
46 | /     List::push(Some(list), 13, |list| {
47 | |         assert_eq!(list.data, 13);
48 | |     })
   | |______^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:44:13
   |
42 |   List::push(None, 3, |list| {
   |                        ----
   |                        |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
43 |       assert_eq!(list.data, 3);
44 | /     List::push(Some(list), 5, |list| {
45 | |         assert_eq!(list.data, 5);
46 | |         List::push(Some(list), 13, |list| {
47 | |             assert_eq!(list.data, 13);
48 | |         })
49 | |     })
   | |______________^ `list` escapes the closure body here
```

흠 아니네요, 코박죽 수준의 뜨거운 쓰레기(hot garbage) 폭발은 여전합니다.

진짜 팩트 폭력(problem)은, 우리 리스트가 사실 (다분히 의도적으로😉) *가변성(variance)* 이라는 마법에 빌붙어 연명하고(relying on) 있었다는 겁니다. [가변성(Variance)은 존나게 복잡한 심연의 주제지만](https://doc.rust-lang.org/nomicon/subtyping.html) 걍 무식하게 단순화해서(simplified terms) 야부리를 좀 털어보죠:

리스트에 속한 각 놈들은 놀랍게도 *자기 자신과 완전히 똑같은 타입(exact same type)*의 List를 가리키는 참조자를 품고 있습니다. 저 가장 안쪽 심연에 처박힌(inner-most) 리스트 새끼 입장에선, 바깥의 모든 리스트들이 자기랑 완벽히 똑같은 수명(same lifetime)을 갖는 동급생처럼 보일 겁니다. 하지만 이건 *객관적으로(objectively)* 좆구라입니다: 각 리스트의 노드들은 무조건 자기가 품은 다음 놈보다 명백히 구질구질하게 더 오래 살아남습니다(strictly longer). 왜냐고요? 애초에 중첩 스코프(nested scopes) 감옥 구조로 찍어냈잖아요!

그럼... 왜 아까 불변 참조(shared references) 쓸 때는 컴파일러 새끼가 아무 말 없이 통과시켰던 걸까요? 왜냐면 컴파일러 놈팽이는 수명이 "드럽게 긴(too long)" 녀석을 훔쳐 쓰는 게 대부분의 경우 안전하다는 걸 짬바로(in many cases) 체득하고 있기 때문입니다! 우리가 리스트 참조자를 다음 놈 혓바닥 위에 슬쩍 올려둘 때마다, 컴파일러 새끼는 뒤에서 조용히 수명(lifetimes)을 "쥐어짜서(shrinking)" 새 리스트 놈이 안심하고 집어삼킬 크기(fit)로 맞춰줍니다. 이렇게 수명을 억지로 쥐어짜는 마술쇼를 우리는 *가변성(variance)* 이라 부릅니다.

이건 쉽게 말해 상속(inheritance) 개념 좀 만져본 놈들이 `Animal` (동물, 부모 타입) 요구 사항에다 무지성으로 `Cat` (고양이)을 들이미는 꼼수(trick)랑 완벽히 일치합니다. 대가리가 굴러간다면 `Animal` 찾는 자리에 `Cat`을 던져줘도 아무 문제 없단(fine) 걸 직감할 겁니다. 애초에 고양이는 동물에다 *"뭔가 귀여운 옵션 좀 더 붙은(and more)"* 녀석이니까요. 잠시 그 "뭔가 옵션 좀 더 붙은" 사실을 머릿속에서 포맷(forget) 해둬도 세상 안 무너지잖아요, 그렇죠?

비슷한 맥락으로, 커다란 수명(larger lifetime) 이란 걍 개미만 한 수명(smaller lifetime)에 *"수명 옵션 더 붙은(and more)"* 것과 같습니다. 고로 여기서도 그 "수명 옵션 더 붙은" 걸 쿨하게 치매 온 듯 까먹어도(forget) 세상 평화롭단 뜻입니다!

근데 님들은 십중팔구 여기서 띠용? 할 겁니다: 씨발 그래서 가변 참조(mutable reference) 버전은 왜 이따구로 쳐 망한 건데요!?

왜냐하면 말입니다, 가변성 마술쇼가 *매번* 안전할 거란 건 니들의 귀여운 착각(isn't always safe)이기 때문입니다. 진짜 만약에 우리 병신 코드가 컴파일*돼버렸다면(did compile)*, 우린 아래처럼 메모리 해제 후 사용(use-after-free)의 재앙을 손수 빚어낼 수 있었을 겁니다:

```rust ,ignore
List::push(None, 3, |list| {
    List::push(Some(list), 5, |list| {
        List::push(Some(list), 13, |list| {
            // HAHAHA all the lifetimes are the same, so the compiler will
            // let me rewrite my parent to hold a mutable reference to myself!
            // I will create all the use-after-frees!!
            *list.prev.as_mut().unwrap().prev = Some(list);
        })
    })
})
```

세세한 사실들을 치매 걸린 듯 까먹을 때의 가장 좆되는 문제점(problem) 은 바로, *다른 어딘가에선 아직 그 세세한 팩트들을 또렷이 기억하며(remember those details) 그게 변함없이 사실일 거라고 맹신(expect) 할 수도 있다*는 겁니다. 특히 여기에 *돌연변이 가변성(mutation)* 같은 치명적인 독을 타는 순간 이건 진짜 빼박 재앙(very big problem) 이 됩니다. 만약 여러분이 좆도 신경을 안 쓴다면, 우리가 지워버린 "옵션 좀 더 붙은(and more)" 사실을 기억조차 못 하는 머저리 코드 놈이, 여전히 진실을 "기억하며" 그 "옵션"이 거기 남아있을 거라고 굳게 *믿는(expect)* 구역에 와서 지 맘대로 똥을 싸질러도(write things) 괜찮다고 깝칠 수 있거든요.

객체지향 상속 나부랭이의 언어로 번역해 드리자면, 지금 이 코드는 무조건 깜빵(illegal)에 가야 합니다:

```rust ,ignore
let mut my_kitty = Cat;                  // 고양이를 한 마리 연성한다 (수명이 긺)
let animal: &mut Animal = &mut my_kitty; // 이게 고양이란 치명적 사실을 포맷시킴 (수명을 강제로 쥐어짬)
*animal = Dog;                           // 뜬금없이 개새끼로 둔갑시켜 덮어씀 (수명 짧음)
my_kitty.meow();                         // 냥냥 짖어대는 개새끼 탄생! 우효옷! (Use After Free 메모리 터짐)
```

그래서 결론을 말하자면, 여러분이 가변 참조의 수명을 칼로 싹둑 잘라내는(shorten) 것 자체는 *가능합니다만(can)*, 그딴 짓거리들을 *중첩(nesting)* 해서 엮기 시작하는 순간 세상 모든 게 "불변(invariant)" 상태로 돌변하며, 더 이상 컴파일러 성님이 수명 칼질을 허락해주지(not allowed to shorten) 않습니다.

조금 그럴싸하게 전문적으로 말하자면, `&mut &'big mut T` 껍데기는 절대 `&mut &'small mut T` 찌끄레기로 강등(converted) 될 수 없습니다 (`'big` 수명이 `'small` 수명보다 무식하게 크다는 가정하에요). 이걸 학회에서나 쓸법한 우아한(more formally) 혀를 굴리며 말하자면, `&'a mut T`는 `'a` 에 대해선 공변적(covariant) 이지만 `T` 에 대해서는 무조건 불변적(invariant) 이어야 합니다.

TMI 하나 투척(Fun fact): Java 이 미친놈들은 놀랍게도 여러분이 저딴 미친 짓거리(this kind of thing)를 대놓고 저지를 수 있게 *허락수용(lets)* 해줬습니다. 대신 냥냥 짖어대는 개새끼가 탄생하는 대참사를 막기 위해 [런타임에 일일이 꼽을 주며 검사(runtime checks)](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html) 하는 쌉비효율적인 짓거릴 하고 있죠.

----

그럼 이딴 망할 상황에서 대체 어떻게 해야 데이터를 주무를(mutate) 수 있을까요? 정답은 꼼수 내부 가변성(interior mutability)을 쓰는 겁니다! 이걸 통해 우리는 컴파일러 바보 병신에게 우린 걍 *데이터(data)* 만 순수하게 더럽히고 싶을 뿐이고, 잘 굴러가는 참조자(references) 구조는 맹세코 털끝 하나 안 건드릴 거라고 사기(tell)를 칠 수 있습니다.

걍 타임머신을 타고 불변 참조(shared references) 구데기를 쓰던 예전 버전(previous version) 코드로 런 쳐서 롤백(revert back) 해버립시다. 그리고 새 테스트 빵틀에다 `Cell` 한 방울을 똑 떨어뜨려 줍시다:

```rust ,ignore
#[test]
fn cell() {
    use std::cell::Cell;

    List::push(None, Cell::new(3), |list| {
        List::push(Some(list), Cell::new(5), |list| {
            List::push(Some(list), Cell::new(13), |list| {
                // Multiply every value in the list by 10
                for val in list.iter() {
                    val.set(val.get() * 10)
                }

                let mut vals = list.iter();
                assert_eq!(vals.next().unwrap().get(), 130);
                assert_eq!(vals.next().unwrap().get(), 50);
                assert_eq!(vals.next().unwrap().get(), 30);
                assert_eq!(vals.next(), None);
                assert_eq!(vals.next(), None);
            })
        })
    })
}
```

```text
> cargo test

running 19 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter_mut ... ok
test fifth::test::iter ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test first::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fifth::test::miri_food ... ok
test silly2::test::cell ... ok
test third::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test third::test::basics ... ok
test second::test::iter ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

재귀 파이 먹기처럼 개꿀이네요! ✨
