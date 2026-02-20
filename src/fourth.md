# 구리지만 안전한 양방향 연결 덱 (A Bad but Safe Doubly-Linked Deque)

우리가 방금 Rc와 내부 가변성(interior mutability)이라는 개념을 접하게 되면서, 한 가지 굉장히 흥미로운 발상(interesting thought) 하나가 떠오릅니다... 어쩌면 우리가 감히 Rc 구조의 껍질 너머로 가변 변형(mutate) 짓까지 저지를 수 *있지 않을까* 하고 말입니다. 그리고 만일 *그게* 정말 가능하다면, 어쩌면 우린 *양방향 연결(doubly-linked)* 리스트를 완전히 100% 안전하게(totally safely) 구축해 낼 수 있지 않을까요!

이 시도를 거치며 우리는 *내부 가변성* 이란 무기의 성질에 완전히 친숙해지게 될 것이며, 아마도 그 혹독한 과정에서 '안전하다(safe)'라는 수식어가 '올바르다(correct)'라는 결실과 결코 일치하지 않음을 뼈저리게 통감(learn the hard way)하게 될 것입니다. 애초에 양방향 연결 리스트 설계는 엄청나게 고된 난제이며, 저 스스로도 이따금 어딘가에서 하나씩 나사를 빠뜨려먹고 실수(always make a mistake somewhere)하곤 할 정도니까요.

자, 일단 새로운 `fourth.rs` 파일을 하나 파 봅시다:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
```

글의 취지 상 이번 작업 역시 완전히 밑바닥부터 파고드는 멸균실(clean-room) 개척 프로젝트가 될 테지만, 예전 그랬듯 늘상 그래왔듯 중간중간 예전 가변 리스트 챕터 때 구상해 둔 재탕 로직 조각(verbatim again)들이 여기저기 재활용되는 모습도 여럿 마주하게 될 겁니다. 

면책 조항(Disclaimer): 사실 이 챕터는 솔직히 말해, 이런 설계 시도 자체가 얼마나 개같이 *지독한 뻘짓(very bad idea)* 생각인지 증명해 보이기 위한 일종의 고해성사 시연 극장(demonstration)과도 같습니다.
