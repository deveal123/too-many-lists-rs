# 영구적인 단방향 연결 스택 (A Persistent Singly-Linked Stack)

좋습니다, 우리는 가변(mutable) 단방향 연결 스택 기술을 완벽히 마스터했습니다.

이제 *단일(single)* 소유권 구조에서 벗어나, *공유(shared)* 소유권을 지원하는 *영구적인(persistent)* 불변 단방향 연결 리스트를 작성해 봅시다. 이것이 바로 함수형 프로그래머들이 그토록 사랑하고 아끼는 완벽한 그 리스트 구조입니다. 
당신은 언제든 리스트의 첫 번째(head) 요소나 나머지 꼬리(tail)를 똑 떼어와 다른 꼬리에 갖다 붙일 수 있습니다... 
음... 그리고 사실 이게 전부입니다. 불변성(Immutability)은 참 매력적인 마약과도 같죠.

이 과정에서 우리는 전반적으로 Rc와 Arc의 활용법에 친숙해질 것이며, 이는 다음번 *상식 파괴의 극치(change the game)* 리스트 구현을 위한 밑거름이 될 것입니다.

새롭게 `third.rs`라는 파일을 하나 추가해 봅시다:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
```

이번엔 복사-붙여넣기는 없습니다. 완전히 백지상태의 멸균실(clean room) 작업입니다.
