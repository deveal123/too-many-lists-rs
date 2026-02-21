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

*찢었다 개멋있어 (Nailed it).*

자 이제 우리는 리스트 바구니의 아가리통에서 기어이 어떤 잡스러운 알맹이 원소 같은 것 덩어리든(things) 자유자재로 모가지 틀어쥐어 적출해 파기 삭제 제거(remove) 뽑아 지우기 조율 청소 권능을 제대로 완전무결 확실하게(properly) 확립 입증 마스터 증명해 냈기 땜시, 이 막대한 영광의 부산물을 기반으로 발판 삼아 드디어 저번 내내 지긋지긋 미뤄뒀던 대망의 원한 해소 파괴신 소멸 `Drop` 분쇄기 시스템 청부업 장치 탑재 강제 도륙 구현(implement Drop) 작업에 손댈 뻗칠 착수 돌입 여유 인가 자격 통행이 가능해졌습니다. 더 웃긴 건 이번 참에 마주할 `Drop` 분쇄기 척결 의무 대면식 임무 동기는 옛날에 만났던 여느 하찮은 잡녀석에 비해 훨씬 철학적으로 어둑하게 음미하게 되는(conceptually interesting) 구미 당기는 흥미로운 이질적인 감상의 재미가 도사립니다. 예전 가변형 스택 철부지 애기덜 시절 챕터의 미동 시기 때는, 기껏 우리가 피땀 흘려 애써 이 Drop 살육 수집 처리 분쇄기를 한 땀 한 땀 코딩 구상 박아주며 전전긍긍 매달렸던 그 유일한 초라한 보잘것없는 동기란 건 기껏 그저 저 망할 어리석은 무한 나락 심연 속으로 끌려 들어가 추락해 스택 메모리를 펑 터뜨리는(avoid unbounded recursion) 재귀 낙하 지옥 붕괴 발작 병크 폭발 참사 하나만큼은 피해보자고 오지랖 불쌍 동정 연민 기어 발버둥(bothered to implement) 쳤던 예방접종 방안 소동에 불과했었지만, 반면 지금 이번 양방향 괴물 혼종 상태에선 이 빌어먹을 Drop 의식 연성 장치가 만에 하나 미비 빠지거나(implement Drop) 오류 삑사리라도 나버린다면 애초에 프로그램 생태계 전체의 사이클 작동 수집 순환 우주 궤도 순리 로직 기능 전체 중 도대체 티끌조차 *그 무엇 하나(anything)* 일말이라도 벌어지고 터져 작동 구실 동작을 하거나 굴러갈 턱 발생 여지 자체가 기저 뇌사 마비 일체 전무 마비 사태로(to happen at all) 올 데스 사망 선고 상태가 돼버리는 돌이킬 수 없는 절대 전제 불판이기 땜시 그 의미가 극도로 남다르기(now we need to) 때문입니다.

망할 `Rc` 이 띨띨한 등신 쫄보 녀석는 도저히 끝없이 쳇바퀴 마냥 빙빙 이어 돌아가는 저 지옥의 뫼비우스 대왕 순환 순회 사이클 구조망(cycles) 현상 앞에만 가면 완전 벌벌 떨며 일절 감당 처방 처리 처신 조치 자체를 불가피 아예 뇌사 손 놔 포기 단절 항복 먹통 상태로 방치(can't deal with) 뻗어버리는 지독한 고질적 천부적 결함 장애를 타고났습니다. 만일 당신 설계 공간 우주 영토 내부 어디 후미진 다락방 한편 구석에라도 이 황당한 뫼비우스 고리 미니어처 사이클(cycle) 함정 파동이 슬그머니 터져 발생해 도사리고 있었다면, 정말 단 한 톨의 구제 가능 희망 변명 자비 가차 없이 우주의 멸망 순간까지 서로가 서로 꼬리에 꼬리를 물고 무한 이빨 인질 삼아 핑퐁 영생 불사 부활 좀비 카운터 동기화(everything will keep everything else alive)를 썩어 나빠지게 기행 유지 영생 수명을 처 발라버리는 우주 영구 미아 좀비 아포칼립스 병크 사태를 유발해냅니다. 그리고 엄청나게 골 때리고 슬프게 밝혀진 더 놀라운 잔인 소름 끼치는 냉정한 대 참사 실체 현실 하나가 뭐냐면(as it turns out), 바로 딱 우리 면상 앞에 놓인 저 떡하니 등빨 세운 장르 본질인 '양방향-연결 리스트(doubly-linked list)' 저 끔찍한 구조물 종족 설계 본질 본태 그 자체가 애달프게도 결연하게 그저 티끌만 한 좁쌀 사이클 소용돌이 순환 고리(tiny cycles) 수천만 겹 사슬을 황당한 듯 기나긴 우주 대왕 동아줄(just a big chain) 포장 껍데기 기차 행렬 이음새 열차 모음집 군락 연골 무리 단지 세트장 진열품 전시관일 뿐이란 이 끔찍한 숨겨진 잔혹 사실입니다! 고로 이 무모한 끔찍 우주적 결함 구조 한가운데서 아무 대책 대비 사전 약 속 보호 제재 안전장치 조끼 조제약 처방 대비 없이 마냥 손 놓고 우리가 신나게 던져버린 척 방관 유기 폐기 시점 낙하(when we drop) 리스트 파괴 호출 이탈 종말 순간 강제 드롭 처형 결재 버튼을 딱 밟았다 칩시다, 그럼 기껏 그 리스트 본체 쪼가리 지분 하나가 손을 떼면서 유발되는 타격 현장 여파 파급 나비효과는 기껏해야 양옆 끄트머리 벼랑 끝단 허공 앞뒤 외곽 문지기 끄트머리 수문장 양단 두 개 노드 녀석들 알맹이의(the two end nodes) 목덜미에나 박힌 그 소중한 지분율 대여 참조 딱지 숫자 전두엽 카운터를 기껏 딱 마이너스 일 까내려 바닥 1단 강하 착륙감봉 조치(decremented down to 1) 하나 고따구로 겨우 깎아 처박아 내리는 하명 하사 짓거리에 만족하고 그칠 뿐이며... 그리고 그 비극의 결말 수순 바로 그다음에 터질 여파 사후 뒤처리 현상이라곤(and then), 이 우주에 진짜 그 누구도 더이상 아무 짓 아무 손도 절대 아무것도(nothing else will happen) 자의로 나서 구제 차단 처리 결박 수습 안 하고 그 상태 영구 방치 지옥 고사 사망 좀비 좀먹는 데드락 영구 멸종 병크로 얼어 멈춰 굳어 스탑 셧다운 고사 미아 방치 뻗어 버립니다. 야, 물론 뭐 씨볼 지극히 확률 희박한 천운의 럭키 박스 우연으로 만에 하나 백번 양보해 당신이 문제뺑이쳐 조각 연마한 이 리스트 보울 안면 통짜 바구니 속 알맹이가 오롯이 딱 단 한 톨의 점박이 구슬 단 1개 티끌 잔해물 노드 단독(exactly one node) 구성 체제 구조였다면, 뭐 그러면 별 탈 없이 순직 처리 깔끔 우수 사살 정제 완전 범죄 파괴 은폐 기적이 가능(we're good to go)하긴 하시겠죠. 하지만 정말 가슴에 진짜 손을 얹어 제발 양심 선별 잣대 이상향 포부 합리적 구상 기원 기조 목적을 가다듬고 통틀어 곱씹어 헤아려 이상적(ideally) 사상 원리 진실로 돌이켜 단언해 보자면, 리스트 생태계 종특 종속 존재 의미 기능 작동 여부 발동 발휘란 자고로 그 뱃속 체내 뱃가죽 안에 다수의 겹겹 불특정 수치 뭉태기 떼깔 병렬 여러 알맹이 파편 다수 구성원(contains multiple elements) 데이터 조직 진입을 묵묵 수용 짬내 배출하고 순환 감당 유도할 때 비로소 그 거룩 영롱 올바름 순기능 정석 순종 작동 부합(should work right) 구실이 빛을 발하는 게 합당 정론이 아닙니까. 어쩌면 제 머리 망상 기준에서만 저런 무지성 돌연변이 사이코패스 잡생각 억까 병맛이 지당(Maybe that's just me)하다고 세뇌 오만 꼴값을 떨고 자빠진 걸지도.

어쨌든 우리가 아까 위에서 피 똥 지리며 절감 통감 통찰 구사 경험 시연 살펴(saw) 보았던 대로 고생 생고생 맛본바, 이 리스트 항문에서 알맹이 쪼가리 하나 쑥 뽑아 배출 추출 솎아 삭제 찌르기(removing elements) 제거 분해 마술 작업 자체가 참 염병 난리맞게 번거롭고 숨통 목 조여 압박(a bit painful) 고단한 기막힌 난이도 장벽 수문장 생채기 마찰을 빚어낸단 사실을 잘 알고 있습니다. 고로 짱구를 이리저리 잔머리 풀가동 회전 모색 고민 궁리한 우리 얄팍 타산 두뇌 결론으로 도출되는 가장 속 편안한 궁극 우주 만사형통 편의 만점 프리패스 기똥찬 치트 지름길 방책 결단(the easiest thing for us to do)은, 뭐 그냥 아주 단순 무식하게 리스트 그 자궁 속에서 None 허공 공허 바닥나 탈탈 털려 긁혀 나올 그 텅 빈 진공의 순간 찰나 타이밍의 끝점 바닥 심연 궤적이 튀어 잡힐 때까지(until we get None) 무념무상으로 그저 딸깍딸깍 고문 펌프 버튼 누르듯 그 불쌍한 `pop` 추출 탈곡 압수 장치 레버만 죽어라 광클 난사 펌프 갈굼 발작 쑤셔 제껴 돌리는(just pop) 야근 꼼수 뺑뺑이 강림 수작업 루프 뿐입니다:

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

(사실 우리 이거 예전 가변형 스택 마루타 실험실 시절 때도(with our mutable stacks), 이딴 개꼼수 편법 수작질(done this) 한 뚝배기 치트 난사 편의 융통(We actually could have) 타작이 충분히 가능하긴 했었습니다만, 그딴 병맛 숏컷 지름길 치트키 남발 기행 요행 따윈 이미 인생관 철학을 마스터 달관한 선현 진리 통찰 오만 지식인 참된 이해자들자(people who understand things!) 도인 고수 상급자 클래스 전유물 기만 용도(shortcuts are for) 허세 자위 전유 부속일 뿐 진짜 학문에 정진 매달리는 우리들에겐 사치라 모른 척 넘어갔었단 거 다 압니다!)

우린 아직 지금 당장 `push` 나 `pop` 이 두 단짝 형제 듀오의 그림자 분신격 파생 복사 찐빠 형태 버전인 반대편 백스윙(back 버젼) 도구 모음 단지 세트장 구현(implementing the `_back` versions) 짬처리 작업에 시력을 돌려 치우기 관망 모색 착수 검토 고찰해(could look at) 볼 수도 충분히 있긴 합니다만, 까놓고 저것들 기능 구현 뼈대야 뭐 어차피 그냥 저 앞 단락 복사 카피 드래그 긁어다 붙여먹기(copy-paste jobs) 한 땀 타자 놀음 폰트 위치 반주기 작업 따위에 불과한 귀찮은 타자기 형편없는 것 허접질일 테니 걍 이번 장막의 쩌~기 뒷단 최후미 분량 찌꺼기로 휙 짬 처리 폐기 미뤄 치워 묻어 미뤄둬버릴(defer to later in the chapter) 작정입니다. 그러니까 당장 이 소중 아까운 현시점 타임 찬스 대목 당장 지금 이 순간(For now)만큼은 좀 더 꿀 떨어지고 뇌 도파민 전율 돌아버릴 흥미진진 군침 황당한 요물 장난감 재미 요소 신기원(more interesting things!) 사변 탐구에 시선 환기 돌려 미쳐 봅시다!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
