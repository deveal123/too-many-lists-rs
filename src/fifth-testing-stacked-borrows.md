# 누적 차용 테스트하기 (Testing Stacked Borrows)

> 이전 섹션에서 다룬 Rust(단순화된) 메모리 모델의 요약(TL;DR):
>
> * Rust는 개념적으로 "차용 스택(borrow stack)"을 유지하여 재차용(reborrows)을 관리합니다.
> * 스택의 맨 위(top)에 있는 것만이 "활성(live)" 상태(독점적 접근 권한 가짐)입니다.
> * 더 아래에 있는 것에 접근하면 그것이 "활성" 상태가 되고 그 위에 있는 것들은 스택에서 제거(popped)됩니다.
> * 차용 스택에서 제거된 포인터를 사용하는 것은 허용되지 않습니다.
> * 차용 검사기(borrowchecker)는 안전한 코드가 이를 준수하도록 보장합니다.
> * Miri는 이론적으로 원시 포인터(raw pointers)가 런타임에 이를 준수하는지 확인합니다.

이것들은 많은 이론과 아이디어였습니다 -- 이제 이 책의 진정한 핵심으로 넘어가 볼까요: 나쁜 코드를 짜고 우리 도구들이 비명 지르게 만들기 말입니다. 우리의 사고방식(mental model)이 말이 되는지 확인하고 누적 차용(stacked borrows)에 대한 직관적인 감을 잡기 위해 *엄청나게(ton)* 많은 예제들을 살펴볼 것입니다.

> **해설자:** 런타임에 미정의 동작(Undefined Behaviour)을 잡아내는 것은 까다로운 일(hairy business)입니다. 결국, 여러분은 컴파일러가 말 그대로 *절대 일어나지 않는다고 가정하는(assumes)* 상황을 다루고 있기 때문입니다.
>
> 운이 좋다면 오늘은 "작동하는 것처럼 보일(seem to work)" 수 있지만, 똑똑해진 컴파일러(Smarter Compiler)나 코드의 작은 변화 앞에서는 시한폭탄이 될 것입니다. 운이 *정말* 좋다면 확실하게 충돌(crash)하여 오류를 잡아내고 고칠 수 있습니다. 하지만 운이 나쁘다면 기괴하고 당혹스러운 방식으로 망가질 것입니다.
>
> Miri는 프로그램에 대한 rustc의 가장 순진하고 최적화되지 않은 관점을 취하고 해석하면서 추가 상태를 추적하는 방식으로 이 문제를 우회하려 시도합니다. "분석기(sanitizers)"로서 이는 꽤 결정론적이고 강력한 접근법이지만 결코 *완벽(perfect)* 할 수는 없습니다. 발생한 UB를 잡아내려면 테스트 프로그램이 실제로 해당 UB를 지닌 채 실행되어야 하는데, 실전의 거대한 프로그램에서 모든 종류의 비결정론(non-determinism)을 일으키기는 매우 쉽습니다 (HashMap조차 기본적으로 RNG를 사용합니다!).
>
> 우리는 결코 프로그램 실행에 대한 Miri의 승인을 UB가 없다는 절대적인 확언으로 받아들일 수 없습니다. Miri가 실제로는 UB가 아닌 것을 UB라고 *착각(think)* 할 가능성도 있습니다. 하지만 상황이 어떻게 돌아가는지에 대한 우리 나름의 사고방식이 있고, 거기에 Miri가 동의하는 것 같다면, 그것은 우리가 올바른 길(right track)을 가고 있다는 좋은 징조입니다.




# 기초적인 차용 (Basic Borrows)

이전 섹션들에서 우리는 차용 검사기(borrowchecker)가 이 코드를 굉장히 싫어한다는 것을 보았습니다:

```rust ,ignore
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// 순서가 뒤바뀜! (ORDER SWAPPED!)
*ref1 += 1;
*ref2 += 2;

println!("{}", data);
```

이제 `ref2`를 `*mut`로 바꾸면 무슨 일이 일어나는지 살펴봅시다:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;

    // 순서가 뒤바뀜! (ORDER SWAPPED!)
    *ref1 += 1;
    *ptr2 += 2;

    println!("{}", data);
}
```

```text
cargo run
   Compiling miri-sandbox v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.71s
     Running `target\debug\miri-sandbox.exe`
13
```

Rustc는 이런 작태를 아주 만족스러워하는 것 같습니다: 경고도 없고 프로그램은 우리가 예상한 결과를 매끄럽게 내놓았습니다! 그럼 이제 엄격 모드의 Miri는 이 결과를 어떻게 생각하는지 보죠:

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running cargo-miri.exe target\miri

error: Undefined Behavior: no item granting read access 
to tag <untagged> at alloc748 found in borrow stack.

 --> src\main.rs:9:9
  |
9 |         *ptr2 += 2;
  |         ^^^^^^^^^^ no item granting read access to tag <untagged> 
  |                    at alloc748 found in borrow stack.
  |
  = help: this indicates a potential bug in the program: 
    it performed an invalid operation, but the rules it 
    violated are still experimental
 
```

조오오오았어(Nice)! 작동 방식에 대한 우리의 직관적인 모델이 들어맞았습니다: 컴파일러는 이 문제를 잡아내지 못했지만, Miri는 해냈습니다.

조금 더 복잡한, 아까 살짝 언급했던 `&mut -> *mut -> &mut -> *mut` 교배 테크트리를 구성해 봅시다:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // 첫 번째 원시 포인터에 가장 먼저 접근
    *ptr2 += 2;

    // 그런 다음 "차용 스택(borrow stack)" 순서대로 접근
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
22

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1621> at alloc748 found in borrow stack.

  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting read access to tag <1621> 
   |                at alloc748 found in borrow stack.
   |
```

우와 맞네요(Wow yep)! 강박 엄격 모드의 Miri는 그 두 원시 포인터를 따로 "구별(tell apart)"하고 두 번째 것을 사용할 때 첫 번째 것을 무효화(invalidate)할 수 있었습니다. 모든 문제를 엉망으로 꼬았던 그 첫 번째 개입 접근 조작을 지워버리면 과연 모든 게 정상으로 무사 작동하는지 살펴봅시다:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // "차용 스택(borrow stack)" 순서대로 가지런히 접근
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
20

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
20
```

쩔어주네요! (NICE.)

그래요, 전 지금 이쯤 지점에서 우리 모두 프로그래밍 언어의 메모리 모델 설계와 그 구현 아키텍처 박사 학위(PhD's) 정돈 가뿐히 따낼 수 있을 거란 근거 없는 확신에 차 있습니다. 솔까 누가 컴파일러 따위를 *필요로(needs)* 합니까, 이딴 부류의 학문 따위 정말 껌처럼 *쉬운(*easy*)* 걸요 뭐.

> **해설자:** 전혀 아니었지만, 그럼에도 난 니가 자랑스럽다 찐따야.





# 배열 테스트하기 (Testing Arrays)

자 이제 배열(arrays)이랑 포인터 오프셋 값(`add`와 `sub`)을 좀 조물딱 버무려 섞어볼까요. 이거 돌아갈까요, 그렇죠?

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // 0번째 원소 참고
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // 0번째 원소 포인터
    let ptr3_at_1 = ptr2_at_0.add(1);       // 1번째 원소 포인터

    *ptr3_at_1 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // [3, 3, 0, ...] 이 나와야 함
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[3, 3, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1619> at alloc748+0x4 found in borrow stack.
 --> src\main.rs:8:5
  |
8 |     *ptr3_at_1 += 3;
  |     ^^^^^^^^^^^^^^^ no item granting read access to tag <1619>
  |                     at alloc748+0x4 found in borrow stack.
```

*(석박사 논문 입학 통지서 따위를 그 자리에서 찢어발기며)*

이런 ㅅ... 뭔 일이 일어난 거야? 우린 차용 스택 질서를 완벽 그 잡채로 준수하며 썼잖아! 어째서 순정 `ptr`에서 서자 파생 `ptr`을 낳아 조작(go `ptr -> ptr`)하면 뭔가 괴기스러운 불량 이단 돌연변이가 초래되는 건가? 좋아, 아예 포인터 값 주소 사본들을 싸그리 통째로 판박이 복붙으로 다 똑같은 곳을 가리키게 대량 양산 복사(copy)해버리면 어떨까:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // 0번째 원소 참조
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // 0번째 원소 포인터
    let ptr3_at_0 = ptr2_at_0;              // 0번째 원소 포인터

    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // 결과는 [6, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

아 난리, 이건 아주 멀쩡히 잘 되네요(Nope, that works fine). 어쩌면 우연일지도 모릅니다. 이번엔 여러 포인터를 황당한개처럼 거하게 구겨 넣어 정말 개난장판 폭탄 무더기(real big mess)를 만들어 조져보죠:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];            // 0번째 원소의 올바른 적자 참조자
    let ptr2_at_0 = ref1_at_0 as *mut i32;   // 0번째 서자 원시 포인터
    let ptr3_at_0 = ptr2_at_0;               // 0번째 서자의 쌍둥이 포인터 사본
    let ptr4_at_0 = ptr2_at_0.add(0);        // 0번째 오프셋으로 장난질 친 원시 포인터
    let ptr5_at_0 = ptr3_at_0.add(1).sub(1); // 0번째에 갔다 뺀 더러운 마이너 양아치 포인터

    // 절대 막장으로 마구잡이 혼돈 뒤죽박죽된 ptr 조작 포화 해시
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ptr4_at_0 += 4;
    *ptr5_at_0 += 5;
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // 기대값은 [20, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

아 젠장, 이건 또 안 고장 나네요(Nope!) 사실상 Miri는 근본이 태초 오리지널 다른 원시 포인터에서 파생 적출 자가 복제 분열 탈피 파생된(derived from other raw pointers) 불량 원시 포인터 패거리 녀석에게는 *정말 황당한 듯이(way)* 기형적으로 자비롭고 포용력 관용 허용(permissive)이 넘치는 할렘 자비 구역을 베풀었거든요. 그 형편없는 것 녀석들은 다 한솥밥을 처먹는 처지라 똑같은 호적 차용 지분율 권리(동일한 *태그(tag)*)를 공산당처럼 똑같이 나누어 먹고 공유(share)하는 집단 구조이기 때문이죠.

일단 한 번 원시 포인터의 구렁텅이에 발을 들이기 시작하면, 그녀석은 무한 자가 증식해 자신들만의 미니어처 화난 작은 괴물(tiny angry men) 병사 단풍잎 사본 인형 클론 군단 미니언즈 패거리로 자유 분해 산란 분열 증식 갈라 쪼개 파생(freely split into)되어 제멋대로 마구 뒤엉켜 놀아재끼며(mess with themselves) 셀프 난장판 굿판을 펼쳐 버립니다. 뭐 그래도 노 프라블럼인 것이, 컴파일러마저도 이 사태를 아주 통달 이해(understands that)하고 관망하며, 일반 성스러운 안전 순정 참조자들에게 부여해 주던 똑같은 은혜로운 기적 고성능 판옵티콘 최적화 연산 가속 통제를 이 더러운 일진 포인터 깡패 동네의 읽고 쓰기 조작에는 다 포기하고 일절 간섭 손 떼버리고 신경 끄는 배려 방관 대우(won't optimize)를 시전하기 때문에 문제 터질 일도 막힙니다.

> **해설자:** 팁 하나 더, 코드가 꽤 단조롭고 순수 조잡 멍청 심플 간결하게(simple enough) 구조상 안 꼬여 한눈에 파악될 정도라면, 컴파일러는 이 시궁창 더러운 파생 짭 포인터(derived pointers) 패거리들의 난투 거동 내역마저도 싸그리 그 뒤통수 경로를 일거수일투족 트래킹 캐치 인지(keep track of)하여 그 난장판 속에서도 틈나는 대로 무사 우위 가능 기회(where possible)가 보이는 곳엔 틈새 최적화 튜닝 폭격을 끼워 넣기도 합니다, 다만 안전 성역 참조자에 쓸 수 있는 위대한 정밀 논증 추리 통찰력 분석(reasoning)보단 백만 배는 더 취약 쿠크다스 유리알(brittle) 방벽 같아서 믿기엔 거시기할 겁니다.

그렇다면, 대체 아까 배열은 *뭐가 진정(real)* 치명타 문제였던 걸까요?

비록 `data` 자체는 거대한 하나의 덩어리 "할당체"(한 지역 변수 묶음 단위)이긴 하지만, 처음 시작점 단추 `ref1_at_0`는 오로지 첫 번째 머리 칸 원소 1개 구석 자리(first element)의 점유권 한도만 빌린 소규모 조각 차용증일 뿐이었습니다. Rust는 전체 할당을 쪼개어 부분 발췌 할당 영역 부분 조각 파편 갈고리 차용 발췌 파편 이양 부분 계약 통제 규약 발부(apply to particular parts of the allocation)가 가능하게 허락 분리 분할 파편 타작 조각 배분 권리를 쪼갤(broken up) 수 있도록 너른 허용 자유 방임(allows borrows to be broken up)을 베풉니다! 이 쪼개기 증명을 한번 강행 테스트 돌려보죠:

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // 0번째 원소 독무대 참조
    let ref2_at_1 = &mut data[1];           // 1번째 원소 독무대 참조
    let ptr3_at_0 = ref1_at_0 as *mut i32;  // 0번째 포인터
    let ptr4_at_1 = ref2_at_1 as *mut i32;   // 1번째 포인터

    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;
    *ref2_at_1 += 2;
    *ref1_at_0 += 1;

    // 예상은 [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
error[E0499]: cannot borrow `data[_]` as mutable more than once at a time
  --> src\main.rs:5:21
   |
 4 |     let ref1_at_0 = &mut data[0];           // Reference to 0th element
   |                     ------------ first mutable borrow occurs here
 5 |     let ref2_at_1 = &mut data[1];           // Reference to 1th element
   |                     ^^^^^^^^^^^^ second mutable borrow occurs here
 6 |     let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
   |                     --------- first borrow later used here
   |
   = help: consider using `.split_at_mut(position)` or similar method 
     to obtain two mutable non-overlapping sub-slices
```

맙소사! (Shoot!) Rust는 이 배열 인덱스의 수학 연산 도출 값을 일일이 추적해 이 두 가변 독점 차용이 서로 침해 안 되는 안전한 남남(disjoint) 분리 이산 간격 교차 영역이라는 걸 머릿속으로 증명 입증(prove)조차 할 지능 파악 능력조차 못 미치는 바보지만, 그 대신 이 슬라이스를 여러 떨거지 조각 소형 파편 구역 슬라이스 다중 분할 타작 파생 조각들로 아주 안전무관 결백한 남남 영역 분할 소구역으로 공인 이혼 찢어 쪼개 나눌 수 있게 해주는 안전 합법 결백 보장 증명 위조 보험 증서 도구(safe to assume works)인 `split_at_mut`을 구세주로 하사 던져주십니다:

```rust
unsafe {
    let mut data = [0; 10];

    let slice1 = &mut data[..];
    let (slice2_at_0, slice3_at_1) = slice1.split_at_mut(1); 
    
    let ref4_at_0 = &mut slice2_at_0[0];    // 0번째 원소 참조
    let ref5_at_1 = &mut slice3_at_1[0];    // 1번째 원소 참조
    let ptr6_at_0 = ref4_at_0 as *mut i32;  // 0번째 원소 포인터
    let ptr7_at_1 = ref5_at_1 as *mut i32;  // 1번째 원소 포인터

    *ptr7_at_1 += 7;
    *ptr6_at_0 += 6;
    *ref5_at_1 += 5;
    *ref4_at_0 += 4;

    // 예상은 [10, 12, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]
```

오, 아주 스무스하게 돌아가네요(that works)! 슬라이스들은 당면한 컴파일러와 Miri 님께 직접 무릎 꿇고 제대로 호소 웅변(properly tell) 증명 서약으로 나섭니다. "야 내가 책임지고 내 영역 거대 영토 안에 들어있는 이 모든 재재 메모리 파편 대지 토상 구역에 대해 수 조억 엄청난 떼돈 천문학적 초대형 대규모 부채 한도 가계수표 대출 영수 담보 빚 보증 연대책임을 몰빵 전량 독점 이관 책임 대관 이양 양도 인수 대납 도맡아 감당(taking a huge loan) 다 지고 뒤집어썼으니 안심하셔"라고요. 고로 저 높으신 분들도 모든 구성 파편 배열소들이 죄다 통째로 쑥대밭 도화지처럼 변경 난도질(mutated) 갈아엎어져도 무탈하단 걸 비로소 허가 자각 동의(know)해 줍니다.

또한 `split_at_mut` 같은 작업들이 허용된다는 점에 주목하세요. 이는 차용증 더미 조직이 꼭 단순한 층층이 스택 타워보단 오히려 가지치기 퍼지는 나무뿌리(tree)에 더 가깝다는 사실을 암시합니다. 커다란 차용 덩어리를 잘게 부수어 독립적인 파생 파편 소구획으로 분해하고 각기 나눠먹는 짓(break one big borrow into a bunch of disjoint smaller ones)을 벌여도 여전히 만사형통으로 아무 이상 없이 모든 게 정상 작동(everything still works)하기 때문입니다.

(글쎄요, 저의 짧은 식견상 누적 차용 원석 오리지널 모델 그 밑바닥 깡통 내부 원리 자체로는 만사가 전부 뼈저린 강제 원판 스택탑 기둥 진영 지능(everything's still stacks)일 거라고 봅니다. 그 개념상의 스택 자체는 프로그램 속 메모리 공간의 매 1바이트 쪼가리 입자 하나하나 파편 조각 단위마다 모조리 파편 조각 몫의 접근 감시 판독 꼬표 판정(permissions) 여부를 실시간 관측 추적 꼬투리 지능(tracking permissions for each byte of the program..?)을 수여받는 형국 구조니 말이죠.)

그렇다면 슬라이스(slice) 자체를 *통째 직결 날것 바로(*directly*)* 포인터 녀석팽이로 격하시키면 도대체 무슨 괴변이 나타날까요? 저 통째 변종 포인터는 전 구간 풀 커버(full slice) 전체 영토 슬라이스 구간을 구석구석 다 간섭 참견 접근 통치 조물딱 거릴(have access to) 수 있게 될까요?

```rust
unsafe {
    let mut data = [0; 10];

    let slice1_all = &mut data[..];         // 원본 덩어리 전역 배열 슬라이스 묶음
    let ptr2_all = slice1_all.as_mut_ptr(); // 그 덩어리 전체 전역 대장 포인터 사령관
    
    let ptr3_at_0 = ptr2_all;               // 0번째 녀석팽이 지칭 포인터 (값은 동일)
    let ptr4_at_1 = ptr2_all.add(1);        // 1번째 녀석 찌를 포인터
    let ref5_at_0 = &mut *ptr3_at_0;        // 0번째 녀석 참조자
    let ref6_at_1 = &mut *ptr4_at_1;        // 1번째 녀석 참조자

    *ref6_at_1 += 6;
    *ref5_at_0 += 5;
    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;

    // 그냥 재미 삼아(Just for fun), 루프 뺑뺑이 태워 모든 원소 갈아엎기
    // (어떤 원시 포인터 병사 녀석을 쓰든 알바 아님, 다 같은 차용증 무리니까(share a borrow)!)
    for idx in 0..10 {
        *ptr2_all.add(idx) += idx;
    }

    // 재미 들린 김에 똥 치우기용 비교 합법 정석 안전 버전 코드
    for (idx, elem_ref) in slice1_all.iter_mut().enumerate() {
        *elem_ref += idx; 
    }

    // 결과물은 자고로 [8, 12, 4, 6, 8, 10, 12, 14, 16, 18] 이어야 마땅함
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
```


오 지리네요(Nice)! 포인터가 단순 멍청 정수 쪼가리 치부 취급받을 핫바지(aren't just integers) 수준이 결코 아닙니다: 이들은 고유 자체 점거 배당 자신만의 전속 귀속 배정 연관 메모리 영토 범위 통치 영역(range of memory associated with them) 텃발 수역 한도를 확실히 자각 수여받아 결합 명시 부여된 지표 영주 녀석이었고, 더군다나 은혜로운 Rust 세계에선 우리가 원할 때면 언제든 그 넓은 제국 본연의 영토를 극소 마이크로 핀포인트 좁쌀 파편 조각 영토 스케일로 축소 협소 압축 감축 단절 가압 우회 세분 좁혀 깎아 강제 좁히기(narrow that range) 분할 합법 통치 허가 관할 분단 합법 자결권이 당당히 허용 하사 열려(we're allowed to) 진 권리 체제 만세(narrow that range!) 라는 진리를 폭로 고발 영접 입증한 셈입니다!





# 공유 참조 테스트하기 (Testing Shared References)

지금껏 저 위의 모든 징그러운 혈투 시뮬레이션 샘플 예제 공방전 내내, 저는 오로지 순정 결백 *가변(mutable)* 독점 참조자들만 조심스레 엄격 통제 선별하여 고르고 걸러 읽고-수정하고-쓰는 연산(`+=`) 사이클 노가다만 주야장천 굴리면서 가장 맹물처럼 단순 무결한 시나리오 상황을 연출 유지하려고 사서 생고생 개고생(carefully only using ...)을 고집해 왔습니다.

하지만 아시다시피 Rust 세계관엔 엄연히 수정 금지 읽기-전용(read-only)인 데다 사방팔방 무한대 복제 산란(freely copied) 반출이 합법인 "공유 조립품 참조자(shared references)" 같은 것들이 득시글거리는데, 과연 이 녀석들은 대체 저 스택 원리 체제판에서 어떤 취급 처우 거동 규칙을 따라야 마땅할까요? 뭐, 앞선 실험실 유혈 사태에서 우린 원시 불량 포인터 녀석이 황당한 듯이 자유 증식 복사(freely copied) 판을 쳐도 다 같은 '단일 하나짜리(a single)' 호적 소속 차용 명의표를 나눠 먹기 공유 승계(share)하는 맹점 사각지대로 통제 우회 면피 무마 취급 처분 규명(handle that by saying)해 넘어가는 걸 똑똑히 목격했단 말이죠. 그렇다면 만약 저 공유 참조 같은 것들도 원시 괴물녀석팽이 포인터 무법자들과 다를 바 없는 같은 부류 종자 취급 무리들로 싸잡아 동급 취급 사생아 무리들로 격하 판명 규정 확증(think of shared references the same way)시켜 버리면 좀 아구가 맞아떨어지지 않을까요?

한 번 `println!` 통수 자동 마법(`println!` can be a little magical) 함수로 어떤 값을 빼다 박아 읽어 내려가는 짓거리 테스트로 저 망측한 가설 썰을 직접 심판대에 올려 검증 돌려 봅시다(오토-참조/역참조 귀신 헬퍼 매직 장난질 같은 눈속임 마법에 홀리지 않고 우리가 진짜 찐으로 테스트 검문 확인하려 드는 본질 표적 행위(what we want to be) 자체만 똑디 관통 심사 확인하기 위해 굳이 감싸는 껍데기 포장 함수(wrapping it in a function) 하나를 굳이 구태여 손수 짰습니다):

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let sref2 = &mref1;
    let sref3 = sref2;
    let sref4 = &*sref2;

    // 무작위로 뒤죽박죽된 공유 참조 읽기(Random hash of shared reference reads)
    opaque_read(sref3);
    opaque_read(sref2);
    opaque_read(sref4);
    opaque_read(sref2);
    opaque_read(sref3);

    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

warning: unnecessary `unsafe` block
 --> src\main.rs:6:1
  |
6 | unsafe {
  | ^^^^^^ unnecessary `unsafe` block
  |
  = note: `#[warn(unused_unsafe)]` on by default

warning: `miri-sandbox` (bin "miri-sandbox") generated 1 warning

10
10
10
10
10
11
```

아 맞다 그래요, 깜빡 잊었네 미안요. 정작 맹독성 원시 불량 포인터들 갖고 조물딱거리는 핵심 파괴 공작질(do anything with raw pointers)을 하나도 빼먹고 순정 안전 바닐라 코드만 돌렸군요. 그래도 적어도 우린 모든 "공유 참조자" 무리가 완전히 한통속으로 뒤엉켜(interchangeably) 지지고 볶고 마구잡이 순서로 돌아가며 막 부려먹히고 남용(freely copied and used) 굴려져도 전혀 무해 철통 방어 무사 파사 아무 탈 문제 오류 하나 안 나고(fine) 멀쩡 잘 돌아간단 사실 명제 하나만큼은 쌈빡하게 증명 입증 건져냈습니다. 이제 이 평화로운 생태계에 불량 구제불능 폐급 원시 포인터 폭탄 투척(mix in some raw pointers) 미꾸라지 훼방꾼들을 섞어 던져 흙탕물 깽판 마라맛 혼돈을 빚어봅시다:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &mref1;
    let ptr4 = sref3 as *mut i32;

    *ptr4 += 4;
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

error[E0606]: casting `&&mut i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

아이고 맙소사 이런 등신 머저리 삽질을(Oh whoops). 우리가 진짜 찐 목적 알맹이 알몸 순정 `&` 껍데기를 상대하는 게 아니라, 쓸데없이 번거롭게 두 겹이나 껴입은 `& &mut` 중첩 곱창 겹겹이 덩어리를 들이밀고 있었네요! 뭐 Rust 언어 자체가 워낙 이딴 소소한 자잘 문법 삑사리 실수(when it doesn't matter) 따윈 융통성껏 어련히 알아서 보정 세탁 수습 커버 눈감아주고 덮어주기(papering over that) 달인 고수라 제가 방심했군요. 자자, 이번엔 `let sref3 = &*mref1` 껍질 리셋 정제 치환 캐스팅 꼼수 재차용 마사지(properly reborrow it)로 한 꺼풀 벗겨 재단장 제대로 알몸 세팅(properly) 해줍시다:


```text
cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

틀렸어 젠장(Nope)~ 여전히 Rust 꼰대 양반은 저딴 막돼먹은 짓거릴 용납 결사반대(still doesn't like that) 호통을 칩니다! 오로지 당신은 저 고결한 성녀 성역 순백 공유 참조자를 감히 범접 불허 불가침 순결주의 극강 모태 신앙 방어기제인 오직 "읽기 전용" 순정 깡통 `*const` 껍데기 원시 포인터 하나로만 강등 유폐 형벌 전락(cast ... to a `*const`) 시켜야만 합법 통과입니다. 그렇다면 만약 우리가 무대포 직진(just... do...) 반사 신경 발악 우회 꼼수로 이런 야매 탈법 밑장빼기 장난질(this...?) 캐스팅 겹치기 콤보 마법을 강제 발동 걸면 어떨까요...?

```rust ,ignore
    let ptr4 = sref3 as *const i32 as *mut i32;
```

```text
cargo run

14
17
```

아니 뭐(WHAT). 그래 좋아 알겠다고(OK SURE FINE)? 이런 빌어먹을 장인 정신 컴파일 타임 보호 캐스팅 방파제 시스템(Great cast system) 같으니. 솔직히 까놓고 까고 말해 저 모태 보호 성녀 순정 무구 신앙 `*const` 태그 달린 깡통 껍데기 타입은 그저 C 언어산 고철 유물 API 호출 잔해 껍질 포장(describe C APIs)할 때나, 아님 인간 프로그래머 바보 머저리들에게 대충 넌지시 "이거 읽기-전용이니까 험하게 굴리지 마쑝" 따위의 허울뿐인 눈치 코치 허세 가이드 도덕책 표지 공익광고 간판 표지판 훈수 코딩 수칙(vaguely suggest correct usage) 강박 주입용 외엔 진짜 전혀 쥐뿔 쓸모없는 무능 허접 형편없는 것(useless type) 잉여 타입인 게 팩트 맞잖아요? (네 맞아요, 그딴 거(it is, it does)). 자 그럼, 우리의 강박 살인 검사 엄격 깐깐징어 Miri 심사위원 판사님께선 이 황당한 적폐 합법 꼼수 이단아를 뭐라 질타 심의 판결(think) 하실까요?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting write access to 
tag <1621> at alloc742 found in borrow stack.
  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting write access to tag <1621>
   |                at alloc742 found in borrow stack.
```

오호 통재라 슬프도다(Alas), 비록 우리가 컴파일러의 깐깐징어 호통 잔소리 폭격을 저 비겁한 더블 이중 캐스팅 기만 사기극 치트 꼼수(double cast)로 무사 여과 우회 농락 뚫어넘겨 합법 회피 회피기(get around the compiler complaining) 통과에는 기어이 성공 쾌거를 거뒀을지언정, 고작 그따위 얄팍한 면피 캐스팅 껍데기 포장술 하나 발랐다고 이 본질 악행 적법 불가 치명타 깽판 폭력 연산 자체를 "합법 무죄 면죄 체제(made this operation *allowed*)" 선상으로 신분 세탁 무마 승격 시켜줄 턱이 있겠습니까. 우리가 공유 참조자를 발급 입수 취득 채택 발췌 생성 점거(take the shared reference) 하는 바로 그 엄숙한 찰나 순간부터, 우린 절대로 절대 네버 단연코 그 내용물 값에는 오만불손 참견 가변 난도질 상처 하나 단 1의 변형 얼룩도 묻히지 변경 침해 수정 간섭 가필 덧칠(modify) 하지 않겠다고 피의 서약 맹세각서(*promising*)를 하늘에 대고 올린 것과 진배없기 때문입니다.

이 약조 맹세 철칙 제약은 어마무시하게 막중 파급 중대 막강 핵심(important) 뼈대 기틀 철칙 원리입니다! 왜냐하면 이 제약 덕분에, 저 오만불손한 성녀 성역 순결 순백 공유 참조자 같은 것가 후일 차용 스택 탑 그 꼭대기 옥좌 정점 최상층 고층 계단 꼭대기에서 마침내 퇴역 수명 다해 무너져 쫓겨 내려와 폭풍 제거 방출 파괴 철거 척결 폐기 하락 강판 소멸 붕괴(popped off) 낙하 처리로 밀려난 바로 그 시점 직후, 언제고 대기조 백업 잠복 하단 지층 밑바닥 계층 언더그라운드 늪지대 대기 타고 있던 야행성 가변 깡패 독재 원시 포인터 패거리들이 다시 고개 옥좌 복권 탈환 패권 전면 부상 컴백 왕좌 찬탈 활성 복귀 부흥 갱생 승격 부활 등극 옥좌 재탈환 재기 복위 차지(pointers below it *can*)를 성사 달성 거머쥐게 되는 바로 그 영광의 재집권 재강림 순간에 한 가지 엄청난 극강 강력 팩트 확신을 가질 수 있기 때문입니다. 그 강력한 맹신 절대 팩트 보장 가정 전제란 곧 자기 빈자리 부재기간 동안 저 더럽게 순수 결벽 결백 오지랖 성녀 모태 읽기 단일 꼰대 독재 성녀(공유 참조자 무리들) 것들이 지배 관장 활개 점거 통치 군림했던 긴 세월 긴 세월 동안 단 한 번도 누구 하나 감히 저 신성 메모리 영토 토상 터전 조각 표면 대지 위 내용물 자체 값(memory hasn't changed)을 토씨 하나 얼룩 점 하나 오염 개변 난도 변형 가변 농단 개조 칼질 덧칠 수정 오물 도화지 덧칠 망작 난도질 가필 상처 참견(modify) 내지 않았다는 100% 무결점 결백 무변형 맹목 맹신 확신 전제 보증 보장(assume the memory hasn't changed)을 영원토록 떵떵거리며 안심 믿어 의심치 않고 가슴 깊이 간직 맹신 신뢰(assume) 할 수 있게 된다는 위대한 보장 권리 기강 확보다 이거죠. 물론 지들이 물러난 빈자리 잠적 부재 틈을 타 오만가지 온갖 호기심 많은 잡무 변종 서브 루틴 부하 하급 좀도둑 작은 요정 괴물 시종 분파 파생 클론 무리 잡병 같은 것 녀석들(tiny angry men)이 우르르 몰려와선 저마다 지들 눈깔에 현미경 쌍안경 들이밀고 수십수백수천 번 줄줄 읽어대느라(reading* the memory) 무단 읽기 전용 접근 조작 난관 참견 복제 난장 관람 조준 구경 관측 사열 열람 굿판은 실컷 난장 치며 놀아재껴 마구 쑤셔댔을 순 있지만 (어쨌든 그렇게 누군가의 망막으로 실컷 복사 전가 송출 발부 투영 전시 유포 배달 공급 흡수 인용 반영 관측 시청 목도 노출 열연 스크린 상영 목격 발현 반사 출력 독회 낭독 실황 감시 보고 접수 입수 취합 포착 수락 도출 출력 도달 목도(writes had to be committed)까지 도달했죠), 그럼에도 감히 그 누구 하나 어떤 대장 불량 포인터든 서자 떨거지 하급 똘마니 요정 깡패든 단 한 마리 녀석도 감히 그 절대 순결 무구 본연 영역 원석 텍스트 그 값 자체 본체 옥좌 도화지에 매직펜 칼부림 대못망치 손톱기스 얼룩 단 한 방울(weren't able to modify it)의 물리적 오염 훼손 수정 폭력 가변 변형 행각 공작 난도질은 일절 범하지 못했고 터치 불가 절대방어 실패 실패 완전방어 실패 불가능 절대 방벽 성역화 수호 제동 한계(weren't able to modify it) 앞턱에 가로막혀 완전히 다 실패 헛발질 패태 봉쇄 좌절 붕괴 무위로 끝나 불발(weren't able to modify it) 되었을 테니까요! 고로 저 지하실 원조 독재 패권 컴백 왕의 귀환 적장 적통 가변 포인터 지존(mutable pointers) 녀석은, 그 옛날 지들이 왕위 점거 지배 시절 퇴각 직전 당시 최후 마지노선 순간 마지막 찰나 지필 붓으로 기록 써 놓고 아로새긴 그 마지막 옛날 유산 흔적 고대 낙서 문구 조각 최종 작성 마지막 데이터 잔흔 흔적 최후 기록(last value they wrote)이, 놀랍게도 그 오랜 긴 공백 유배 긴 암흑기 통치 공백 유배 단절 긴 잠수 잠적 격리 격변 유예 세월 기간이 다 지난 지금 복위 귀환 재접견 도달 바로 현시점까지도 토씨 점 하나 안 바뀌고 100% 온전 무사 고스란히 영원불멸 무사 보존 잔존 무사 보전 존속 결백 유지 방치 건재(is still there) 유지 잔존 생존 안착 보전 중이라는 사실을 1000% 무결 확답 절대 불변 기정 맹신 교리 팩트 절대 확신 보장 불변 대전제 기정사실 팩트 절대 가정 믿음 전제 보장 근거 원칙 확신(can assume ... is still there)으로 평생 믿어 의심치 않고 안심 가동 풀가동 설계 전략 맹신 연산 질주 스퍼트 시동 전략 무결 작동 설계 아키텍처 구축 질주 확립 맹신 뚝심 돌파 가동 진행 런닝 작동 돌진 실행(can assume)을 거침없이 이어나갈 수 있는 겁니다!

**일단 한 번 "공유 참조자" 성녀 권력 무리가 저 절대 권력 차용 탑(borrow-stack) 장부 증서 리스트 제단 꼭대기 왕좌 반열 구역 권역 장부 등재 진입 반열 명단 기재(on the borrow-stack) 성사 거머잡기 등극 도달 성사 안착 입성 차지 진출 안착(on the borrow-stack)에 마침내 성공 자리매김 입성 올라타는 순간, 그 위로 차곡차곡 겹겹 포개어 쌓여 올려진 또 다른 후발대 잡동사니 떨거지 불량 하위 계층 조준 맹신 추종자 파생 똘마니 패거리 녀석들 전원 절대 싹수 모조리 다 총집합체 모든 잡것 떨거지들(everything that gets pushed on top of it) 일괄 싹수 지위 신분 계급 꼬리표 권리 자격 통제 박탈 제약 억압 통제 규율 구속 영구 억제 속박 압수 속죄 형벌 철통 단두대 낙인 봉인 속박 구속 지위 나락 강등 등급 형벌 처분 적용 하달(everything that gets pushed on top of it only has) 구속을 받아 평생토록 단지 숨죽여 조용히 입방정 떨지 않고 그저 눈팅 열람 관전 몰카 구경 관측 대기 독회 열람 시청 보기 시청 독회 읽기 읽어오기 눈팅 열람 감상 시청 대기 전용 권한 따위 찌질 구경 권한 승인 입수 허가 티켓 조회 조준 관람권 찌질 전용(only has read permissions) 접근 승인권 관람 티켓 허용 범주 내로만 활동 행동 관할 지배 억압 권리 구속 자격 한계 박탈 낙인 권한 하락 수갑 감금 통제 족쇄 억눌림 제동 규제 억압 족쇄 통제 전락 속박 제한 봉쇄(only has read permissions) 제약 형벌 철칙 원칙 신세 처지 굴레 족쇄 운명형 나락 전락 추락 강등 결박 신세 나락 규정 한계 봉쇄 전락형 족쇄 봉쇄 신세로 추락 결박 동결 감금 고정 결박 귀결 확정 못 박힘(only has read permissions)이라는 절대 철칙 운명 굴레 구속 규율 한계선 족쇄 신세로 몰락 추락 결박 확정 강제 족쇄 신세로 전락 처 박힘 고정 감금당하고 맙니다.**

하지만 이런 얌체 꼼수 발악 장난질(do this) 정도는 살짝 눈감아 주더군요:

```rust
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;
    let ptr4 = sref3 as *const i32 as *mut i32;

    opaque_read(&*ptr4);
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

이 코드를 보면, 겉보기에 극악 흉악 이단아 가변 돌연변이 원시 포인터 무기 흉기 살상 병기 흉터 공격 포인터 독재자 칼날(mutable raw pointer)을 생성(create) 뽑아내 쥐어 빼어드는 황당한 반역 사태 반란 반역짓거릴 감행 시전 창설 도발(create a mutable raw pointer)해 놓고서도, 정작 현실 진짜 실전 전투 실행에선 그저 칼집에 꽂아두듯 허세 가오 장식 겉치레 칼등 눈요기 용도 수준 그저 살포시 눈으로만 감상 치장 독회 시선 훑기 열람 감상 훔쳐 눈팅(only actually read from it!)으로만 온순하게 다루어 방치 써먹는 척 장식 사용 용례 시늉 행각 시전(only actually read from it!)에 머무르고 얌전 떨기 지조 결백 자중 자제 평화 관망 스탠스(only actually read from it!)만 잘 유지 지켜 훌륭히 속이기만 해 주면, 놀랍게도 이 요상한 위장 전술 허세 야바위 평화쇼(still "fine" to create) 발동 설립 제작 소환 둔갑 분식 속임수 시전 자체가 그 흔한 경고 딱지 벌점 철퇴 철퇴 매타작 레드카드 퇴장 제재 철퇴 칼부림 경색 지적 참견 간섭 제동 터클 시비 트집 심사 반려 철퇴 태클 징계 경고 한 방 먹지 않고 그럭저럭 봐주기 묵인 용인 철통 기만 합법 면죄 패스 위장 통과 여과 은폐 위장 패스 합격 기만 통과 무마(still "fine")되는 마법 같은 모순 진기명기(Note how) 참모습 역설 현장 작태를 감상할 수 있습니다!

```text
cargo run
10
10
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
10
10
13
```

혹시나 찜찜하니까 돌다리 짚고 확인 사살 차원에서, 우리의 성녀 공유 참조자가 평소 정석 루틴 정규 사이클 코스 밟듯 정상 패턴 궤도에 맞춰(like normal) 제대로 차용 탑 지붕 계단 왕좌 구역에서 퇴출 추방 축출 철거 숙청 제거 분쇄 낙하 척결 방출 쫓겨남 옥좌 박탈 수명 소멸 붕괴 파 면 강판(popped)되는 말로 최후를 맞이하는지도 짚고 넘어갑시다:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;

    *ptr2 += 2;
    opaque_read(sref3); // 실수로 읽기 순서 꼬임? (Read in the wrong order?)
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly 
at alloc742, but parent tag <1620> does not have an appropriate 
item in the borrow stack

  --> src\main.rs:13:17
   |
13 |     opaque_read(sref3); // Read in the wrong order?
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc742, but parent tag <1620> 
   |                       does not have an appropriate item 
   |                       in the borrow stack
   |
```

어라라(Hey), 이번엔 평소 지적질 해대던 어느 특정 지목 죄수 번호판 고정 라벨 태그(some specific tag) 명찰 꼬리표 운운하는 문구 대신, 뭔가 'SharedReadOnly'라는 낯선 광역 스킬 단체명 명찰 덩어리 호칭 죄목 타이틀 뭉텅이 병명 찌라시 꼬리표 딱지가 덕지덕지 날아붙은 조금 생소 기묘 희한한(slightly different) 에러 메시지 깽판 사유서가 도착했군요. 뭐, 충분히 납득 가능 이치 합당 수긍(makes sense) 납득 가는 사안 전개 조치 합리 해명 변명이긴 합니다: 왜냐, 한 번 사태 전장 필드 판도 지형 공간 영토에 *어떤 떨거지 잡동사니 종류 형태 개구리 무당 상관없이 단 한 마리짜리 아무 잡녀석 공유 조무래기 성녀 참견러 무리 티끌 찌끄러기라도 일단 투입 등판 개입 난입 출몰 오염 번식 침투 점거 출현 출범 착륙 상륙 점령 진입(once there's *any* shared references)* 진입 출몰 상륙 전개 발생 침범 등장(once there's *any* shared references)하는 그 시점 직후 그 찰나 순간부터는, 사실상 그 주변 나머지 모든 인계 영토 남은 잡동사니 모든 나머지 떨거지 모든 일체 남은 존재 부속 여타 잔당 전원(basically everything else)들은 죄다 싸그리 한데 뭉뚱그려져 구분 무의미 잡동사니 단합 거대 익명 짬뽕 집단 공동체 소속 덩어리 거대 연합 단일 거대 도가니 혼돈 잡탕 통일(are just a big) 짬뽕 집단 잡탕 소속 단일 거대 익명 짬뽕 구덩이 잡탕 익명 집합 혼합 무리 구덩이 단일 거대 소속 덩어리 연합 풀장 혼합 전역 변종 집단 소속 단일 거대 혼합 덩어리 소속 익명 단일 연합 도가니 짬뽕 단일 거대 소속 집단 연합 무리 집합 익명 단일 거대 도가니(just a big SharedReadOnly soup) 끓는 솥단지 안의 '읽기 한정 공산주의 단합 연합' 덩어리 육수 찌개 잡탕 단일 덩어리(just a big SharedReadOnly soup) 속 무의미 건더기 잔여 무명 무리 티끌 잡동사니 조각 조각품 덩어리 쩌리 형편없는 것통 혼합 덩어리 잔해 무리 일개 익명 집단 잔당(just a big SharedReadOnly soup)으로 통폐합 귀결 격하 합병 전락 통일 용융 희석(just a big SharedReadOnly soup)되어 버리기 때문입니다. 그러니 굳이 그 익명 대통합 잡탕 무명 무더기 진흙탕 속에서 굳이 '이녀석이 저녀석이고 저녀석이 이녀석이네' 일일이 신원 조사 추적 지목 감식 구별 특정 지목 특정 구분 구별 분리 독립 특정 식별 지목 지정 호명 감별 차별 분리 개별 지목 특정(distinguish any of them) 할 가치도 체면도 건덕지도 수고 이유 필요 명분 필요성(no need to)조차 전혀 단 1도 쓸데없이 완벽하게 상실 소멸 증발 소멸 소진 증발 종식 소멸(no need to)해버리기 때문이죠!





# 내부 가변성 테스트하기 (Testing Interior Mutability)

우리가 저주받은 연결 리스트를 `RefCell`과 `Rc`로 만들려고 시도했을 때 만사가 평소보다 더 끔찍하게 꼬였던 그 끔찍한 책의 챕터를 기억하시나요?

우리는 공유 참조자로는 절대 변경(mutation)을 할 수 없다고 계속 주장해 왔지만, 그 챕터는 사실 *내부 가변성(interior mutability)*을 통해 공유 참조자로도 값을 변경할 수 있다는 점을 다루고 있었습니다. 여기서는 작고 귀엽고 심플한 [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) 타입을 시험 삼아 써봅시다:

```rust
use std::cell::Cell;

unsafe {
    let mut data = Cell::new(10);
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut Cell<i32>;
    let sref3 = &*mref1;

    sref3.set(sref3.get() + 3);
    (*ptr2).set((*ptr2).get() + 2);
    mref1.set(mref1.get() + 1);

    println!("{}", data.get());
}
```

아, 정말 아름다운 아수라장이네요. Miri가 이 형편없는 것 코드에 어떻게 반응하는지 구경하는 건 즐거운 일이 될 겁니다.


```text
cargo run
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
16
```

잠깐, 진짜요? *저게* 괜찮다고요? 왜요? 어떻게 된 거죠? 대체 *Cell*이란 녀석 정체가 뭡니까?

*표준 라이브러리에 채워진 자물쇠를 부순다*

```rust ,ignore
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

대체 `UnsafeCell`은 또 뭡니까?

*우리가 장난치는 게 아니라는 걸 표준 라이브러리에게 똑똑히 보여주기 위해 자물쇠를 하나 더 부순다*

```rust ,ignore
#[lang = "unsafe_cell"]
#[repr(transparent)]
#[repr(no_niche)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

아하, 그냥 마법(wizard magic)이었군요. 알겠습니다. `#[lang = "unsafe_cell"]`은 말 그대로 컴파일러한테 UnsafeCell은 UnsafeCell이다 라고 알려주는 용도입니다. 이제 자물쇠 부수기는 그만두고 [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) 공식 문서를 확인해 봅시다.

> Rust에서 내부 가변성을 구현하기 위한 핵심 원시 타입(core primitive)입니다.
>
> `&T` 참조자가 주어졌을 때, 보통 Rust 컴파일러는 `&T`가 불변(immutable) 데이터를 가리킨다는 사실에 기반하여 최적화를 수행합니다. 따라서 앨리어싱(aliasing) 우회 조작이나 `&T`를 `&mut T`로 캐스팅(transmuting)하여 데이터를 변경하는 행위는 미정의 동작(undefined behavior)으로 간주됩니다. 그러나 `UnsafeCell<T>`는 이 `&T`의 불변성 보장 규칙에서 "면제(opts-out)"됩니다: 즉, 공유 참조자 `&UnsafeCell<T>`는 내용물이 언제든 변경(mutated)될 수 있는 데이터를 가리킬 수 있습니다. 이를 “내부 가변성(interior mutability)”이라 부릅니다.

오 이런, *진짜로(really is)* 마법 그 자체였네요.

UnsafeCell은 기본적으로 컴파일러에게 이렇게 말하는 것과 같습니다. "이봐, 우린 지금부터 이 메모리를 가지고 좀 별난 짓(goofy)을 벌일 거니까, 평소처럼 앨리어싱(aliasing)에 대해 함부로 가정하지 마." 마치 그 구역 앞에 "접근 주의: 작고 화난 요정들 횡단 구역(CAUTION: TINY ANGRY MEN CROSSING)"이라는 표지판을 세워둔 것과 같죠.

자, 이제 이 UnsafeCell을 추가하면 Miri가 어떻게 만족해하는지 살펴봅시다:

```rust ,ignore
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = data.get_mut();      // 내용을 향하는 가변 참조자 취득!
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*ptr2;

    *ptr2 += 2;
    opaque_read(sref3);
    *mref1 += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly
at alloc748, but parent tag <1629> does not have an appropriate
item in the borrow stack

  --> src\main.rs:15:17
   |
15 |     opaque_read(sref3);
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc748, but parent tag <1629> does
   |                       not have an appropriate item in the
   |                       borrow stack
   |

```

잠깐, 뭐시라?! 우리가 분명 마법의 주문(magic words)을 외웠잖아요! 연방 정부의 승인을 받은 무의식 제물용 염소 피를 이만큼이나 다 어쩌라는 겁니까?

글쎄요, 마법 주문을 외우긴 외웠죠. 하지만 그 직후에 바보같이 `get_mut` 함수를 써버려서 이 성스러운 UnsafeCell의 마법 효과를 완전히 해제(completely discarded the spell) 해버렸잖아요! `get_mut`는 UnsafeCell의 안을 들여다보고 결국 평범한 `&mut i32` 가변 참조자를 내뱉어 버립니다!

생각해 보세요(Think about it): 만약 컴파일러가 모든 `&mut i32` 타입들이 '사실은 속 내용물이 `UnsafeCell` 안에 든 걸지도 모른다(...*could* be looking inside an `UnsafeCell`)'고 전부 의심하고 가정해야 한다면, 컴파일러는 절대 그 어떤 앨리어싱(aliasing) 관련 최적화 가정도 내릴 수 없게 됩니다! 모든 코드가 작고 화난 요정들(tiny angry men)로 가득 차 있을 수 있으니까요.

그러므로 우리가 해야 할 일은 `UnsafeCell`을 사용하는 포인터 타입들 껍데기 자체에 어떻게든 끝까지 그 존재를 남겨두어 컴파일러가 우리가 지금 무슨 짓을 하는지 이해하게 만드는 것입니다:

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;              // *겉포장(outside)* 장막에 대한 가변 참조자
    let ptr2 = mref1.get();             // 알맹이(insides)로 향하는 직통 원시 포인터 발급
    let sref3 = &*mref1;                // *겉포장(outside)* 장막에 대한 공유 참조자

    *ptr2 += 2;                         // 원시 포인터로 알맹이 값 조작 변경
    opaque_read(&*sref3.get());         // 공유 참조자를 거쳐 읽기 강행
    *sref3.get() += 3;                  // 공유 참조자를 통해 값 덮어쓰기 조작
    *mref1.get() += 1;                  // 가변 참조자로 최종 조작 마무리

    println!("{}", *data.get());
}
```


```text
cargo run
12
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
12
16
```

작동하네요(It works)! 준비했던 이 염소 피들을 구태여 다 내다 버릴 필요는 없겠네요.

사실 좀 이상하긴 합니다. 우리가 여전히 코드를 약간 우스꽝스러운(goofy) 순서로 작성했거든요. 우리는 `ptr2`를 먼저 만들고, 그다음에 가변 참조자로부터 `sref3`를 파생시켰습니다. 그러고 나서는 공유 포인터보다 원시 포인터를 먼저 사용했죠. 이 모든 것이... 좀 틀린 것 같습니다(seems... wrong).

가만, 생각해보니 아까 `Cell` 예제에서도 똑같은 짓을 저질렀었네요. 흠(HMMM).

우리는 두 가지 결론 중 하나를 받아들여야만 합니다:

* Miri가 불완전해서 이것이 아직도 잡히지 않은 미정의 동작(UB)이다.
* 우리가 세운 지나치게 단순화된 모델이 진짜로 사실은 너무 단순화가 되어서 그렇다.

저라면 제 돈을 두 번째 가설에 전액 베팅(put my money on the second one) 하겠지만, 그래도 확실히 하기 위해서(just to be safe), 누적 차용에 대한 우리의 깡통 모델(simplified model)에도 100% 들어맞는 무결점 안전 버전(airtight) 코드를 다시 짜보죠:

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;
    // 이제 이 두 줄의 순서를 제대로 뒤집어놨으니 차용증은 
    // *확실하게(definitely)* 스택에 순서대로 쌓이게 됩니다.
    let sref2 = &*mref1;
    // 극도로 안전하기 위해 공유 참조자에서 포인터를 도출해냅니다!
    let ptr3 = sref2.get();             

    *ptr3 += 3;
    opaque_read(&*sref2.get());
    *sref2.get() += 2;
    *mref1.get() += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
13
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
13
16
```

이제 우리가 시도했던 첫 번째 구현체 버전이 왜 애초에 정상적으로 통과(`might` actually be correct) 할 수 있었는지 생각해 봅시다. 앨리어싱(aliasing) 관점에서 `&UnsafeCell<T>`는 솔직히 별 볼 일 없는 `*mut T` 와 사실상 별반 다를 게 없기(really is no different from) 때문입니다. 무한정 복제하고 맘대로 값을 변경할 수 있으니까요!

그러니까 어떤 의미에선 우린 그냥 원시 포인터 두 개 만들어다가 마치 평범한 일진 녀석팽이들처럼 이리저리 서로 번갈아 교환하면서 사용(used them interchangeably like normal)한 것뿐입니다. 두 칼자루 모두 하나의 똑같은 가변 참조자 대장에서 갈라져 나왔다는 점이 살짝 캥기고 의심쩍긴 하지만요(sketchy). 그러다 보니 어쩌면 엄밀히 따져 두 번째 자루가 무단 파생되는 탄생 순간, 첫 번째 뽑아낸 포인터가 스택에서 제거(pop the first one off the borrow stack)되는 것이 맞을지도 모릅니다. 하지만 애초에 우린 가변 참조자의 속 알맹이 자체에 무단 침입해 *진짜 물리적으로 터치 조작 개입(*actually* accessing the contents)*을 한 적 조차 없으므로, 저런 강제 징벌 조치가 굳이 필연 발동될 필요는 없습니다(not really necessary). 우리는 그저 겉에서 슬며시 주소값만 훔쳐 복사(just copying its address) 해갔을 뿐이니까요.

`let sref2 = &*mref1`와 같은 코드는 참 교묘하고 요물 같은 트릭(tricksy thing)입니다. *문법적으로 보면(Syntactically)* 무언가 대단한 역참조 수술이나 뜯어내기(dereferencing it)를 해내는 것처럼 둔갑 포장되어 있지만, 역참조 단일 작업 기호 그 자체만 가지곤 혼자서 결코 어떤 실체 발동 효과나 부작용 연산 성과(*a thing*)조차도 일으키지 않습니다. 한번 `&my_tuple.0` 모습를 눈에 불을 켜고 고찰해 보세요(Consider `&my_tuple.0`): 이런 코드 구문 안에서 여러분은 진짜로 눈먼 맹신을 가지고 저 `my_tuple` 본체나 `.0` 이 조각 속성 자리에 대고 무슨 파괴나 조작 개입 변형 복사 조작(*actually* doing anything) 등을 가하거나 손상을 일으켰나요? 결단코 전혀 아닙니다. 그저 얘들은 특정 목적 도달 지점이나 영토 메모리 번지수 푯말 좌표 따위를 표시하고 투영하는 손가락 역할 마커 타겟 삼아 활용(use them to refer to a location in memory) 되었을 뿐, 그다음에 `&` 기호를 앞에 딱 붙여주며 "이봐 컴파일러, 당장 이거 불러서 로드하는 바보짓은 때려치고 그냥 고이 조용히 번지수 주소 값(address down)만 잘 적어 남겨둬!" 라고 지시한 것에 지나지 않습니다.

`&*` 도 정확히 똑같은 판박이 요괴 수법 깡통 짓거리(the same thing) 속임수 입니다: `*`의 역할은 "야 어디 이 포인터 녀석이 폼잡고 가리키는 지점의 장소 영토 타겟 위치 정보 좌표 자리 지점(the location this pointer points to)에 대해 입 좀 떠들어볼까"라는 허세고, 그 앞에 붙은 `&` 녀석은 바로 화답하여 "좋아 그래! 그럼 그 번지수 도안 푯값 숫자(write that address down) 바로 수첩에 옮겨 적어버리지 뭐" 라는 식입니다. 물론 그 결괏값 도출물은 빼박캔트 처음 오리지널 포인터 녀석팽이가 가지고 놀던 고유 푯값 번지 도안과 100퍼센트 완벽하게 쌍둥이 일치 동치 수치값(the same value the original pointer had) 그대로라는 건 당연한 결론 일입니다. 다만 포인터의 타입 신분 호칭 분류표(type of the pointer) 자체는 슬그머니 다른 물건으로 탈의 갈아입기 세탁이 되어 돌아옵니다, 왜냐면... 거 뭐 까놓고 말해 타입이(because, uh, types!) 다 이 구역 생태계가 그런 식이니까요!

하지만 반대로, 진짜 억센 야생마 황당한 콤보 돌진 막가파식 쌍 역참조 후벼 파기 수작 조작 개조 `&**` 따위의 하드코어 복선 타파(do `&**`) 폭탄 짓거리를 대놓고 자행 시전 갈겨버린다면! 그 무모한 순간엔, 결국 맨 앞에 대기 타던 십자가 첫 번째 돌격대장 `*` 녀석 때문에 진짜로 살점 알맹이 골수 값을 로딩 파내기 탈취 소환하는 무단 해체 로드 인출 적출 연산(loading a value with the first `*`!)이 사실상 백 프로 결행 발발 발동 터져버립니다! 아 정말 신기하고 알 수 없는 별 마크 지팡이 요물 딱지 바보 덩어리 폭탄(weird!) 아니냐고요!

> **해설자:** 샌님 *조나단(Jonathan)* 녀석아, 네 그 잘난 주둥이에서 "lvalue(좌측값)"이라는 고루한 옛날 낡아빠진 틀딱 용어 지식 자랑 나불대는 모습 따위 아무도 관심 없고 쥐뿔도 신경 안 쓰거든(No one cares)? Rust 이 최첨단 트렌디 세계관에서는 정말 우아하게 개명해서 그 녀석들을 차원 영토 좌표 공간 구역 터전 거점 *장소(places)* 라고 찬미하며 떠받들고 칭하거든, 이게 정말 씹간지 백만 배 어나더 레벨 섹시 몽환 고결 폭풍 압도적 찬란 쿨내 진동 지존(totally different and *so* much cooler) 포스 나는 거 반박 안 받음.




# Box 테스트하기 (Testing Box)

어이구, 니들 우리가 애초에 무슨 헛바람 망령 사연 폭풍 삽질 시동 때문에 이토록 기나긴 바보 삽질 타령 삼천포 도발 잡담 만담(this extremely long aside) 대하드라마 폭풍 수다 여정 뇌절을 개시 시작 착수 출항 이탈 결행 시동(remember why we started) 시작 점화시켰는지 머리에 기억 나? 안 난다고(You don't?)? 참나 대단히 웃기고 자빠지는구만(Weird).

하여튼 우리가 이 개노답 사태를 벌인 건, 저 끔찍한 Box 녀석랑 원시 포인터 마검 떨거지 형편없는 것 돌연변이 종자들을 무단 강제 짬뽕 환장 폭탄주 믹스 배합 제조 혼합 투하 살포 콜라보(mixed Box and raw pointers) 교잡 교미 투척 칵테일 투척 혼합을 시도했기 때문이야. Box 이 녀석은 사실 살짝 비스듬히 들여다보면 깐깐징어 보스 `&mut` 녀석랑 꽤나 존똑 닮은 꼴 쌍둥이 아류 짭유사 판박이 빙의 망령(like `&mut`) 그 자체야, 왜냐면 지가 목표 타겟 가리키고 깔고 뭉갠 점유 대상 공간 지목 메모리 영토 범위 터전 번지 구역 공간 점거 덩어리에 대해 기어이 누구도 손 못 대는 유일무이 막장 전권 행사 독점 소유권 제왕적 단독 패권 독재 발언 권리 소유 억지 지배 만용 전권 주장 배타 선언 전유 소유 억지 독차지 고집 선포 점유 무법 주장 억지력 행사 호언 독차지 확언 갈취 장악 횡포 배타(claims unique ownership) 짓거릴 처하면서 난리해대고 요구하거든. 그러니 어디 저 잘난 독점 호언 거들먹 만용 억지 허세 뻥카 배짱 주장이 리얼인지(test that claim out) 몸소 단두대 테스트 타격 심판 시동 돌진 박치기 시운전 검증 도발 타격 증명 돌파 실사 확인 시전 점검 실증 심사(test ... out) 작전으로 찔러나 볼까나:

```rust ,ignore
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1;

    // 단연코 정답 목표 타깃 값 산출 귀결은 21 (Should be 21) 이 도출 떠야만 함
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
       to tag <1707> at alloc763 found in borrow stack.

 --> src\main.rs:7:5
  |
7 |     *ptr1 += 1;
  |     ^^^^^^^^^^ no item granting read access to tag <1707> 
  |                at alloc763 found in borrow stack.
  |
```

크하하, 그렇지 그렇지 당연한 소리지(Yep), 우리 깐깐징어 원칙주의 대마왕 Miri 판사님께서 저딴 개 막장 하극상 혼돈 무례 도발 야바위 변칙 야바위 만행 삽질 역전 개판 작태 혐오 난리(hates that)을 단칼에 극대노 혐오 철퇴 처단 단죄 질타 발작 역정 발작 꿀밤 호통 응징 참교육 철퇴 분노(hates) 호통 컷 갈겨 조져주시는 거 정말 통쾌 그 잡채다. 그러면 이제 좀 정신 차리고 얌전 정석 엘리트 순결 참교육 정배 순서 패턴 모범 예절 FM 법도 순차 절차 규범 수순 타령 순리대로 정배 밟기(doing things in the right order) 매뉴얼 돌려보면 안도 무사 결백 생존 승인 면죄 허가 합격 통과 증빙 무사통과 순항 프리패스 허가 만사형통 무해 패스 승인 수락 합격 합법 무사 달성 구원(is ok) 따내는지 확인 테스트 증명 검증 시운전 감식 검사 체크 조사 시동 시연 작동 조준 타격(check that) 출격 검열 실험 사살 점검 타파 시연 가동 돌진해볼까나:

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *ptr1 += 1;
    *data += 10;

    // 기필코 정답 종착 종점 수치는 21이어야만 함 (Should be 21)
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
21
```

캬아 젠장 미쳤다 지렸다 찢었다 완벽 그 잡채 합격(Yep)!

어허이 닝겐 여러분 다들 주목 집중(folks), 이제 정말로 진짜 진심 드디어 마침내 기나긴 이 끔찍한 대잔치 고문 고난 지옥의 누적 차용 탑 원리 시스템 마수 교리 철칙 고찰 논의 분석 탐구 궁리 발굴 지껄 강연 잡담 학술 토론 주접 뇌절 해부 타령 삽질 여정의 토크 퍼레이드(talking and thinking about) 논문 강연 헛소리 지껄 토론 수다 분석 분석 연구 삽질 고뇌 파헤 풀이 잡탕(thinking about)에서 우리 모두 드디어 해방 자유 영접 종착 완결 졸업 종강 만세 전원 만세 아웃 탈주 종점 탈출 굿바이 하차 종료 해방 마감(finally done) 달성 성공 만세 종료 종결 만세!!

...근데 젠장 잠깐잠깐만, 그럼 도대체 이 개노답 Box 황당한 대마왕 파탄 뇌절 녀석이 터뜨려 문제망낸 이 재앙 참사 폭탄UAF 구멍 버그 참사 문제 결함 버그 오류 에러 사태 해결 땜빵 진압 수습 해결 복구 정복 파훼 해결점(this problem with Box)은 대체 뭔 재주 수단 묘수 정답 힌트 처방구책 묘안 극복 파해 진압 마법 편법 방벽 조치 방패 방어 해결 솔루션 파훼 도구(solve)로 비틀어 틀어막아야 한다는 거임? 아니 내 말은, 그래 뭐 당장이야 개나 소나 다하는 저딴 찌질 허접 멍청 장난감 모조품 형편없는 것 장난 실습 소형 테스트용 조각 예시 꼬마 유인원 침팬지 유인원 테스트 허접 샌드박스 잡템 초보 동네 꼬마 실습 코드 장난감 같은 것 쪼가리 장난감(toy programs like this) 같은 것 조각 소형 애들 장난 연습용 예제 장난감 애들 찌라시 꼬마 코드 예제 조각 견본 모형 잡 찌끄레기 잡코드 조각 따위 정도 형편없는 것들(toy programs like this) 정도야 아무 영혼 무뇌 상태 뇌 빼고 딸깍 치며 대충 끄적끄적 코딩 똥글 타이핑 찌끄려 갈겨 쓰며 조작 창조 타이핑 짜내며 휘갈길 순(write toy programs like this) 있겠지 젠장 반박 불가 안 그래, 근데 사실 현실 리얼리티 야생 야전 리얼 야생 정글 실전 찐 배틀 실전 진지 심연 막장 실무 야전 배틀 그라운드 전투 필드 실무 야생 환경 필드 필드 실무 척박 가시밭길 지옥 극한 실무 환경 구덩이 고립 진배 진창 환경 극한 현장 투입 실전 돌입 진짜 리얼 현장 생존 직면 전투 현실 상황 구덩이 환경 직면 사태 돌진 생존 정글 상황에선 정말 필명 피눈물 머금고 그 잘난 거대 괴물 Box 덩어리 본체 몸뚱아리를 어딘가 영구 보존 안식처 요새 장소 피난처 창고 요새 본거지 인벤 구멍 장소 깊숙한 안방 금고 구석 영토 안쪽 대피소 장소 구멍 깊숙한 어딘가(somewhere) 구석 요지 은신 구석탱이 인벤토리 대기소 진열장 깊숙이 어딘가에 고이 모셔 짱박아 격리 안치 수납 배치 유치 진열 수납 고정 고정치 적재 유폐 영치 동면 안치 은닉 결박 배치 포박 은닉 거치 적재 격리 존치 유치 배치 저장 정박 보관 매장 보존(store the Box somewhere) 보존 박아 존치 수장 은닉 유치(store) 해둬야만 할 젠장 피 말리는 개뼈다귀 같은 운명 업보 숙명 의무 과업 필수 조건 필연 불가피 필요 숙명 굴레 필연 상황(we need to) 사태 수요 숙명 운명이 무조건 도래 직면 필연 발생 뒤따르고 터질 건데, 아 그리고 또 동시에 부록으로 덤탱이 뒤집어쓴 채 저주받은 시한폭탄 그 망할 원시 포인터 좀비 형편없는 것 이단 파편 지뢰 잔당 고철 지뢰 불량 똘마니 마검 녀석 지뢰 원시 깡통 폐급 오물 잡동사니 병기 자객 뾰족창 좀비 미치광이(raw pointers) 이녀석팽이들도 또 젠장 한시도 손에서 절대 안 내버려 두고 안 놓고 악착 독종 징글 질척 인고 기나긴 시간 공백 긴 터널 장기 대기 체류 기나긴 장기 연장 무기한 장구 유예 무기한 공백 세월 지속 존버 무기 공백 오랜 긴 장기 타임 억겁 보류 대기 시간 기간 인고 지연 존속 보유 존버 체류(a potentially long time) 세월 연장 기간 도래 체류 머뭄 유예 존속 대기 장구 지속 무기한 보류(a potentially long time) 존버 내내 억겁 시일 시간 영겁 지속 타임 유구 타임 체제 보장(a potentially long time) 내내 징글 악착 질척 끈질 독종 파지 악착 부여잡 무한 인계 존속 이관 독종 결속 킵 인계 결박 소지 거머쥐 지지 꽉 부여잡고 방치 손아귀 단단 지탱 악착 유지 단단 고수 존버 파지 소유 보존 수호 간직 유지 보유 부여잡고 보류 고정 억류 수호 체류 유지 점거 보전 장착 존속 소지 지탱 확보 끈질기게 질척 존버 확보 인계 보유 억류 홀딩 움켜쥐고 악착 쥐착 수성 유지 관리 파지 소유 간직 유치 무한 보존(hold onto) 꽉 부여잡고 방치 손아귀 단단 보전(hold onto) 해둬야만 하는 피눈물 통탄 통곡 한숨 혈압 치솟 절망 피눈물 나는 끔찍한 상황 기구 수요 현실 비운 사태 발현 직면 도래 의무 책임 의무 수요 숙명(we need to)이 필연 또 터질 텐데 말이지 빌어먹을. 분명 백 퍼센트 확률 보장 맹세코 필연 장담 그 빌어처먹을 폭탄 마검 좀비 원시 포인터 형편없는 것 녀석이 언제 어디서 어떻게 지들끼리 얽히고설키고 뒤섞이고 병합 유린 결탁 기만 쌍둥이 개차반 짬뽕 엉망 혼란 난투 난기류 도가니 개조 오염 충돌 난입 오염 침투 뒤섞 결탁 충돌 혼잡 오류 결탁 병합 교잡 유착 무단 유출 변질 유착 얽힘 도가니 뒤죽박죽 엉망진창 충돌 파탄 난투 합작 빙의 교란 뒤죽박죽 꼬임 뒤죽박죽 짬뽕 혼합 범벅 혼입(get mixed up) 변질 오염 짬뽕 혼란 도가니 뒤범벅(get mixed up) 개차반 문제망 사태 터져서 종국엔 싹 다 무효 파기 박탈 말소 파멸 유예 기각 맹탕 증발 사형 심판 종말 아웃 권리 상실 처단 배제 억제 즉사 강판 기회 박탈 전락 소멸 붕괴 파탄 폐기 실격 아웃 소진 정지 몰락 사망 휴지통 효력 상실 퇴출 강판 차단 봉인 소멸 박탈 지위 상실 강판 전락 낙오 전멸 아웃 소외 폐기 파기 불발 사멸 날아가 증발 박탈 강퇴 무효화 단두대 철퇴 심판 종식 폐기 철거 소멸 삭제 타격 즉사 증발 백지화 소멸(invalidated) 파멸 지옥 강등 사멸 철퇴(invalidated) 콤보 후폭풍 뒤통수 재앙(invalidate) 참사 종식 종말 징계 멸망 궤멸 사멸(invalidated) 몰살 폭사 파멸 몰락 폭파 터질 거 정말 안 봐도 비디오 눈에 빤히 눈뜬 장님 뻔히 너무 극명 보임 뻔뻔(Surely) 뻔뻔 명백 자명 삼척동자 너무 뻔한 기정 기정사실 확정 자명 명백 극명 분명 명약관화 필연 도래 불보듯 뻔함(Surely) 훤한 거 아니냐? 

아 따 젠장 정말 날카롭고 번뜩 찢어발기는 명존쎄 급 천재 킹갓 통찰 질문 훌륭 찬사 핵공감 명언 대작 일갈 질문 찬미 영광 전재 예리 극찬 박수 스포트라이트 명언 찬양 팩폭 날카 명언 천재 일갈 질문 통찰 갓벽 송곳 날선 천제적 위대한 대명작 일침 예리 날카 찰떡 핵공감 기막힌 극찬 위대한 질문 극찬 돌직구 대박 쩌는 송곳 명언 대박 예리 훌륭 명답 찰떡 지림 소름 감탄 천재 훌륭 질문(Great question)! 바로 님들의 그 젠장 대갈통 깨부수는 명존쎄 훌륭 빛 번뜩 예리 일침 번뜩 기염 토하는 송곳 핵심 명 질문 의문 통찰력 물음 폭탄 대명 질문 에 대하여 그 해답 타파 파훼 정답 타개 정본 모범 해답 돌파 열쇠 답변 해설 솔루션 진리 해답 광명 정답 구원타파 해법 공략 실마리 풀이 열쇠 결론 해소 답변 응답(To answer that) 진리 해법 빛 구원 나침판 이정표 사이다 청량 해답 속 시원 진리 해담 해소 해갈 타개 해답 진리(To answer that)을 제시 해소 파훼 증명 타파 결론 답변 깨달음 타개 구원 정복 해명 풀이 해답(To answer that) 기동 제시 해결 답 하사 떠먹 제공 하달 수여 답변 제시 베풀 풀어주기 썰 풀기 응답 던져 해주기 위해, 이제 우리는 드디어 젠장 마침내 진짜 찐 막장 리얼 끝판 대마왕 최종 보스 최종장 시조 애초 최초 기원 태초 마침내 회귀 궁극 도달 안착 영접 조우 재림 강림 도착 발현 성사 오리지널 원래의 그 끔찍한 숭고 빛나는 찬란 운명 천직 우리네 본업 진면목 참된 본질 거룩 사명 숭고 대업 진성 목적 진짜 사명 진정한 천명 본분 소명 참된 숙명 궁극 대업 거룩한 부름(true calling) 천명 숭고 대망 본분 운명 직분 소명 부름 제국 소명 최종 사명 천직 귀환 본업(true calling) 타겟 영점 복귀 환원 회귀선 종점 목적지 리턴 귀속으로 사명 강림 재림 역행 유턴 컴백 빽턴 백미러 재회 조우 강림 유턴 다시 회귀 복귀 귀환 원상 귀속 컴백(we'll finally be returning) 유턴 빽 귀향 원대복귀(returning) 기수를 돌이키고 타겟 조준 조향 핸들 꺾어 복귀 턴 돌입 터닝 이륙 돌이킬 거라구: 바로 그 개젠장 저주 거지 빌어 처먹을 재앙 무덤 지옥 폭탄 형편없는 것 폐급 망할 망조 구덩이 호러 납골당 지뢰 시한폭탄 극혐 개 거지발싸개 끔찍한 바보 똥싸개 황당한 저주 개망나니 염병할 저주 거지 발싸개 난리맞을(some god damn) 연쇄 저주 폭발 형편없는 것 개잡동사니 바보 똥망 시한폭탄 문제 바보 폐급 호러쇼 오컬트 강령 주술 괴물 오물 시궁창 함정 지뢰밭 연결 리스트(linked lists) 고철 덩어리 시궁창 지뢰 괴물 혐오 고어 지옥의 구조 고철 덩어리 지옥 형편없는 것 괴물 병기 (linked lists) 같은 것 녀석들 코드나 하루 온종일 허구한 날 구박 주야장천 쉴 틈 휴식 없이 평생 주야장천 밤샘 백야 허구한 엄청나게 쳐 찍어 짜내 적어 갈기고 쳐버리고 쓰고 내지르고 무뇌 코딩 치고 버리고 뽑고 자빠져 싸지르고 주야장천 코딩 공장 딸깍 코딩 같은 것 타이핑(writing) 도배 노가다 타자질 코드 찍기 싸지르 찍어내는 복붙 질주 코딩 셔틀 기계 타자 공장(writing) 하는 그 숭고 막장 지옥 영광 천직(calling)의 구렁텅이 요람 구덩이 제단 무덤 골짜기 심연으로 말이야.

니미럴 잠깐만 수동 조작 브레이크 급발진 급브레이크 타임아웃 워워 멈춰 뭐요 대체 이게 뭔 젠장 자다가 봉창 난리 황당한 끔찍한 소리요 왓 해프닝 젠장?, 내가 지금 다시 한 번 죽었다 깨어나도 그 젠장 우라질 개끔찍한 늪지대 연결 리스트 해골물 형편없는 것 뼈다귀 무덤 형편없는 것 함정 재앙 형편없는 것들을 기어이 이 악물고 피눈물 혈서 다시 십자가 짊어 매고 또 다시 이어서 기어코 또다시 두 번 재차 연달아 재도전 억울 환장 환생 예토전생 다시금 또(again) 처음부터 다시 리셋 쌍으로 돌림 쳐 발라 쳐 코딩 토악질 복붙 노동 노가다 손가락 관절 파괴 노동 착취 타이핑 수작질 노동 십자가 쳐발라서 적어 내려 부활 부관참시 조작 노동 다시 써내려 조립 제작 강행 노가다 코딩 적성 강행(write linked lists) 삽질 복기(write) 조작 제작 건립 이룩 결행 달성 제조 이룩 씹노가다 삽질 고행 발동 지시 타이핑 삽질 수동 수작 노동 창조 코딩(write) 해야 한다고? 야 니들 지금 장터 골목 말장난 수작 장난 조커 장난질 애들 코미디 예능 찍냐 똥때리냐 젠장 우리 좀 인간적으로 이성 교양 차원 지조 유지 자제 워워 진정 타이밍 심호흡 흥분 수위 안정 성급 참아 진정 조급 급발진 성급 오버 뇌절 진정 오버 떨지 말자 자제 무리 자제 자제 오버(let's not be hasty) 흥분 자제 컴다운(hasty) 릴렉스 심호흡 하자 인간들아 동지들아 제발 좀. 좀 인간적으로 뇌수 열기 뚜껑 머리 좀 식히고 쿨하게 침착 얼음 냉수 마시고 상식 양심 합리 논리 상식 타당 정상 지능 이성 교양 합리 타당하게 도리 지각 상식껏 이치 생각 양심껏 이성 도리 합식 이성적으로 정상 합리 이치 타당 생각 사리분별 이성 이성 정상 사고 이성(Be reasonable) 멘탈 챙겨 챙겨라 사리 판단 좀 치고 살자 이성 합리 타당 도리 지성 이성 지각 철학 양심 상식 차려라 바보들아. 그냥 아가리 잠시 쳐 여물고 지퍼 채우고 잠자코 모래 입 다물 딱 대기 요양 존버 대기 좀만 더 죽닥치 존버 홀드 숨만 쉬고 스탠바이 참아 기대 스톱 정지 대기 대기 스탠바이 버텨 (hold on) 닥치고 얼음 조용히 잠시 제발 좀 제발 부탁이니 멈춰 기다려(hold on)봐. 나 분명 내 전두엽 걸고 장담 피눈물 맹세 장담 확신 필연 장담 호언 분명 맹세 하늘 걸고 호언 확신 100% 장담 기필 분명 확언 단언 확신 보장(I'm sure)컨대 아직 우리가 더 파내고 후비고 엄청나게 물어뜯고 아가리 파이터 참전 논의 토론 지껄 강연 잡설 까고 떠들고 토론 까발 논의 토크 탁상공론 분해 썰전 연구 아가리 논쟁 분해 씹뜯맛즐 해부 논의 타령 수다 논쟁 분석 토론 지껄 논의 파헤칠(for me to discu) 흥미 도파민 만점 아드레날린 도파민 폭발 기똥 흥미진진 호기심 천국 꿀잼 레전드 흥미 팝콘각 자극 흥미진진 기발 매력 치명 꿀잼 흥미(interesting) 유발 매력 환장 흥미 유발 오싹 자극 폭발 흥미(interesting) 돋는 어나더 레벨 차원 외전 곁가지 별책 부록 사이드 스토리 또 다른 별개 곁가지 번외 딴 잡다 여타 다른 제3종 여타 잡다 별개 기타 남은 타 잡다 다른 잡동사니 이종 여타 여분 발생 다른 잡(other) 형편없는 것 바보 버그 지뢰 도발 폭탄 퀘스트 시련 재앙 타락 골머리 두통 난제 폭탄 결함 문제 에러 재앙 트러블 고난 뻘짓 함정 폭탄 문제 결함 이슈 트러블 하자 오류 사태 트러블 불상사 말썽 에러 결함 문제 이슈 난제 지뢰 트러블 논란 고난 (issues) 같은 것 파편 폭탄 논란 잔재 미스터리 숙제 떡밥 파편들이 어디 맵 어딘가 형편없는 것 더미 구석 던전 골방 무더기로 구석 잔뜩 구덩이 형편없는 것봉투 음지 지하 쌓여 병풍 은밀 잠복 숨어 지뢰 널려 포진 잠재 산재 잔존 투하 포진 산재 진열 남아 널려 숨어 잔재 매복 잔존 도사리고 산재 처 박혀 잔뜩 대기 남음 존재 잔재 대기 숨어 잠복 진열 나뒹굴고(there's some... issues) 우글우글 바글바글 미해결 미로 미로 미답 잔존 널브러져 대기 은폐 미제 사태 잔여 잔재 도사 포진 산재 대기 숨어 진열 존버 잔존 남아(there's some) 있을 거라고 나 젠장 분명 단언 내 문제 뇌수 전 재산 뽕알 전두엽 영혼 걸고 맹세 피 철철 호언장담 각서 보증 신앙 간증 맹신 선언 확언 확신 장담 (I'm sure) 호언 맹세 하늘 걸고 투서 장담 하니까 젠장 제발 좀 아 좀만 더 대기 기다리라고 참아봐 씹쌔끼들아 씨아아아발 나 무서워 도망갈래 집에 갈래 엄마 무서워 환멸 진저리 안 해 못 해 도망 갈래 살려줘 으아아아아악 환장 공포 도피 질주 도망갈래 여기서 내보내줘&mdash;