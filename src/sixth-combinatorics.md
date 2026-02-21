# 지루한 조합론 (Boring Combinatorics)

좋습니다, 이제 늘 하던 대로 다시 연결 리스트로 돌아가 보죠!

우선 `pop` 덕분에 거저먹기 수준이 된 `Drop`부터 후딱 해치웁시다:

```rust ,ignore
impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // 더 이상 뽑아낼 게 없을 때까지 모조리 Pop 합니다
        while let Some(_) = self.pop_front() { }
    }
}
```

우리는 이제부터 `front`, `front_mut`, `back`, `back_mut`, `iter`, `iter_mut`, `into_iter`, ... 등등 엄청나게 지루하고 뻔한 조합론적(combinatoric) 구현체들을 그야말로 산더미처럼 작성해 넣어야 합니다.

물론 매크로(macros) 같은 걸 써서 뚝딱 처리해버릴 수도 있겠지만, 까놓고 말해서 그건 복사 붙여넣기 노가다보다도 훨씬 더 끔찍한 운명을 초래합니다. 우린 그냥 무식하게 복사 붙여넣기를 썩은 문둥이처럼 반복할 겁니다. 제가 아까 `push/pop`을 구현할 때 *아주 심혈을 기울여서* 짰기 때문에, 이제 우린 *글자 그대로(literally)* `front`와 `back` 단어만 서로 싹 스와핑해주면 코드가 알아서 찰떡같이 맞아떨어질 겁니다! 이 모든 게 뼈아픈 과거의 경험 덕분이죠 만세! (노드들을 가리킬 때 "이전(prev)"이나 "다음(next)" 같은 수식어를 쓰고 싶은 강렬한 충동이 들 때가 많지만, 제 경험상 웬만하면 무조건 "앞(front)"이랑 "뒤(back)"로만 일관되게 칭하는 게 실수를 줄이는 데 훨씬 이득입니다.)

자, 그럼 첫 번째 타자, `front`입니다:

```rust ,ignore
pub fn front(&self) -> Option<&T> {
    unsafe {
        self.front.map(|node| &(*node.as_ptr()).elem)
    }
}
```

어 사실, 이 튜토리얼 책이 워낙 옛날에 쓰인 구석기 유물이라서 그동안 Rust에도 `Option::None`에서 일찍 반환(early return) 빵을 때려주는 `?` 연산자 같은 신문물이 많이 추가되었는데, 과연 이걸 쓰면 우리 코드가 좀 더 이쁘장해질까요?


```rust ,ignore
pub fn front(&self) -> Option<&T> {
    unsafe {
        Some(&(*self.front?.as_ptr()).elem)
    }
}
```

글쎄요? 이렇게 단순한 코드에선 차라리 썼다 안 썼다 별 차이도 없는 무승부(wash)에 가깝고, 게다가 바로 전 챕터에서 우리가 조기 반환 빔을 얼마나 호러 틱하게 무서워했는지 구구절절 써놨기 때문에, 여기선 차라리 묵묵하게 명시적으로 쑤셔 넣는 편을 택하겠습니다 (전 그냥 기존의 `map` 떡칠 구현을 유지할 겁니다). 다음 타자, `front_mut`:

```rust ,ignore
pub fn front_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.front.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

나머지 찌꺼기 `back` 패밀리 버전들은 싹 다 맨 뒤쪽에 뭉치로 짬처리해두겠습니다.

다음 타석은 반복자(iterators)입니다. 지금까지 짰던 여타 허접 리스트들과는 달리, 우리는 드디어 마침내 [양방향 반복자(DoubleEndedIterator)](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html)의 꿀맛을 볼 제한을 해제(unlocked)했습니다. 이왕 상용 프로덕션 퀄리티를 지향하는 김에 [정확한 크기 반복자(ExactSizeIterator)](https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html)까지 깔쌈하게 구현해 보겠습니다.

그러니까 기존의 `next`랑 `size_hint`에 짬짜면으로 마저 얹어서, `next_back`이랑 `len`까지 추가로 지원해 줄 겁니다.

아마 눈썰미 쩌는 분들이라면 "어 잠깐만, IterMut 이 녀석 이거 양쪽에서 뜯어먹는 양방향 이터레이션 돌리면 정말 위험한 거 아니야?" 하고 의심의 눈초리를 보내실 수도 있겠지만, 놀랍게도 이건 완벽하게 안전(sound)합니다!

...아 씨바 이거 보일러플레이트(boilerplate) 찍어내는 모습 보니까 토 나오네. 진짜로 매크로 짜야 하나... 아냐, 아냐, 그건 더 끔찍한 파멸로 가는 지름길이야.

```rust ,ignore
pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

impl<T> LinkedList<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }
}


impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    
    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}
```

...방금 쓴 게 고작 `.iter()` 하나 분량입니다...

`IterMut`은 무식하게 싹 다 복붙한다음 필요한 곳곳에 `mut`만 가득 채워 대충 맨 끝바닥에 처박아 던져둘 거고, 우선 `into_iter`부터 후딱 치워버립시다. 너무나 다행스럽게도 얘는 그저 우리 컬렉션을 통째로 감싼 다음 `next` 부를 때마다 `pop`을 호출해서 뱉어주는, 뼈대 굵은 근본 해결책(tried-and-true solution)을 그대로 우려먹을 수 있습니다:

```rust ,ignore
pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}


impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}
```

여전히 토악질 나오는 보일러플레이트 덩어리긴 하지만, 그래도 적어도 이 녀석은 속이 편안해지는 *사이다(satisfying)* 보일러플레이트군요.

자, 이리하여 모든 빌어먹을 조합론 찍어내기가 끝난 최종 전체 코드 명세입니다:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}

pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn push_back(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None,
                front: None,
                elem,
            })));
            if let Some(old) = self.back {
                // Put the new back before the old one
                (*old.as_ptr()).back = Some(new);
                (*new.as_ptr()).front = Some(old);
            } else {
                // If there's no back, then we're the empty list and need 
                // to set the front too.
                self.front = Some(new);
            }
            // These things always happen!
            self.back = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a back node to pop.
            self.back.map(|node| {
                // Bring the Box front to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new back.
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).back = None;
                } else {
                    // If the back is now null, then this list is now empty!
                    self.front = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn front(&self) -> Option<&T> {
        unsafe {
            self.front.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.front.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn back(&self) -> Option<&T> {
        unsafe {
            self.back.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.back.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Pop until we have to stop
        while let Some(_) = self.pop_front() { }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}


#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
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