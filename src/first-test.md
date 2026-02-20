# 테스트 (Testing)

좋습니다, 우리는 이제까지 `push`와 `pop` 작성을 무사히 잘 끝마쳤으니 이제 이걸로 뼈 빠지게 고생하며 만들어 낸 우리의 스택이 과연 원하던 대로 안 터지고 잘 돌아가는지 야무지게 직접 찔러 테스트해 볼 시간입니다! 자랑스럽게도 Rust와 cargo는 태초부터 자체적인 테스트 전용 환경 생태계 구성을 언어 문법 내 1등 시민격의 중요 핵심 기능으로서 기본 지원하고 있으므로, 막연하게 느껴지는 이 일련의 검증 작업이 실로 정말 그저 숨 쉬듯 초간단해질 수 있습니다. 우리가 진짜로 손가락 움직여해야 할 유일무이한 잡일이라곤 그저 검증 함수를 하나 불쑥 던져놓고 겉에다 `#[test]`라는 마술의 주석만 알콩달콩 달아주기만 하면 끝입니다.

일반적으로, 우리 Rust 커뮤니티 개발 철학 바닥에선 항상 테스트하려는 원본 시스템 코드와 그 코드를 검증하는 테스트 코드 검사기를 한 공간상에 딱 달라붙어 이웃하게 바짝 기생시켜두는 관습적 트렌드를 추구하고 있습니다. 하지만 그러면서도 굳이 그와 동시에 진짜 동작하는 "실제 상용 로직" 부분과 테스트가 서로 더러운 코드 찌꺼기로 뒤섞여 충돌오류를 내지 않고 철저하게 격리될 수 있게끔, 우리는 새하얀 격벽을 쳐 이 테스트 전용 공간만을 별도의 청정 네임스페이스 영역 구석으로 밀쳐놓습니다. 아주 오래전 먼 옛날에 우리가 `mod` 블록 개념을 처음 사용해서 `first.rs`를 `lib.rs` 쪽에 어설프게 끼워맞춰 포함시켰던 전적과 아주 소름 돋게 비슷한 메커니즘으로, 여기서도 우리는 `mod`를 사용해 현재 다루는 이 파일 저 밑 단락 한 구석탱이에 전혀 눈길도 안채어지는 새로운 *분리된 가상의 독립 내장(inline) 파일* 구획을 통째로 오려 갈라 낼 권능이 생깁니다:


```rust ,ignore
// src/first.rs 파일 맨 밑자락

mod test {
    #[test]
    fn basics() {
        // TODO 코딩
    }
}
```

이제 저걸 야심 차게 구동하기 위해 여러분의 소중한 터미널에 `cargo test` 마법 주문을 한 자 한 자 소리쳐 봅니다.

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

아자아아무 짓도 안 하는 텅텅 빈 공백 백지 테스트 검사가 아주 대견하고 기만적으로 통과 판정을 받아냈습니다! 그럼 이번엔 제법 뼈와 살이 있는 기특한 검증 기능을 하나둘씩 모아 추가해서 더 이상 멍청하게 가만히 아무 일도 안 하고 노는 무용지물 월급루팡 테스트가 안 되게 뜯어고쳐 봅시다. 이 담대한 포부는 영원불멸 무적의 깡패 `assert_eq!` 매크로 하나를 당차게 동원하면 무척 짜릿하고 손쉽게 달성이 가능해집니다. 사실 이 매크로는 무언가 절대 상상도 못 할 고차원적인 외계 마법 기술이 걸린 신비의 심판 명기가 아닙니다. 이게 실제로 하는 유일한 병크짓이라곤 여러분들이 슬며시 인자로 쥐여준 두 개의 데이터 값을 건네 받아 단지 서로 나란히 세워두고 일일이 값을 들여다보며 비교 대조해서, 만일 둘의 수치가 한 치라도 맞물리지 않아 불일치를 발생시키면 즉시 분노를 참지 못해 발작하며 프로그램의 명줄을 그 자리에서 즉각 칼같이 잘라버리고 비참하게 붕괴(panic) 시켜버리는 파괴적인 행위가 전부입니다. 네 그래요, 여러분들의 기대를 저버리지 않게 테스트 구동 엔진은 이렇게 장렬하고 성대하게 고래고래 찢어지게 고함을 질러대다 장렬히 자폭해줘야 겨우 무사히 테스트가 실패 판정을 받아냈다는 안타까운 굉음 소식을 바깥으로 전달할 수 있거든요!

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // 텅 비어 깡통이 된 리스트가 제대로 빈 반응을 작동하는 뱉어내는지 점검합니다.
        assert_eq!(list.pop(), None);

        // 쓸데없이 리스트의 배때지에 무수한 값들을 왕창 밀어 넣습니다.
        list.push(1);
        list.push(2);
        list.push(3);

        // 올바른 차례순으로 이쁘게 정상 제거가 잘 이루어지는지를 깐깐하게 검증합니다.
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // 무결성과 데이터의 조밀함이 우발적 잔혹 행위로 인해 훼손되지 않았는지 이중 확인 차 위에다 한 번 더 몇 개 내팽개쳐봅니다.
        list.push(4);
        list.push(5);

        // 다시 한번 올바른 정상 제거 메커니즘 검증에 착수.
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // 모조리 영혼까지 끌어모아 탈탈 털어서 뽑아가 남은 게 이젠 정말 없는 징후를 추적.
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

이런이런 큰일 났군! 우리는 `test` 모듈이라는 좁디좁은 새로운 고립된 우주의 4차원 외부 환경 격리 섬을 하나 통째로 파버렸기 때문에 우리가 방금 이 섬 밖의 널찍한 육지에서 직접 죽어라 열심히 고생하며 개발해 만든 List 핵심 모듈을, 원칙상 명시적으로 선언을 내려다가 무식하게 강제로 외부에서 이 은밀한 테스트 섬 안마당으로 이끌고 내려와 다방면으로 가져와 써야만 합니다.

```rust ,ignore
mod test {
    use super::List;
    // 그 외 기타 모든 이전 내용 일괄 동일 유지
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

예에에!

그런데 저기 멀뚱히 뒤에 남겨져 홀로 붉은빛을 뿜는 무시무시한 경고 문구 자식은 대체 왜 남은 뭡니까...? 우리는 눈을 비벼가며 코드를 백 번 뜯어봐도 분명히 방금 막 짠 우리의 테스트 안에다가 저 빌어먹을 외부 List 모듈을 의기양양하게 땡겨 가져와서 너무나 잘 소화해 내며 멀쩡히 사용했단 말입니다!

...허나 슬프게도 그 경고는 어찌 보면 오직 "우리만" 아는 진실이고 오직 전적으로 "우리가 테스트를 진행하는 상황 중에만!" 발동한다는 반쪽짜리 진실에 불과했습니다. 컴파일러 입장에선 영 맘에 안 드는 불협화음의 원흉이 된 우리의 까탈스러운 컴파일러의 언짢은 기운 노이로제를 조용히 다독여 잠재워 주고 (겸사겸사 훗날 이걸 무턱대고 아무 때나 사용하게 될 불특정 다수의 다른 멍청한 소비자들의 눈을 찌르지 않도록 부드럽게 보호해주기 위해), 우리 스스로가 이 거대하게 포장된 `test` 서브 모듈 구조 전체 껍데기는 제발, 오로지 오직 우리가 진짜 굳게 마음먹고 빌드 단계 설정에서 `cargo test` 모드가 발동될 상황일 경우 한에서만 은밀하게 부분 격리되어 묶임 컴파일을 거치도록 강제되어야 한다는 사실을 매크로 라벨로 친절히 표기해 다뤄 둡시다.


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // 그 외 기타 모든 이전 내용 일괄 동일 유지
}
```

자 이제 이게 테스트 과정의 파란만장한 스펙터클함에 관하여 여러분이 필수적으로 알아가야 할 모든 내용의 A to Z 전부입니다!
