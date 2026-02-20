# 좋지 않은 단방향 연결 스택 (A Bad Singly-Linked Stack)

이 챕터는 *단연코* 가장 긴 챕터가 될 것입니다. 왜냐하면 여기서 사실상 Rust의 모든 것을 소개해야 하고, 언어를 더 잘 이해하기 위해 굳이 "어려운 방식"으로 일일이 바닥부터 쌓아 올릴 것이기 때문입니다.

우리의 첫 번째 리스트는 `src/first.rs`에 작성할 것입니다. 우리가 작성하는 라이브러리(`lib.rs`)가 `first.rs`를 사용한다는 사실을 Rust에게 알려주어야 합니다. 이를 위해 Cargo가 만들어준 `src/lib.rs` 파일의 최상단에 다음 코드를 추가하기만 하면 됩니다:

```rust ,ignore
// in lib.rs
pub mod first;
```
