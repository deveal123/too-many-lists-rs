# 제법 괜찮은 안전하지 않은 단방향 큐 (An Ok Unsafe Singly-Linked Queue)

오케이, 앞서 다뤘던 참조 카운터와 내부 가변성 조합은 솔직히 좀 통제 불능(out of control) 수준이었습니다. Rust가 정말 모든 상황에서 대체로 저런 식의 코딩을 요구하는(expect) 걸까요? 글쎄요, 맞기도 하고 틀리기도 합니다. `Rc`와 `RefCell`은 단순한 케이스를 다루기엔 훌륭하지만 너무 거추장스러워(unwieldy)질 수 있습니다. 특히 그 내부에서 벌어지는 그런 짓거리들을 겉으로 완전히 숨기고(hide) 싶을 때 더욱 그렇죠. 분명 더 나은 방법이(better way) 있을 겁니다!

이번 장에서 우리는 다시 단방향 연결 리스트 시대로 회귀(roll back)하여 단방향 연결 큐(queue)를 구현함으로써, *원시 포인터(raw pointers)* 와 *안전하지 않은 Rust(Unsafe Rust)* 의 세계에 살짝 발을 담가(dip our toes) 볼 겁니다.

> **해설자:** 그리고 전 네가 무슨 실수를 저지르는지 친절히 짚어줄 거지.

걱정 마쇼 우린 *단 한 치의* 실수(make *any* mistakes)도 저지르지 않을 겁니다.

`fifth.rs` 라는 새 파일을 만들어 보죠:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
pub mod fifth;
```

연결 리스트 생태계에서 큐라는 구조는 본질적으로 스택의 확장(augmentation)판에 불과하기 때문에, 이번 코드는 상당 부분 `second.rs` 에서 차용(derived from)해 올 겁니다. 그럼에도 불구하고, 레이아웃 등 여러 가지 근본적인 결함 구조들을 다시 밑바닥부터 뜯어고치기 위해(fundemental issues we want to address) 백지상태에서 처음부터(from scratch) 다시 시작할 겁니다.
