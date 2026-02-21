# 부수고 해체하기 (Breaking Down)

`pop_front` 녀석은 원리상 아까 짰던 `push_front` 랑 근본 뼈대 논리(basic logic)가 똑같아야 정상인데, 방향만 180도 뒤집어서 거꾸로(backwards) 주행하는 짓거리일 뿐입니다. 어서 한 번 기안서 뽑아서 밀어붙여(try) 봅시다:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // 낡은 머리 옛 머리를 멱살 잡아 빼내 통째로 뜯어옵니다, 이때 이녀석한테 꽂혀있던 바깥줄 -2개가 마이너스 차감(ensuring it's -2)되는 걸 명확히 증명해야 합니다
    self.head.take().map(|old_head| {                         // 여기서 -1 old 차감
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // 여기서 -1 new 차감
                // 아직 리스트 안에 잡다한 생존자가 남아있군요(not emptying list)
                new_head.borrow_mut().prev.take();            // 여기서 다시 -1 old 차감
                self.head = Some(new_head);                   // 여기서 +1 new 획득 보강
                // 총 누계: -2 old 손실, +0 new 똔똔 유지
            }
            None => {
                // 리스트 바닥나서 완전 빈 깡통화(emptying list)
                self.tail.take();                             // 여기서 -1 old 차감
                // 총 누계: -2 old 손실, (new라는 건 애초에 없음)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

아악. (ACK). *RefCells* 이 망할 형편없는 것. 하, 까먹을 뻔했네요 또 `borrow_mut` 함수 버튼 밟아서 빌려 난리해야(Gotta borrow_mut again I guess...) 하는 모양이군요...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```


```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*빌어먹을 하아 깊은 한숨 (sigh)*

> 남한테 빌려온 전세 물품 내용물을 함부로 무단 적출해 훔쳐 이동 반출 이별(move out of) 소유권 이전시키는 도난 범죄는 절대 금지 철회 무단 차단(cannot move out of borrowed content) 

흠... (Hrm...) 예전 박스(Box) 시절에 만능 편의 도구들에 우리가 진짜 정말  철저히 온실 속 화초처럼 너무 편하게 응석받이(spoiling us)로 살아왔나 봅니다. 이 구두쇠 같은 `borrow_mut` 무인 발급기 창구 직원은 단호하게 우리 손에 겨우 빌려 단기 락 대출 차용 딱지인 `&mut Node<T>` 따위만 허락 쥐여주며 내보낼 뿐인데, 우리는 지금 그 귀한 타인의 귀속 대여 물건의 배때지를 칼로 쑤셔 속 알맹이를 강제로 뜯어 쥐고 소유권 들고 야반도주(move out of that) 이탈하겠다는 희대의 말도 안 되는 짓를 당당히 뻔뻔히 들이밀고 있었네요!

우리에게 지금 가장 절실한 동아줄(We need something) 직방 해결책은, 바로 저 골칫덩어리 `RefCell<T>` 포장용기 껍데기를 통째로 받아먹곤 그걸 대충 씹어 부수고 분해해 내부 순수 알맹이 심장 `T`를 뱉어주는 마법의 착즙 분쇄기 장치입니다. 다시금 그 위대한 성서 [사전 공식 문서 (docs)][refcell] 를 한 번 더 뒤적여 쓸만한 착즙 기능 부속(something like that)이 있나 물색 사냥해 봅시다:

> `fn into_inner(self) -> T`
> 
> RefCell 이라는 헌 포장재를 입에 넣고 와작와작 흔적기관 소화 파괴(Consumes) 삼켜 버린 뒤, 안에 꼭꼭 감춰 수여 포장되어있던(wrapped) 순결한 알맹이 가치 덩어리(value) 자체만 뿅 역참조 반환(returning) 추출 뽑아냅니다.

이야 기가 맥히군 이빨이 아주 잘 맞물려 들어가는(promising) 찰떡 파츠 조각 도구네요!

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

아 젠장 (Ah dang). 저 이쁜 `into_inner` 착즙기 함정 카드는 뭣도 모르고 덥석 황당한개처럼 RefCell 포장재를 무단 씹어 삼켜 해체 강탈 이동 소유권 적출권(move out)을 무력 강제 시도 발동하지만, 우리가 이 불합리한 세계 철칙 구조상 지금 당장 소화가 불가능(can't) 한 이유, 바로 그 망할 RefCell이 여전히 이상한 사람의 독재 전당포 소속 `Rc` 철창 우리 안에 갇혀(in an Rc) 저당 잡혀있기 때문이라는 끔찍한 함정 제약이 또 존재했기 때문입니다. 이전 장(previous chapter)에서 이미 피눈물 흘리며 절실히 절감 깨닫고 배운 대로, 무자비한 철판 딱지 `Rc<T>` 녀석은 오직 오로지 자신의 아가리 철창 배때기 속(its internals) 피사체 알맹이에 대해 단지 무기력한 관람 공유 참조 투시 시야(`&`) 차용권(shared references) 접근 같은 것 인가 선심만 생색 허락(only lets us) 베풀 뿐입니다. 그도 그럴 게 뭐 어찌 보면 그따구로 생겨 처먹은 억까 제한 통치 시스템 자체가, 이 잘나신 무적 참조 횟수 카운터 계기판 포인터 시스템(reference counted pointers)들의 영광스러운 존재 구실 지향 본질 목표 목적(the whole point) 그 자체이기 때문이기도 하죠: 여하간 걔넨 기본 패시브 태생 본질 자체가 너구리 굴 파고 뭉쳐 공동 분담 대여 빌붙어 나눠 쓰라(shared)는 종자 무리니깐요! 

옛날 우리가 이 참조 카운팅 리스트 같은 것 집단을 상대로 재귀 파괴 붕괴 오버플로우를 막고자 수동 강제 해체꾼(Drop) 트레이트를 손수 몸소 장인 구현 제정 시연 박아주려 생쑈 덤벼들었을 당시에 이미 절실히 한 번 부딪혀봤던 유구한 난치병 질환(a problem)이며, 그에 대한 즉효 만병통치약 해독제 해결 방안 도구 또한 예의 여전히 완전 100% 한결같은 토씨 하나 안 틀린(is the same) 똑같은 파츠 주사기입니다: 바로 `Rc::try_unwrap`, 이 녀석만이 감히 `Rc` 독재자 철창 소속 수감자 알맹이의 소유권 이동 해체 끄집어 뽑아 반출(moves out) 적출술 마술 기믹을, 오직 단 한 가지 조건 하에 인가 부팅 승인 실행시켜 줍니다: 그건 바로 무려 이 Rc 죄수 껍데기에 연루된 저당 참조 공유 카운터 딱지 숫자가 정직히 단 1개뿐으로 나락에 떨어져 깨끗이 자신 혼자만의 세상 천상천하 나 홀로 고립 고독 독거 독점(if its refcount is 1) 신빙성 증빙 일치 투명 확인 결재가 자명히 검증된 바로 단 그 찰나의 순간 말입니다.

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

참고로 이 빌어먹을 마술 부속 `Rc::try_unwrap` 녀석은 다짜고짜 즉방 결실 값을 무자비로 안겨주는 대신 `Result<T, Rc<T>>` 이라는 거지 같은 판도라의 뽑기 보물 상자 봉투 껍질 규격 통보를 한 번 더 거쳐 우리 손에 던져 내동댕이 반환시켜 줍니다. 사실 이 으스대는 척하는 Results 덩어리는 그 본모습의 민낯 뼈대 밑천 자체가 앞서 주야장천 빨아댄 옵션 요정 `Option` 기믹 체계를 살짝 모양내 덧댄 짝퉁 보급형 강화 포장재 패키지 에디션 단말기(generalized Option) 정도에 불과하며, 대신 굳이 차이 위용을 좀 꼽자면 옵션 요정 곁에선 그저 매몰차게 아무것도 빈털터리 공허함뿐이었을 `None` 나락 실패 쪽박 케이스 절망 상황 그 구멍 한 켠 자리에 굳이 부득불 찌질한 변명거리 인자 따위 무거운 낙오 부적격 실연 데이터(data associated with it) 위로 상품 찌꺼기를 부가 첨부 보충(associated) 제공해 우수리를 쥐어 구겨 넣었다는 차별점 정도입니다. 이 구체적 상황 케이스 건방 상자(In this case) 룰에 대조 빗대자면, 그 찌질한 핑계 낙오 데이터 따위야 결국 너님녀석팽이가 되지도 않는 뻘짓 수작질로 포장 껍질 까 분해(unwrap)하려 대들다 퇴짜 거절 실패로 박살 난 방금 그 원래 원 주인 등껍질 포장지 `Rc` 본체 깡통 고철 덩어리 찌꺼기 시체 그 이상 그 이하도 아니라 이겁니다. 허나 뭐 어차피 이미 다 아시겠지만 우리 성향이 그렇듯 여기서 실패 삑사리 나서 박살 나건 말건 실패 따위 쪽박 반환상자 구멍 핑계(case where it fails)엔 1절 눈곱 띠끌 1도 전혀 일말의 관심도 아예 없이 안중에도 없습니다 (심지어 우리가 만일 설계 로직상 우리 손으로 한치의 오차 없이 코드를 잘 짰다면(wrote our program correctly), 그건 무조건 100% 필사적으로 강제 성공 도장 결재를 따낼 기막힌 상황 시점 세팅(has to succeed)이 확약된 상태니깐요), 고로 그러니 그저 편하게 앞뒤 잴 것 없이 대책 무지성 `unwrap` 강제 개봉 망치 버튼을 상자 머리통에 대고 호탕하게 후려 까 내려 찍어(call `unwrap` on it) 후드려 박아 깨부수면 심기일전 아주 편안히 내용물을 탈취 획득 쓱싹할 수 있습니다.

자 어찌 됐든 이 잡설 다 뒤로 물리고, 다음번엔 이 개고집불통 컴파일러 심판관 녀석이 무슨 창의적 재수 없는 에러 불호령 통보증을 뽑아 제끼며 우리를 엿 멕이고 괴롭힐지(let's face it) 기쁜 인내심으로 기대 꼴찌 관전이나 마사지해 봅시다 (there's going to be one). 

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

아이고 두야 (UGH). 진짜 기겁을 하겠네, 기껏 Result 상자 머리통 뚝배기 박살 내려 무지성 내리꽂은 이 우월한 `unwrap` 강제 해체 도끼 스킬 사용 규정 조건 약관 요구 사항에 따르면 무려, 재수 옴 붙어 혹여나 그게 실패 시 실패 딱지 찌끄러기 불량 케이스 값(error case)으로 튀어나올 그 데이터 같은 것라도 반드시 기어이 무조건 콘솔창 로그 기록에 디버그-프린트 출력 발자취(you can debug-print) 증거 기록 인쇄 텍스트 활자를 싸지르며 남길 수 있어야 한단 고약한 의무 구속 철칙 제약 문턱(requires) 장벽이 도사리고 있었던 겝니다. 근데 `RefCell<T>` 종자 이 녀석들은 태생 성향 자체가 그 뱃속 단전 밑바닥 근간 내장 타입 소속 `T` 녀석이 애초부터 디버그(`Debug`) 기록 출력을 완화 허락 규약(`implements Debug`) 인가 증명 도래 동기화 만족 상태여야지만 겨우 꼽사리로 덩달아 연대 자격 승진 위임 부여 디버그(`Debug`) 출력이 해금(only implements) 가능한 바보 족쇄 한계점 찌질이라구욧. 그리고 결정적으로 우리가 직조 수공예 연마 빚어낸 이 근본 깡통 덩어리 빈껍데기 `Node` 도화지 녀석한테는 우린 아직 단 한 번도 귀찮아서 저딴 빌어먹을 디버그 출력 자격증(`Debug`) 공인 연동 부착 인가 허가 딱지(doesn't implement) 트레이트 구현 따위를 달아준 이력 전과 시도 흔적 이단조차 한 번도 뚫은 적이 전무하거늘!

에라 씨볼 진짜 성질 같아선 저 이상한 사람의 오지랖 참견쟁이 구문 비위 다 맞추러 죄다 파고들어서(Rather than doing that) 지저분하게 그 디버그 트레이트를 일일이 손수 파고 깎아 탑재 개조시켜주러 삽질 도배 발라 들어가는 것도 끔찍한 막노동이니, 그딴 쓸모없이 미련 덜떨어진 헛짓거리를 하느니 걍 훨씬 간단하고 무식하면서 잔머리 개꼼수 야매 회피 술법 통과 루트(work around it) 우회로 지름길을 골라 뚫어 제낄 튈 작정입니다: 기껏 힘들게 반환상자 Result로 힘겹게 변환돼 포장되어 온 규격 패키지 덩어리를 바로 그 순간 요술 변환 필터 망 `ok` 도구 필터 촉매제를 냅다 치트 급유 장착 비벼 처발라 우겨 부어서(converting), 아예 그 깐깐하고 말 많은 에러 케이스를 다루던 그 망할 Result 봉투 구조 자체를 그냥 싹 다 허무하게 뭉개버려 강제 리셋 증발 휘발 소멸 포멧 세탁시켜서는, 오직 알맹이에 실패 확률은 배제 묵살하고 단박 무결성 Option 요정 상자 단벌 껍데기 기믹 에디션 환골탈태(to an Option with `ok`) 통과 필터 환전 변조 마술 결합술을 우회 투척 시전 부어버리는 만행을 저지르는 겁니다:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

살려줍쇼 통과시켜 주소서 제발요 (PLEASE).

```text
cargo build

```

YES! (아싸!)

*깊은 안도의 한숨 (phew)*

해냈습니다. (We did it.)

진짜 드디어 그 지긋지긋한 `push` 랑 `pop` 녀석을 기어이 양방향 생태계에 피사체 안착시켜버렸습니다. (We implemented `push` and `pop`.)

자 어디 그 옛날 박제로 담가 감금시켜뒀던 케케묵은 가변 스택(stack) 동네의 기본 기초 템플릿 테스트(basic test) 원상태 마루타 원본 사본을 통째로 압수 불펌 도둑질 카피 스틸 조달(stealing) 납치해다가 당장 당면 눈앞의 가동 구동 생체 테스트 실험 현장으로 굴피 밀어 테스트 성능 검증 착취 조리돌림 시련을 거쳐 내쳐 보겠습니다 (애초에 왜 하필 스택 녀석이냐 당황스럽겠지만, 가슴에 손을 얹고 잘 생각해 보면 이 챕터 내내 허송세월 다 바쳐 우리가 지금까지 조물딱 거려 겨우 간신히 토대 기반 간신히 올려세운 기능 전체 총합(all that we've implemented so far) 결과물 덩어리란 게 결국 당장 저거 딸랑 하찮은 모습의 구색 스택 동작 정도 수준의 찌끄레기 가동 한계 반쪽짜리 기능 상태가 현실이기 때문입니다):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        // 텅 빈 바구니 리스트 녀석이 정신 안 차리고 버벅거리는지 아닌지 군기 확인 기강 잡기 작동 여부 올바름 행동 검증
        assert_eq!(list.pop_front(), None);

        // Populate list
        // 리스트 아가리에 바득바득 원형 숫자 파츠 알맹이 데이터 영양소 구겨 넣어 힘겹게 풍선처럼 속살 증식 살찌우기
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        // 넣은 고대로 잘 소화시켜 정상 기작 반환 패턴 순서 맞춰 예쁘게 잘 토해 내는지 순방향 일반 도출 이탈(removal) 추출 검사
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        // 혹시 그동안 우리가 모르는 새 안쪽 복막 어디 창자 내장 구석 하나가 비틀려 훼손 오작동 썩어 문들어지지(corrupted) 않았나 못 미더우니 재차 다짐 압박 확인 사살차 무심하게 데이터 추가로 더 밀어 구겨 박아 넣어봄
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        // 또다시 넣은 고대로 잘 소화시켜 정상 패턴 순서 맞춰 예쁘게 잘 토해 내는지 후방 연속 2차 도출 이탈(removal) 추출 재검사
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        // 바닥 밑바닥까지 박박 영끌 최후의 심연 마지막 마른오징어 국물 한 방울까지 빈틈없이 바닥 고갈 탈수 쪼그라트리기 완전 착즙 박멸 고갈(exhaustion) 한계점 바닥 판별 생체 파괴 테스트 1차 투입 및 2차 종말 결론 단죄
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*끝내주네요. (Nailed it.)*

자 이제 우리는 리스트 양 끝에서 요소를 자유자재로 제거(remove)할 수 있게 되었습니다. 이 성과를 발판 삼아 드디어 그동안 미뤄뒀던 대망의 `Drop` 구현(implement Drop) 작업에 돌입할 수 있게 되었습니다. 양방향 연결 리스트에서 `Drop` 구현은 개념적으로도 꽤 흥미로운(conceptually interesting) 주제입니다. 이전에 가변형 스택을 다룰 때는 `Drop`을 안 구현하면 단순히 재귀 한계를 넘어 스택 메모리가 터지는 것을 막기(avoid unbounded recursion) 위해 마지못해 구현(bothered to implement)했었지만, 이번에는 `Drop`을 제대로 구현하지 않으면 아예 처음부터 아무런 동작도(anything) 정상적으로 일어나지 않기(to happen at all) 때문입니다.

`Rc`는 끝없이 이어지는 순환 참조(cycles) 앞에서는 아무것도 스스로 처리하지 못합니다(can't deal with). 만약 여러분의 시스템 어딘가에 이 작은 순환 고리(tiny cycles)가 발생했다면, 이 고리에 속한 모든 요소들은 우주가 끝날 때까지 서로가 서로의 수명을 연장하며(everything will keep everything else alive) 살아남아 거대한 메모리 누수 좀비가 되어버립니다. 
그런데 안타깝게도 양방향 연결 리스트라는 구조 자체가 사실상 하나의 거대한 순환 고리(just a big chain)에 불과하다는 점(as it turns out)입니다! 따라서 만약 우리가 아무런 조치도 취하지 않고 리스트가 파괴되도록 내버려 둔다면(when we drop), 시스템에서 줄어드는 참조 카운트는 기껏해야 양 끝쪽 두 노드(the two end nodes)에 대한 카운트가 1 감소하는(decremented down to 1) 것에 그칠 뿐입니다. 그리고 그 이후에는 아무 일도 일어나지 않은 채(and then nothing else will happen) 거대한 순환 고리들은 영원히 메모리를 차지하게 됩니다. 
물론 리스트 내에 노드가 단 1개(exactly one node)뿐이었다면 운 좋게 정상 파괴되어 문제없이 통과(we're good to go)하겠지만, 보통 리스트란 내부에 여러 개의 원소를 포함하도록(contains multiple elements) 만들어지며 그래야 정상적으로 작동할 테니까요(should work right). 뭐, 어쩌면 저만의 착각일 수도 있지만요(Maybe that's just me).

어쨌든 앞서 뼈저리게 살펴봤듯이(saw), 리스트 중간에서 원소를 쏙쏙 빼내는 작업(removing elements)은 상당히 귀찮고 고통스럽습니다(a bit painful). 그러므로 이번에도 가장 단순하고 속 편한 방법(the easiest thing for us to do), 즉 `pop` 작업에서 텅 빈 `None`이 튀어나올 때까지(until we get None) 무념무상으로 그저 계속 `pop`을 호출하는(just pop) 방식을 선택하도록 하죠:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(사실 이 방식은 예전 가변형 스택 시절(with our mutable stacks)에도 충분히 써먹을 수 있는 꼼수였습니다(done this). 하지만 그때의 우리는 진짜 찐 고수들처럼 깊이 있는 학구열을 뽐내야 했기 때문에(shortcuts are for people who understand things!) 그 쉬운 지름길을 애써 모른 척 지나갔던 겁니다!)

이번 장의 남은 부분을 통해 `push`와 `pop`의 반대편 꼬리 방향(back) 버전들(implementing the `_back` versions)을 계속해서 살펴볼 수도(could look at) 있겠습니다만, 어차피 앞서 만든 로직을 그대로 복사해서 붙여넣기(copy-paste jobs) 하는 것에 지나지 않으니 그 귀찮은 작업은 챕터 맨 마지막으로 미뤄두겠습니다(defer to later in the chapter). 그러니 당장 지금 이 순간(For now)만큼은 좀 더 새롭고 재밌는 내용들(more interesting things!)에 시선을 돌려봅시다!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
