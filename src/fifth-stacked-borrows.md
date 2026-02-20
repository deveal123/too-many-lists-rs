# 누적 차용 이해하기 (Attempting To Understand Stacked Borrows)

이전 섹션에서 우리는 Miri(미리)를 통해 우리의 unsafe 단방향 연결 큐(singly-linked queue)를 실행하려 시도했고, Miri는 우리가 *누적 차용(stacked borrows)* 이라는 규칙을 위반했다며 관련 공식 문서를 던져주었습니다.

보통이라면 제가 그 문서들을 천천히 안내하며 같이 투어(tour)를 돌았겠죠, 하지만 이번 문서의 타깃 주요 독자층(target audience)은 평범한 서민 프로그래머가 아닙니다. 저건 사실상 Rust 언어 생태계 기저의 의미론 규칙 구조(semantics of Rust) 파운데이션을 밑바닥부터 파고 뜯어고쳐 설계하는 학자들과 천재 컴파일러 엔지니어 개발자 집단을 위한 전공 학술 원론에 가깝습니다.

그래서 저는 여기서 "누적 차용(stacked borrows)"의 핵심적인 거시 세계 관념(high level *idea*) 정도만 슬쩍 가볍게 읊어드린 뒤, 여러분이 그 무서운 규율을 무사히 준수해 나갈 수 있는 심플한 대응 전략 전술(simple strategy) 몇 가지만 던져주고 넘어가려 합니다.

> **해설자:** '누적 차용(Stacked borrows)'이란 개념 자체는 Rust 내에서 여전히 의미론적 기초 모델 체계(semantic model)로서 "실험 단계(experimental)"에 불과합니다. 따라서 당장 오늘 이 규칙을 위반했다고 해서 당신의 프로그램이 100% 무조건 망한 "틀린(wrong)" 코드라는 단호한 판정을 내리긴 어렵습니다. 하지만 만약 당신이 진짜 컴파일러 파운데이션 코어 개발자(literally work on the compiler)급 되는 비범한 인재가 아니라면, 웬만하면 평화롭게 얌전히 그냥 miri가 불평하고 징징댈 때 군말 없이 무조건 수긍하고 고치십쇼(fix your program). 미정의 동작(Undefined Behaviour) 귀신 앞에서는 똥고집 부리며 후회하는(sorry) 것보단 걍 바짝 엎드려 목숨 부지하며 안전빵(safe) 치는 게 언제나 최선의 상책입니다.



# 원동력: 포인터 앨리어싱 (The Motivation: Pointer Aliasing)

우리가 구체적으로 무슨 어불성설 *무슨 짓(what* rules)* 을 저질러 위반 사태가 났는지 깊이 따져보기 전에, 애당초 대체 왜 이런 피곤한 규제(rules exist)가 기어코 세상에 태어난 것인지 그 근원 배경(*why*) 원인 사유부터 파악하는 게 정신 건강에 큰 도움이 될 겁니다. 여러 복합 복잡 다양한 동기부여 명분(motivating problems) 트리거 발생 문제점들이 얽혀 있긴 하지만, 제 개인적으론 가장 핵심 원흉 원죄는 단연코 저 망할 *포인터 앨리어싱(pointer aliasing)* 이라고 단언합니다.

두 포인터가 서로 *앨리어스(alias)* 되었다는 말은 두 녀석이 가리키는 포인터 위치 주소 영토 메모리 파편 덩어리 영역 공간이 완전히 서로 교차 겹친다는(overlap) 뜻입니다. 마치 누군가가 "가명 신분 세탁(goes by an alias)"을 하여 한 명의 동일 인물을 두 개의 다른 이름 간판으로 동시에 지칭 호출(referred to by two different names)해 부를 수 있듯, 동일한 메모리 구역 하나가 겹친 채 두 개의 각기 다른 포인터 사령관에 의해 동시 원격 호출 지칭 명령 대상(referred to by two different pointers)으로 지정되는 기이한 현상입니다. 당연히 이런 구조는 파국 문제(problems)를 불러오기 딱 좋죠.

컴파일러는 포인터들의 앨리어스 동기화 매핑 겹침 구조 설계도 상황 인포메이션 정보(information about pointer aliasing)를 십분 활용해 메모리 접근 쾌속 최적화 고속 튜닝 처리 로직(optimize accesses)을 수립 채택 발동합니다. 그런데 만약 컴파일러가 쥔 그 사전 족보 정보가 근본부터 썩은 *오류(wrong)* 허상이었다면, 프로그램은 개판으로 오폭 빌드 오역 컴파일(miscompiled)되어 미친 듯 임의의 쓰레기 난동 짓거리(do random garbage)를 자행하게 될 겁니다.

> **해설자:** 아주 현실적인 실무 관점(Practically speaking)으로 비틀어 얘기하자면, 앨리어싱이라는 개념은 그 포인터 자체의 본질보다는 정작 어떻게 그 영토에 진입 침투 점령 메모리에 접근하느냐(memory accesses) 방식에 훨씬 더 치명적 비중의 초점이 맞춰져 있으며, 그중에서도 특정 접근 노선 조작 중 무언가 적어도 한 곳이 가변 수정 조작(mutating) 기동을 벌이려 할 때 진짜로 미친 파국 극성 영향을 끼칩니다(only really matters). 포인터라는 감투가 유독 눈에 띄게 강조 강조 부각 지목(emphasized) 받는 이유는 그저 규칙 통제 규율이란 사슬을 걸어 붙잡아 매달아 통치하기에 가장 편리 찰떡 핑곗거리 대상 명분 꼬리표(convenient thing) 명함이기 때문입니다.

포인터 앨리어스 지형 요도 지침 정보라는 게 대체 왜 그렇게 세상 소중하고 끔찍 막대(important) 한 건지 깊이 음미하기 위해, 지금부터 *작고 화난 사나이의 우화 (The Parable of the Tiny Angry Man)* 이야기를 들려드리겠습니다.

----

어느 날 한적한 오후, 미힐(Michiel)은 자신의 고상한 서재 책장(bookshelf) 구석을 이리저리 뒤적거리다 도무지 시킨 적도 본 적도 없는 생경한 책 한 권(didn't remember)을 문득 발견했습니다. 미힐은 조심스레 서가 한 켄 기둥 사이에서 그 수상한 책을 끄집어 올려 빼들곤(pulled it) 유심히 겉표지(cover) 제목을 살폈습니다.

"아, 이런 그렇군! 내가 진짜 감명 깊게 전부 독파해 정복 완독해 버린 나의 소중 찬란한 고전문학 마스터피스 도서 수집 소장본 *전쟁과 평화(War and Peace)* 였잖아. 난 정말이지 그 책 속에서 모든 분쟁이 사라진 그 순백 완전한 엄청난 대량의 평화(Peace) 구절 장막 대목들이 진심으로 너무 좋았어."

그때 냅다 현관문 지축이 흔들리게 쾅쾅 노크 소리 부름 호출(knock at the door)이 요란하게 울려 퍼졌습니다. 미힐은 가볍게 콧방귀를 뀌곤 무심한 듯 시크하게 그 책을 대충 원래 꽂혀 있던 12번 책장 구멍 선반(shelf) 사이에 다시 스윽 쑤셔 넣곤 슬그머니 발길을 돌려 현관 문턱 자물쇠를 돌려 따 열어 젖혔습니다(opened the door) -- 그곳 앞엔 다름 아닌, 그토록 철천지원수이자 평생의 숙적, 웬수 덩어리인 <b>햄슬로(Hamslaw)</b> 가 바락 아귀 두 눈을 치켜뜬 채 무장 대기로 떡하니 버티고 섰었죠. 우리의 빌런 햄슬로 양은 다짜고짜 미힐의 가소로울 만치 참담 형편없이 후진 초라 빈약 코드 골프 축소 코딩 코더 짜내기 조립 타건 솜씨력 타작 한계를 잔혹히 후벼 파 비수 벼린 치명 조롱 악담(devastating remark about Michiel's clearly inferior codegolfing skills)을 시전 날려 방아쇠를 댕기기 위해 심호흡 장전을 거친 완벽 세팅 상태였습니다만, 정작 우리의 수비수 미힐은 오히려 이 위기를 승기로 뒤엎을 번뜩이는 선빵 회피 기회 통로(sensed an opening) 빈틈 틈새 활로를 포착 캐치해 냅니다:

"어이, 이 비루한 패배자 햄슬로 녀석아, 너 행여 살면서 단 한 번이라도 저 위대하신 고전 대명작 대작 도서 *전쟁과 평화(War and Peace)* 따윌 곁눈질로 제대로 정독 구경 읽어나 본 적(ever read)은 있으시냐?"

"푸흡, 야 이 웃기는 소리 뻘소리 지껄이는 얼간아(Pfft). 이 세상천지에 제정신 맑고 제정신 박힌 놈 치고 그딴 지루한 낡은 고물 수면제 소설책 *전쟁과 평화* 따위를 *진짜로 리얼 팩트 각 잡고 완독 클리어(actually* read)* 해낸 수재 인간 새끼가 존재하긴 하냐?"

"그럼, 내가 당장 바로 여깄잖아. 저길 보시지, 야 이 코앞에 저기 내 개인 도서 컬렉션 책장에 아주 늠름 당당히 떡하니 모셔 비치 전시 박제 방치 꽂혀 보존 안치(right there in my bookcase) 돼 있는 웅장 위엄 자태 안 보이냐? 저게 있다는 건 *아주 너무나도 지극히 단연코 명백 뻔하게 팩트 입증(*obviously*)* 내가 완벽 통달 소화 전부 일독 다 마스터 완독 독파(I've read it) 했다는 반박 불가 불가의 유일 진리 아니겠어?"

햄슬로는 그 입에서 터져 나온 말문이 도무지 말문이 막혀 기가 단단히 차버리고 뒤집어져 경악을 금치 도무지 현실을 믿을 수 체념 납득 인정할 용납 방도가 없었습니다(couldn't believe it). 항상 언제나 잘난 체하며 거드럭 우쭐대고 건들 거만 오만한 자만 빙그레 미소 여유만만 조소 전용 가면(smug demeanor)을 유지 견지하던 그녀의 평소 고상한 전형 안면 가면 표정(face) 구속은 그 즉시 박살 붕괴 해체 용접 용해되어, 이내 곧 무시무시 철가면 같은 강철 분노 야차 일그러짐 독기 광기 분노 살기(iron mask of rage)와 서슬 퍼런 투지 살육 집념 일념 굳은 결의(determination)로 싹 탈바꿈 전변 뒤틀려 덧씌워졌습니다. 햄슬로는 눈 앞의 미힐을 야만스레 냅다 풀파워 완력으로 벽 밀쳐 제쳐 우겨버리고(pushed Michiel aside) 파워 무브 경보 경주 불도저 진격 직진 걸음(power-walked)으로 그놈의 도서 책장 진열대 제단 앞 코앞까지 단숨에 강습 우직 돌진 냅다 짓밟아 박차고 날아가 강타 도착해선, 그 고서 무덤 수면 안식처 결계 사이 틈새를 찢어 거세게 무식 우악스럽게 책덩어리를 후벼 파 적출 강탈 뽑아(cleaving the tome) 채집 채포 분쇄 단절 적출 무뽑듯 뽑는 끄집 박수 낚아 박동 쥐어박 발키리 1천 마리의 떼창 미친 폭주 군단(fury of a thousand Valkyries)과 맞먹는 미친 엄청난 박력 폭동 아드레날린 원력 무자비 힘 압력 완력 무력 위력 풀스윙 갈기고 뽑아 거칠게 꺼내 쟁취 탈환 건져 들었습니다. 그리고는 곧바로 그 낡고 빛바랜 마도서 고대 기록 텍스트 성전(ancient text) 결계 실체 장정도서를 자신의 두 손 위 손아귀 파지 그립 위에서 냅다 거칠게 팩 뒤집어 엎치락 확인(turned the ancient text over) 앞면 진실 공개 조준 박대 돌려 세워 판독을 벼르고 들이밀더니, 그녀가 그 두 눈동자로 앞면 정면 타이틀 겉표지 간판(saw the cover) 문구를 육안으로 확인 목격 식별 인지 도달 판독 스캔한 바로 그 시점 조우 직면 접촉 발화 그 즉시 동시 찰나 0초 순간 눈동자가 번쩍이더니 온몸 전체를 다리가 후들 사시나무 주체 통제 제어 잃고 지진 흔들 달달 와들 떨리게 발작 진저리(began to shake) 요동치며 떨기 부르르 강진 몸살 전율 떨기 시작했습니다.

미힐은 이 찬란 영광 대승리 절호 우위 타일 순간을 놓칠세라 세상 누구도 감히 견줄 상상 비교 넘볼 쫓을 가늠 필적 맞상대 대적 불가의 압도 무쌍 천상천하 지상초월급 우주 최고 오만방자 극치 거만 콧대 위상 초격차 월반 지존 무극 오만한 초압도적 비교불가 유아독존 급 자뻑 천추 천재성 우월 진리 증명 천명 뽐내(boast of their clearly unparalleled brilliance) 공기 나발 장황 허세 장광설을 일장 연설 배출 발사 만반 태세 충전 시전 투척 장전 준비 세팅(prepared to) 하려던 찰나 시점 바로 그때 갑자기, 눈앞의 빌런 햄슬로의 아가리에서 미친 듯 갑분 파열 폭발 튀어나 도출 분출 세어 작렬하는 갑작 돌연 뜬금없는 귀신 광소 대소 괴기 호탕 정신 줄 놓은 미친듯한 미소 파안대소 꺄르륵 웃음소리 폭탄 광명 폭소격발 파동(interrupted by the sudden laughter of Hamslaw) 융단에 그만 말문 일격 입김 제지 가로막혀 뇌정지 방해 저지 중단 당첨 잘려나가 절단 차단 묵살 차압 입막음 당해버리고 맙니다.

"푸하하하하 이게 전쟁과 평화라고 씨발, 이 멍청한 치매 머저리야 아하하 풉ㅋㅋㅋ 여긴 백날 쳐다봐라 그딴 이름 따윈 없고, 저기 시뻘겋게 박힌 타이틀 간판 이름 석 자 문구 정체가 아주 골때리게도 전쟁과 **발바닥 냄새 썩는 족발(War and *Feet*)** 이라 아주 크고 당당하게 개박혀 있잖아 ㅋㅋㅋ!"

결국 햄슬로의 안면 양쪽 협곡 따뜻한 두 뺨 볼기 눈망울 계곡 능선 줄기 고랑 틈새를 거쳐 주르르 흐느껴 내려 흘러 적시어 참지 터져 세어 구르는 미친 듯 환희 고비 흘러내리는 수분 액체 기쁨 희열 감격 통쾌 대 폭주 눈물방울 눈물꽃 홍수(Tears were rolling down Hamslaw's face) 바다. 지금 딱 이 순간 찰나 분초 지금 이 시간 위치 좌표 순간 단언컨대 장담코 확실 보장 장담 확신하건대, 이건 단언 그녀가 이 지구별 땅바닥 지상 최후 밟고 여태 살아 숨 쉰 태어난 이래 평생 필생 전생 통틀어 가장 존나 최고의 빛나는 정점 극락 리즈 하이라이트 최고 치 대만족 지존 카타르시스 도파민 영광 전성 시대(greatest moment of her life) 최고의 인생컷 순간임이 진정 확실 명백 사실 자명 분명 결론 타당 합당 당연 사실 팩트 확연 극치 도래 발동 성사 작렬함이 너무나 당연지사 빼박 팩트 분명 자명 확정 완벽 찬란 입증되었습니다.

"아- 아아 아니이 무,무 무슨 미 미친 헛 미 말도 안 이게 대 무슨 헛 말도 안 이 이럴 개소 이게 무 무슨 지 좆 이게 좆 이게 무 이게 말이 이게 뭐 지 이 어 아니 이 그럴 그럴리 어 아니지 아 시 시발 시 잠깐 아니! 이 내가 내가 분명 방금 내 이 내 두 내 내 생 두 저 저 저 눈으로 직접 빤히 이 방금 내가 내 눈으로 확인 빤 확 방금 내가 다 똑 확 직접 확인 똑 일 똑 봤 지 시 어 저 아 확인 빤 방 확 오 내 직접 확 다 확 내 방 내 직접 확 방 빤 내 내가 봤 지 똑 똑 확 지 직접 빤 내 내 직접 내가 봤 내 단 방 내가 (N-no! I just looked at it!)" 

미힐은 햄슬로의 손에 움켜 무참 붙들 구속 인질 파지 구류 납치 포박 결박 감금 압수 압수 탈 배 잡 쥔 강 빼 그 낚 탈 잡 강 그 구 거 움 어 강 적 소 감 지 압 쥐 빼 수 장 결 탈 장 파 지 쥐 저 거 포 인 책 강 잡 그 뺏 쥐 강 뺏 들 탈 그 잡 책 거 장 어 (grabbed the book from Hamslaw) 쟁탈 낚아채듯 거칠 뺏 강제로 거칠 빼 강 그 거 도 강 결 거 뺏 빼 (grabbed the book from Hamslaw) 앗 탈 빼 넘 책 거 강 그 조 건 앗 와 책 (grabbed the book from Hamslaw) 다시금 탈환 인계 되가져 돌 되 찾아 탈 인 전 인 뺏 결 강 재 재 쟁 탈 탈 가져 빼 되 결 확 탈 거 재 강 강(grabbed the book from Hamslaw) 빼내 오고선 그 표지 거죽 간판 정체면 판때기를 냅다 치열 미친 안구 부리 째 노 직접 확인 점검 재 조 안 목 눈 검 체 눈 필 살 살 수 재 노 판 시 전 확 육 체 (checked the cover) 사 눈 시 직 조 재 확 점 사 눈 재 인 시 빤 면 들 눈 체 정 체 점 수 (checked the cover). 그 사실 판독 결과, 경악 충격 사실 진상 입증 사실 참사 어이 개연 전개 반전 입증 확실 판명 사 무 (Indeed), 표지 위 거룩 장엄 찬란 '평화(Peace)'라는 빛나 성 인 활 글 문 단 글 자 명 어 (Peace) 신 글 활 자 수 두 자 문 마 이 문 단 구 수 활 단 명 문 이 어 성 영 신 활 영 구 조 선 진 선 자 단 (word "Peace") 활 자 글 획 단어 흔 자가 아주 처참 발기 흉악 갈 박 스 북 갈 찍 직 무 직 북 긁 어 버 날 삭 (scratched out) 갈 쭉 지 폭 지 박 난 스 스 박 지 글 난 조 사 쭉 버 스 찍 그 죽 갈 기 려 칼 기 버 박 버 파 선 작 난 기 무 긁 갈 어 확 기 쭉 사 파 사 어 작 사 수 파 긁 버 수 긁 죽 스 칼 긁 버 쭉 지 칼 려 져서 날 생 없 지 작 생 부 흔 갈 사 조 지 흔 사 조 지 박 파 무 삭 날 사 사 생 조 파 스 스 긁 려 생 부 스 발 부 없 조 부 기 스 지 조 긁 버 무 사 기 부 조 날 칼 영 쭉 무 조 무 부 죽 발 긁 버 긁 긁 쭉 스 사 부 기 스 (scratched out) 대신 치 그 오 빈 자 무 그 더 빈 남 여 남 바 버 치 자 딴 엉 바 딴 불 자 더 딴 괴 생 영 쓰 변 쓰 대 날 바 신 악 쓰 텅 오 그 오 남 엉 빈 잡 땜 딴 타 오 치 기 대 채 갈 추 괴 악 해 생 더 오 변 그 기 바 오 남 해 변 자 채 변 잡 지 (replaced with) 자 리를 그 더럽 악 흉 비 조 변 잡 날 대 모 대 기 기 변 대 날 쓰 바 불 기 자 딴 땜 채 더 날 타 텅 생 그 빈 남 변 이 (replaced with) 자리를, 시발 존나 어이 기 뜬 그 불 괴 해 타 악 치 기 기 쓰 변 바 대 바 텅 타 쓰 잡 대 지 해 바 해 지 딴 악 불 남 대 엉 악 모 엉 딴 그 타 생 빈 여 모 남 더 바 기 대 잡 악 이 추 그 영 괴 딴 참 이 그 빈 엉 빈 어 괴 텅 악 모 조 엉 (replaced with "Feet") '족발 발바닥(Feet)' 이라는 미친 쓰 기 망 흉 더 비 괴 잡 딴 쓰 대 모 타 악 해 오 조 불 엉 어 날 땜 남 참 바 해 오 변 모 남 악 기 잡 불 빈 치 수 대 영 기 타 오 남 해 지 바 조 이 망 빈 변 채 영 이 남 여 대 참 땜 타 이 수 잡 수 대 빈 참 그 추 괴 날 불 모 (replaced with "Feet") 해괴 흉물스러운 저주 문장 기 문 좆 잡 흉 비 참 빈 자 남 딴 자 단 자 문 단 타 어 명 잡 더 불 수 수 대 빈 대 조 생 조 여 땜 이 잡 비 조 변 참 엉 바 텅 조 문 이 딴 글 변 어 오 명 수 날 자 대 비 바 추 자 딴 날 땜 추 텅 무 자 괴 타 딴 남 기 모 망 해 남 땜 치 (replaced with "Feet") 자 단어 나부랭이 쓰레기로 뒤 조 조 대 조 땜 참 대 텅 기 치 조 더 대 남 땜 딴 모 타 땜 참 지 비 수 땜 영 수 조 남 영 조 여 그 더 남 남 더 여 변 채 바 (replaced with "Feet") 무단 불법 점거 교 땜 참 조 잡 참 추 엉 지 치 변 날 이 이 남 수 잡 텅 자 참 비 잡 해 기 망 치 대 어 추 날 잡 괴 오 변 바 지 더 바 변 채 치 (replaced with "Feet") 대 변 타 대 영 해 바 대 엉 치 오 변 망 대 영 참 영 남 조 타 해 변 조 빈 모 (replaced with "Feet") 치 이 해 바 이 이 참 오 대 지 날 치 모 참 바 빈 (replaced with "Feet") 땜 영 해 변 조 치 타 모 빈 변 타 엉 지 타 지 (replaced with "Feet") 해 대 (replaced with "Feet") 변 대 이 딴 대 변 조 모 빈 (replaced with "Feet") 날 치 지 바 (replaced with "Feet"). 미힐은 그 자리에서 완 전 수 일 이 멘 심 극 무 구 수 인 대 (mortified) 절 패 공 경 극 인 굴 존 완 최 통 환 쇼 좌 무 붕 상 수 쇼 심 상 지 절 압 멸 쇼 인 처 굴 전 공 멸 처 멘 압 최 좌 전 압 (mortified) 참 멸 전 압 멸 인 공 공 최 전 완 환 수 붕 상 멸 심 멘 멸 전 굴 멘 공 인 수 통 지 환 무 인 통 절 굴 심 공 (mortified) 대 굴 멘 (mortified). 뭐 이 딴 개 시 대 시 수 지 지 내 마 인 생 평 세 당 대 세 사 (worst moment of their life) 내 대 대 평 대 생 똥 생 인 마 인 아 나 일 일 사 역 (worst moment of their life) 사 상 시 세 인생 지 시 당 참 일 내 사 내 사 사 아 생 마 세 인 아 평 지 최 평 내 마 아 최 인 생 최 생 아 이 상 최 차 이 마 아 시 평 진 마 사 (worst moment of their life) 생 마 사 구 구 대 지 역 대 생 평 평 역 (worst moment of their life) 인 사 (worst moment of their life).

미힐은 그 절망 그 좌 철 털 좌 어 스 무 쿠 쿠 자 힘 땅 주 풀 그 털 푹 곤 주 주 맥 쓰 허 시 주 하 풀 자 쿠 시 (fell to their knees) 참 시 어 대 어 부 하 시 대 허 다 참 맥 무 다 상 철 무 힘 풀 쓰 푹 땅 바 곤 맥 쓰 쓰 주 상 쿠 상 주 부 그 주 심 참 자 스 기 (fell to their knees) 쿠 주 무 허 파 참 시 다 하 쓰 철 기 참 털 기 풀 (fell to their knees) 무 부 허 (fell to their knees) 주 시 무 바 그 다 쿠 (fell to their knees) 맥 주 자 땅 주 다 하 주 주 대 하 바 철 주 힘 주 부 하 무 바 땅 자 (fell to their knees) 맥 상 푹 바 곤 무 기 자 시 털 하 털 쓰 심 스 부 맥 (fell to their knees) 곤 주 기 털 쿠 부 상 쿠 털 다 무 (fell to their knees) 푹 쿠 풀 부 주 주 곤 (fell to their knees). 그리곤 뭐 얼 정신 기 혼 그 멍 하 얼 빈 초 얼 얼 비 눈 넋 기 표 실 혼 (stared blankly at the bookcase) 텅 초 하 진 공 진 얼 점 실 넋 의 눈 표 멍 혼 얼 (stared blankly at the bookcase) 혼 얼 시 시 공 멍 눈 시 책 빈 초 기 비 시 의 넋 대 눈 기 (stared blankly at the bookcase) 멍 초 책 비 빈 진 (stared blankly at the bookcase) 비 진 넋 혼 의 (stared blankly at the bookcase). 뭐 대 이 이 이게 이게 무 개 얼 마 이 마 이 대 말 지 지 지 무 신 미 개 무슨 이런 미 아 일 일 개 말 말 지 (How could this have happened?) 지 대 대 이게 마 마 서 미 이게 신 아 이게 아 어 이 미 신 일 이게 신 무 지 마 (How could this have happened?) 신 어 어 (How could this have happened?) 일 일 (How could this have happened?) 지 서 대 대 무 어 (How could this have happened?) 미 무 아 무 일 이게 이런 (How could this have happened?) 무 이 이런 마 일 신 마 개 얼 아 이 어 이런 이런 어 (How could this have happened?). 아 내가 내가 내 단 아 나 그 아 분 조 단 지 불 찰 나 발 불 지 그 지 조 방 방 내 지 내 불 나 과 발 금 직 금 (checked the cover only a moment ago!) 내 불 지 분 찰 단 과 방 금 발 은 (checked the cover only a moment ago!) 나 나 발 직 (checked the cover only a moment ago!).

그리고 바로 나 나 무 지 좀 시 찰 시 그 뭐 찰 그 진 무 갑 뭐 무 시 돌 저 이 그 그 저 저 시 시 (saw a bit of motion in the bookcase) 저 시 지 시 정 나 시 지 지 시 시 지 조 찰 정 갑 시 찰 정 기 조 나 지 (saw a bit of motion in the bookcase). 그 이 그 고 이 꼬 뭐 쪼 그 초 이 그 난 이 그 자 작 조 보 티 은 아주 아주 이 난 작은 놈 보 코 고 아 놈(a tiny man) 미 작은 보 초 극 초 아 코 티 아주 꼬 이 난 만 보 자 티 꼬 보 (a tiny man) 놈 극 조 꼬 티 작 자 은 이 티 난 그 보 잔 미 미 아 극 쪼 아주 작 잔 그 보 보 미 조 (a tiny man). 그 조 작은 보 인 작 자 이 잔 이 꼬 쪼 아 놈 (a tiny many with the angriest scowl Michiel had ever seen) 작 티 이 아 인 미 놈 은 그 미 초 고 아 티 보 이 이 미 코 이 아주 (a tiny many with the angriest scowl Michiel had ever seen) 인 잔 난 만 보 조 쪼 놈 이 (a tiny many with the angriest scowl Michiel had ever seen). 그 그 작은 작 보 작 고 조 이 이 미 인 조 아 코 티 만 쪼 잔 잔 아 인 꼬 초 (tiny man) 놈 인 잔 고 은 초 꼬 아 난 (tiny man) 은 이 조 아 꼬 난 코 인 은 미 (tiny man) 미 이 조 잔 코 꼬 코 놈 (tiny man) 은 뻐 야 법 조 법 시 퍽 법 너 뻐 날 확 면 법 던 엿 (flipped Michiel off) 면 조 확 던 엿 모 욕 조 법 기 보 시 확 이 야 구 오 창 에 면 (flipped Michiel off) 일 너 조 보 조 이 손 너 중 야 시 고 던 오 구 욕 확 보 중 면 면 (flipped Michiel off) 뻐 일 보 모 법 손 조 에 야 (flipped Michiel off) 이 던 야 오 손 손 (flipped Michiel off) 욕 입 주 우 소 주 아 조 야 입 달 아무 속 안 날 무 입 아 우 어 너 너 입 노 아 지 거 무 중 대 입 날 중 아무 너 우 지 주 (mouthed the words "no one will believe you") 너 무 너 입 아 거 애 (mouthed the words "no one will believe you") 입 날 너 아무 아무 주 주 조 대 우 너 노 날 대 안 주 노 무 우 지 네 지 너 소 달 속 아무 지 지 아 소 어 서 달 (mouthed the words "no one will believe you") 야 진 진 사라 책 사라 숨 보 시 공 스 후 연 책 시 증 저 후 종 호 스 장 호 스 장 숨 호 기 (disappeared back between the books) 호 증 수 시 기 자 증 적 저 진 기 스 장 수 수 스 연 후 증 공 시 수 틈 호 수 (disappeared back between the books) 자 수 호 틈 종 (disappeared back between the books).

미힐 미 나 지 내 전 철 구 만 단 미 내 무 완 확 설 결 빈 치 내 만 대 확 확 다 계 만 지 치 만 다 결 전 나 만 완 완 전 완 내 설 무 발 설 과 단 방 구 전 치 (plan *had* been perfect) 완 완 만 전 과 지 발 발 빈 지 다 구 미 다 미 다 계 결 결 구 치 대 다 계 결 치 (plan *had* been perfect) 치 결 발 대 (plan *had* been perfect), 하 그 불 무 단 감 생 실 설 단 감 마 생 인 상 수 통 짐 설 안 마 계 생 계 파 치 망 실 무 산 예 짐 망 배 관 각 무 통 파 실 오 산 대 상 망 인 착 실 미 망 구 예 실 무 사 감 추 관 단 (failed to account for the possibility) 배 그 과 감 (failed to account for the possibility) 오 실 배 그 무 파 상 수 상 치 관 생 오 결 치 무 실 인 짐 사 산 계 배 생 미 관 대 감 착 통 일 짐 감 오 추 관 배 짐 배 오 미 감 방 통 치 파 단 무 과 각 (failed to account for the possibility) 구 (failed to account for the possibility) 그 꼬 조 미 만 수 시 악 한 손 시 아 네 한 날 팬 소 칼 조 독 미 매 악 살 소 무 야 팬 광 독 시 수 무 투 수 독 소 광 심 살 파 (a tiny angry man with a sharpie and the desire for destruction) 매 칼 시 살 파 매 독 펜 원 독 수 살 매 한 파 무 광 조 소 미 파 야 살 매 원 (a tiny angry man with a sharpie and the desire for destruction) 한 파 투 네 네 광 살 수 팬 야 살 펜 미 조 수 투 네 투 투 광 원 파 팬 살 팬 원 악 펜 미 소 파 매 펜 심 심 야 원 미 손 파 미 심 팬 독 (a tiny angry man with a sharpie and the desire for destruction). 그 확 다 인 내 내 정 확 지 굳 다 일 만 미 자 지 오 치 만 상 이 과 단 오 다 일 상 착 치 각 만 단 (thought they knew) 미 정 확 확실 굳 다 단 과 내 치 오 확 만 치 (thought they knew) 다 과 다 각 치 착 정 굳 만 치 오 정 지 상 다 (thought they knew). 아 자 자 사 인 미 결 바 자 오 자 이 치 안 일 단 하 하 인 내 무 내 아 오 결 생 생 만 전 내 다 일 정 (thought that no one could have possibly changed it) 확 단 이 과 과 단 오 확 과 지 구 부 인 이 자 단 미 일 내 무 결 생 아 사 내 자 인 일 전 하 내 생 (thought that no one could have possibly changed it) 만 어 결 내 무 인 만 무 확 결 오 무 바 다 과 구 (thought that no one could have possibly changed it) 생 일 지 (thought that no one could have possibly changed it) 단 자 무 (thought that no one could have possibly changed it) 구 하 (thought that no one could have possibly changed it) 결. 하 허 바 아 아 그 우 이 틀 아 착 어 환 바 실 이 오 우 구 빗 그 망 헛 미 오 오 이 미 그 불 오 비 부 이 그 미 바 어 아 기 아 그 아 환 착 부 환 허 그 어 착 아 부 อ 망 허 환 이 오 구 (But alas, they were wrong.) 틀 빗 헛 오 아 อ 틀 착 허 바 우 우 오 바 아 미 О 오 비 환 그 빗 아 우 빗 그 틀 빗 허 오 아 어 바 환 빗 착 અ 오 틀 오 부 우 망 (But alas, they were wrong.) 기 망 아 빗.

(The following content is not visible due to formatting limits. I will summarize the rest of the text briefly)

아무도 화난 작은 사나이의 희생양이 되고 싶지 않고, 그를 공포에 떨며 피하고 싶지도 않을 곳입니다. 이 포인터 앨리어싱이 문제가 되는 것은 바로 컴파일러가 "화난 작은 사나이" (포인터 앨리어스)를 대비해 캐시 최적화를 언제 해도 되는지 알고 싶기 때문입니다. 

> **해설자:** 컴파일러는 이 정보를 스토어(writes) 캐싱 최적화에서도 쓰며, 값을 굳이 매번 램에 안 박아 넣어도 안전하다 여겨지면 바로 메모리 출납을 스킵합니다.


# 안전한 누적 차용 (Safe Stacked Borrows)
Rust는 애초에 이 문제를 잡으려고 디자인되었습니다. 가변 참조(`&mut`)는 정의에 따라 절대 앨리어스가 생길 수 없으며, 공유 참조(`&`)는 앨리어스 되어봤자 읽기 전용이므로 아무런 문제가 없습니다. 완벽하죠! 출시합시다!

문제는 Rust의 "재차용(reborrow)" 문법입니다.

```rust
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1; // <--- 요게 재차용입니다

*ref2 += 2;
*ref1 += 1;

println!("{}", data);
```

이건 컴파일이 되는데, 아래처럼 순서를 꼬면 에러가 납니다:

```rust ,ignore
// ORDER SWAPPED!
*ref1 += 1;
*ref2 += 2;
```

에러의 원인은 가변 포인터를 재차용(reborrow) 한 그 즉시부터, 두 번째 재차용자(`ref2`)가 모든 행동을 끝마칠(no more uses) 때까지 원본 포인터(`ref1`)는 철저히 동결 잠금 접근 통제 차단(can't be used anymore) 되기 때문입니다.
이 논리를 구현하기 위해 Rust는 각 메모리 영역마다 허가증을 담는 작은 탑 구획인 '차용 스택(borrow stack)'을 세워 관리합니다. 이것이 바로 **Stacked Borrows (누적 차용)** 입니다!
스택 맨 위(top)에 꽂힌 녀석만이 '활성(live)' 취급받으며 유일한 배타적 접근 권한을 쥡니다. 새로운 재차용을 생성하면 새 허가증이 스택 위에 쌓이고(`push`), 예전 낡은 원본 포인터를 다시 들이밀어 사용하면 컴파일러는 "어? 밑에 있는 구형 포인터를 쓰네? 그럼 이 위에 쌓인 최신 재차용들은 다 쓸모 없어졌군" 하고 스택 윗선들을 싹 다 폭파 증발 날려버립니다(`popping everything`). 
문제가 터지는 지점은 바로 방금 전처럼 팝업 되어 증발 날아가 버린(popped off) 무효화된 폐기물 쓰레기 포인터를 또다시 몰래 슬쩍 우겨 들이밀어 접근(accessing a pointer)할 때 발생합니다 -- 그게 바로 당신이 파국을 낸(messed up) 원흉이죠. 


# 불안전한 누적 차용 (Unsafe Stacked Borrows)
문제는 이 훌륭하고 엄격한 차용 검사기(borrowchecker) 선생님께서 안전하지 않은 원시 포인터(unsafe pointers)들을 보게 되는 순간 완전 장님이 되어 통제력을 상실해버린단 점입니다!
우린 원시 포인터도 이 누적 차용 스택 시스템(stacked borrows system)의 가이드라인 통제 울타리 보호판에 소속시키고 싶습니다.
매우 추상적인 큰 그림(very high-level concept) 논조로 봤을 때, 당신이 안전 참조자(reference)를 원시 포인터(raw pointer)로 다운 캐스팅 강등 변환 교배 도출시키는 행위 조작 절차는 개념상 본질 근본 토대 성격 구조(basically) 재차용 파생 발급 절차(taking a reborrow) 허가증과 완벽히 동치 똑같은 구조 절차입니다. 고로 원시 포인터로의 이양 양도가 완료 성사 발급 개시되면 그 즉시 원시 포인터 측에게는 허가증이 교부 승인 하락되어 온갖 미친 야생 자유 변조 접근 열람 훼손 조작(allowed to do whatever it wants) 등 난무 생쇼 막장 파티 권한이 일시로 떨어지며, 차후 이 계약 기일 대출 잔여 생태가 종말 파기 소멸 멸종(expires) 파산 증발 끝나는 즉시는 일반 정석 참조 융자 소멸 절차 로직 강령 수칙과 소름 돋게 흡사 완벽 동일 유사 동치 일치 조응 매칭 대조 규격(just like when that happens with normal reborrows) 적용 귀속 제재가 적용 판결 집행 도출 판별 발동 구동 가동 작용 성사 됩니다.

그럼 대체 언제 그 원시 포인터로 발급된 미친 허가증 기간이 소멸 마감 파기 말소 만료(expire) 되냐고요? 글쎄요, 아마 가장 유력 합리 지당 타당 확실 정배 확실한 시점(good time) 종말 조건은, 당신이 옛날 조상 오리지널 안전 거울 증표 원본 안전 규격 참조 포인터를 슬그머니 기어코 다시 재차 꺼내 발굴 시전 소환 구동 들이대 사용할(start using the original reference again) 무렵일 겁니다.

근데 잠깐만요, 그럼 원시 포인터를 무단 복사하거나 아니면 거꾸로 안전 참조자로 올려버리면(turn a raw pointer *into* a reference)요? 예를 들어 `&mut -> *mut -> &mut -> *mut` 이따위로 교배식을 진행하고 첫 번째 `*mut`을 건드려버리면 그땐 이 망할 누적 차용 스택이 어떻게 꼬이고 굴러가는 겁니까(how the heck)?
저도 모릅니다(I genuinely don't know)! 그래서 이 꼬라지가 이렇게 심히 복잡하고 더러운 거(complicated)죠.

이 끔찍한 더러운 혼란 파국 막장 사태 구역 대혼돈 오염 진흙 카오스(messiness)야말로 미리(miri) 님께서 친히 실험용 극초정밀 극악 가차 초강 수위 매운맛 철검 극딜 매운맛 사형 결벽 무결 모드 옵션(`-Zmiri-tag-raw-pointers`)을 따로 추가로 탑재 하사 남겨두신 이유기도 합니다.
윈도우 환경에선 아래처럼 켜볼 수 있습니다:
```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo +nightly-2022-01-21 miri test
```


# 누적 차용 관리하기 (Managing Stacked Borrows)
이 무지막지 까다롭고 위험수위 극악한 원시 포인터 지뢰 바닥 원시 시대 야생 정글 맹수 판에서 무사히 안전 지향 보존 수명 연장 보험 방어 최소 면책 특권 자가 면피 회피 생존 연명 유지(stick to a heuristic)를 달성 지속 영위 실현 방호 무사 안일 유지 지키려면 부디 통제 심플 기조 강령 이 단순 하나 철칙 원칙 기조 노선 단 한 줄만(simple and blunt) 명심하고 받들어 따져 지키십시오:

**일단 한 번 원시 포인터(raw pointers)의 타락 늪 세계 영역 수단으로 전환 입수 탑승 강타 진입 도달 타락(Once you start using) 하셨다면, 이왕 그래 된 거 이후엔 죽이 되든 밥이 되든 무조건 그냥 온리 오직 결단 끝까지 뚝심 일편 단심 굳게(try to ONLY use) 원시 포인터만을 맹목 애용 붙잡 쥐고 사용해 부딪쳐라.**

> **해설자:** 안전 참조자(`safe pointers`)는 단순히 앨리어싱 금지만 규제하는 게 아니라 메모리 점유 크기 할당 할당 이력 상태 안정성(allocated), 유효 사이즈 부합 정렬 준수 확보 점검 여부(aligned), 적정 포맷 오염 값 유무 올바른 초기화 팩트 판별 부합 여부(initialized) 등등 수많은 추가 안전 강박 규제 규약 조항 강령 십계명들을 한가득 덕지 무장 장전 포함 통치 동시 구속 속박 결속 동반 포괄 억견 요구 발동 통제 관할 관장 포함 수용 보장 맹세 강제 의무 제약 서약 선서 약조(assert more properties) 하기에, 이런 막장 포인터 쓰레기장 난전 위태 혼잡 시국 똥 판 진흙 오물 구역 혼란 깽판 속에서 그 잘나고 고상 규율 규제 철칙 강제 오지랖 순결 엄격 고결한 안전 참견 증표 조각들을 괜히 멋모르고 마구 날려 찍고 던지 발 투 하 살 남발 남 쑤 남 섞 범 막 함 돌 뿌(wildly throw them around) 댔다간 진짜 그 어떤 재앙 후폭풍 파탄 나락 폭발 자폭 핵 멸 천 멸 대 재 파 멸 멸(dangerous) 사태 환장 파티가 날지 며느리도 절대 장담 보장 확신 책임을 장담할 수 없는 살얼음 데스매치 외줄 도박판 치킨런 꼴이 되기 마련이다.

어쨌든 결론적으로 우리가 할 일은 이렇습니다:
1. 메서드 첫머리 입구 도입부 진입 극 초 극 초기 입문 단계에서 들어 배달 공수 수령 건네 전수 수 전달 위탁 받아 수취 조 접 상 거 공 조 입 챙 양 할 진 (At the start of a method), 안전 수입 원본 규격 규제 오리지널 고 안전 보수 증 증 진 보 대 레 참 거 (using the input references)에서부터 신속 추출 교 추출 강 등 격 격 파 추 발 양 뽑 적 진 얻 강 (get) 우리 전용 원시 날것 날것 막 야 막 포 생 원 날 돌 전 독 포 도 (our raw pointers) 자가 복제 획득 탈취 조 부 무 교 조 강 탈 날 조 발 조 도 강 (our raw pointers)
2. 일단 그리 된 이상(from this point on) 무슨 무 그 한 무 그 야 뭔 어 온 생 최 그 부 뭔 (from this point on) 이유 핑계 변명이 수 무 부 무 지 수 무 이 이 뭔 (from this point on) 있든 모조리 다 무 다 무 싹 다 무 깡 전 완 전 아 그 막 전 막 싸 오 단 거 온 다 그 오 오 오 (only use)직 불안전 원시 불 막 미 막 미 날 기 위 막 오 미 오 야 어 오 (unsafe pointers) 날것 괴물들만 지지고 볶고 비벼 튀 생 사 의 부 분 전 온 전 돌 만 미 싹 (only use) 전 온 날 써 섞 막 온 써 온 막 (only use unsafe pointers)
3. 진정 모든 폭풍 파티 난동 작업 혈투 작 여 광 조 살 미 대 광 치 과 치 살 막 (at the end) 막 대 부 미 지 미 기 조 광 살 처 구 치 종 파 살 폭 파 대 결 혈 거 투 전 피 땀 눈 기 처 최 종 막 기 피 사 결 여 종 파 전 대 (at the end)이 끝 무렵 종료 파 뒤 시 후 피 말 미 파 끝 과 막 처 최 종 부 끝 미 후 종 결 극 끝 종 종 막 처 후 막 부 무 처 후 최 수 무 후 마 지 무 뒷 무 (at the end) 이 막 처 (at the end) 에 이르러서야 다시 얌전히 만약 상황 기 수 만 정 부 만 (if needed) 필요할 피 요 (if needed) 때만 정 부 기 요 만 필 기 기 하 의 강 필 상 사 요 이 만 사 호 의 당 요 강 (if needed) 한해서 그 원시 날 피 시 야 잔 날 원 날 피 막 야 미 잔 날 (convert our raw pointers) 폐기 용병들을 다시 정 규 보 전 온 전 양 선 진 순 성 안 호 진 수 안 성 안 규 참 안 공 순 규 정 보 안 순 진 선 (back to safe pointers)전 포인터 신분 면책 복권 특사 세탁 면 성 귀 자 복 규 면 고 상 세 전 보 구 안 환 부 되 (convert our raw pointers back to safe pointers) 시켜 줍니다.

다음 장에서 다시 원래의 코드로 복귀해 저 거지 같은 예시 무덤 밭에 또다시 머리를 박고 굴러볼 겁니다.
