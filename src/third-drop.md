# 메모리 해제 (Drop)

가변 리스트를 만들 때 맞닥뜨렸던 것처럼, 이번에도 우리에겐 재귀 해제(recursive destructor)로 인한 스택 오버플로우 문제가 도사리고 있습니다.
물론 인정하건대, 불변 리스트 구조에서는 이 문제가 예전만큼 치명적이지는 않습니다: 만일 우리가 역추적해 들어가다 우연히 *어딘가에* 존재하는 다른 리스트의 첫머리(head) 노드와 딱 마주친다면, 우린 그 너머를 더 이상 재귀적으로 깊이 파고들어 소멸시키지 않을 확률이 높기 때문입니다. 하지만 그럼에도 불구하고 여전히 우린 이 잠재적 시한폭탄을 신경 써야 하며(care about), 이걸 대체 어떤 수단으로 우아하게 해결할지(how to deal with) 직관적으로 잘 떠오르질 않습니다. 자, 예전 가변 리스트 땐 이걸 어떻게 빠져나갔었는지 다시 기억을 되감아봅시다:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

사건의 발단이자 근본적인 장벽은 바로 저 루프의 심장부(body)에 자리 잡은 이 구문입니다:

```rust ,ignore
cur_link = boxed_node.next.take();
```

이 코드는 Box 껍데기 안에 들어차 있는 Node 자체를 파괴적으로 조작(mutating)해 까집는 짓인데, 이제 우리의 포장재인 Rc를 상대로는 이런 강제 개봉 짓거리가 절대 용납되지 않습니다; 앞서 누누이 말했듯 이녀석은 오직 읽기 전용의 공유된 접근 권한(shared access)만을 제공할 뿐 가변 권한이 없기 때문입니다. 언제 어디서 눈치 밥상에 숟가락 얹으려는 수많은 다른 Rc 꼬맹이들이 이걸 꿰뚫어 보고 가리키고 있을지(any number of other Rc's could be pointing at it) 아무도 장담할 수 없으니 말이죠.

하지만, 만일 우리가 온 하늘을 우러러 맹세코 우리가 쥐고 있는 이 리스트 녀석이 저 노드의 존재를 기억하고 가리키는 이 세상 최후의 1인(last list)이라는 사실을 확실하게 알아낼 수만 있다면, 비로소 그제서야 저 꼴 보기 싫은 Rc의 족쇄에서 Node를 갈라 빼내 뜯어 옮기는(move) 파괴적 짓거리마저도 *충분히 합법적이고 무방(would actually be fine)*하다고 승인될 수 있을 것입니다. 덤으로 그 단서를 역이용한다면, 우리가 어느 시점에 멈춰야 할지도(when to stop) 아주 자명해집니다: 즉 우리가 더 이상 안쪽에서 Node 알맹이를 무사히 호이스트(hoist out)해 끄집어낼 수 없는 그 어떠한 순간이 바로 탈출 시점인 셈이죠.

그리고 놀라지 마시라, 마치 이 상황을 꿰뚫고 있었던 것처럼 우리 Rc 군은 정확히 100% 이 기능만을 위해 탄생한 필살기 메서드 하나를 숨겨두고 있었습니다: 바로 `try_unwrap` 입니다:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ADesires/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

대단해! (Great!)
멋집니다. (Nice.)
