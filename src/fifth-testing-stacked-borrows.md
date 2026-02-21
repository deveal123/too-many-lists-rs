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

오호 통재라 슬프도다(Alas), 비록 우리가 컴파일러의 경고를 꼼수 캐스팅으로 무사히 피했을지언정, 고작 그따위 얄팍한 캐스팅 하나 발랐다고 이 조작이 합법화될 리는 없습니다. 우리가 공유 참조자를 만드는 바로 그 순간, 우리는 절대로 그 값을 변경하지 않겠다고 굳게 약속한 것(promising)과 다르지 않기 때문입니다.

이 약속은 매우 중요합니다! 왜냐하면 공유 참조자가 차용 스택 최상단에서 수명을 다해 제거(popped off)된 직후, 그 아래에 있던 가변 원시 포인터들이 다시 활성화될 때 한 가지 굳은 확신을 가질 수 있기 때문입니다. 바로 자신들이 비활성화되어 있던 기간 동안 오직 '읽기' 접근만 허용되었기 때문에, 원래 메모리 값이 전혀 변하지 않았다고(assume the memory hasn't changed) 믿을 수 있다는 점입니다. 누군가가 무수히 메모리를 읽어 갔을 순 있지만(writes had to be committed), 그 누구도 감히 그 메모리의 값을 수정하지는 못했을(weren't able to modify it) 테니까요! 고로 가변 포인터들은, 자신들이 마지막으로 썼던 값(last value they wrote)이 여전히 그 자리에 그대로 남아있다고 확신(can assume ... is still there)할 수 있는 겁니다!

**일단 한 번 "공유 참조자"가 차용 탑(borrow-stack)에 올라가는 순간, 그 위로 쌓이는 모든 포인터들은 오직 읽기 권한(only has read permissions)만 갖도록 제한됩니다.**

하지만 이런 꼼수(do this) 정도는 눈감아 주더군요:

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

이 코드를 보면, 가변 원시 포인터(mutable raw pointer)를 생성하기는 하지만 실제로 사용할 때는 단순히 값을 읽어오기만 합니다(only actually read from it!). 이렇게 읽는 데만 쓴다면 컴파일러의 징계를 피하고 무사히 스택에 등록(still "fine" to create)되는 것을 볼 수 있습니다!

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

혹시 모르니 확인 차원에서, 우리의 공유 참조자가 평소처럼(like normal) 제대로 스택에서 제거(popped)되는지도 한 번 짚고 넘어갑시다:

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

어라(Hey), 이번엔 특정 태그(some specific tag)를 언급하는 대신, 'SharedReadOnly'라는 낯선 에러 메시지가 나왔군요. 꽤 합리적(makes sense)인 조치이긴 합니다. 왜냐하면 일단 공유 참조자가 하나라도 생겨나게 되면(once there's *any* shared references), 그 상황에서는 더 이상 누가 누구인지 구분할 필요 없이 전부 '읽기 전용 공동체(just a big SharedReadOnly soup)'로 취급 합병되기 때문입니다. 굳이 그 안에서 개별적으로 식별(distinguish any of them)할 필요가 전혀 없어지는(no need to) 것이죠!





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

애초에 왜 이 길고 긴 삼천포로 빠지게 되었는지 기억나시나요? 안 나신다고요? 이상하네요.

우리가 이 이야기를 시작한 건 Box와 원시 포인터를 섞어 썼기 때문이었습니다. Box는 대상을 독점적으로 소유(unique ownership)한다고 주장하기 때문에 여러모로 `&mut`와 비슷합니다. 정말 그런지 확인해 볼까요:

```rust ,ignore
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1;

    // 21이 나와야 함 (Should be 21)
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

네, Miri는 이걸 아주 싫어합니다. 그럼 올바른 순서대로 접근하면 괜찮은지 확인해 보죠:

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *ptr1 += 1;
    *data += 10;

    // 21이 나와야 함 (Should be 21)
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
21
```

옙, 통과네요!

자, 여러분, 이제 길고 길었던 차용 스택(borrow-stack)에 대한 논의가 드디어 끝난 걸까요? 네!!

...잠깐만요. 그럼 이 Box 문제를 도대체 어떻게 해결하란 말인가요? 이런 장난감 같은 예제 프로그램이야 대충 순서를 맞춰 쓸 수 있겠지만, 실전에서는 Box를 어딘가에 저장해 두고 원시 포인터들을 꽤 오랜 시간 동안 쥐고 있어야 할 텐데요. 분명 이 둘의 접근이 뒤섞여서 포인터가 무효화(invalidate)되는 일이 터지지 않겠어요?

아주 좋은 질문입니다! 그 해답을 찾기 위해, 우리는 드디어 우리의 진정한 사명으로 돌아가려 합니다: 바로 연결 리스트 작성하기 말입니다.

잠깐만 워워 진정해 아니 내가 무슨 소릴 듣고 있던 건지 완전히 까먹었는데 제발 다시 연결 리스트를 짜게 만들진 마— 우리 너무 성급하게 굴지 맙시다. 이성적으로 생각해보자구요. 잠깐만요. 분명 제가 아직 설명해야 할 다른 흥미로운 문제들이 더 남아있을 거라고 확신합니다, 제발 안돼—