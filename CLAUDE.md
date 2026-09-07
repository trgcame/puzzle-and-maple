# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

<!-- >>> managed by mswai >>> -->
@AGENTS.md
<!-- <<< managed by mswai <<< -->

---

## Project: 퍼즐앤메이플 (Puzzle & Maple)

A **방치형 매치3 퍼즐 RPG** built on MapleStory Worlds. 무기 원소를 매치3로 터뜨려 5인 파티의 스킬을 발동하고, 스테이지(5층 단위)를 돌파하며, 뽑은 영웅을 성급·스타포스로 키운다.

> 🖥️ **타겟 플랫폼은 PC 전용입니다** (2026-09-07 확정). UI 캔버스는 1920×1080 고정이라 화면 구조는 그대로지만, **모바일 기준(터치 타겟 88×88 · SafeArea)은 적용 대상이 아니고**, 대신 **PC 시스템 UI 예약영역**이 실제 제약이다 — 좌상단 260×170(채팅)·우상단 220×130(친구·메뉴)에 엔진이 항상 UI를 그리고 **끌 API가 없다**. 여기 겹치는 화면 요소 7곳이 미해결(GDD Phase 5 G17).

> **기획 문서가 소스 오브 트루스입니다.** 이어서 작업할 때는 `msw-planning` 스킬을 먼저 로드하고 resume flow를 따르세요.
> - `Docs/PuzzleMaple-M1-GDD.md` — 현재 마일스톤(v1.0 출시) 계약서 + Phase 체크리스트
> - `Docs/PuzzleMaple-Roadmap.md` — 마일스톤 로드맵 · 백로그 (마일스톤 밖 작업의 단일 관리처)
> - `Archive/As-built.md` — 구현 현황 지도 · 상시 규칙

### Maps & UI

게임플레이는 **전부 UI 공간(`BattleGroup`)** 에서 동작합니다. `.map`은 룸 컨테이너 역할이라 `TileMapMode`를 바꿀 일이 없습니다.

| File | Purpose |
|---|---|
| `map/Lobby.map` | 로비 (스크립트에서 `MoveUserToStaticRoom("Lobby")`로 참조) |
| `map/BattleMap.map` | 전투 룸 |
| `map/Boss.map` · `Rest.map` · `Shop.map` · `UITest.map` | **스크립트 참조 0건** — 2026-09-07 사용자 결정으로 **삭제하지 않고 남긴다**(Phase 6 맵 점검 대상에는 포함) |
| `ui/RobbyGroup.ui` · `Lobby.ui` | 로비 화면 |
| `ui/BattleGroup.ui` | 전투 HUD (퍼즐 보드 · 영웅/몬스터 슬롯 · 보스 오버레이). 영웅 슬롯 `Hero1~5`에 `mainpuzzle`/`subpuzzle` — `PlayerManager:SetupPartyDisplay`가 무기타입으로 채움 |
| `ui/HeroInventory.ui` | 영웅 인벤토리 · 상세(초상·이름·등급·**다음 성급까지 남은 장수 게이지**(`DetailPanel/StarProgress` = 라벨 + `SliderComponent` + 수치, 2026-09-07 신설)·공격력·체력·주/부속성 퍼즐·스타포스 게이지·스킬·파티 토글). **성급 ★ 표기는 2026-09-05 제거** — 스타포스 게이지와 헷갈려서. 공격력·체력·스타포스 25칸 게이지는 **표시 전용**으로 유지. **강화 실행 UI는 2026-09-05 제거** — 강화는 `Enhance.ui` 전용 |
| `ui/Enhance.ui` | 스타포스 강화 전용 화면. 로비 `Enchant` 버튼 → `EnhanceUI:Open()`. 좌 대상(`PortraitRUID`) · 중앙 전신(`ThumnailRUID`) · 우 정보(공격력/체력/성공/실패/하락/파괴) · 하단 비용·파괴 방지(토글, 끌 때까지 유지)·강화하기. 결과 이펙트(`Result` 스프라이트, animationclip 3종)와 사운드 4종 배선됨 |
| `ui/Shop.ui` | 가챠(소환) 화면. 로비 `상점` 버튼 → `UIShopOpen`이 그룹 Enable, `BackButton` → `UIShopClose`. 1회/10연차 결과 팝업 · 확률 팝업 포함 |
| `ui/Ranking.ui` | 랭킹 화면. 로비 `RightAside/Inventory` 버튼 → `UIRanking` → `RankingUI:Open()`. **팝업 형태**(`BackGround`=딤 + `Popup`=패널). 탭 2개(스테이지 최고 층 / 시즌보스 누적 딜량) + **`GridView` 가상 스크롤 목록(100위)** + 내 순위. 목록 행은 `ListPanel/RowTemplate` 하나를 복제해 쓴다 |
| `ui/SceneTransition.ui` | 씬 전환 연출 (Transition / BossTransition / FadeOut) |
| `ui/PopupGroup.ui` · `ToastGroup.ui` · `DefaultGroup.ui` | 공용 모달 · 토스트. 🔴 **`UIPopup`(공용 모달)은 화면에 뜨지 않는다** — `PopupBack`이 `enable=false`로 저장돼 있는데 `UIPopup:Open()`은 루트 `popupGroup`만 켜고, `PopupBack`을 켜는 코드가 없다. 시즌보스 도전 횟수 소진 안내(2곳)가 조용히 사라진다. **켜면 로비 UI 뒤에 그려져** z-order까지 같이 고쳐야 하므로 M1에서는 손대지 않기로 했다 |
| `ui/AugmentGroup.ui` · `MapGroup.ui` | **스크립트 참조 0건** — Phase 1에서 컨트롤러만 지우고 `.ui`는 남은 고아 파일. Phase 6 삭제 대상 |

### Script Architecture

**Managers** — 엔티티 경로 `/common/Manager/` (등록 위치는 `Global/common.gamelogic`, **AI 편집 금지** — 추가/삭제는 Maker 작업)

| Script | Role |
|---|---|
| `GameManager` | 진입점. `StartRun()` → 파티 HP 초기화 → `CurrencyManager:GetMaxClearedStage()+1` 스테이지로 시작 |
| `StageManager` | **진행 축.** `StageSet` 로드, 5층 구조, 층/스테이지 클리어 판정. `absoluteFloor = (stage-1)*5 + floor` |
| `BattleManager` | 전투 오케스트레이션(최대 파일). 스킬 스텝을 힐/버프/디버프/광역/단일로 분류 후 `_Queue:RunSequence`로 순차 연출 |
| `TurnManager` | Player → Resolving → Enemy 순환 |
| `PlayerManager` | 파티 5인 HP·쉴드·타겟팅·피격 애니. 슬롯 UI + `HeroInfo` 자식 슬롯 이원 구조 |
| `MonsterManager` | 몬스터 슬롯 5개, 소환/부활, 히트·사망 애니. `slotIdx` 오름차순 = 전방→후방 |
| `EnemySkillManager` | 적 스킬 트리거 3종: `every_n_turns` · `hp_threshold` · `phase_change` |
| `SeasonBossManager` | 3페이즈 보스, 파츠 4개 조립, 일일 도전 횟수 |
| `CurrencyManager` | 골드·티켓·강화석, 방치보상(10분 단위, 최대 12h). 🔴 `_RewardRateForStage`의 시급 2줄은 **경제의 기준점**이라 `StarforceSet.GoldCost`와 한 쌍으로만 의미가 있다 |
| `UIManager` | `navStack` 기반 패널 전환 + `SceneTransition` 연출 |

**기타 `@Component`**: `PuzzleBoard`(5×6 보드) · `PuzzleController`(타일 드래그) · `MainCamera` · `UI folder/Components/*`(버튼 글루)

**Global Logics** (`_PuzzleLogic` 형태로 접근):

| Script | Key responsibility |
|---|---|
| `PuzzleLogic` | 드래그 상태, 캐스케이드 resolve 체인, 완료 시 `BattleManager:ExecutePlayerAttack(MatchResult)`. **드래그 10초 제한**(`DragTimeLimit`) — 표시는 `PuzzleBoard/DragTimer` 시계 게이지(Radial360 `FillAmount`)로, **드래그 중인 타일을 따라 움직인다**(같은 부모라 `anchoredPosition` 복사) |
| `HeroDataManager` | 영웅 정적 데이터 + DataStorage 3키(`heroes`/`party`/`starforce`). 더티 플래그 + 60초 flush |
| `HeroInventoryUI` | 영웅 카드 동적 생성, 상세 패널, 파티 편성. 강화는 `EnhanceUI` 담당 |
| `EnhanceUI` | 강화 전용 패널 컨트롤러. `panel` UUID 1개만 프로퍼티로 받고 나머지는 `_Find("A/B/C")` 경로 조회. **패널 내부 버튼은 첫 `Open()`의 `_EnsureBound()`에서 지연 연결**(`DefaultShow=false` 그룹의 자식은 `OnBeginPlay` 때 안 잡힘). 재빌드 시 루트 UUID가 바뀌므로 빌더 `bind`로 자동 주입 |
| `RankingManager` | 랭킹 서버 로직. 보드당 `SortableDataStorage` 1개(`stage_rank`/`boss_rank`). 닉네임은 `KeyInfo.Tag`에 저장. **`GetSortedAndWait`의 min/max는 점수 범위**라 전 구간을 받아 잘라야 한다 |
| `RankingUI` | 랭킹 화면 컨트롤러. 탭 전환 + `GridView` 목록 렌더. `ItemEntity`/`OnRefresh`는 화면 파일에 저장되지 않아 `_EnsureGrid()`가 첫 `Open()`에서 붙인다. 1~3위는 메달 스프라이트, 4위부터 숫자 |
| `GachaResultUI` | 가챠 결과 팝업(1회/10연차) · 소환 확률 팝업. 루트 3개만 프로퍼티로 받고 자식은 `GetChildByName`으로 조회. 확률은 `GachaSet`/`GachaConfigSet`을 런타임에 읽음 |
| `Effect` | `Model_Effect` 스폰 → 재생 → 자동 Destroy. Sync/Async 두 계열 |
| `Queue` | `RunSequence(steps)` — `action()` 실행 후 `duration` 대기. `StopSequence()`로 전투 종료 시 중단 |
| `Resource` · `Tween` · `Math` | 유틸 |
| `UIPopup` · `UIToast` | 공용 모달 · 토스트 |
| `UISoundManager` | **UI 버튼 공용 클릭음**(RUID `7eda004d…`). 화면마다 붙이지 않고 `/ui` 트리를 재귀로 훑어 `ButtonComponent` 보유 엔티티에 `ButtonClickEvent`를 연결한다. 🔴 **2초 주기 재스캔 필수** — `DefaultShow=false` 그룹의 자식은 `OnBeginPlay`에 없다(실측 166 → 171개) |

### Dataset (`RootDesk/MyDesk/Dataset/`, 20종)

CSV가 소스 오브 트루스입니다.

- **영웅**: `HeroSet` · `HeroSkillSet` · `HeroSkillEffectSet` · `HeroSkillVisualSet` · `HeroSlotSet` · `StarLevelSet`(등급별 ★1~★5 필요 **총 장수**. SSR 1/3/6/10/15 · SR 1/8/15/25/38 — 2026-09-07 신설) · `StarforceSet`(성공+유지+하락+파괴=100%, 유지는 CSV에 없이 유도. **`Level` 열은 목표 강수다**(`GetStarforceData(현재+1)`). **12강 도전까지는 실패해도 유지 / 13강 도전(12→13)부터 하락 / 16강 도전부터 파괴**, 안전 구간 없음. 10성 이상 성공률 30% 평탄(24·25강만 10%·5%). **파괴 = 스타포스 12강 복구**, 영웅은 안 사라짐. `ProtectStoneCost` = **파괴 방지**(파괴가 1강 하락으로 대체됨, 16강↑에서만 과금))
- **적**: `MonsterSet`(Boss 4행은 `HPMult 2.5` — 보스는 1마리라 파티 광역 딜이 낭비되므로 **단일 대상 딜로 길이를 계산할 것**) · `EnemySkillSet` · `EnemySkillEffectSet` · `EnemyEffectSet` · `BossSet`(P1/P2/P3 = 60k/60k/130k · 공격력 **2,000** · **`EndlessAtkGrowth` 1.194** = 무한 페이즈 턴당 복리, **상한 없음** · **`EndlessHP` 1억**) · `BossPartSet`
- **스테이지**: `StageSet`(그룹×층 구성 — 스테이지 번호가 아니라 **그룹 단위**라 그룹을 돌려쓰면 행이 안 늘어난다) · `GroupSet`(배경 + `StageSpan` = 그룹이 담당할 스테이지 수) · `StageCurveSet`(몬스터 스탯 곡선 공식 1행 — `HP=(1350+절대층×45)×1.002007^절대층` · `DEF=2+절대층×0.01`. **`MonsterManager`에 같은 값의 폴백이 하드코딩돼 있어 항상 같이 고쳐야 한다**)
- **퍼즐**: `PuzzleSet`
- **가챠**: `GachaSet`(가중치 풀) · `GachaConfigSet`(단가·연차·보장)

**퍼즐 원소 6종 = 무기타입**: `Greatsword` · `Bow` · `Dagger` · `Wand` · `Knuckle` · `Shield` (Shield는 데미지 대신 쉴드 부여)

### 데미지 공식

```
atk    = HeroDataManager:CalcATK(BaseATK, 성급, 스타포스) × 파티버프 × 자기버프 × 보조무기배율
         -- CalcATK = BaseATK × 2^(성급-1) × StarforceSet.AtkMult  (성급 ×16 · 스타포스 ×32.33 = 최대 ×517)
         -- ⚠️ 최종값이므로 스타포스를 따로 더하면 이중 계산이다
damage = atk × (AtkCoeff + 스타포스스킬계수)
       × (1 + (매치수-3) × MatchCoeff)
       × (1 + 강화타일수 × EnhanceCoeff)
       × (1 + (콤보-1)  × ComboCoeff)
       × 최종배율 × 적받는피해배율
       × 10/(10+DEF)          -- 고정피해(convert_to_fixed)면 생략
```

원소↔영웅 매칭: 주무기 ×1.0 / 보조무기 ×0.3 (다른 타입) 또는 ×0.5 (동일 타입)

### Game Flow

```
로비 → UIGameStart → GameManager:StartRun()
  → StageManager:StartStage(stage, groupId) → StartCurrentFloor()
  → BattleManager:StartBattle(battleData)
      ├ UIManager:ShowBattle()  (SceneTransition 연출)
      ├ MonsterManager:Init()   (CSV Slots "2,3" 파싱해 슬롯 배치)
      └ PuzzleLogic:StartGame() → TurnManager:StartBattle()
  → 타일 드래그 → TurnManager:EndPlayerTurn()
  → PuzzleLogic:StartResolve() → 캐스케이드(Remove→Drop→Fill)
  → BattleManager:ExecutePlayerAttack(matchResult)   -- 힐→버프→디버프→광역→단일
  → TurnManager:StartEnemyTurn() → ExecuteEnemyAttack()  -- 소환→버프→디버프→공격
  → 층 클리어 → 5층(보스) → StageManager:CompleteStage() → 로비
  → 패배: 파티 전멸 → 결과 연출 → 로비

보스: UIBossStart → SeasonBossManager:TryStartBossBattle(bossId) → 3페이즈
```

### Key Wiring Notes

- `PuzzleBoard.Puzzles[row][col]` is 1-indexed; Row = Y (top-down), Column = X (left-right).
- **`GetGridFromUIPosition` returns `Vector2(row, column)`** — 축 순서가 일반적 `(x,y)`와 반대.
- `MonsterManager`는 `battleGen` 카운터로 이전 전투의 애니메이션 콜백을 무효화한다.
- 파티 HP는 `PlayerManager._T.heroHP`(런 중)와 `HeroDataManager._T.partyCurrentHP`(층 이동 유지)에 **이중 보관**된다.
- **성급(차수) 필요 장수는 등급마다 다르다** — `StarLevelSet.csv`가 소스. **SSR 1/3/6/10/15장 · SR 1/8/15/25/38장 = ★1~★5**(총 획득 장수 기준). 🔴 SR은 특정 1명이 뽑힐 확률이 SSR의 **2.5배**(SR 10%÷4명 vs SSR 1%÷1명)라 그만큼 더 필요하다 — 이렇게 맞추면 **파티 5인 ★5 달성이 약 1,500뽑기로 동시에 떨어진다**. `CalcStarLevel(heroId, duplicates)`는 `AddHero`가 세는 **중복 카운터**(= 총 장수 − 1)를 받으므로 내부에서 +1 해 총 장수로 비교한다. **★0은 존재하지 않는다** — `BattleManager`가 `for tier = 0, starLevel - 1`로 스킬을 모으므로 ★0이면 스킬이 0개가 된다.
- **데미지 숫자는 스프라이트라 자릿수가 하드 캡이다** — `UIDamageNumber`의 `D0`~`D<n>` 칸 수가 곧 표시 한계이고, 넘으면 9로 채운 값이 나온다(2026-09-07 기준 **8칸**, 실측 최대 피해 1,067,948). 밸런스를 올렸다면 칸 수를 같이 볼 것. 칸을 늘릴 때 **컨테이너 `RectSize`는 건드리지 말 것** — 칸이 오른쪽 가장자리 기준이라 폭을 바꾸면 숫자 위치가 통째로 이동한다.
- 몬스터 스탯은 `StageCurveSet` 1행 + `MonsterManager:CalcBaseStats(절대층)`. **난이도를 볼 때는 HP가 아니라 유효 체력 `HP×(10+DEF)/10`으로 볼 것** — DEF는 데미지 공식의 `10/(10+DEF)`를 통해 조용히 2차항으로 작용한다.
- **밸런스는 추정하지 말고 실측한다.** 플레이어는 한 턴에 평균 **7~10콤보**를 내고, `comboExp`·`atkMult` 누적·5인 동시 발동이 겹쳐 곱연산이 4~5겹이다. 기준값: **8콤보 1턴 파티 총 피해 = ★1 0강 1,916 / ★5 25강 3,451,690**(실전력 ×1,801). 재려면 `_HeroDataManager._T.inventoryData`를 임시로 덮어쓰고 몬스터 HP를 1e12로 올린 뒤 `BattleManager:ExecutePlayerAttack`에 합성 `MatchResult`를 넣는다 — **전투 시작 직후에 쏘면 `_Queue`가 겹쳐 공격 스텝이 잘린다.**
- `_PuzzleLogic`은 `@Logic` — `@Logic`은 `OnMapEnter`/`OnMapLeave`를 받지 않는다. 맵별 셋업은 맵 엔티티 `@Component` 또는 `OnUpdate` 폴링으로.
- **가챠**: 서버 로직은 `HeroDataManager`에 있다(`RequestGachaPull` → `DoGachaPull` → `ReceiveGachaResult`).
  확률 SSR 1% / SR 10% / 메소 60% / 결정석 29%. **천장·보장 없음**(10연차도 순수 확률 — `GachaConfigSet.GuaranteedRewardType` 공란). UI는 `Shop.ui` + `GachaResultUI`로 완성됨(진입·확률·뽑기·결과 표시)
- **랭킹**: 자체 구현 완료(2026-09-05). 스테이지 = 최고 클리어 층, 시즌보스 = **누적 딜량**(무한 모드 전제). `boss_rank_s<N>`은 3일 시즌, `stage_rank`는 통산.
- **시즌보스 난이도**: 페이즈 HP는 약한 파티도 P3에 닿게 낮게 두고, 압박은 **무한 페이즈 공격력 복리**(`EndlessAtkGrowth`)로 만든다. `SeasonBossManager:NotifyTurnEnd`는 **`OnEnemyTurnEnd`보다 먼저** 불려야 진입 턴이 0턴으로 잡힌다.
  🔴 **무한 페이즈는 페이즈 반복이 아니다** — `_EnterEndless`가 `EndlessHP`(1억)를 **한 번만** 채우는 단일 구간이고, 기믹은 켜진 채 유지되며 공격력만 턴마다 오른다. `_EnterPhase`를 부르면 기믹이 재무장된다. 무한 구간은 **초과 딜을 버리지 않는다**(버리면 랭킹이 붕괴).
  🔴 **보스 밸런스를 잴 때는 `EnemySkillSet` 기믹 행을 먼저 읽을 것**: `shield_magic`(P2 진입, 플레이어 딜 0.5배)은 **판 끝까지 안 꺼지고**, `attack_all`(P3 진입, 보스 공격력 2배)도 **끝까지 유지**된다(2026-09-07 확정). 둘 다 파티 스탯만큼 결과를 흔든다.
- **경제 밸런스 기준**(2026-09-07 확정): 목표는 **200스테이지에서 0→22강 19일**(하락 경계를 12강으로 올린 뒤 30일에서 내려온 값을 **그대로 수용**하기로 결정 — 시급은 안 낮춘다). 23강↑는 운의 영역. 병목은 **메소**(19일 vs 결정석 17일). 강화 메소 비용 = 표의 1/10 · 방치보상 메소 `112800+(s-1)*290`/h · 결정석 `217+(s-1)*0.56`/h. 🔴 **방치보상 기울기가 완만한 건 의도** — 세우면 "수급을 위해 200스테이지까지 밀어야 하는" 강제가 되살아난다(1스테이지 45일 → 500스테이지 20일이 현재 폭). 🔴 **비용표와 시급은 한 쌍** — 한쪽만 고치면 목표가 즉시 깨진다. 🔴 강화 비용을 볼 때는 단가가 아니라 **레벨별 기대 방문 횟수 × 단가**로 볼 것(하락이 붙는 **13~15강**이 비용의 대부분). 🔴 확률이 고정이면 시도 횟수도 고정 — 0→22강 **40,866회**라 하루 2,151회 클릭이고, **일괄 강화(취소됨) 없이는 재화 기준 일수가 곧 도달 시점이 아니다**
- **미구현**: 상점은 M1에서 제외(로비 `상점` 버튼은 가챠 진입점으로 전용).
  **티켓 획득처가 0건**이다 — 신규 지급 3장이 전부이고 시즌보스 랭킹 보상으로 붙일 예정 → GDD Phase 3

### 답변 스타일
"최대한 한글을 사용해서 답변 어쩔 수 없는 경우 영어 사용"
