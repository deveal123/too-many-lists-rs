# 상용 수준의 안전하지 않은 이중 연결 덱 (A Production-Quality Unsafe Doubly-Linked Deque)

드디어 여기까지 왔습니다. 제 최대의 적수: **[std::collections::LinkedList][linked-list], 이중 연결 덱(Doubly-Linked Deque)** 말입니다.

제가 파괴하려고 그토록 시도했지만 결국 실패했던 바로 그 녀석이죠.

우리의 이야기는 2014년이 저물어가고, Rust의 첫 안정화 릴리스인 Rust 1.0 출시가 코앞으로 다가오던 무렵으로 거슬러 올라갑니다. 당시 저는 `std::collections`, 혹은 우리가 그 시절 애정 어린 별명으로 부르던 'libcollections'의 관리를 도맡는 역할을 맡게 되었습니다.

libcollections는 수년 동안 온갖 사람들이 아무렇게나 던져놓은 '귀여운 아이디어들(Cute Ideas)'이나 '대충 어디엔간 유용할 것 같은 것들(Vaguely Useful Things)'이 뒹구는 쓰레기장이었습니다. Rust가 막 날갯짓을 시작하던 실험적인 언어 시절엔 이것도 꽤 괜찮았습니다만, 제 새끼들이 둥지를 떠나 안정화(stabilized) 라벨을 달려면 스스로의 가치를 증명해 보여야만 했습니다.

그때까지 저는 그 구조체들을 모두 감싸 안고 보살펴 왔습니다만, 이제는 그들의 부족함에 대한 심판을 받을 시간이 다가온 것입니다.

저는 기반암에 발톱을 깊숙이 박아 넣고, 제가 거느린 가장 미련하고 어리석은 자식들을 위해 비석을 깎아내리기 시작했습니다. 그리고 만천하가 볼 수 있도록 마을 광장 한복판에 그 소름 끼치는 기념비를 세웠죠:

**[Kill TreeMap, TreeSet, TrieMap, TrieSet, LruCache and EnumSet(TreeMap, TreeSet, TrieMap, TrieSet, LruCache, 그리고 EnumSet을 사살함)](https://github.com/rust-lang/rust/pull/19955)**

제 말이 곧 법이었기에, 그들의 운명은 영원히 봉인되었습니다. 다른 컬렉션(collections)들은 저의 그 잔혹함에 경악을 금치 못했지만, 그들 역시 아직 어미의 분노로부터 완전히 안전한 것은 아니었습니다. 곧이어 저는 두 개의 비석을 더 들고 돌아왔으니까요:

**[Deprecate BitSet and BitVec(BitSet과 BitVec을 지원 중단함)](https://github.com/rust-lang/rust/pull/26034)**

저 'Bit' 쌍둥이 녀석들은 먼저 쓰러져간 전우들보다는 조금 더 교활했지만, 제 손아귀를 빠져나갈 만큼 강하진 못했습니다. 대부분의 사람들은 이걸로 제 학살극이 끝났을 거라 여겼겠지만, 저는 이내 하나를 더 앗아갔습니다:

**[Deprecate VecMap(VecMap을 지원 중단함)](https://github.com/rust-lang/rust/pull/26734)**

VecMap은 은신을 통해 어떻게든 살아남으려 발버둥 쳤습니다 &mdash; 그 녀석은 워낙 작고 무해했으니까요! 하지만 제가 그리는 미래의 libcollections 비전 앞에서 그 정도로는 어림도 없었습니다.

저는 대지를 굽어살피며 살아남은 생존자들을 확인했습니다:

* Vec과 VecDeque - 강건하고 단순한, 컴퓨팅의 심장.
* HashMap과 HashSet - 강력하고 현명한, 컴퓨팅의 두뇌.
* BTreeMap과 BTreeSet - 좀 어색하지만 필수 불가결한, 컴퓨팅의 간(liver).
* BinaryHeap - 교묘하고 날렵한, 컴퓨팅의 발목.

저는 만족스러운 표정으로 고개를 끄덕였습니다. 단순하고 효과적입니다. 이제 내 일은 다 끝났&mdash;

안 돼, [DList](https://github.com/rust-lang/rust/blob/0a84308ebaaafb8fd89b2fd7c235198e3ec21384/src/libcollections/dlist.rs), 이럴 순 없어! 넌 분명 그 끔찍했던 가비지 컬렉션 폭발 사고 때 뒈졌잖아! 실수인 척 위장했지만 절대 고의로 일으킨 게 분명했던 그 대형 사고에서 말이야!

놈들은 자신들의 죽음을 위장한 채 새 이름으로 신분 세탁을 감행했지만, 본질은 여전히 놈들이었습니다: 바로 `LinkedList`, 컴퓨팅 세계의 그 음흉하고 믿을 수 없는 음모꾼 새끼 말이죠.

저는 마주치는 모든 이들에게 놈들의 악행을 떠벌리고 다녔지만, 사람들의 마음은 끄떡도 하지 않았습니다. `LinkedList` 이 은장도 같은 세치혀를 놀리는 악마 새끼가, 주변의 모든 이들에게 자신이 마치 이 바닥 컴퓨팅 세계의 근본적이고 자연스러운(fundamental and natural datastructure) 무언가인 양 뇌내망상 세뇌를 시켜놨기 때문이었습니다. 이 새끼는 심지어 C++마저 홀려서 자기가 [*바로 그* 리스트](https://en.cppreference.com/w/cpp/container/list)의 본가라고 믿게 만들었으니까요!

"어떻게 표준 라이브러리에 *LinkedList* 하나 안 박아두고 근본을 챙길 수 있습니까?"

아주 쉽게! 껌이지!

"이건 구현하기 까다로운(non-trivial) 안전하지 않은(unsafe) 코드 덩어리니까 다칠까 봐 표준 라이브러리에서 안전하게 관리하는 게 이치에 맞다고요!"

지랄, GPU 드라이버랑 비디오 코덱도 구현하기 까다롭지만 표준에 안 넣잖아, libcollections는 미니멀리즘이 모토란 말이다!

하지만 슬프게도, 제가 그놈들의 핏줄 찌꺼기 친척들을 도륙하는 데 정신이 팔려있는 동안 `LinkedList`는 벌써 너무 많은 동맹 규합 프락치들을 양성하고 힘을 무지막지하게 키워버린 상태였습니다.

저는 제 비밀 실험실로 도망쳐 들어가, 그 더러운 새끼에게 대적하고 처참히 파괴해버릴 모종의 [사악한 클론(evil clone)](https://github.com/contain-rs/linked-list)이나 [강화 사이보그 리플리컨트(enhanced cyborg replicant)](https://github.com/contain-rs/blist) 따위를 만들어내기 위해 밤낮으로 머리를 싸맸습니다만, 내 연구가 "너무 살인적이고 악의적"이라는 둥의 궤변을 늘어놓으며 지원금 줄을 끊어버리는 바람에 프로젝트는 무기한 엎어지고 말았습니다.

`LinkedList`의 승리였습니다. 저는 결국 참패하여 유배지로 쫓겨나고 말았죠.

하지만 이제 여러분이 이 자리에 서 있습니다. 이곳까지 도달하셨군요. 이쯤 되면 당연히 저 오만방자한 `LinkedList` 새끼의 타락과 방종의 수위 심해를 똑똑히 이해하셨을 겁니다! 오십시오, 그 새끼를 이 세상에서 완전히 끝장 복수 파멸 내버리는 데 필요한 모든 것 &mdash; 즉, 상용 프로그램에 당장 투입해도 손색없는(production-quality) 최고봉 안전하지 않은(unsafe) 이중 연결 덱(Doubly-Linked Deque)을 손수 구현해 내는 데 필요한 모든 지식을 여러분께 남김없이 전수해 드리겠습니다.

어느 정도로 상용 수준이냐고요? 글쎄요, 옛날 옛적 제가 썼던 고대 유물 Rust 1.0 시절의 그 linked-list 크레이트를 밑바닥부터 깡그리 다시 다 엎고 새로 쓸 겁니다. std에 병신같이 박혀있는 그 쓰레기보다 객관적으로 백배는 더 우월한 바로 그놈 말입니다. 2015년 안정화 Rust 환경에서 이미 커서(Cursors) 기능까지 빵빵하게 지원했던 바로 그 전설의 레전드! 무려 2022년의 자랑스러운(?) stdlib조차도 아직 꼬무룩 구경도 못 해본 그 환상의 최첨단 기능을 떡칠해서 말이죠!




[linked-list]: https://github.com/rust-lang/rust/blob/master/library/alloc/src/collections/linked_list.rs
