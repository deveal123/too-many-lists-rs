# 커서 구현하기 (Implementing Cursors)

좋습니다, 여기서부턴 편하게 불변성(immutable) 버전은 싹 다 갖다 버리고 std의 `CursorMut`만 야무지게 구현해 볼 겁니다. 왜냐면 불변성 버전은 솔직히 노잼(isn't actually interesting)이거든요. 제가 처음 제안했던 오리지널 디자인과 똑같이, 여기엔 리스트의 젠장점과 끝점(start/end)을 알리는 `None`을 품은 "유령(ghost)" 원소 하나가 덩그러니 놓여있을 거고, 우린 녀석 위를 당당히 "밟고 지나가는 (walk over it)" 방법으로 리스트의 반대쪽 끝으로 빙글 돌아갈(wrap around) 수 있습니다. 이걸 구현하려면 다음 부품들이 필요합니다:

* 현재 노드를 가리키는 포인터
* 리스트 본체를 가리키는 포인터
* 현재 인덱스(index)

잠깐, 우리가 "유령"을 가리키고 있을 때 인덱스 값은 도대체 뭐가 돼야 하죠? 

*미간을 찌푸리며(furrows brow)* ... *std 코드를 까본다(checks std)* ... *std의 끔찍한 답변을 보고 개정색한다(dislikes std's answer)*

아 오케이, 꽤나 합리적이게도(reasonably) Cursor의 `index`는 `Option<usize>`를 뱉어내는군요. std 구현체는 내부적으로 এটাকে Option으로 저장하는 걸 어떻게든 피해보려고 온갖 잡다한 형편없는 것 짓거리(bunch of junk)를 다 해놨지만... 뭐 젠장 우리는 어차피 문제망한 연결 리스트니까 이 정도는 괜찮습니다. 게다가 std엔 `cursor_front`/`cursor_back` 같은 녀석이 있어서 커서를 처음부터 맨 앞/맨 뒤 원소에 스폰시켜주는 개꿀 직관적인(intuitive) 기능도 있는데, 문제는 리스트가 텅텅 비어있을 땐 이게 또 정말 기괴하게(weird) 동작해야만 한다는 겁니다.

여러분이 꼴린다면 저런 것들도 다 손수 빚어 만드셔도 좋지만, 전 저딴 반복적인 형편없는 것 노가다(repetitive gunk)랑 자잘한 코너 케이스(corner cases)들은 다 갖다 버리고, 걍 깔끔하게 유령 위치에서부터 무조건 시작하는 쌩짜배기(bare) `cursor_mut` 메서드 딱 하나만 만들 겁니다. 사람들은 알아서 `move_next` 나 `move_prev`를 써서 자기가 원하는 곳으로 뚜벅뚜벅 기어가면 됩니다 (물론 여러분이 정 원하신다면 겉에다 저걸 예쁘게 포장해서 `cursor_front`라고 구라 쳐서 내놓으셔도 됩니다).

자, 당장 코딩하러 가보죠:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

아주 직관적이쥬? 우리가 앞서 나열한 불릿 리스트(bulleted list)의 부품명단 하나당 필드 하나씩 정직하게 들어갔습니다! 자 이제 `cursor_mut` 메서드입니다:

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

어차피 우린 유령 위에서 태어나니까, 걍 모든 걸 `None`으로 쿨하게 세팅하고 시작하면 됩니다. 참 쉽고 아름답죠! 다음, 무빙(movement) 타임입니다:


```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }
}
```

그래서 정리하자면 4개의 흥미진진한 케이스가 존재합니다:

* 정상적인 일반 케이스
* 정상적인 케이스지만, 걷다 보니 유령을 밟게 된 케이스
* 이미 유령을 밟고 있던 상태에서, 리스트 맨 앞구멍(front)으로 워프하는 케이스
* 똑같이 유령을 밟고 있는데 리스트가 텅텅 비어있어서 걍 암것도 안 하는(do nothing) 찐따 케이스

`move_prev`도 논리는 1도 안 틀리고 완벽히 똑같습니다. 그냥 앞/뒤(front/back) 방향만 반대로 뒤집고, 인덱스 덧셈 뺄셈만 거꾸로 엎어버리면 그만입니다:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // We're on a real element, go to its previous (front)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // We just walked to the ghost, no more index
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // We're at the ghost, and there is a real back, so move to it!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // We're at the ghost, but that's the only element... do nothing.
    }
}
```

그다음엔 커서 주변에 어슬렁거리는 원소들을 엿볼 수 있는(look at) 염탐 메서드들을 마저 쑤셔 넣어 보죠: `current`, `peek_next`, 그리고 `peek_prev`. **여기서 엄청나게 중요한 주의사항 하나(A Very Important Note):** 이 메서드들은 무조건 우리 커서를 가변 참조 `&mut self`로 차용(borrow)해야만 하고, 거기서 뱉어낸 결과물들 역시 무조건 그 차용(borrow)에 생사가 묶여있어야(tied) 합니다. 절대 유저 이 이상한 사람들이 가변 참조 복사본을 여러 개 복사해서 싸들고 돌아다니지 못하게(multiple copies) 막아야 하며, 그런 불법 참조자를 손에 쥐고 있는 상태에선 우리의 자랑스러운 insert/remove/split/splice API들을 1미리도 건드리지 못하게 원천 봉쇄해야 합니다!

정말 다행스럽게도, rust 컴파일러가 수명 생략(lifetime elision) 규칙을 적용할 때 기본값(default assumption)으로 알아서 이렇게 처리해 주기 때문에, 우린 걍 평소대로 대충 쓰면 숨 쉬듯이 알아서 올바른 결과(right thing)가 튀어나옵니다!

```rust ,ignore
pub fn current(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

머리 비우고 걍 타이핑 칩니다. Option 메서드들과 컴파일러가 알아서 쌍욕 박으며(omitted compiler errors) 토해내는 피드백들이 제 뇌를 대신해서 모든 걸 다 생각(all thinking) 해주고 있습니다. 솔직히 처음에 `Option<NonNull>` 이딴 형편없는 것 같은 짓거리(stuff)를 쓸 때는 긴가민가(skeptical) 했었는데, 와 시바 진짜 이건 걍 코드가 알빠노 자동주행(autopilot) 켜놓은 것처럼 막힘없이 줄줄 써지네요. 그동안 망할 녀석 배열 기반 컬렉션들만 주야장천 짜느라 Option의 달콤함을 제대로 맛볼 길조차 없었는데(never get to use), 와 이거 개미치게 좋네요! (물론 저 `(*node.as_ptr())` 떡칠은 여전히 토악질 나오게 참담(miserable)하지만, 젠장 뭐 Rust 로우 포인터가 다 그런 거죠...)

자 다음은 분실물 선택의 시간입니다: 당장 이 API의 꽃이자 존재 이유 그 자체인 분할(split)과 접합(splice) 코딩으로 바로 다이빙할 건가요? 아니면 단일 원소 삽입/삭제(insert/remove) 따위나 만지작거리며 찐따처럼 아기 걸음마(baby-step)나 뗄 건가요? 왠지 제 뇌피셜로는 결국 삽입/삭제 구현도 나중 가면 분할/병합 로직 갖다 쓰면서 돌려막기(implement in terms of) 할 거 같은 삘링이 강하게 오는데... 걍 남자답게 분할/병합부터 조져보고 운명에 몸을 맡겨 봅시다 (나발이고 타자 치는 이 순간에도 솔직히 전혀 모르겠습니다).




# 분할 (Split)

첫 번째 빠따는 `split_before` 랑 `split_after` 입니다. 이 녀석들은 현재 원소 위치를 기준으로 그 앞/뒤에 있는 내용물을 몽땅 긁어모아다 새로운 LinkedList로 토해냅니다 (정확히는 반대편에서 유령 원소를 만날 때까지 긁어모으는데, 만약 처음부터 유령 위에 올라타 있었다면 걍 리스트 전체 내용물을 통째로(whole List) 헌납하고, 본인 커서는 이제 텅 빈 깡통 리스트를 가리키게 됩니다):

*눈을 가늘게 뜨며(squints)* 오케이 이 녀석 이거 로직이 은근히 만만찮네요(non-trivial). 꼬인 실타래 풀듯이 한 번에 한 땀 한 땀 차분히 입으로 되새김질해가며 풀어나가야겠습니다.

일단 `split_before`를 짤 때 부딪칠 수 있는 재밌는(interesting) 케이스 4가지가 눈에 선하네요:

* 정상적인 일반 케이스
* 정상 케이스이긴 한데, prev가 하필 유령인 케이스
* 처음부터 유령 위에 올라타 있어서, 리스트 전체를 통째로 뱉어내고 본체는 텅 빈 깡통으로 돌변하는 케이스
* 똑같이 유령 위에 있는데 리스트마저 텅텅 비어있어서, 걍 암것도 안 하고 빈 깡통 리스트나 하나 툭 던져주고 땡치는 찐따 케이스

가장자리(corner) 케이스부터 먼저 족쳐보죠. 세 번째 케이스 녀석은 아마 제 생각건대 그냥 이거면 끝일 겁니다.

```rust
mem::replace(self.list, LinkedList::new())
```

그렇죠? 우리 본체는 텅 빈 깡통(empty)이 되고, 우리가 들고 있던 전체 리스트 데이터는 통째로 쿨하게 반환해 버리는 거. 게다가 애초에 우리 필드 값들도 다 `None`이었기 때문에 딱히 귀찮게 재설정 업데이트해 줄 건더기도 없습니다. 캬 달달하구만(Nice). 오, 심지어 이 코드는 덤으로 네 번째 찐따 케이스까지 찰떡같이(Does The Right Thing) 한방에 해결해 주네요!

자, 이제 남은 일반(normal) 케이스들을 조져봅시다... 흠 이건 시각화가 좀 필요하겠는데, 대충 아스키아트 다이어그램 좀 찍어내야겠습니다. 가장 일반적인 베이직 케이스(most general case)의 모습는 대충 이러합니다:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

그리고 우린 이걸 아래의 끔찍한 혼종(this)으로 마개조해야 합니다:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

고로 우린 `cur` 랑 `prev` 사이에 이어진 밧줄을 뚝 끊고(break the link), 그리고... 젠장 세상에 뜯어고쳐야 할 게 산더미처럼 널렸네요. 알겠습니다 걍 제가 이거 어떻게 돌아가는 로직인지 스스로 납득할 수 있게 여러 단계(steps)로 쪼개야겠습니다. 좀 오버해서 말이 길어지는 감이(over-verbose) 있긴 한데 그래도 이렇게 해야 최소한 머리는 돌아가니까요:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // We are pointing at a real element, so the list is non-empty.
        unsafe {
            // Current state
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // What self will become
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // What the output will become
            let output_len = old_len - new_len;
            let output_front = self.list.front;
            let output_back = prev;

            // Break the links between cur and prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            }

            // Produce the result:
            self.list.len = new_len;
            self.list.front = new_front;
            self.list.back = new_back;
            self.index = new_idx;

            LinkedList {
                front: output_front,
                back: output_back,
                len: output_len,
                _boo: PhantomData,
            }
        }
    } else {
        // We're at the ghost, just replace our list with an empty one.
        // No other state needs to be changed.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

참고로 이 `if let` 구문은 방금 말했던 "정상 케이스이긴 한데, prev가 하필 유령인 케이스" 상황을 전담 마크하고 있습니다:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

만약 *여러분*이 꼴리신다면, 저 장황한 코드를 찰흙 뭉치듯 한데 짓이겨 압축(squash)하고 이런 식의 최적화(optimizations) 기교를 부리셔도 됩니다:

* `(*cur.as_ptr()).front` 접근을 두 번이나 하는 짓거리 대신 깔끔하게 `(*cur.as_ptr()).front.take()` 하나로 후려치기(fold)
* `new_back` 변수가 사실상 아무 쓸모 없는 형편없는 것(noop)란 걸 눈치채고 과감히 들어내기

제가 보는 한, 그 외 나머지 자잘한 것들은 어찌어찌 우연의 일치로 알아서 찰떡같이 맞아떨어지는(incidentally Does The Right Thing) 것 같습니다. 뭐 나중에 테스트 돌려보면 견적 나오겠죠! (이제 복붙해서 `split_after` 만들러 갑시다)

전 더 이상 끔찍한 실수(Making Mistakes) 따윈 하기 싫기 때문에, 이 세상에서 제일 무식할 정도로 안전한(foolproof) 코드를 짜보려고 발악하는 중입니다. 이게 진짜 제가 실전에서(actually) 컬렉션 코드를 짜는 찐 방법론입니다: 걍 멍청한 제 뇌 용량으로도 100% 한 치의 의심 없이 완벽하게 이해될(foolproof) 때까지, 모든 로직을 하찮고 자잘한(trivial) 단계와 케이스들로 개박살 내서 잘게 쪼개버립니다. 그리고 나서 "아 씨바 이젠 진짜 더 이상은 문제될 수가 없겠구나" 하는 확신이 들 때까지 무지성으로 테스트 코드를 산더미처럼 때려 박는(write a ton of tests) 거죠.

왜냐면 그동안 제 인생에서 만져본 컬렉션 작업물들의 대다수가 *개황당한듯이 위험천만한 언세이프(extremely unsafe)* 부류였기 때문에 기본적으로 컴파일러가 제 똥을 치워줄 거란(catching mistakes) 헛된 기대 따윈 접어야 했고, 심지어 라떼는 miri 같은 꿀 빠는 기계도 없었거든요! 고로 머리가 깨질 것처럼(until my head hurts) 모니터를 게슴츠레 째려보며(squint), 그저 순수 피지컬로 단 하나의 삑사리도 내지 않기(Never Ever Ever Make A Mistake) 위해 온 영혼을 갈아 넣으며 똥꼬쇼를 하는 수밖에 없었습니다.

제발 쓸데없이 언세이프(Unsafe) Rust 코드 짜지 마세요! 세이프(Safe) Rust가 씹압살하게 우월합니다!!!!




# 접합 (Splice)

이제 딱 한 녀석, 역대급 보스 몹인 `splice_before`와 `splice_after` 패거리만 잡아 족치면 끝납니다. 감히 예상컨대 이 녀석들이야말로 온갖 끔찍한 코너 케이스(corner-casiest)의 집약체일 겁니다. 이 두 함수는 외부 스와트 팀 마냥 다른 LinkedList를 하나 인자로 꿀떡 집어삼킨(take in) 다음, 그 내장 내용물들을 우리 리스트 한복판에다 수술하듯 꿰매어 이어붙입니다(grafts). 이 과정에서 우리 쪽 리스트가 텅텅 비었을 수도 있고, 상대 리스트가 깡통일 수도 있고, 심지어 유령 녀석까지 설쳐대는 난장판을 수습해야 합니다... *휴(sigh)* 일단 심호흡 좀 하고 `splice_before` 부터 천천히 한 땀 한 땀 뜯어봅시다.

* 만약 저쪽에서 들고 온 리스트가 텅 빈 깡통이면, 우린 걍 파업하고 놀면 됩니다. 
* 만약 우리 리스트가 텅텅 비었다면, 부끄러운 일이지만 우리 껍데기가 그대로 저쪽 리스트로 변이해 버리면(becomes their list) 됩니다.
* 만약 우리가 애먼 유령 녀석를 가리키고 있다면, 이건 사실상 리스트 맨 끄트머리에 엉덩이를 이어 붙이는(appends to the back) 것과 같습니다 (list.back 조작).
* 만약 우리가 영광스러운 리스트 맨 앞구멍 원소(0)를 가리키고 있다면, 이건 사실상 머리 앞에다 무식하게 들이박는(appends to the front) 꼴입니다 (list.front 조작).
* 그 외의 가장 일반적인(general) 낭만 케이스엔, 온갖 더럽고 난잡한 포인터 서커스 똥꼬쇼(pointer fuckery)를 씨게 한바탕 벌여야 합니다.

대망의 일반 케이스(general case) 모습는 대충 이렇습니다:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

그리고 우린 이걸 아래의 프랑켄슈타인(this)으로 진화시켜야(Becoming) 합니다:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

자 알겠죠? 그래요 해봅시다. 코드로 뽑아내 보죠... *최대한 깊게 심호흡(HUGE BREATH) 한 번 땡기고 지옥 불로 다이빙합니다(PLUNGES IN)*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                *self.list = input;
            }
        }
    }
```

하, 이 새낀 진짜배기 오리지널 개노답 끔찍한 형편없는 것(horrendous)군요. 이번엔 진짜 그 `Option<NonNull>`의 끔찍한음과 고통(pain)을 폐부 깊숙이 실감하고 있습니다. 그래도 불행 중 다행으로 나름 청소하고 각 잡을 만한(cleanups) 구석이 제법 쏠쏠하게 있습니다. 우선, 이 더러운 코드 덩어리 하나를 멱살 잡고 저기 맨 밑바닥 끝자락으로 집어던집시다. 왜냐면 어차피 무슨 난리이 나든 무조건 실행시켜야 하는 범지역구(always want to do) 로직이니까요. 솔직히 이 짓거리 별로 맘에 들진 않지만(don't *love*) (가끔 보면 쓸데없는 헛발질(noop)일 때도 있고, 뭐 `input.len = 0` 세팅해놓는 짓도 걍 미래에 코드 더 확장할 때 문제되지 않으려고 떠는 심각한 강박증(paranoia)에 가깝긴 합니다만):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> Use of moved value: `input` (이동된 값 사용 에러: 바보아)

아 씨바 맞다, 우린 방금 "본체 리스트가 깡통(we're empty)일 때" 통째로 녀석을 갈아 치우면서 소유권 이동(moving)을 내버렸었죠. 쿨하게 스왑(swap) 펀치로 땜빵해 줍시다:

```rust ,ignore
// We're empty, become the input, remain on the ghost
std::mem::swap(self.list, &mut input);
```

이 케이스에서 변수에다 뭐 끄적거리며 써대는 짓거리(writes)는 개뻘짓(pointless)이 되긴 하겠지만, 어쨌든 터지진 않고 기동은 하니까 넘어갑시다 (컴파일러 녀석 불평대는 거 닥치게(appease) 하려면 저 분기점에서 걍 조기 반환(early-return) 때려버려도 될 거 같긴 하네요).

저 뜬금없는 `unwrap` 뇌절은 사실 제가 무식하게 머릿속에서 케이스 분류를 역순으로 구상하면서 똥을 싸질러 둔 대가(consequence)일 뿐입니다. `if-let` 이 씹녀석한테 제대로 똑바른 질문을 던지게끔 타일러 주면 쉽게 치료(fixed) 할 수 있습니다:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

인덱스를 조물딱거리는 짓(Adjusting)이 분기문(branches)들 안에 쓸데없이 도배되어(duplicated) 있으니, 이 녀석도 위로 시원하게 끄집어냅시다(hoisted out):

```rust
*self.index.as_mut().unwrap() += input.len;
```

좋아요, 이 모든 피와 땀과 똥꼬쇼들을 한데 모아 빚어내면 드디어 대망의 걸작이 나옵니다:

```rust
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // We're appending to the front, see append to back below
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // We can either `take` the input's pointers or `mem::forget`
    // it. Using take is more responsible in case we do custom
    // allocators or something that also needs to be cleaned up!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

좋아요 뭐 여전히 끔찍한이 생겨먹긴 했지만(sucks), 나발이고 간에 방금 버그 하나 정말 치명적인 거 찾았습니다:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

우리는 `input.front`를 무작정 `take`로 스틸해버리고선, 놀랍게도 바로 다음 줄에서 머리에 총 맞은 원숭이마냥 그걸 쌩으로 `unwrap` 해대고 있습니다! *하아 젠장(sigh)* 심지어 똑같은 판박이 분신술 거울(mirror) 케이스에서도 똑같은 개똥 같은 짓을 반복했네요. 사실 뭐 나중에 테스트 돌려보면 1초 만에 정말  털렸을(caught) 허접한 버그긴 하지만, 지금 명색이 "완벽주의(Perfect)" 컨셉 잡고 라이브 방송(live) 뛰듯 타이핑 치고 있는 마당에 변명의 여지가 없네요. 평소의 제 깐깐하고 숨 막히는 찐따(tedious) 같은 텐션을 버리고 차분히 단계를 밟지 않은 제 오만함에 대한 처절한 대가입니다. 더 끔찍한을 만큼 노골적으로 가자고요(More explicit)!

```rust
// We can either `take` the input's pointers or `mem::forget`
// it. Using `take` is more responsible in case we ever do custom
// allocators or something that also needs to be cleaned up!
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // No prev, we're appending to the front
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

오케이 이 정도면, 이 정도 폐기물이면 저도 나름 참고 봐줄 아량(tolerate)이 생기네요. 굳이 꼬투리 하나 잡아 트집 잡자면 `in_front`랑 `in_back` 변수를 재사용(dedupe)해서 합치지 않았다는 불만(complaints) 정도인데 (아마 조건문 논리 설계(conditions) 자체를 옹이구멍 파듯 다시 재조립(rejig)하면 안 될 것도 없겠지만 젠장 알빠노(eh whatever)). 솔직히 까놓고 말해서 이건 걍 여러분이 C 언어로 더럽게 짰을 법한 구닥다리 로직에다가, 망할 귀찮음증(tedious)을 유발하는 `Option<NonNull>` 껍데기 형편없는 것들을 잔뜩 처발라놓은 거랑 다를 바가 없습니다. 뭐 이 정도 똥이라면 참고 안고 갈 수 있습니다(I can live with that). 아뇨 젠장 사실은 이런 애미리스한 상황을 막기 위해 쌩 로우 포인터 자체를 더 우월하게 진화시켜야만 합니다. 하지만 슬프게도 그건 이 문제망한 튜토리얼 책의 한계 범위(out of scope)를 아득히 벗어난 우주적 과업이죠.

어찌 됐든, 전 방금 이 처절한 똥꼬쇼를 치르느라 진심 영혼까지 기 빨려서 너덜너덜(exhausted) 해졌기 때문에, `insert` 나 `remove` 따위를 포함한 떨거지 여타 API들은 유능하신 독자 여러분의 귀염뽀짝한 연습문제(excercise) 몫으로 과감히 짬 때리겠습니다.

자, 이리하여 제가 무지성 노가다 복사 붙여넣기 컴비네이션 타법을 구사하며 처절하게 발악한 최후 커서 전체 코드가 완성되었습니다. 과연 이 코드가 올바르게 돌아가긴 할까요? 젠장 저도 몰라요, 다음 챕터에서 이 괴물(monstrosity) 녀석를 테스트라는 이름의 감옥에 처박아봐야 알겠죠!


```rust ,ignore
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }
}
```