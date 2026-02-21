# 기초 (Basics)

좋습니다, 이제 이 책에서 제일 짜증 나는 파트이자, 제가 이 챕터 하나 쓰는 데 무려 7년이나 걸린 이유에 도달했습니다! 우리가 이미 5번이나 우려먹었던 진짜 개노잼 작업들을 또다시 불태울 시간인데, 이번엔 심지어 모든 걸 두 배로 길고 번거롭게(extra verbose and long) 처리해야 합니다. 왜냐하면 모든 걸 앞뒤 두 번씩 처리해야 하고, 게다가 그녀석 `Option<NonNull<Node<T>>>` 포장지까지 일일이 다 까야 하니까요!

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }
}
```

PhantomData는 필드가 아예 하나도 없는 이상한 타입이라서, 그냥 저렇게 타입 이름만 띡 불러주면 알맹이가 하나 뚝딱 만들어집니다. *어쩌라고요(shrug)*.

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    // SAFETY: 이건 연결 리스트잖아, 대체 바라는 게 뭐야?
    unsafe {
        let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
            front: None,
            back: None,
            elem,
        })));
        if let Some(old) = self.front {
            // 새로운 front를 옛날 front보다 앞에 끼워 넣습니다
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
        } else {
            // 만약 front가 없다면, 이건 지금 빈 리스트라는 뜻이고 
            // back 역시 똑같이 맞춰줘야 합니다. 덤으로 우리가 뭔가 
            // 뻘짓할 때를 대비해서 무결성 검사(integrity checks)도 몇 개 깔아둡니다.
            debug_assert!(self.back.is_none());
            debug_assert!(self.front.is_none());
            debug_assert!(self.len == 0);
            self.back = Some(new);
        }
        self.front = Some(new);
        self.len += 1;
    }
}
```

```text
error[E0614]: type `NonNull<Node<T>>` cannot be dereferenced
  --> src\lib.rs:39:17
   |
39 |                 (*old).front = Some(new);
   |                 ^^^^^^
```


아 맞다 젠장, 전 진짜 이 포인터 찌꺼기 자식 녀석들이 너무 혐오스럽습니다. 우리는 NonNull 껍데기에서 원시 포인터를 끄집어내기 위해 `as_ptr`을 명시적으로 일일이 박아줘야 합니다. 왜냐하면 저녀석 DerefMut이라는 마법이 빌어먹을 `&mut`을 기반으로 정의되어 있는데, 우린 안전하지 않은 코드 구역 안에 함부로 그 안전한 참조자(safe references) 같은 것들을 마구잡이로 풀어놓고 싶지 않기 때문입니다!


```rust ,ignore
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
```

```text
   Compiling linked-list v0.0.3
warning: field is never read: `elem`
  --> src\lib.rs:16:5
   |
16 |     elem: T,
   |     ^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: `linked-list` (lib) generated 1 warning (1 duplicate)
warning: `linked-list` (lib test) generated 1 warning
    Finished test [unoptimized + debuginfo] target(s) in 0.33s
```

나이스, 이제 pop (그리고 len) 차례입니다:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    unsafe {
        // 뽑아낼 front 노드가 있을 때만 깔짝거려 주면 됩니다.
        // 그리고 이제 더 이상 그 빌어먹을 `take` 같은 걸로 똥꼬쇼할 필요가 없어졌음을
        // 주목하세요. 왜냐하면 모든 게 다 Copy 값들뿐이고 우리가 망치더라도 실행될
        // 소멸자(dtors) 따윈 일절 없기 때문이죠... 맞죠? :) 그쵸오오오? :)))
        self.front.map(|node| {
            // 안의 값을 쑥 뽑아내고 마무리를 짓기 위해(Drop) Box 껍데기를 되살려냅니다
            // (Box는 놀랍게도 이 짓거리를 마법처럼 완벽히 이해해 줍니다).
            let boxed_node = Box::from_raw(node.as_ptr());
            let result = boxed_node.elem;

            // 다음 노드를 새로운 front 자리에 꽂습니다.
            self.front = boxed_node.back;
            if let Some(new) = self.front {
                // 뽑혀 나간 삭제된 노드를 향한 남겨진 미련(reference)을 깔끔하게 지워줍시다.
                (*new.as_ptr()).front = None;
            } else {
                // 만일 front가 Null이 되어버렸다면, 이 리스트가 텅텅 비었다는 뜻입니다!
                debug_assert!(self.len == 1);
                self.back = None;
            }

            self.len -= 1;
            result
            // 여기서 Box는 알아서 내장 파괴(freed)수순을 밟게 되고, T가 없어졌다는 사실 역시 알아서 잘 압니다.
        })
    }
}

pub fn len(&self) -> usize {
    self.len
}
```

```text
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
```

꽤 그럴싸해 보입니다, 이제 테스트를 돌려볼 시간입니다!

```rust ,ignore
#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // 텅 빈 리스트 괴롭히기
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // 아이템 하나짜리 리스트 괴롭히기
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // 막 굴려보기 (Mess around)
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```


```text
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src\lib.rs

running 1 test
test test::test_basic_front ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

만세, 우린 완벽합니다!

...정말 그럴까요?(Right?)