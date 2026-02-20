# 커서 테스트하기 (Testing Cursors)

자, 이제 이전 섹션에서 제가 얼마나 수치스럽고 끔찍한(horribly embarassing) 똥을 싸질러 놨는지 까발려 볼 시간입니다!

아 맙소사, 우린 std 랑도 다르고 제 예전 구현체랑도 다르게 API를 완전히 마개조해 버렸잖아요. 뭐 알겠습니다, 일단 어떻게든 두 곳에서 조금씩 긁어모아다 급하게 누더기 기워보죠(hastily cobble together). 그래요, 겸사겸사 std에 있는 이 테스트 코드들부터 좀 "빌려와(borrow)" 봅시다:

```rust ,ignore
    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[7, 1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]);
        
        /* remove_current not impl'd
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(tmp.into_iter().collect::<Vec<_>>(), &[102, 103, 8, 2, 3, 4, 5, 6]);
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[200, 201, 202, 203, 1, 100, 101]);
    }

    fn check_links<T>(_list: &LinkedList<T>) {
        // would be good to do this!
    }
```

운명의 시간!(Moment of truth!)

```text
cargo test

   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 1.03s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_debug ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_ord ... ok
test test::test_cursor_move_peek ... FAILED
test test::test_cursor_mut_insert ... FAILED
test test::test_iterator ... ok
test test::test_mut_iter ... ok
test test::test_eq ... ok
test test::test_rev_iter ... ok
test test::test_iterator_double_end ... ok
test test::test_hashmap ... ok
test test::test_ord_nan ... ok

failures:

---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `None`,
 right: `Some(1)`', src\lib.rs:1079:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1153:9


failures:
    test::test_cursor_move_peek
    test::test_cursor_mut_insert

test result: FAILED. 12 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

솔직히 말해서 참회하자면, 저 스스로 완벽하게(nailed it) 짰을 거라며 근자감(hubris)에 좀 심하게 쩔어 있었습니다. 뭐 이 맛에 테스트 코드 짜는 거 아니겠습니까 (아니면 그냥 제가 원래 있던 테스트를 복붙(porting)하면서 병신같이 옮겨 적었을 수도 있고요..?).

첫 번째 연쇄 추돌 사고(first failure) 원인이 뭐죠?

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 2, 3, 4, 5, 6]);
let mut cursor = m.cursor_mut();

cursor.move_next();
assert_eq!(cursor.current(), Some(&mut 1));
assert_eq!(cursor.peek_next(), Some(&mut 2));
assert_eq!(cursor.peek_prev(), None);
assert_eq!(cursor.index(), Some(0));

cursor.move_prev();
assert_eq!(cursor.current(), None);
assert_eq!(cursor.peek_next(), Some(&mut 1)); // DIES HERE
```

이런 젠장, 진짜 생초보도 안 할 가장 기초적인 기능(basic functionality)부터 좆창내놨었네요. 잠깐만,

> 머리 비우고 걍 타이핑 칩니다. Option 메서드들과 컴파일러가 알아서 쌍욕 박으며(omitted compiler errors) 토해내는 피드백들이 제 뇌를 대신해서 모든 걸 다 생각(all thinking) 해주고 있습니다.

솔직히 제가 정직함(honest) 하나 빼면 시체이긴 합니다.

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

...네 그냥 쌩으로 틀렸습니다. 만약 `self.cur`가 `None`이라면 걍 무식하게 포기할(give up) 게 아니라, 당연히 우리가 유령 위에 서 있다는 뜻이니까 `self.list.front`도 찔러봐야(check) 했건만! 걍 메서드 체이닝 끝자락에 `or_else` 하나만 스윽 끼워 넣으면 다 해결될 일입니다:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .or_else(|| self.list.front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .or_else(|| self.list.back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

이걸로 고쳐졌으려나요?

```text
---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `Some(6)`,
 right: `None`', src\lib.rs:1078:9
```

아니 잠깐, 이번엔 아예 *더 앞에서부터(further back)* 터지잖아. 오케이 인정 반성합니다. 이제 `peek` 짤 땐 제발 뇌 빼고 치는 거 그만둬야겠습니다. 겉보기와 달리 이 녀석 제가 생각했던 것보다 훨씬 더 딥다크(lot harder)한 새끼였네요. 무지성으로 체이닝만 덕지덕지 이어 붙이는 건 걍 파멸로 가는 지름길(disaster)이니, 제대로 빡 각 잡고 유령일 경우와 아닐 경우를 `if` 문으로 갈라치기 해줍시다:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        let next = if let Some(cur) = self.cur {
            // Normal case, try to follow the cur node's back pointer
            (*cur.as_ptr()).back
        } else {
            // Ghost case, try to use the list's front pointer
            self.list.front
        };

        // Yield the element if the next node exists
        next.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        let prev = if let Some(cur) = self.cur {
            // Normal case, try to follow the cur node's front pointer
            (*cur.as_ptr()).front
        } else {
            // Ghost case, try to use the list's back pointer
            self.list.back
        };

        // Yield the element if the prev node exists
        prev.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

이번엔 진짜 삘이 오네요, 근자감 풀충전 완료!(Feelin' confident about this one!)

```text
failures:

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1168:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    test::test_cursor_mut_insert

test result: FAILED. 13 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

좋았어(Yesss). 자 이제 딱 하나만 더 잡으면... 어라.

아까 제가 `remove_current` 기능 테스트하려고 짜둔 코드 주석 처리(commented out) 해버린 거 눈치채셨나요? 네, 이 망할 테스트 코드가 내부적으로 상태값을 유지하며 파멸로 치닫고 있는(stateful) 연쇄 구조란 걸 빡대가리처럼 놓치고 있었죠. 걍 미련 없이 `remove_current` 가 휩쓸고 지나간 파멸 후의 상태값으로 리스트를 뚝딱 새로 하나 창조해 줍시다:

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 8, 2, 3, 4, 5, 6]);
```

```text
 cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.70s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_cursor_move_peek ... ok
test test::test_eq ... ok
test test::test_cursor_mut_insert ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_ord_nan ... ok
test test::test_mut_iter ... ok
test test::test_hashmap ... ok
test test::test_debug ... ok
test test::test_ord ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

오우예에에에에에 다 통과했죠오오... 자 이쯤 되니 오히려 이제 슬슬 피해망상(paranoid)이 도지기 시작하네요. 기왕 하는 거 꼼꼼하게 `check_links` 까지 마저 채워 넣고 Miri 검사관님 모셔다가 심판을 받아봅시다:

```rust ,ignore
fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
    let from_front: Vec<_> = list.iter().collect();
    let from_back: Vec<_> = list.iter().rev().collect();
    let re_reved: Vec<_> = from_back.into_iter().rev().collect();

    assert_eq!(from_front, re_reved);
}
```

이게 진짜 최선의(best) 구현 방식입니까? 아뇨. 그래도 쓸만(fine) 합니까? 예쓰.

```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo miri test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_cursor_move_peek ... ok
test test::test_cursor_mut_insert ... ok
test test::test_debug ... ok
test test::test_eq ... ok
test test::test_hashmap ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_mut_iter ... ok
test test::test_ord ... ok
test test::test_ord_nan ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

완벽.(DONE.)

진짜 끝났습니다.(Done.)

우리가 해냈습니다 씨발. 우리가 마침내 std 구현체랑 비비는 깡패 같은 성능과 기본기를 두루 갖춘 상용(production-quality) LinkedList를 이 두 손으로 직접 빚어냈습니다. 중간중간 쓸데없는 자잘한 편의성 메서드(convenience methods)들을 꽤 빼먹긴 했지만요? 당연히 빼먹었죠 시발. 나중에 제가 이 크레이트를 직접 릴리스(published)할 날이 오면 마저 다듬어 넣을까요? 아마도 삘 꽂히면 그러겠죠!

하지만, 전 지금, 말도 안 되게 개같이 지쳤습니다(So Very Tired).

어쨌든. 우리의 승리입니다.

아 잠깐 썅. 우리 상용(production quality) 퀄리티 지향한다 하지 않았음? 오케이 진짜 찐막 리얼 마지막 숨겨진 최종 보스: 클리피(clippy) 님 모시겠습니다.

```text
cargo clippy

cargo clippy
    Checking linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:189:19
    |
189 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: `#[warn(clippy::redundant_pattern_matching)]` on by default
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: method `into_iter` can be confused for the standard trait method `std::iter::IntoIterator::into_iter`
   --> src\lib.rs:210:5
    |
210 | /     pub fn into_iter(self) -> IntoIter<T> {
211 | |         IntoIter {
212 | |             list: self
213 | |         }
214 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::should_implement_trait)]` on by default
    = help: consider implementing the trait `std::iter::IntoIterator` or choosing a less ambiguous method name
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait

warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:228:19
    |
228 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: re-implementing `PartialEq::ne` is unnecessary
   --> src\lib.rs:275:5
    |
275 | /     fn ne(&self, other: &Self) -> bool {
276 | |         self.len() != other.len() || self.iter().ne(other)
277 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::partialeq_ne_impl)]` on by default
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#partialeq_ne_impl

warning: `linked-list` (lib) generated 4 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
```

자 들어오시죠 클리피 성님, 한 번 해봅시다.

불만 사항 1 (그리고 3): `while .is_some()` 놔두고 왜 굳이 `while let Some(_) = ` 같은 똥꼬쇼를 하냐는군요. 어차피 루프 속이 텅텅 비어서(empty) 진짜 단 1미리의 속도 차이도 안 나는 개뻘짓(doesn't matter)이긴 한데, 알겠습니다 쒸발, 클리피 성님이 하라면 성님 방식대로(your way) 기어야죠.

불만 사항 2: 우리가 자체적으로(inherent) `into_iter` 메서드를 박아뒀다는군요. 아니 잠깐, 왜 뭐가 문젠.. *std 코드를 까본다* 아 오케이, 이번 건 클리피 성님 1점. `IntoIterator`는 근본 중의 근본 프렐류드(prelude) (사실상 언어 자체 내장품(lang item))에 박혀있기 때문에, 우리가 굳이 허세 부리며 자체 버전까지 쑤셔 넣을(inherent version too) 필욘 없죠.

불만 사항 4: 우리가 std에서 이상한 카고컬트(cargocult) 버릇을 훔쳐 왔다는군요. *어깨 으쓱(shrug)* 알빠노 걍 지워줄게.

```text
cargo clippy
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

개꿀. 이제 "이거 진짜 상용 퀄임" 하고 자랑스럽게 떠벌리기 전에 딱 하나 남은 마지막 관문: `fmt` 타임입니다.

```text
cargo fmt
```

...예 뭐 알아서 줄 바꿈(newlines) 좀 스까주고 뒤쪽 잔여 공백(trailing whitespace) 찌끄레기들 좀 쓸어 담아 주네요. 노잼(Nothing interesting).

**드디어 진짜 리얼루다가 모든 게 끝났습니다!!!!!!!!!!!!!!!!!!!!!**

