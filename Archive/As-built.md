# 퍼즐앤메이플 — As-built log

> 월드 구현 상태의 러닝 기록. 마일스톤을 가로질러 유지됨. ⚠️confirm = 서베이 추정, 사용자 확인 필요.
> AI/핸드오프용 참조 문서이며 사용자용 기획 문서가 아님.

## 현재 상태 (시스템별)

| 시스템 | 구현 형태 | 위치 (핵심 파일) | 비고 / 함정 |
|---|---|---|---|
| 게임 진입 · 런 시작 | `@Component` | `Component folder/Manager/GameManager.mlua` | 14줄. `CurrencyManager:GetMaxClearedStage()+1`로 진입 스테이지 결정 |
| 스테이지 진행 | `@Component` | `Manager/StageManager.mlua` | **현재 진행 축**. 5층 고정, `absoluteFloor = (stage-1)*5 + floor` |
| 전투 오케스트레이션 | `@Component` | `Manager/BattleManager.mlua` (1,173줄) | 프로젝트 심장. 스킬 스텝을 힐/버프/디버프/광역/단일로 분류 후 `_Queue:RunSequence`로 순차 연출 |
| 턴 사이클 | `@Component` | `Manager/TurnManager.mlua` | Player → Resolving → Enemy. `Win()`/`Lose()`는 빈 껍데기 |
| 퍼즐 보드 | `@Component` + `@Logic` | `Manager/PuzzleBoard.mlua` · `Manager/PuzzleController.mlua` · `Logic folder/PuzzleLogic.mlua` · `Objects/Puzzle.mlua` | 5×6. **`GetGridFromUIPosition`은 `Vector2(row, col)` 반환 — 축 순서가 일반적 (x,y)와 반대** |
| 파티 HP/쉴드 | `@Component` | `Manager/PlayerManager.mlua` | 슬롯 5개 + `HeroInfo` 자식 슬롯 이원 구조. HP는 `_T.heroHP`(런 중) ↔ `HeroDataManager._T.partyCurrentHP`(층 이동 유지) 이중 보관 |
| 몬스터 슬롯 | `@Component` | `Manager/MonsterManager.mlua` | `slotIdx` 오름차순 = 전방→후방. `battleGen` 카운터로 이전 전투의 애니 콜백 무효화 |
| 적 스킬 | `@Component` | `Manager/EnemySkillManager.mlua` | 트리거 3종: `every_n_turns` · `hp_threshold` · `phase_change`. 효과 타입: `damage_single`/`damage_all`/`damage_random` · `block_heal` · `summon_minion` · `buff_enemy_*` · `debuff_party_*` · **`heal_self`**(2026-09-06 신설, `MonsterManager:HealMonsterAt`). ⚠️ **`SkillCategory=passive`는 no-op** — `applySkillEffects`가 즉시 return한다. 패시브 성격이어도 `buff`로 넣을 것. **보스도 같은 파이프라인을 탄다**(F4) — `ExecuteEnemyAttack`이 보스를 "몬스터 1마리"로 위장한 합성 `info`로 만들어 넣는다. `hp_threshold`는 페이즈 전환마다 재무장(F3) |
| 시즌 보스 | `@Component` | `Manager/SeasonBossManager.mlua` | 3페이즈, 파츠 4개(`BossPart0~3`) 조립. **일일 3회 제한이 테스트용으로 우회 중** (L138) |
| 영웅 데이터 · 저장 | `@Logic` | `Logic folder/HeroDataManager.mlua` | DataStorage 키 `heroes`/`party`/`starforce`. 더티 플래그 + 60초 flush + `UserLeaveEvent` 즉시 저장. 성급은 **`CalcStarLevel(heroId, 중복 카운터)`** — **★1~★5, ★0 없음**. 🔴 **필요 장수는 등급마다 다르다**(`StarLevelSet`): SSR 총 1/3/6/10/15장 · SR 1/8/15/25/38장. 등급 조회는 클라 `heroBase` / 서버 `heroRarity` **양쪽에 따로** 있다(`LoadHeroBase`가 ClientOnly라서) |
| **가챠** | `@Logic` (`HeroDataManager` 내부) | `Logic folder/HeroDataManager.mlua` | `RequestGachaPull`(Server) → `DoGachaPull`(ServerOnly) → `ReceiveGachaResult`(Client). 서버 등급 색인은 `LoadHeroRarityIndex` — **`LoadHeroBase`가 ClientOnly라 서버엔 영웅 표가 없어서 별도로 필요**. 결과는 중첩 테이블이라 `_HttpService:JSONEncode` 사용(`TableToString`은 중첩을 조용히 버림). UI는 `Shop.ui` + `GachaResultUI`로 완성 |
| 재화 · 방치보상 · 클리어 보상 | `@Component` | `Manager/CurrencyManager.mlua` | DataStorage 키 `currency`. 10분 단위 수령, 최대 12h 누적. 신규 유저 초기 지급 골드 5000/티켓 3/강화석 10. **스테이지 클리어 보상은 2026-09-06 폐지** — 남은 것은 진행도 갱신 전용 `_AdvanceMaxStage`(진입 스테이지·방치보상 시급·스테이지 랭킹이 전부 이 값을 본다). 상한·단위는 `IdleCapHours`(12)·`IdleUnitSeconds`(600) 프로퍼티 |
| 인벤토리 UI | `@Logic` | `Logic folder/HeroInventoryUI.mlua` (483줄) | 영웅 카드를 `SpawnByModelId`로 동적 생성. `HeroCardUI`가 엔티티명 `HeroCard_<heroId>`를 파싱해 자기 데이터를 역조회하는 구조 |
| 패널 전환 | `@Component` | `Manager/UIManager.mlua` | `navStack` 기반. `SceneTransition` 4초 연출 + 보스 전용 전환 분기 |
| 이펙트 | `@Logic` | `Logic folder/Effect.mlua` | `Model_Effect` 스폰 → 재생 → 자동 Destroy. Sync/Async 두 계열 (`PlayEffect` vs `PlayEffectSync`) |
| 시퀀스 러너 | `@Logic` | `Logic folder/Queue.mlua` | `RunSequence(steps)` — `action()` 실행 후 `duration` 대기. `StopSequence()`로 전투 종료 시 중단 |
| 데이터 | UserDataSet ×18 | `RootDesk/MyDesk/Dataset/` | CSV가 소스 오브 트루스. Phase 1에서 `AugmentSet`·`NodeImageSet` 삭제, Phase 2에서 `GachaSet`·`GachaConfigSet` 신설. `StageRewardSet`은 2026-09-06 클리어 보상 폐지로 삭제, `BossRewardSet`·`StageCurveSet` 신설. **`StarLevelSet` 2026-09-07 신설**(등급별 성급 필요 장수) |
| UI | `.ui` ×12 | `ui/` | 반드시 `msw-ui-system` UIBuilder 경유. raw JSON 편집 금지. `Shop.ui`(가챠 화면)·`Enhance.ui`(강화 화면)는 2026-09-04 신설 — 루트 UIGroup·`DefaultShow=false`로 `HeroInventory.ui`와 동일 형태. ⚠️ **`AugmentGroup.ui`·`MapGroup.ui`는 아직 남아 있다** — Phase 1에서 컨트롤러 스크립트·모델·데이터셋만 지웠고 `.ui` 파일은 그대로다(고아 상태, 참조 0건) → Phase 6 정리 대상 |
| 영웅 카드 | `.model` `Model_HeroSlot` (루트 Sprite+Button, 자식 `Advance`/`Particle`) | `Model folder/` | `HeroInventoryUI`가 이 모델을 스폰. **`Model_HeroCard`는 구버전 디자인이며 현재 참조 0건** — 이름이 헷갈리니 주의 |
| 맵 | `.map` ×6 | `map/` | `Lobby` · `BattleMap`만 스크립트 참조됨. `Boss`/`Rest`/`Shop`/`UITest`는 참조 0건 ⚠️confirm |

## 상시 이슈 & 핸드오프 규칙

| 이슈 / 규칙 | 대응 | 횟수 | 최초 → 최근 |
|---|---|---|---|
| **스크립트 파일 삭제 직후 Maker 전체 저장 → 스크립트 컴포넌트 전량 소실** | `.mlua` 삭제와 Maker 저장을 같은 세션에서 연달아 하지 말 것. 삭제 전 `.ui`/`.model`/`common.gamelogic` 백업 필수. 증상: `componentNames: ""`, `@components: []` | 1 | 09-03 → 09-03 |
| **모델 UUID가 안 맞아 보이면 다른 모델의 ID인지 먼저 확인** | 전체 `.model`의 `EntryKey`를 대조한 뒤 판단. `bc86ca02`를 "없는 ID"로 오판해 구버전 모델로 바꾼 사고 있었음 | 1 | 09-04 → 09-04 |
| **`@EventSender("Self")` 핸들러는 `ButtonComponent`와 같은 엔티티에 있어야 함** | 스크립트 부착 위치는 이름이 아니라 `ButtonComponent` 보유 엔티티 기준으로 정할 것. 컨테이너(부모)에 붙이면 클릭이 안 됨 | 1 | 09-03 → 09-03 |
| **UIBuilder는 새로 만드는 모든 스프라이트에 어두운 반투명 기본 스킨을 입힌다** | `Color = RGBA(26,26,26,60)`. 이 프로젝트의 기존 스프라이트는 Color 필드가 없어(=흰색) 원본대로 보이므로, 새 스프라이트에는 **반드시 `color:"#FFFFFF", alpha:1.0`**(버튼은 `bg_color`)을 넘길 것. 안 넘기면 배경·아이콘이 통째로 안 보이는 것처럼 된다 | 1 | 09-04 → 09-04 |
| **`PreserveSprite=AspectOnly(1)`는 아이콘을 부모 밖으로 밀어낸다** | 슬롯 안에 아이콘을 넣을 때 쓰면 위로 삐져나감. `None(0)`으로 두고 **런타임에 `RectSize`로 종류별 크기를 지정**하는 편이 안전(영웅 전신 이미지는 여백이 많아 재화 아이콘보다 크게 잡아야 비슷해 보임) | 1 | 09-04 → 09-04 |
| **`ui_lint`가 node를 segfault시킴** | 크래시는 **쓰기 성공 뒤 lint 단계**에서 발생 — 파일 자체는 정상적으로 써진다(untouched 원본으로도 재현되므로 특정 변경 탓이 아님). `write(path, { lint: false })`로 우회하고, 백업 대비 전 엔티티 `componentNames` diff로 구조 검증할 것. `node -e` 인라인은 더 불안정하니 **스크립트 파일로 실행**. ⚠️ **`bind`는 lint 통과 후에 실행되는 마지막 단계라, lint가 죽으면 `.mlua` UUID 주입이 통째로 건너뛰어진다**(`.ui`는 멀쩡히 써져서 성공처럼 보인다 — 2026-09-06 `Mail.ui`에서 재현). `Lobby.ui` 외 새 파일에서도 발생 | 2 | 09-04 → 09-06 |
| **`mouse_input`은 `ButtonComponent` UI 버튼을 클릭하지 못함** | 도구 명시 제한(엔진 이벤트 시스템 경유 UI는 대상 외). 버튼 배선 검증은 `entity:SendEvent(ButtonClickEvent())`를 클라 컨텍스트에서 보내 핸들러 발화 로그로 확인할 것 | 1 | 09-04 → 09-04 |
| **매니저에 새 메서드를 추가하면 호출부에 `LIA-1115`가 남는다** | `BattleManager` → `MonsterManager:HealMonsterAt`에서 재현(2026-09-06). **`os.utime`/`fs.utimesSync` touch 로는 안 사라진다 — 파일 *내용*을 바꿔야 재임포트된다**(끝 개행 토글로 충분. 2026-09-07 정정: 이 방법으로 LEA-1102/1103·LIA-1115 가 전부 해소됐다) — 기존 메서드(`SummonMinions` 등)는 같은 형태인데 경고가 없으니 **LSP 인덱스가 새 심볼을 못 따라오는 것**이다. Info 등급이라 빌드는 통과하고 **런타임 호출은 정상**이다. 판정은 로그로 할 것(여기서는 `자가 회복 +239` 확인) | 1 | 09-06 → 09-06 |
| **`*EffectSet` 계열은 `skillId`당 레코드 1개다 — 배열이 아니다** | `EnemyEffectSet` / `HeroSkillVisualSet`은 `_LoadData`에서 `effectData[skillId] = { … }`로 **덮어쓰기** 저장한다(`skillEffects`만 `table.insert` 배열). `ipairs`로 돌면 **정수 키가 없어 조용히 0회** — 에러도 경고도 없다. 필드명도 `effectRUID`가 아니라 `monsterEffectRUID` / `heroEffectRUID` / `projectileEffectRUID` / `screenEffectRUID` 4종 | 1 | 09-06 → 09-06 |
| 🔴 **`OnEndPlay`에서 DataStorage에 저장하면 안 된다** | MSW 공식 가이드: `OnEndPlay`에서 `SetAndWait`/`SetAsync`를 호출하면 **유저 재입장 시 이전 저장이 제대로 로드되지 않을 수 있다**(잘못된 데이터를 받는다). 저장 자리는 **`UserLeaveEvent` 핸들러**다. `OnEndPlay`는 타이머 정리만. 이 프로젝트는 3곳(`CurrencyManager`·`HeroDataManager`·`RankingManager`)이 이 패턴이었고 2026-09-06에 전부 옮겼다 | 1 | 09-06 → 09-06 |
| **`@EventSender("Service")`는 두 번째 인자가 필수** | 서비스 **타입명**을 함께 줘야 한다 — `@EventSender("Service", "UserService")`. 빠뜨리면 `[LEA-3005] InvalidArgument : 'EventSender' Attribute의 인수가 부족합니다`. `"Entity"`/`"Model"`은 두 번째 인자가 **Entity Id**, `"Self"`/`"LocalPlayer"`는 두 번째 인자 없음 | 1 | 09-06 → 09-06 |
| 🔴 **`io.open(p, "w")`는 여는 순간 파일을 0바이트로 만든다** | 파이썬으로 문서를 통째로 다시 쓰다가 `write()`에서 예외가 나면 **원본이 이미 날아간 뒤**다. 2026-09-07 `Archive/As-built.md`가 실제로 0바이트가 됐다(원인: 소스에 `🔴` 같은 **서러게이트 이스케이프**를 써서 `UnicodeEncodeError: surrogates not allowed`). 대책 ① 문서 편집은 **Edit 도구**를 쓸 것 ② 굳이 파이썬을 쓴다면 이모지는 **이스케이프가 아니라 문자 그대로** 넣고, 새 내용을 임시 파일에 쓴 뒤 `os.replace`로 갈아끼울 것 ③ 이 저장소는 git이 아니라 **되돌릴 수단이 없다** | 1 | 09-07 → 09-07 |
| **계정 리소스 스토리지에 올린 스프라이트 RUID는 월드에서 안 그려진다** | `asset_create_account_resource_storage_item`으로 올린 RUID를 `ImageRUID`에 넣으면 **경고 한 줄 없이 아무것도 렌더되지 않는다**(엔티티·좌표·정렬은 전부 정상). 같은 엔티티에 공개 검색 RUID를 넣으면 즉시 보이므로 그것으로 판별할 것. 커스텀 스프라이트가 꼭 필요하면 Maker에서 월드에 직접 등록해야 한다. 2026-09-07 드래그 타이머 링에서 재현 | 1 | 09-07 → 09-07 |
| **`DataRef`는 `tostring()`으로 비교할 수 없다** | `ImageRUID` 등 `DataRef` 프로퍼티를 `tostring()` 하면 값과 무관하게 **항상 `"MOD.Core.MODDataRef"`**(타입 이름)가 나온다. 서로 다른 RUID도 같아 보여 "안 바뀌었다"고 오판하게 된다. 실제 값은 **`ref.DataId`**(string)로 읽을 것 | 1 | 09-06 → 09-06 |
| **`_Effect:PlayEffect`는 비동기다** | 내부가 `_ResourceService:PreloadAsync` 콜백에서 스폰하므로 **같은 프레임에 자식 수가 안 늘어난다**(실측 +0.6s). 부착 검증은 반드시 타이머로 폴링할 것 — 호출 직후 1회만 세면 "이펙트 없음"으로 오판한다 | 1 | 09-06 → 09-06 |
| **매니저는 `@Component`라 `_XxxManager` 전역이 없음** | `_CurrencyManager`처럼 쓰면 `LEA-2007`(nil 인덱싱). `@Logic`(`_HeroDataManager`·`_PuzzleLogic` 등)만 전역이고, `@Component` 매니저는 `_HeroDataManager.CurrencyManager`처럼 프로퍼티를 타고 갈 것 | 1 | 09-04 → 09-04 |
| **`maker_execute_script`는 큰 루프에서 조용히 멈추고 다음 스크립트까지 막음** | ≈10,000회에서 재현. 증상은 "출력이 안 나옴" + 이후 스크립트 무반응. **빈 스크립트를 보내 중단**시킨 뒤 N을 수천 이하로 줄일 것 | 1 | 09-04 → 09-04 |
| **버튼에 글루 스크립트를 붙이면 `ButtonComponent`가 교체될 수 있다** | 빌더 `script()`로 커스텀 컴포넌트를 붙인 뒤 `ButtonComponent`가 사라진 사례(로비 랭킹 버튼). 클릭이 아예 안 먹는데 **`SendEvent(ButtonClickEvent())` 검증은 통과**하므로 놓치기 쉽다 — 부착 후 `componentNames`를 직접 확인할 것 | 1 | 09-05 → 09-05 |
| **UI 텍스트 컴포넌트가 두 종류(`TextGUIRendererComponent` / `TextComponent`)** | 한 화면 안에 섞여 있다. 엉뚱한 쪽을 읽으면 `nil`이라 `if c ~= nil` 가드에 걸려 **오류 없이 표시만 죽는다**(전투 HUD 영웅 이름이 `모험가`로 고정된 원인). 둘 다 시도하고, 없으면 `log_warning`으로 드러낼 것. 타입 확인은 빌더 `componentNames` | 1 | 09-05 → 09-05 |
| **`DefaultShow=false` 그룹의 자식 엔티티는 `OnBeginPlay` 시점에 안 잡힘** | 패널 내부 버튼을 그때 연결하면 전부 `nil`이 된다(에러 없이 배선만 죽음). 그룹을 켠 **직후**에 한 번만 연결하는 지연 바인딩(`_EnsureBound`)으로 처리할 것 | 1 | 09-04 → 09-04 |
| **`.ui` 재빌드는 루트 UUID를 새로 발급한다** | `.mlua`에 하드코딩한 패널 UUID가 조용히 끊긴다(증상: `Open()`이 아무 일 없이 return). 빌더 `write(..., { bind: { mlua, props: {...} } })`로 매번 자동 주입할 것 | 1 | 09-04 → 09-04 |
| **`GetChildByName`은 경로를 못 받고, 이름만 주면 동명 엔티티를 잘못 잡는다** | `"A/B"`를 통째로 넘기면 항상 `nil`. 재귀 검색(`true`)은 트리 어디의 `Label`이든 먼저 걸린 것을 반환한다. `string.gmatch(path,"[^/]+")`로 구간을 잘라 단계별로 내려갈 것 (`string.find`는 두 값을 반환해 `LWA-1111`) | 1 | 09-04 → 09-04 |
| **UIBuilder `top-center` 앵커는 아래로 내릴 때 y가 음수** | 양수를 주면 패널 밖 위로 올라간다. `middle-right`도 안쪽으로 넣으려면 x가 음수 | 1 | 09-04 → 09-04 |
| **빌더 전체 재빌드는 사용자가 Maker에서 손본 레이아웃을 통째로 날린다** | 2026-09-04 `Enhance` 화면에서 실제로 발생(사용자 저장 19:05 → 재빌드 19:11에 소실, 복구 불가). 대책 3가지: ① 빌드 스크립트가 쓰기 전에 타임스탬프 백업(`scratchpad/backup/`)을 남기게 할 것 ② 레이아웃이 자리 잡은 뒤에는 **전체 재빌드 금지, `patch`로 부분 수정** ③ 재빌드 전 `scratchpad/snap_ui.py`로 `{U}` 폴더 전체 스냅샷 | 1 | 09-04 → 09-04 |
| **Maker는 UI 그룹 썸네일을 남긴다 — 사고 후 "무엇이 있었는지" 확인용** | `~/AppData/LocalLow/nexon/MapleStory Worlds/Screenshots/<worldId>/Thumbnails/UIGroup/<루트UUID>` (확장자 없는 PNG). 저장 시각에 갱신되므로 소실 직전 화면을 볼 수 있다. **다만 190×110 수준이라 좌표 복원은 불가** — 구조 확인까지만 | 1 | 09-04 → 09-04 |
| **파일을 되돌렸는데 Maker가 계속 옛 내용을 쓴다 → mtime을 확인할 것** | `refresh_workspace`는 **수정 시각으로 재임포트를 판단**한다. `shutil.copy2`(또는 원본 타임스탬프를 보존하는 복사)로 백업을 덮어쓰면 mtime이 과거로 돌아가 Maker가 변경을 무시한다(증상: 되돌렸는데 play에서 옛 구조 그대로). 복사 후 `os.utime(dst, None)`로 현재 시각으로 올릴 것 | 1 | 09-04 → 09-04 |
| `.ui` 파일은 직접 수정 금지 | UIBuilder 경유, 또는 Maker 에디터에서 사용자가 직접 편집 (복구 등 대량 작업은 사용자 승인 후 예외) | 2 | 09-03 → 09-04 |
| 컴포넌트 접근은 `entity.ComponentType` 형식 | `GetComponent()` 사용 시 `LIA-1114` 경고 | 1 | 09-03 → 09-03 |
| `handler`는 mLua 예약어 | 변수명은 `evtHandler` 등으로 회피 (`LEA-1001`) | 1 | 09-03 → 09-03 |
| Entity/Component 프로퍼티 선언 시 기본값 필수 | `property Entity X = ""` — 없으면 빌드 오류 | 1 | 09-03 → 09-03 |

## Log

### 2026-09-03 Seed — 툴킷 도입 시 서베이

브라운필드 서베이 결과. 스크립트 49 · 데이터셋 17 · 모델 7 · 맵 6 · UI 10.

**동작 확인된 것**: 매치3 퍼즐 → 원소↔무기타입 매칭 → 영웅 스킬 발동 → 적 턴 → 층/스테이지 클리어 → 재화 저장까지 전 루프가 이어짐. 시즌보스 3페이즈, 스타포스 25강, 방치보상까지 구현 완료.

**미연결로 확인된 것 (호출처 grep 0건)**:
- 증강 시스템 전체 (`AugmentManager:ShowAugmentSelect`/`GetBonus`/`ResetRun`) — 데이터셋·UI까지 있으나 전투에 미연결. `ApplyAugment`도 `hp_bonus`만 반영
- 로그라이트 노드맵 — `MapManager:CompleteNode` 호출처 0건이라 노드 진행 불가. 메인 흐름이 `StageManager`로 이전되며 우회됨
- 구 전투 잔재 `Objects/Player.mlua` · `Objects/Monster.mlua` — 갱신 대상인 `/ui/BattleGroup/UserState`를 `BattleManager:StartBattle`이 명시적으로 비활성화
- `MonsterManager` 5종 · `EnemySkillManager` 2종 · `BattleManager:RequestReturnToLobby` · `PlayerManager:AddShieldToHeroes` · `Effect:PlayAoeHitEffect` · `UIToast:ShowMessage`

**출시를 막는 구멍**: 영웅 획득 정상 경로가 없음. 디버그 `RequestDebugAddAllHeroes`만 존재하고 `btnGacha`는 빈 문자열, `ticket` 재화는 소비처 0건.

**버그**: `CalcATK`가 `starLevel`을 무시하고 `BaseATK` 반환 → 성급이 ATK에 미반영, 인벤토리 "다음 강화" 표기가 항상 동일. `GetSkillCoeffBonus`는 정의만 되고 미호출 → 스타포스 스킬 계수 보너스 미적용.

**문서 불일치**: `CLAUDE.md`가 초기 프로토타입 시점 기준 — 맵 이름·원소 4종·데미지 공식·매니저 구성이 전부 현재와 다름.

**사용자 확정 (동일자)**: 증강 시스템 제거, 노드맵 폐기. 이후 확장 → v1.0 출시 목표.

### 2026-09-03~04 사고 — 스크립트 컴포넌트 전량 소실 및 복구

**무슨 일**: Phase 1에서 스크립트 파일 15개를 삭제한 직후 Maker가 전체 재직렬화(19:17:34)를 하면서, `common.gamelogic`의 매니저 11개 · `.ui` 8개(엔티티 473) · `.model` 4개의 **`script.*` 컴포넌트가 전부 사라짐**. `@Logic`은 무사, `@Component`만 전멸. 빌드 로그에는 Error가 0건이라 한동안 원인을 못 찾았고, 증상은 "심볼/엔티티를 찾을 수 없음"과 UI 무반응으로 나타남.

**왜 못 잡았나**: 빌드 콘솔의 LIA-1113/1114는 원래부터 있던 정적분석 노이즈였고, 매니저 `OnBeginPlay` 로그가 **0건**인 것이 유일한 실질 신호였음. `common.gamelogic`의 before/after grep이 결정적 증거가 됨.

**복구**: 사용자가 Maker에서 매니저 11개 재부착 → 저장 후에도 유지되는 것 확인(= 저장이 더 이상 파괴적이지 않음) → 나머지 `.ui` 19곳 + `.model` 1곳은 UIBuilder/ModelBuilder로 부착. 부착 위치는 `.mlua`의 UUID 프로퍼티에서 역산.

**복구 중 발생시킨 2차 사고**: `heroCardModelId`의 `bc86ca02`를 없는 ID로 오판해 `Model_HeroCard`(구버전)로 변경 → 구버전 카드가 렌더링됨. 실제로는 `Model_HeroSlot`의 ID였고 원래 값이 정답. 되돌림.

**부수 변경**: `HeroInventoryUI:UpdateGrid`가 모델을 `_EntryService:GetModelIdByName("Model_HeroSlot")`로 조회하고 프로퍼티를 폴백으로 쓰도록 변경(식별자 재생성에 대한 내성).

**미해결로 확인된 것**: `btnEnhance`(스타포스 강화 버튼)는 연결 스크립트가 프로젝트에 아예 없음 — `HeroInventoryUI:OnClickEnhance()` 호출처 0건. 사고와 무관한 기존 미구현. → GDD Phase 2 항목으로 등록.

**미사용 자산**: `Model_HeroCard`(구버전 카드 디자인, 자식 `imgIcon`/`txtName`/`txtRarity`/`txtStars`)는 이제 어디서도 안 쓰임 → Phase 6 정리 대상.

### 2026-09-04 Phase 1 완료

증강 시스템과 로그라이트 노드맵을 폐기하고, 성장 공식 버그를 바로잡았다. 자산 15개(스크립트 11 · 모델 2 · 데이터셋 2쌍) + 미사용 메서드 11개 제거. 매니저는 12 → **10개**, 데이터셋 17 → **15종**.

> ⚠️ **2026-09-04 정정**: 여기 원래 "`.ui` 10 → 8개"라고 적혀 있었으나 **사실이 아니다.** `AugmentGroup.ui`·`MapGroup.ui`는 삭제되지 않았고 지금도 `ui/`에 있다(컨트롤러 스크립트가 없어 참조 0건인 고아 파일). `.ui`는 여전히 **10개**다 → Phase 6 정리 대상.

**성장 공식 수정**: `CalcATK`가 성급을 무시하던 버그 → 성급당 +10%(HP 보정과 동일 규칙). 검증 `star0=1000 / star1=1100 / star5=1500`. 방치보상 `_RewardRateForStage(stage)` 신설로 서버 지급식과 클라 표시식을 한 함수로 통일.

**순서 의존성 2건** (다음에 유사 작업 시 재현될 수 있음): ① 증강 전용 API는 `AugmentManager` 삭제 **이후**에 제거해야 함(유일 호출처) ② `UIDifficulty`/`UIMap`은 `UIManager.ShowMap`/`MapControl` 제거와 **동시에** 지워야 빌드 유지.

**중간에 스크립트 컴포넌트 전량 소실 사고** 발생 — 위 항목 참조. 복구 완료 후 사용자가 스테이지 클리어까지 관통 확인.

**이관**: 스타포스 스킬 계수 실측 검증 → Phase 5(밸런스). 배선·함수 검증은 완료 상태.

### 2026-09-04 Phase 2 — 가챠 서버·데이터 완료 (A블록)

가챠를 **서버 권위**로 구현하고 스테이지 클리어 보상을 붙였다. UI는 아직 없어 실제 호출처는 0건 — 여기서 사용자 Maker 작업(B1)으로 넘어간다.

**신설 데이터셋 3종**: `GachaSet`(가중치 풀 4행·합 100) · `GachaConfigSet`(단가·연차·보장) · `StageRewardSet`(클리어 보상 공식 1행). 셋 다 25행짜리 표 대신 **공식 설정 1행 + 코드 계산** 방식 — 선형 증가라 표로 둘 이유가 없었다.

**확률**: SSR 영웅 1% / SR 영웅 10% / 메소 60%(5000) / 결정석 29%(20). **천장·보장 모두 없다** — 10연차도 순수 확률(2026-09-04 사용자 결정으로 보장 폐지). 끄는 방식은 `GachaConfigSet.GuaranteedRewardType`를 **공란**으로 두는 것이고, `RollGuaranteedHero`와 분기 코드는 남아 있어 값만 `hero`로 되돌리면 부활한다.

**검증 (전부 `maker_execute_script` server_main)**:
- `RollGachaOnce` 300회 분포 — hero 40 / meso 171 / stone 89, 최대 1.3σ
- `RandomIntegerRange(1,100)` 균일성 3000회 — **범위 1~100 양끝 포함 확인**, 구간별 최대 1.7σ. (중간에 영웅률이 높아 보여 편향을 의심했으나 근거 없음. 누적 385/3300 = 11.7% vs 기대 11%)
- 10연차 100회 — 영웅0 18회 발생 → (당시) 보장 18/18 성공. **보장은 이후 폐지됨**
- **전체 경로 실행** — 티켓 13→3(-10) · 골드 +10000 · 결정석 +120이 결과 10건과 정확히 일치. 클라 `ReceiveGachaResult`가 JSON 10건 복원. *티켓은 테스트 전 10장 지급해 순변동 0으로 맞춤*
- 스테이지 보상 4케이스 — 재클리어 0/0 · 최초클리어 1250/50 · 반복요청 0/0 · 2단계 건너뛰기 2950/118. 테스트 후 유저 상태 원복

**설계 판단 — 클리어 보상은 최초 1회만**: `CompleteStage`가 ClientOnly라 클리어마다 지급하면 같은 요청 반복으로 재화를 무한 증식할 수 있다. 기존 `RequestUpdateMaxStage`(단조 증가)에 지급을 접붙여 서버 추가 검증 없이 파밍을 차단했다. 대가로 **재도전은 보상 0**이며, 상시 수입은 방치보상이 담당한다.

**실전 함정 2건**:
- `_CurrencyManager` 같은 전역은 없다. `CurrencyManager`는 `@Component`라 `_HeroDataManager.CurrencyManager`처럼 참조를 타고 가야 한다 (`LEA-2007`)
- `maker_execute_script`는 루프가 크면(≈10,000회) 조용히 멈추고 **다음 스크립트도 막는다**. 빈 스크립트를 보내 중단시킨 뒤 N을 줄일 것. 체감 한계는 수천 회

**막힌 곳**: 티켓 획득처가 0건이다. 초기 지급 3장이 전부라 가챠는 동작해도 3회면 멈춘다 → Phase 3(시즌보스 랭킹 보상).

### 2026-09-04 가챠 진입점 — 로비 상점 버튼 → `Shop` 그룹

사용자가 Maker에서 `ui/Shop.ui`를 신설했고(루트 UIGroup `Shop` · `DefaultShow=false` · `GroupOrder=8`, `HeroInventory.ui`와 같은 형태), 여기에 진입/이탈 배선을 붙였다.

**부착**: `script.UIShopOpen` → `/ui/Lobby/RightAside/Shop`(이미 `ButtonComponent` 보유) · `script.UIShopClose` → `/ui/Shop/BackGround/BackButton`. 둘 다 `ShopPanel`(= `/ui/Shop` 루트 `b02c279e…`)의 `Enable`만 건드린다.

**navStack을 쓰지 않은 이유**: Stage 패널과 같은 토글형 오버레이라 `UIManager:GoBack()` 경로에 올라가지 않는다. Phase 1에서 스테이지 뒤로가기가 이 이유로 안 먹은 전례가 있어 같은 실수를 피했다.

**검증**: 시작 `Enable=false` → 상점버튼 `ButtonClickEvent` → `[UIShopOpen]` 발화 + `Enable=true` + 화면 렌더 확인 → BackButton → `[UIShopClose]` 발화 + `Enable=false`. 빌드 0건.

**작업 중 부딪힌 도구 문제 2건** (상시 이슈 표에 등록): `ui_lint`의 `Lobby.ui` segfault, `mouse_input`의 UI 버튼 클릭 불가.

**뽑기 버튼 배선(같은 날 이어서)**: `UIGachaPull1`·`UIGachaPull10`을 `1Gatcha`·`10Gatcha`에 부착. 검증은 티켓 11장을 선지급하고 전부 소모해 **순변동 0**으로 맞춘 뒤 진행했다 — 1회 `티켓-1 메소+5000`, 10연차 `티켓-10 영웅1 메소+30000 결정석+60`, 최종 티켓 3(시작값). `.ui`의 script 컴포넌트는 `{"@type","Enable"}`만 담고 프로퍼티 값은 전부 `.mlua` 기본값에서 오므로, 버튼마다 카운트가 다른 경우 **인스턴스 오버라이드 대신 스크립트를 분리**했다.

**남은 것**: 결과 팝업이 없다(사용자 확인). 클라 결과 데이터는 `GetLastGachaResult()`로 이미 꺼낼 수 있으므로 팝업만 생기면 B3는 배선 작업이다. `Gatcha` 버튼은 현재 배선 대상 아님.

### 2026-09-04 가챠 결과·확률 UI 구성

사용자가 Maker에서 껍데기(`ShowResult(1Gatcha)` 900×1000 · `ShowResult(10Gatcha)` 1600×1050 · `ShowProbability` 버튼)를 만들고, 내부 구성을 AI가 UIBuilder로 채웠다. `Shop.ui`는 20 → **94 엔티티**.

**스타일 판단**: 프로젝트에 이미 고유 비주얼(메이플 골드 프레임)이 있으므로 템플릿 번들 RUID를 쓰지 않고 **기존 프로젝트 RUID를 재사용**했다 — 슬롯 칩은 로비 헤더 재화 칩(`3acfe09296…`, Sliced), 확인 버튼은 "1회 소환"의 파란 버튼(`3143e935b4…`), 창 배경은 `ShowResult(1Gatcha)` 배경(`9ba0a01311…`), 메소/결정석 아이콘은 헤더 것 그대로.

**구성**:
- **1회**: 큰 일러스트(`PortraitRUID`) + 이름 + `등급 · 신규/중복(성급 +1)` + 확인
- **10연차**: 5×2 슬롯 그리드(각 아이콘/이름/부가정보) + 요약 `영웅 n명 · 메소 · 결정석` + 확인
- **확률 팝업**: 6행(미사용 행 자동 숨김). **하드코딩 없이 `GachaSet`/`GachaConfigSet`을 런타임에 읽는다** → CSV를 고치면 화면도 따라간다
- 영웅·메소·결정석을 **같은 슬롯 형태로 통일**해 재화가 섞여도 어색하지 않게 했고, 참고 이미지의 NEW 폭죽·파티클 같은 화려한 연출은 넣지 않았다(사용자 요청)
- **영웅 이미지는 `ThumnailRUID`**(= 로비 `Center/HeroThumnail`과 같은 전신 아트). `PortraitRUID`는 쓰지 않는다. 로비와 동일하게 **`PreserveSprite=NativeSize` + `LocalScale` 축소 + CSV의 `LocalX/LocalY` 오프셋**으로 그린다 — `RectSize`로 늘리면 전신 아트가 찌그러진다. 배율은 단일 0.48 · 10연차 슬롯 0.17 · 확률행 0.07(로비는 0.6), 오프셋은 `배율/0.6`으로 환산한다. 재화 아이콘은 정사각이라 기존대로 `RectSize`로 지정. **영웅 아트 원본 세로는 약 1100px** — 배율을 정할 때 기준으로 삼을 것
- **10연차 그리드 좌표 주의 2가지**: ① 패널 아트(1600×1050)의 실제 금테 안쪽은 약 **x ±620**이라 폭 250·간격 280 배치는 양끝 열이 프레임을 넘어갔다 → 슬롯 200×290 · 열 `[-470,-235,0,235,470]`로 축소. ② 패널 중심이 `y=-55`라 **패널 하단 40px은 화면 밖**이다 — 확인 버튼을 -455에 두면 잘린다(-390이 상한). 행은 `[200,-110]`, 요약 -300, 버튼 -390
- **신규/중복 표기는 넣지 않는다**(2026-09-04 사용자 결정). 부가정보는 등급만 노출. 서버가 보내는 `isNew`는 그대로 남겨 두었다

**컨트롤러**: `GachaResultUI`(@Logic) 신설. UUID를 70개 주입하는 대신 **루트 3개만 프로퍼티로 받고 자식은 `GetChildByName`으로 조회**한다. 버튼은 `@EventSender` 글루 스크립트 대신 Logic에서 `ConnectEvent`로 묶고 `OnEndPlay`에서 해제(글루 파일 4개를 아낌). `HeroDataManager:ReceiveGachaResult`가 성공 시 `_GachaResultUI:Show(items)`를 호출하고, 실패 시 `_UIToast:ShowMessage`로 사유를 알린다(티켓 부족).

**시행착오 2건** (상시 이슈 표에 등록): 빌더 기본 스킨 때문에 첫 빌드에서 배경·칩·아이콘이 전부 안 보였고, 이를 고친 뒤 `PreserveSprite=AspectOnly`가 아이콘을 슬롯 밖으로 밀어냈다.

**검증**: 빌드 0건 · 구조 diff(원본 20개 손실 0 · ID변경 0 · desync 0) · 확률 팝업 수치 일치 · 실제 1회/10연차 뽑기(티켓 선지급 후 전액 소모, 순변동 0) · 합성 데이터로 영웅 5명 섞인 최악 배치 미리보기 · 티켓 부족 경로(차감 없음 + ToastGroup `false→true`). 토스트는 2.1초 페이드라 스크린샷에는 안 잡히므로 로그로 확인했다.

### 2026-09-04 버그 — 신규 영웅이 공격기 0개 (성급 off-by-one)

**증상**: 가챠로 처음 얻은 영웅이 전투에서 아무 스킬도 쓰지 않는다. 사용자가 신규 계정으로 플레이하다 발견했다.

**원인**: `CalcStarLevel`이 받는 값은 `AddHero`가 세는 **중복 카운터**(첫 획득 = 0)인데, 임계값이 `>= 1`부터 ★1이라 **첫 획득 영웅은 ★0**이 됐다. 그리고 `BattleManager`가 스킬을 `for tier = 0, hero.starLevel - 1`로 모으기 때문에 ★0이면 `for tier = 0, -1` → **루프가 한 번도 돌지 않는다**. `HeroSkillSet.csv`에 StarLevel 0(기본 공격기)이 멀쩡히 있는데도 쓰이지 않았다.

**원래 의도**: `CalcStarLevel`의 주석이 `1성: 1명 / 2성: 3명 / 3성: 6명 / 4성: 10명 / 5성: 15명`(삼각수, 단위가 **명**)이었다. 즉 **총 획득 장수** 기준으로 설계했는데 구현이 중복 카운터와 비교하며 한 칸 밀린 것.

**수정(A안)**: 임계값을 1씩 낮추고 하한을 ★1로 뒀다 — `14/9/5/2/else 1`. 총 장수 기준 1/3/6/10/15장 = ★1~★5가 되어 주석과 일치하고, ★5에서 CSV의 스킬 5개가 전부 열린다. **★0은 더 이상 존재하지 않는다.**

**연쇄 정리**: `HeroInventoryUI`의 `if starLevel < 1 then starLevel = 1 end`을 제거했다 — ★0 버그를 카드 표시에서만 가리던 임시 보정이라, 근본 원인이 사라지며 불필요해졌다.

**안전성**: 호출부 7곳을 모두 확인했고 전부 보유/파티 영웅에만 적용된다(미보유는 `continue`하거나 `GetHeroStar`가 −1로 먼저 거른다). 성급은 저장값이 아니라 중복 수에서 매번 계산하므로 **기존 세이브 마이그레이션이 필요 없다**.

**검증**: 경계값 11개(중복 0·1·2·3·4·5·8·9·13·14·20) → 총 1장★1 / 3장★2 / 6장★3 / 10장★4 / 15장★5 정확. `BattleManager`의 수집 루프를 그대로 재현해 ★1=1개(슬래시 블러스트) … ★5=5개 확인.

### 2026-09-04 강화 전용 패널 (`Enhance`) 신설

**의도**: 인벤토리 상세 안의 작은 강화 버튼과 별개로, `Docs/강화패널.png`(무기 강화 화면) 형태의 **독립 강화 화면**을 만든다. 로비 `Enchant` 버튼에서 연다.

**구성**: `ui/Enhance.ui`(53엔티티, `GroupOrder=9`, `DefaultShow=false`) + `EnhanceUI.mlua`(@Logic). 좌 = 강화 대상(**`PortraitRUID`** 상반신 + 보유 영웅 5칸 페이지 + `강화 대상 변경` 버튼) · 중앙 = **`ThumnailRUID`** 전신 + 성급 게이지 + `n강 → n+1강` · 우 = 공격력/체력 변화 + 성공·실패·하락 확률 · 하단 = 비용 · 강화 보호 토글 · 강화하기.

- **두 RUID를 용도별로 나눠 쓴다**(2026-09-04 사용자 결정): 좌측 대상 프레임은 `PortraitRUID`(상반신, `PreserveSprite=None`으로 `RectSize`에 맞춤), 중앙 큰 그림은 `ThumnailRUID`(전신, `NativeSize` + `LocalScale` + CSV `LocalX/LocalY`). 전신 아트를 `RectSize`로 늘리면 찌그러지므로 헬퍼를 `_SetPortrait` / `_SetThumbnail` 둘로 분리했다.
- **전투력은 넣지 않는다**(2026-09-04 사용자 결정). 한때 넣었던 `HeroDataManager:CalcCombatPower`도 함께 제거했다.

**왜 @Logic인가** (사용자 질문에 대한 답을 남김): ① UI 상태는 맵을 넘어가도 살아 있어야 한다 ② `HeroDataManager:ReceiveStarforceResult`(@Logic)가 이름(`_EnhanceUI`)으로 호출해야 한다 ③ **`DefaultShow=false` 그룹 안의 `@Component`는 `OnBeginPlay`를 아예 못 받는다.**

**배선 함정 2건**:
- 패널 내부 버튼 8개가 `OnBeginPlay`에서 전부 `nil`이었다 — 그룹이 꺼져 있어 자식 엔티티가 잡히지 않는다. `_EnsureBound()`를 첫 `Open()`(패널 `Enable=true` 직후)에서 호출하는 지연 연결로 해결.
- `.ui`를 재빌드하면 **루트 UUID가 새로 발급**돼 하드코딩한 `panel` 프로퍼티가 끊긴다(증상: `Open()`이 조용히 return). `b.write(..., { bind: { mlua, props: { panel: "Enhance" } } })`로 매 빌드마다 자동 주입한다.

**데이터**: `StarforceSet.csv`에 `DowngradeOnFail` · `ProtectStoneCost` 2열 추가. 1~10강은 실패해도 유지, 11~25강은 실패 시 1강 하락이며 보호석 비용은 해당 단계 `StoneCost`와 같다. 서버는 `RequestStarforceEnhance(heroId, useProtect)`로 확장하고 결과코드를 `success/keep/down/max/cost/data`로 세분화했다.

**파괴는 없다**(2026-09-04 사용자 확정). 처음엔 `DestroyRate` 열과 `파괴 확률` 행을 붙였으나, 영웅은 뽑기로 모아 성급을 올리는 자산이라 파괴 손실이 과하다는 판단으로 **페널티 축을 하락 하나로 통일**했다. 열은 CSV에서 제거했고, 3번째 행은 `하락 확률`, 보호 버튼은 `하락 방지`가 됐다.

**확률 3행의 의미**: 실패 = `100 − 성공`(성공하지 못할 확률), 하락 = **하락 구간이면서 보호를 안 켰을 때만** 실패와 같은 값이고 그 외에는 0%. 셋을 더해 100이 되게 하지 않은 이유는, 그러면 하락 구간에서 "실패 0% / 하락 20%"처럼 보여 실패를 안 하는 것처럼 읽히기 때문이다. 보호를 켜면 하락만 0%로 떨어져 **보호의 효과가 숫자로 드러난다**.

**중앙 별 게이지(25칸)는 제거했다**(2026-09-04 사용자 요청) — 바로 아래 `n강 → n+1강`과 정보가 중복되고 두 줄로 접혀 지저분했다.

**시행착오**: 카드 프레임 `1a46a42d`는 260×300·80×100으로 줄이면 흰 판으로 뭉개져 9-slice 칩 `3acfe092…`로 교체했다. `top-center` 앵커는 **아래로 내리려면 y가 음수**라는 부호 규칙을 두 번 틀렸다. `_Find`에서 `string.find`가 두 값을 반환해 `LWA-1111`이 떴고 `string.gmatch` 경로 순회로 바꿨다(`GetChildByName`은 경로를 못 받고, 이름만 주면 재귀 검색이 동명 엔티티를 잘못 잡는다).

**검증**: 빌드 0건 · 패널 오픈 로그 · 정보행 실측 4경로 — 0강(성공100/실패0/하락0, 비용 1) · 11강 보호OFF(80/20/20, 결정석 3) · 11강 보호ON(80/20/**0**, 결정석 6) · 25강 만렙(전부 `-`, 보호 "사용 불가") · `StarforceSet` 값 일치 · 스크린샷 렌더 확인(별 게이지 사라짐).

**강화 경로는 실제로 동작한다** — 사용자가 Maker에서 직접 3회 강화한 흔적을 확인했다(0강→3강, 골드 −4500, 결정석 −3 = CSV 1~3강 비용 `1000+1500+2000` / `1+1+1`과 정확히 일치). AI가 누르면 사용자 재화가 소모되므로 클릭 자체는 하지 않았다.

> 클라 표시 검증은 `_HeroDataManager._T.inventoryData.starforce[heroId]`를 잠시 바꾸고 `Refresh()`만 부르는 방식으로 했다 — **RPC를 타지 않으므로 서버 저장값은 그대로**다. 재화를 쓰지 않고 11강·25강 구간 표시를 확인하는 데 쓸 수 있다.

### 2026-09-04 강화 대상을 파티 5인 → 보유 영웅 전체로

**문제**: 대상 선택이 파티 슬롯 5칸뿐이라 **6번째 영웅부터는 강화할 방법이 아예 없었다**. 사용자가 발견.

**해결** — UI는 **버튼 1개만 추가**하고 나머지는 스크립트로 처리했다(사용자가 Maker에서 잡은 레이아웃 보존이 최우선):
- `_Party()` → `_Targets()`: 보유 영웅 전체를 `HeroSet.csv` 행 순서로 반환. **보유 판정은 `owned[heroId] ~= nil`** — `heroes[heroId]`는 중복 카운터라 첫 획득이 `0`이고, `> 0`으로 보면 새 영웅이 통째로 빠진다.
- 슬롯 5칸은 이제 **전체 목록의 5개짜리 창(페이지)**. `_PageStart() = floor((idx-1)/5)*5`로 선택된 대상이 있는 페이지를 연다. 슬롯 클릭은 `SelectSlot(1~5)` → `_PageStart() + 슬롯번호`로 절대 인덱스 환산.
- `NextTarget()` — 새 `강화 대상 변경` 버튼. 보유 영웅을 순환하며(마지막 → 1) 페이지도 따라 넘어간다. 라벨에 `n/총` 표시.
- 선택 표시는 **새 엔티티 없이 슬롯 스프라이트 밝기만** 조절(선택 `Color(1,1,1,1)` / 나머지 `0.55`).

**UI 패치 방식**: `UIBuilder.read()` → `button()` 1개 추가 → `write()`. **전체 재빌드가 아니다.** 패치 전후를 엔티티 단위로 비교해 `추가 1 / 삭제 0 / 변경 0`을 확인했다(`scratchpad/diff_enh.cjs`).

**자리 함정**: `TargetPanel`의 `RectSize`는 485×760이지만 **패널 아트의 안쪽 바닥은 로컬 y ≈ -312**로, RectSize 기준 -380보다 훨씬 위다. 처음에 -320에 80 높이로 넣었더니 테두리에 걸려 잘렸다. 슬롯(-170~-270) 아래 남은 띠가 42밖에 안 돼 **300×40 / y=-291 / 폰트 22**로 눌러 넣었다. RectSize를 패널 경계로 믿지 말고 스크린샷으로 확인할 것.

**검증**: 빌드 0건 · 보유 5명 목록 정확 · `NextTarget` 6회 순환(5→1) · 슬롯 하이라이트가 선택 칸만 밝음 · 페이지 계산(idx 1·5→0, 6·10→5, 11·13→10) · `SelectSlot` 환산(2페이지 슬롯3 → 8번째) · 스크린샷으로 버튼이 패널 안에 들어온 것과 대상 전환(히어로→신궁: 초상·전신·이름·확률·비용 전부 갱신) 확인.

**후속 (같은 날)**:
- 좌측 `StarText`에서 **성급 별(★★☆☆☆) 표기 제거** → `"스타포스 N강"`만 표시(사용자 결정). 호출처가 사라진 `_StarStr`도 삭제했다. 성급 자체는 `CalcATK`·스킬 수집에 계속 쓰이므로 `starLv` 계산은 남아 있다.
- 대상 슬롯 아이콘을 `ThumnailRUID` → **`PortraitRUID`**로 교체. 66×78 칸에 전신 아트를 0.07로 줄이면 얼굴이 안 보인다. 기존 `_SetPortrait()` 헬퍼를 그대로 재사용해 슬롯 루프가 짧아졌다.
- **6명 이상 동작을 실측**했다. `HeroSet.csv`에 임시로 3행(`hero_test1~3`)을 붙이고 클라 캐시에만 보유 처리해 8명 상황을 만든 뒤 확인 — `1/8`에서 버튼 5번 → `6/8`로 넘어가며 **슬롯이 2페이지(6·7·8번째 + 빈칸 2개)로 자동 전환**됐다. 테스트 후 CSV를 백업본으로 원복하고 5명·등급색인(SSR 1/SR 4)·스타포스 값이 그대로인지 재확인했다. 서버에는 아무것도 쓰지 않았다(클라 캐시만 조작).

> 데이터 확장이 필요한 검증은 이 패턴을 쓸 것: **CSV 백업 → 임시 행 추가 → 클라 캐시로 보유 처리 → 확인 → CSV 원복 → 재확인.** 서버 RPC를 타지 않으므로 세이브가 오염되지 않는다.

### 2026-09-04 대상 초상 계층 뒤집기 · 슬롯 선택 표시를 RUID 교체로

**1. `TargetPanel/Frame/Icon` → `TargetPanel/Icon/Frame`** — 해봤으나 **같은 날 원복했다**(초상이 가려져서). 아래는 시도 기록이며 현재 구조는 `Frame/Icon` 그대로다. `UIBuilder`에 reparent API가 없어 **remove + 재생성**으로 처리하되, 두 엔티티의 `@components`를 통째로 복사해 넣어 값 손실을 막았다. 화면 위치가 그대로 남도록 좌표를 서로 맞바꿨다(Frame이 갖고 있던 `(0,70)`을 Icon이 받고, Frame은 `(0,0)`으로). 앵커 설정에 상관없도록 `anchoredPosition`뿐 아니라 `OffsetMin/Max`도 같은 delta만큼 밀었다. 형제 그리기 순서가 바뀌지 않게 옛 Frame의 `displayOrder`를 새 Icon에 옮겼다.
- **함정(실제로 밟음)**: `framePos`/`iconPos`를 컴포넌트 객체의 **참조**로 잡았더니 첫 `shift()`가 원본을 고쳐 두 번째 delta가 0이 됐다(Frame이 `(0,70)`에 그대로 남음). 숫자를 먼저 복사해 두고 계산해야 한다. 스냅샷에서 되돌린 뒤 재실행했다.
- **원복 사유**: 자식이 부모 위에 그려지므로 **Frame이 초상을 덮었다.** 프레임 스프라이트(`d0de6750…`) 가운데가 불투명이라 초상이 사라졌다. 런타임에 Frame만 꺼서 초상이 정상 표시되는 것을 확인해 **코드 문제가 아니라 계층 변경의 필연적 결과**임을 가렸고, 사용자 판단으로 되돌렸다. 나중에 다시 뒤집으려면 테두리 전용(가운데 투명) 이미지가 먼저 필요하다.
- **원복 방법**: 뒤집기 직전에 떠 둔 `scratchpad/uisnap/<시각>/` 스냅샷을 덮어쓰고 `.mlua`의 초상 경로만 `TargetPanel/Frame/Icon`으로 되돌렸다. `_Find` 직계 우선 조회와 슬롯 RUID 교체는 그대로 유지했다.

**2. 슬롯 선택 표시를 밝기 → 배경 RUID 교체로**(사용자 요청). 선택 `ef1dce6eaf904ce39694b6a1be344fb5`, 비선택은 **화면 파일에 들어 있는 값 그대로**. 비선택 RUID를 하드코딩하지 않고 `_EnsureBound()`에서 슬롯마다 한 번 읽어 `_T.slotBaseRUID`에 캐시한다 — Maker에서 RUID를 바꿔도 코드를 고칠 필요가 없다(실제로 사용자가 `3acfe092…` → `d0de6750…`로 바꿨다). `Color`는 이제 건드리지 않는다(반투명 처리 제거).
- 런타임에서 `SpriteGUIRendererComponent.ImageRUID`는 **문자열이 아니라 `MOD.Core.MODDataRef` 객체**다. `tostring()`으로 찍으면 타입 이름만 나오니 로그로 비교하지 말 것. 대입은 문자열·객체 둘 다 받는다.

**3. `_Find`가 직계 자식을 먼저 찾도록 수정.** 계층을 뒤집으면서 `TargetPanel/Icon`이 생겼는데 `Slot1~5/Icon`과 이름이 겹친다. 기존처럼 `GetChildByName(seg, true)`(재귀)만 쓰면 엉뚱한 손자를 잡을 수 있어, **비재귀 → 실패 시 재귀** 순서로 바꿨다.

**검증**: 빌드 0건 · 패치 전후 엔티티 비교(추가 2 / 삭제 2 / 나머지 변경 0) · 런타임 계층 확인(`TargetPanel` 직계 `Icon` ✓, `Icon` 직계 `Frame` ✓, 옛 `TargetPanel/Frame` 없음 ✓) · `_Find`가 대상 Icon과 슬롯 Icon을 각각 정확히 찾음 · 스크린샷으로 슬롯 3번만 선택 RUID로 바뀌고 나머지는 기본 RUID·불투명 유지 확인.

### 2026-09-04 스타포스 확률표 재조정 (메이플 원본 → 자체 곡선)

메이플 원본표를 잠깐 썼다가 **사용자가 직접 지정한 곡선으로 교체**했다. 원본의 어색한 지점 두 곳(17→18, 20→21에서 성공률이 되레 오르던 것)을 없애고, **10성 이상은 성공률을 30%로 평탄화**한 뒤 파괴 확률만 계단식으로 올리는 형태다.

| 구간 | 성공 | 실패(유지) | 하락 | 파괴 |
|---|---|---|---|---|
| 1~10강 | 95 → 55% | 나머지 전부 | 0 | 0 |
| 11~13강 | 50 / 45 / 40% | 0 | 50 / 55 / 60% | 0 |
| 14~15강 | 30% | 0 | 70% | 0 |
| 16~17강 | 30% | 0 | 67% | 3% |
| 18~19강 | 30% | 0 | 65% | 5% |
| 20~21강 | 30% | 0 | 63% | 7% |
| 22강 | 30% | 0 | 59% | 11% |
| 23강 | 30% | 0 | 50% | 20% |
| 24강 | 10% | 0 | 50% | 40% |
| 25강 | 5% | 0 | 46% | 49% |

- **안전 구간 폐지**: 원본의 16·21강(하락 없이 파괴만) 예외가 사라져 **10성 이상은 실패하면 전부 하락 또는 파괴**다. 규칙이 하나로 단순해졌다.
- **하락 방지는 하락이 있는 전 구간(11~25강)에서 켜진다.** `ProtectStoneCost`를 그 구간 전부에 해당 단계 `StoneCost`와 같은 값으로 넣었다(원본표 때는 17~20·22~25에만 있었다). 여전히 **파괴는 막지 못한다.**
- 마지막 두 단계 성공률은 사용자가 두 번 조정했다: 24강 3→10%, 25강 1→3→**5%**.

**검증**: 빌드 0건 · 25행 전부 합 100% 확인(이상 0) · 런타임 표본 lv10/11/14/16/21/23/24/25 값 일치 · 이전 CSV는 `scratchpad/backup/StarforceSet.prev.csv`에 백업.

### 2026-09-04 보호 아이템: 하락 방지 → 파괴 방지 (+ 토글 유지)

**왜**: 새 확률표에서 하락은 10성 이상 전 구간에 항상 있어서, 그걸 막아 주면 페널티가 통째로 사라진다. 실제로 아픈 것은 **12강까지 밀리는 파괴**라 보호 대상을 그쪽으로 옮겼다.

- **파괴를 "없애는" 게 아니라 "1강 하락으로 대체"한다.** 서버 판정에서 파괴 구간에 걸렸을 때 `protecting`이면 `code = "down"`으로 내려간다. 그래서 화면에서도 파괴 몫이 하락에 합산돼 보인다(25강: 하락 46 → 95%, 파괴 49 → 0%).
- `ProtectStoneCost`를 **파괴가 있는 구간(16~25강)에만** 넣었다. 11~15강은 하락만 있고 파괴가 없어 보호 비용이 0 → `canProtect`가 false다.
- **버튼을 토글로 바꿨다.** `useProtect`를 초기화하던 4곳(`Open` · `SelectTarget` · `NextTarget` · `OnStarforceResult`)을 전부 제거해, 다시 끄기 전까지 켜진 채로 남는다. 대상이나 단계가 바뀌어도 유지된다.
- **대상을 바꿨는데 그 단계에 파괴가 없으면 토글이 자동으로 꺼진다**(`_ClearProtectIfNoDestroy`, `SelectTarget`·`NextTarget`에서 호출). 켜져 있어도 아무 일이 없는 상태로 남아 헷갈리는 것을 막는다. **같은 대상에서 단계만 오르내리는 동안에는 건드리지 않는다** — 강화 도중 임의로 꺼지면 안 되기 때문이다. 25강 만렙(다음 단계 없음)도 꺼진다.
- `ToggleProtect`의 "이 단계는 보호 의미 없음" 차단(토스트)을 없앴다. 파괴가 없는 단계에서도 켤 수 있고, 상태만 남아 있다가 16강↑에 오면 저절로 적용된다. 그 동안 State는 `ON (이 단계는 파괴 없음)`으로 표시하고 **추가 비용을 받지 않는다.**
- `.ui`는 `ProtectToggle/Label` 텍스트 하나만 patch(`하락 방지` → `파괴 방지`).

**검증**: 빌드 0건 · 표시 6경로 — 23강 OFF(30/0/50/20, 결정석 8) / 23강 ON(30/0/**70**/**0**, 결정석 **16**) / 대상 변경 후 `useProtect=true` 유지 / 11강에서 `ON (이 단계는 파괴 없음)` + 결정석 3(추가 없음) / 23강 복귀 시 재적용 / 다시 OFF 복귀 · 서버 판정식 시뮬 lv25 OFF `성공5.2 파괴49.1 하락45.7`(파괴결과 12강) vs ON `성공5.3 파괴0 하락94.7` · 스크린샷으로 버튼 라벨 `파괴 방지` 확인.

**자동 해제 검증 4경로**: 파괴 구간(21강 시도)에서 ON → 파괴 없는 대상(6강 시도)으로 변경 시 **자동 OFF** ✓ / 다시 파괴 구간 대상으로 돌아와도 **자동으로 켜지지는 않음**(OFF 유지) ✓ / 파괴 구간 → 파괴 구간(23강 시도) 이동은 **ON 유지** ✓.

### 2026-09-05 강화 결과 이펙트(Result) 배선

사용자가 화면 루트에 만든 `/ui/Enhance/Result` 스프라이트에 결과별 RUID를 물렸다. `OnStarforceResult`에서 `success` → 성공 / `keep`·`down` → 실패 / `destroy` → 파괴 이펙트를 띄우고 **1.5초 뒤 자동으로 숨긴다**. 재화 부족·만렙처럼 요청 자체가 무산된 코드에서는 띄우지 않는다. 패널을 열거나 닫을 때도 지난 이펙트를 지운다.

- 연속 강화 시 앞선 타이머가 새 연출을 지우지 않도록 **세대 카운터**(`resultGen`)로 무효화한다 — `MonsterManager.battleGen`과 같은 방식.
- 같은 결과가 연달아 나와도 처음부터 재생되도록 `Enable`을 껐다 켠다(클립이 `Onetime`이라 재대입만으로는 다시 재생되지 않을 수 있다).
- **크기·비율은 코드에서 건드리지 않는다** — `Result` 엔티티의 `RectSize`/`PreserveSprite`를 그대로 따른다.

**함정 — 안 보이는 이유를 세 번 잘못 짚었다.** 최종 원인은 **이펙트 원본이 작아서** `1100×100` 렉트 안에서 실선처럼 보인 것이었다. 확인 순서를 남긴다:

1. **엔티티/그리기 순서 의심** → 검증된 스프라이트(제목 배너)를 같은 자리에 넣어 보니 정상 출력. 엔티티·위치·`displayOrder`(BackGround 0 < Result 1) 모두 문제 없음.
2. **RUID 형식 의심** → 평문 문자열·`DataRef(...)`·`thumbnail://` 셋 다 결과 동일. (참고: 이 프로젝트에서는 `ImageRUID`에 평문 문자열 대입도 동작한다.)
3. **리소스 종류 확인** → 공개 검색 인덱스에는 없고(`getResourcesBatch` notFound), **계정 자산**이었다. `asset_get_account_resource_metadata_bulk`로 조회하니 셋 다 `animationclip`(성공/실패/파괴 이펙트).
4. **재생 타입** → `AnimClipPlayType = 0`은 "끄기"가 아니라 **`Onetime`**이다. 한 번 재생하고 끝나므로 1~2초 뒤 찍은 스크린샷에는 이미 안 남아 있었다.
5. **최종** → `PreserveSprite=NativeSize` + `LocalScale 6`으로 키우니 "SUCCESS" 텍스트와 파티클이 선명하게 보였다. 원본이 작을 뿐 클립은 정상.

> 계정에 직접 업로드한 자산은 `msw-search`(공개 인덱스)로는 조회되지 않는다. **`asset_get_account_resource_metadata_bulk`로 `resourceType`을 확인할 것.**

**검증**: 빌드 0건 · 패널 열 때 `Result.Enable=false` · 결과 수신 직후 `true` · 검증용 스프라이트로 위치/순서 정상 출력 · 성공 이펙트 실제 렌더 확인(6배 확대 스크린샷).
⚠️ **남은 것**: `Result` 렉트가 `1100×100`이라 `AspectOnly` 기준으로 이펙트가 100px 높이로 작게 나온다. 사용자가 렉트를 키워야 제 크기로 보인다.

### 2026-09-05 강화 패널 사운드 배선

사용자 업로드 `audioclip` 4종을 물렸다(`asset_get_account_resource_metadata_bulk`로 종류·이름 확인).

| 용도 | RUID | 이름 |
|---|---|---|
| 공통 버튼 클릭 | `ac0a6bab44444690b1f4248fdbc768a8` | `Enchant` |
| 강화 성공 | `87e19c7bfdd14c94914cdf3f6061e756` | `EnchantSuccess` |
| 강화 실패(유지·하락) | `3cbefdfa4eb5450c89b948dfb64ba23a` | `EnchantFail` |
| 파괴 | `0a586d08d09e4a2a9849020efd59b860` | `EnchantDestroyed` |

- ~~클릭음을 `_Bind`에 붙여 모든 버튼에 적용~~ → **되돌렸다**(아래 항목 참조). `Enchant`는 범용 클릭음이 아니라 **강화 액션 소리**다.
- 결과음은 이펙트와 같은 분기에서 짝지어 재생한다. **유지(`keep`)도 실패음**을 쓴다. 재화 부족·만렙(`cost`/`max`)은 연출·소리 없이 토스트만.
- 재생은 프로젝트 기존 방식 그대로 `_SoundService:PlaySound(ruid, 1.0)`.

**검증**: 빌드 0건 · 실제 `ButtonClickEvent` 2건(ChangeTarget → idx 1→2, Slot2 → idx 2)으로 클릭음+콜백 경로가 함께 도는 것 확인 · 결과 5종(`success`/`keep`/`down`/`destroy`/`cost`) 호출에 오류 로그 0건. **소리 자체는 MCP로 들을 수 없어 재생 여부는 사용자 확인이 필요하다.**

### 2026-09-05 강화 연출 순서 정리 (클릭음 → 결과) + 연타 차단

**되돌린 것**: `Enchant`를 공통 클릭음으로 보고 `_Bind`에 붙였더니 **로비 진입 버튼·뒤로·슬롯 등 아무 버튼에서나 울렸다**. `Enchant`는 범용 UI 클릭음이 아니라 **강화 시도음**이었다. `_Bind`에서 빼고 `OnClickEnhance`에만 남겼다.

> 교훈: 사운드 이름(`Enchant`)이 곧 용도다. "기본 버튼 소리"라는 설명을 화면 전체의 공통 클릭음으로 넓혀 해석하지 말 것 — 붙일 범위를 먼저 확인한다.

**연출 순서**: 클릭 → `Enchant` 재생 → **소리가 끝난 뒤** 결과 이펙트 + 결과음 → 1.5초 후 정리.

- `PlaySound`에는 **재생 종료 콜백이 없다.** `SoundPlayStateChangedEvent`는 BGM(`SoundService`)과 `SoundComponent`용이라 일회성 `PlaySound`에는 오지 않는다. 그래서 클립 길이만큼 타이머(**1.1초**, `OnClickEnhance` 안의 상수)로 기다린다. **소리를 교체하면 이 값도 같이 맞출 것.**
- 서버 응답이 소리보다 **빨리 올 수도, 늦게 올 수도** 있다. `sfxDone` 플래그와 `pendingResult` 보관으로 양쪽 순서를 모두 처리한다 — 소리가 아직이면 결과를 보관했다가 타이머가 꺼내 연출하고, 이미 끝났으면 즉시 연출한다.
- **숫자(공격력·확률·비용)는 응답 즉시 갱신**하고, 이펙트·소리·토스트만 미룬다.

**연타 차단**: `_T.enhancing` 플래그. 클릭 → `true`, 결과 연출이 끝나는 1.5초 뒤 `false`. 세대 카운터(`seq`)로 이전 시도의 타이머가 새 시도의 잠금을 풀지 못하게 막았다. 안전장치 3개:
- 재화 부족·만렙·데이터 없음(`cost`/`max`/`data`)은 **기다리지 않고 즉시 해제**(연출도 없음).
- 응답이 유실돼도 영구 잠김이 없도록 **8초 실패 타이머**.
- 패널을 닫으면(`Close`) 잠금과 보류 결과를 함께 비운다.

**검증**: 빌드 0건 · `enhancing=true`에서 클릭 → `강화 요청` 로그 없음(차단) · 소리 재생 중 응답 → 보류 `true` / `Result` 표시 `false` · 소리 종료 후 → `Result` 표시 `true` · `cost` 응답 → 즉시 `enhancing=false` · 1.5초 뒤 → `Result` 숨김 + `enhancing=false`. **소리 자체와 체감 타이밍(1.1초)은 사용자 확인 필요.**

### 2026-09-05 재화 부족 선차단 · 같은 RUID 재대입으로 인한 애니메이션 재시작 수정

**1. 재화가 없어도 강화음이 났다.** 클라가 비용을 확인하지 않고 소리부터 내고 요청을 보냈기 때문이다. 서버가 `cost`를 돌려줄 때는 이미 소리가 난 뒤였다. **클릭 시점에 클라에서 먼저 확인**하도록 고쳤다 — 부족하면 소리도 요청도 내지 않고 토스트만 띄운다. 부족한 쪽과 모자란 양을 같이 알려 준다(`골드가 N 부족합니다` / `강화 결정석이 N개 부족합니다` / 둘 다).
> 서버의 `cost` 판정은 그대로 남겨 두었다 — **권위는 서버에 있고**, 클라 확인은 헛된 연출을 막는 용도다.

**2. 강화할 때마다 중앙 전신 아트가 처음부터 재생됐다.** 원인은 `Onetime` 자체가 아니라 **같은 RUID를 다시 대입한 것**이다. `Refresh()`는 강화 결과마다 불리는데 `_SetThumbnail`이 `ImageRUID`를 무조건 다시 넣었고, animationclip은 재대입하면 재생이 프레임 0으로 돌아간다. `Onetime`이라 한 번 재생 후 멈춰 있으니 재시작이 더 눈에 띄었을 뿐이다(Loop였다면 티가 덜 났을 것).
- `_SetImageRUID(key, renderer, ruid)` 헬퍼 신설 — `_T.lastRuid[key]`와 비교해 **값이 실제로 달라졌을 때만 대입**한다. `_SetThumbnail`·`_SetPortrait` 둘 다 이걸 쓴다.
- 결과 이펙트(`_ShowResult`)는 **일부러 다시 재생시켜야 하므로** 캐시를 거치지 않고 직접 대입한다.

> 규칙: **animationclip을 매 프레임/매 갱신마다 대입하지 말 것.** 화면 갱신 함수에서 무심코 `ImageRUID`를 다시 넣으면 애니메이션이 계속 처음으로 튄다.

**검증**: 빌드 0건 · 재화 부족 3경로(둘 다 부족 / 골드만 / 결정석만) 전부 `enhancing=false`이고 `강화 요청` 로그 없음 — 요청·소리 모두 나가지 않음 · RUID 캐시 3단계(첫 Refresh 기록 → 반복 Refresh 시 재대입 건너뜀 → 대상 변경 시 새로 기록) 확인. 재화는 클라 캐시만 조작했고 검증 후 원복.

### 2026-09-05 결과 이펙트가 한 바퀴 더 돌던 문제

**증상**: 결과 이펙트가 끝난 뒤 **다시 시작하다가 사라졌다**.

**원인**: `SpriteGUIRendererComponent.AnimClipPlayType`의 **기본값이 `Loop`**다(`.d.mlua` 확인). `Result` 엔티티도 `Loop`로 저장돼 있어(런타임 확인) 클립이 끝나면 곧바로 다시 재생을 시작했고, 1.5초 숨김 타이머가 그 두 번째 재생을 중간에 잘랐다.

**수정**: `_ShowResult`에서 `AnimClipPlayType = SpriteAnimClipPlayType.Onetime`을 지정한다. **결과 이펙트에만** 적용되고 다른 스프라이트는 건드리지 않는다. 화면 파일은 손대지 않았다(런타임 지정).

> 앞선 기록에 "세 클립이 Onetime"이라고 적었던 것은 틀렸다 — 그건 다른 컴포넌트(슬롯)를 보고 넘겨짚은 것이고, `Result`는 `Loop`였다. **애니메이션 재생 방식은 클립이 아니라 렌더러 컴포넌트 속성이다.**

**검증**: 빌드 0건 · 저장값 `Loop` 확인 → `_ShowResult` 후 `Onetime` + 표시 `true` 확인. **체감(한 번만 재생되고 멈추는지)은 사용자 확인 필요.**

### 2026-09-05 재화 부족 시 강화하기 버튼 비활성 + 중앙 안내

토스트로만 알리던 것을 **버튼 상태로** 바꿨다. 누를 수 없는 상황이면 애초에 눌리지 않는다.

- **`_SetEnhanceEnabled(on, warning)`** 신설 — `ButtonComponent.Enable`로 클릭을 막고, 스프라이트 `Color`를 `0.45` 회색으로 죽이고, 중앙 안내 문구를 함께 세팅한다.
- **`Refresh()`에서만 부른다.** 대상·단계·보호 토글·재화가 바뀌면 화면 갱신이 어차피 일어나므로, **충족되면 저절로 되살아나고 문구도 지워진다.** 대상을 바꿔도 여전히 부족하면 그대로 유지된다 — 별도 상태 관리가 필요 없다.
- 안내 문구는 부족한 쪽을 구분한다: `골드가 부족합니다` / `강화 결정석이 부족합니다` / `골드와 강화 결정석이 부족합니다`.
- **25강 만렙에서도 버튼을 잠근다**(문구는 비움 — 중앙에 이미 `최고 단계 달성`이 뜬다).
- 화면 파일은 텍스트 엔티티 하나만 추가(`CenterPanel/CostWarning`, 800×60, `#FF6B6B`). 패치 전후 비교 **추가 1 / 삭제 0 / 변경 0**.

> `OnClickEnhance`의 재화 확인은 그대로 남겨 뒀다. 버튼이 잠겨 있어도 **경로가 하나 더 있는 편이 안전**하고, 서버의 `cost` 판정이 최종 권위인 것도 변함없다.

**검증 6경로**: 충분(활성·문구 없음) / 골드만 부족 / 결정석만 부족 / 둘 다 부족 (셋 다 비활성·회색 `r=0.45`·해당 문구) / **부족 상태로 대상 변경 → 유지** / **충족 후 대상 변경 → 다시 활성·문구 삭제**. 스크린샷으로 회색 버튼과 빨간 안내 문구 확인. 재화는 클라 캐시만 조작 후 원복.

⚠️ `CostWarning` 위치가 모루 그림과 `n강 → n+1강` 사이라 다소 빽빽하다 — 사용자가 옮길 수 있다.

### 2026-09-05 영웅 인벤토리에서 스타포스(강화) 제거

강화가 전용 패널로 옮겨가면서 인벤토리 상세의 강화 UI가 중복이 됐다. **스타포스만** 걷어내고 **성급(★)은 남겼다**(2026-09-05 사용자 확정) — 성급은 중복 획득으로 오르는 인벤토리 고유 정보이지, 강화 패널로 옮긴 것이 아니다.

**화면 엔티티 37개 삭제** (118 → 81):
- `HeroDetail/DetailPanel/` 의 `SFLevel` · `SFAtk` · `SFHp` · `SFCost` · `SFRate`
- `HeroDetail/panelButtons/btnEnhance`
- `HeroDetail/StarForcePanel` (Row1~5 × Star1~5 = 25칸 게이지 포함 31개)

**스크립트** (`HeroInventoryUI` 588 → 483줄):
- 프로퍼티 8개(`btnEnhance` · `txtSFLevel` · `txtSFAtk` · `txtSFHp` · `txtSFCost` · `txtSFRate` · `txtMyStones` · `txtMyGold`) 삭제. 뒤 둘은 애초에 `nil`로 연결된 적이 없었다.
- `OnClickEnhance` · `OnStarforceResult` 메서드 삭제.
- `RefreshDetail`의 스타포스 블록(별 25칸 갱신 루프 포함) 삭제.
- `HeroDataManager:ReceiveStarforceResult`에서 `_HeroInventoryUI:OnStarforceResult` 호출 제거 — 이제 `_EnhanceUI`만 결과를 받는다.

**남긴 것**: `txtDetailStars`(★★★★☆) · `panelNextEnhance`(`다음 강화 ★★ → ★★★`) · 파티 토글 · 스킬 5칸.

> ⚠️ **`UIEnhance.mlua`는 지우지 않고 남겼다.** 부착 대상(`btnEnhance`)이 사라져 참조 0건인 고아 파일이다. 상시 이슈("스크립트 파일 삭제 직후 Maker 전체 저장 → 컴포넌트 전량 소실")를 피하려고 파일 삭제는 하지 않았다 — **Phase 6 정리 대상**.

**조사 중 알아낸 것 (중요)**: `property TextGUIRendererComponent`/`property Entity`에 들어 있는 UUID는 **컴포넌트 id**라, 화면 파일의 **엔티티 id와 다르다.** 빌더로 엔티티 id를 훑으면 "어느 화면에도 없음"이 나오지만 실제로는 멀쩡히 살아 있다. **살았는지 죽었는지는 런타임 `isvalid()`로 판정할 것.** 엔티티는 이름·경로로 찾아야 한다.

**검증**: 빌드 0건 · 삭제 5종 전부 `false`, 유지 4종 전부 `true`(런타임 조회) · `SelectHero`로 상세를 실제로 그려 히어로 `★★☆☆☆` / 비숍 `★★★★☆` 정상 출력, 오류 로그 없음 · 스크린샷으로 인벤토리 렌더 확인. 삭제 전 화면·스크립트 백업 완료.

### 2026-09-05 인벤토리 상세 후속: 별 게이지 복구 · 다음 강화 제거 · 공격력/체력 추가

앞선 삭제에서 **`StarForcePanel`까지 지운 것은 과했다**(사용자 지적). 25칸 게이지는 강화 기능이 아니라 **현재 강화 수치를 읽는 표시**라 인벤토리에 필요하다.

- **`StarForcePanel` 31개 복구.** 삭제 직전 스냅샷(`scratchpad/uisnap/`)에서 서브트리를 읽어 부모부터 다시 만들고 `@components`를 통째로 덮어썼다 — 좌표·RUID·`displayOrder`까지 원본 그대로다. 스크립트의 별 갱신 루프도 되살렸다(강화 버튼 없이 **표시 전용**).
- **`panelNextEnhance` 삭제**(3개) — `다음 강화 ★★ → ★★★`와 `ATK 1200 → 1205`. 강화 패널에 같은 정보가 있어 중복이었다. 프로퍼티 `txtNextStar`·`txtNextStatChange`와 해당 블록도 제거.
- **공격력·체력 추가**(2개) — 등급(`txtDetailRarity`)과 성급 별(`txtDetailStars`) 사이. 계산식은 강화 패널과 동일하게 **성급 + 스타포스를 모두 반영**한다.

> 새 텍스트는 프로퍼티(UUID) 배선 대신 **`panelDetail:GetChildByName(name, true)` 경로 조회**(`_SetDetailText`)를 쓴다. 새로 만든 엔티티는 UUID가 없어 프로퍼티를 손으로 이어야 하는데, 이름 조회면 재생성해도 안 끊긴다.

엔티티 81 → 111 (+31 복구 −3 삭제 +2 추가).

**검증**: 빌드 0건 · `StarForcePanel` 존재 `true` / `panelNextEnhance` 부재 `true` · 히어로 `★★☆☆☆ 공격력 1271 / 체력 12710`(스타포스 13강), 비숍 `★★★★☆ 공격력 1428 / 체력 14280` 실측 · 스크린샷으로 별 게이지 5칸 점등과 능력치 배치 확인.

### 2026-09-05 인벤토리 상세에서 성급 별 표기 제거

`txtDetailStars`(★★★★☆ = 성급)를 지웠다. **화면에 별이 두 종류라 헷갈렸기 때문**이다 — 바로 아래 25칸 스타포스 게이지도 별이라, 어느 쪽이 강화 수치인지 구분이 안 됐다(사용자 지적). 이제 상세 화면의 별은 **스타포스 게이지 하나뿐**이다.

- 화면 엔티티 1개 삭제(111 → 110), 프로퍼티 `txtDetailStars`와 대입 1줄 제거.
- 호출처가 사라진 `StarStr()` 헬퍼도 삭제(프로젝트 전체 grep으로 다른 호출처 0건 확인).

> 성급은 여전히 스킬 해금 기준으로 동작한다(잠긴 스킬은 `SkillLock`으로 표시). 상세 화면에서 **숫자로 보이지 않을 뿐**이다. 성급을 다시 보여줘야 하면 별이 아닌 다른 형태(예: `차수 4/5`)를 쓰는 편이 안전하다.

현재 상세 구성: 초상 · 이름 · 등급(SR/SSR) · **공격력 · 체력** · 스타포스 25칸 게이지 · 스킬 5칸 · 파티 토글.

**검증**: 빌드 0건 · `txtDetailStars` 부재 확인 · 비숍 `SR 공격력 1428 체력 14280`, 히어로 `공격력 1271 체력 12710` 정상 갱신(오류 0) · 스크린샷으로 별이 게이지 하나만 남은 것 확인.

### 2026-09-05 영웅 상세에 주/부속성 퍼즐 표시

공격력·체력 아래에 **주속성 / 부속성 퍼즐 조각**을 넣었다. 무기타입이 곧 퍼즐 원소라(`HeroSet.WeaponType` / `SecondaryWeaponType`), `PuzzleSet.csv`의 `ElementType → SpriteRUID`로 실제 퍼즐 스프라이트를 그린다.

- 화면 엔티티 4개 추가(110 → 114): `txtPuzzleMainLabel` · `imgPuzzleMain` · `txtPuzzleSubLabel` · `imgPuzzleSub`.
- `_PuzzleRUID(elementType)` — `PuzzleSet` 6행을 한 번 읽어 캐시. 원소를 추가해도 코드 수정이 필요 없다.
- `_SetPuzzleRow(label, img, elementType)` — **원소가 비어 있으면 라벨과 퍼즐을 함께 숨긴다.** 지금은 5영웅 모두 부속성이 있지만, 없는 영웅이 생겨도 빈 칸이 남지 않는다.
- 기존 `txtDetailElement` 프로퍼티는 선언만 있고 **쓰인 적이 없다**(대입 0건). 이번에 대체된 셈이라 정리 대상이다.

**함정**: `DetailPanel`이 화면 오른쪽에 붙어 있고 폭이 500이라, 아이콘을 `x=470`(폭 44)에 두니 **패널 밖으로 잘렸다.** 라벨을 20pt·100폭으로 줄이고 아이콘을 `x=412`(40×40)로 당겨 안쪽에 넣었다. 이 패널에 무언가를 더할 때는 **로컬 x + 폭 ≤ 500**을 지킬 것.

네 엔티티는 **화면 파일에서 `enable: false`로 시작**한다. 영웅을 고르기 전에는 상세 패널이 비어 있는데 라벨과 퍼즐만 떠 있었기 때문이다(사용자 지적). `_SetPuzzleRow`가 선택 시점에 켜므로, "선택 전 숨김 → 선택 후 표시"가 한 곳에서 결정된다.

**검증**: 빌드 0건 · `PuzzleSet` 6종(Greatsword/Bow/Dagger/Wand/Knuckle/Shield) 매핑 전부 존재 · 히어로(대검+방패) · 신궁(활+단검) · 바이퍼(너클+대검) 3영웅 주/부 모두 표시 `true` · 원소 빈 문자열이면 라벨·퍼즐 둘 다 `false`로 숨김 · 스크린샷으로 신궁의 초록(활)·보라(단검) 퍼즐이 패널 안에 정상 렌더되는 것 확인.

진입 직후 4개 모두 `false`, 영웅 선택 후 4개 모두 `true` 확인.

### 2026-09-05 전투 HUD 영웅 슬롯에 주/부속성 퍼즐

사용자가 `BattleGroup/Hero/Hero1`에 만든 `mainpuzzle`·`subpuzzle`을 **Hero2~5로 복제**하고, 파티 영웅에 맞춰 퍼즐이 채워지도록 배선했다.

- 복제는 Hero1의 `@components`를 통째로 복사하는 방식이라 위치(`main` 170,-9 / `sub` 217,-9, 55×55)·스프라이트 설정이 원본과 동일하다. 엔티티 221 → 229.
- `PlayerManager:SetupPartyDisplay()`의 슬롯 배치 루프에서 `_SetSlotPuzzle`을 호출한다. 원소 매핑은 인벤토리 상세와 같은 방식(`PuzzleSet` 6행 캐시)이고, **원소가 비면 숨긴다.**
- 헬퍼(`_PuzzleRUID` / `_SetSlotPuzzle`)는 `HeroInventoryUI`에도 같은 이름으로 있다. 지금은 각자 들고 있는데(둘 다 20줄 남짓), 세 번째 사용처가 생기면 공용 `@Logic`으로 빼는 편이 낫다.

**검증**: 빌드 0건 · Hero1~5 슬롯 전부 `mainpuzzle`/`subpuzzle` 존재·활성 확인 · 파티 5인 원소 매칭 정확(히어로 대검+방패 / 비숍 완드+방패 / 신궁 활+단검 / 섀도어 단검+너클 / 바이퍼 너클+대검) · 스크린샷으로 5줄 모두 색이 원소와 일치하는 것 확인.

### 2026-09-05 버그 — 전투 HUD 영웅 이름이 "모험가"로 고정

**증상**: `BattleGroup/Hero1~5`의 `txtClass`가 영웅 이름으로 안 바뀌고 화면 기본값 `모험가`가 그대로 남았다.

**원인**: 이 엔티티는 **`MOD.Core.TextComponent`**를 쓰는데 코드가 `TextGUIRendererComponent`를 찾고 있었다. `nil`이 나오니 `if c ~= nil then` 가드에 걸려 **조용히 건너뛰었다** — 오류도, 경고도 없다.

같은 파일 안에서 HP·실드 텍스트(`Hp/text_value`, `Shield/text_value`)는 **이미 `TextComponent`로 읽고 있었다.** 즉 한 화면에 두 종류가 섞여 있는데 한쪽만 맞춰 쓴 것이다.

**수정**: `TextGUIRendererComponent`를 먼저 보고 없으면 `TextComponent`로 넘어가게 했다. 둘 다 없으면 `log_warning`으로 드러낸다 — 조용히 넘어가는 게 이번 버그의 본질이었다.

> **규칙: MSW UI 텍스트 컴포넌트는 두 종류다.** `TextGUIRendererComponent`와 `TextComponent`가 화면 안에서 섞여 있을 수 있다. `nil` 가드만 두고 넘어가면 **아무 신호 없이 표시가 죽는다.** 새 텍스트를 만질 때는 `componentNames`로 실제 타입을 먼저 확인할 것 — 빌더로 `b.find(path).componentNames`를 찍으면 바로 보인다.

**검증**: 빌드 0건 · 5슬롯 전부 `히어로 / 비숍 / 신궁 / 섀도어 / 바이퍼`로 표시 · 스크린샷 확인.

**후속**: 이름이 글자 수에 따라 좌우로 밀렸다. `TextComponent.Alignment`가 **`5`(MiddleRight)** 라 오른쪽 끝에 맞춰져, 2글자(비숍·신궁)와 3글자(히어로·섀도어·바이퍼)의 시작점이 달랐다. **`3`(MiddleLeft)** 로 바꿔 전부 왼쪽에 맞췄다(5슬롯 patch, 다른 값은 손대지 않음). `TextAlignmentType`은 `UpperLeft=0 … MiddleLeft=3, MiddleCenter=4, MiddleRight=5 … LowerRight=8`.

### 2026-09-06 F2 — 보스 기믹 이펙트/사운드 복구

`EnemySkillManager:_ActivateGimmick`의 연출 블록이 **두 겹으로 틀려 있었다.**

1. `self._T.effectData[skillId]`는 **레코드 1개**인데 `for _, eff in ipairs(effects)`로 순회 → **루프 0회**.
2. 그 안에서 읽던 `eff.effectRUID`는 **스키마에 없는 필드**다(실제 이름은 `monsterEffectRUID` / `heroEffectRUID` / `projectileEffectRUID` / `screenEffectRUID`).

둘 다 nil 인덱싱이 아니라 "값이 없으니 아무것도 안 함"으로 끝나서 **로그도 오류도 0건**이었다. F1(배선)을 고친 뒤에도 게임 효과만 걸리고 화면은 조용했던 이유.

**수정**: 레코드를 직접 읽어 `soundRUID` + `screenEffectRUID`만 재생한다. 기믹은 특정 몬스터 슬롯이 아니라 **화면 전체에 거는 연출**이라 `monsterEffectRUID`(몬스터 스프라이트를 통째로 교체하는 필드)와 발사체는 쓰지 않는다. `PlaySkillEffects`에 위임하지 않은 것은 그쪽이 `screenEffectRUID`를 `skillCategory == "attack"`일 때만 재생하기 때문 — 기믹은 `buff`도 있다.

CSV 행이 없을 때 `기믹 연출 없음 skillId=… (EnemyEffectSet 행 미등록)` 로그를 남긴다. 앞으로 "이펙트가 안 난다"가 **데이터 미등록인지 코드 버그인지** 로그만 보고 구분된다.

**검증**: 임시 레코드를 런타임 주입 → `NotifyPhaseChange(2)` → `기믹 연출 재생 skillId=gimmick_phase2 sound=false screen=true` · `BattleRootPanel` 자식 **7 → 8**(t=0.6s) → **8 → 7**(t=2.4s 자동 소멸) · 빌드 오류 0건.

**남은 것**: `EnemyEffectSet.csv`에 `gimmick_phase2` / `gimmick_phase3` 행이 **없다.** 코드 경로는 살았지만 화면에는 아직 아무것도 안 난다 — 사용자가 RUID를 주면 행만 추가하면 된다.

### 2026-09-06 F3 — `hp_threshold` 페이즈 리셋

`EnemySkillManager._T.firedHpThresholds`는 `hp_threshold` 트리거의 **"이미 터졌음" 장부**다. 매 턴 중복 발동을 막는 게 목적인데, **`InitForBattle`(전투 시작)에서만** 비워졌다.

시즌보스는 페이즈가 넘어갈 때마다 **HP가 새로 가득 차고**(마지막 페이즈는 무한 재충전) 임계값을 다시 지난다. 그런데 장부가 그대로라 임계값 기믹이 **판당 딱 한 번**만 걸렸다 — 페이즈가 3개여도 1회.

**수정**: `NotifyPhaseChange` 맨 앞에서 장부를 비운다. 상태 소유자인 `EnemySkillManager` 안에 두어 `SeasonBossManager`가 남의 `_T`를 건드리지 않게 했다. `phase_change` 기믹은 별도 경로라 이 초기화의 영향을 받지 않는다.

**검증**: `EnemySkillSet.csv`에 `hp_threshold` 행이 0개라 임시 스킬(임계 50%, `shield_magic` 0.7)을 런타임 주입해 4단계로 확인 — ① HP 40% → `damageMult=0.7` 발동 ② 재차 40% → **로그 없음**(중복 차단 유지) ③ `페이즈 2 — hp_threshold 재무장` ④ HP 40% → `damageMult=0.7` **재발동**. 임시 스킬 제거 후 원래 2개 복귀, 빌드 오류 0건.

⚠️ **`damageMult` / `extraAtkMult`는 슬롯 1개씩이다 — 누적이 아니라 덮어쓰기.** 검증 중 `gimmick_phase2`의 0.5가 임계값 기믹의 0.7로 덮이는 게 그대로 재현됐다. 같은 `gimmickType`을 쓰는 기믹을 둘 이상 넣으면 **마지막에 발동한 쪽만 남는다.** 겹칠 수 있는 기믹을 설계할 때 주의(C4).

> 이 버그는 `EnemySkillSet.csv`에 `hp_threshold` 행이 0개라 **겉으로 드러나지 않는 잠복 상태**였다. 데이터가 비어 있어 안 터지는 코드 버그는 CSV에 행이 생기는 순간 살아나므로, 데이터 추가 전에 정리해 두는 편이 싸다.

### 2026-09-06 F4 — 보스전에 적 스킬 파이프라인 연결

`ExecuteEnemyAttack`은 맨 앞에 **보스 전용 분기**를 두고 기본 공격만 처리한 뒤 `return`했다. 그 아래에 있는 소환→버프→디버프→스킬공격 4단계에 **도달 자체를 못 해서**, `EnemySkillSet`에 `seasonboss_1`용 `every_n_turns` 스킬을 넣어도 한 줄도 돌지 않았다. 지금까지 보스 기믹 2개가 동작한 건 `phase_change`가 `SeasonBossManager`→`EnemySkillManager`로 직접 가는 **별도 경로**였기 때문이다.

**수정 — 보스를 "몬스터 1마리"로 위장시켜 같은 길로 보낸다.**

보스 분기를 통째로 지우고, `monsterAttacks` 목록을 만들 때 보스면 합성 레코드 하나를 넣는다:

```lua
{ monsterId = GetActiveBossId(), monsterIndex = 1, atk = GetBossAttackDamage(),
  attackTarget = "all", attackEffect = GetAttackEffectRUID() }
```

`monsterId`가 `EnemySkillSet.EnemyId`와 같은 값이라 `GetSkillsForMonster`가 그대로 먹는다. 이후 스킬/기본공격 분리, 4단계 실행, `skipMids`(스킬 쓴 턴엔 기본 공격 생략)가 전부 공짜로 따라온다.

곁들여 정리한 것:
- `GetSkillCasterSlot(info)` 신설 — 보스전은 `MonsterManager:GetSlotForMonster`가 항상 nil이라 연출 기준 슬롯을 보스 몸통으로 대체한다. 스킬 이펙트와 소환 이펙트 두 군데가 이걸 쓴다.
- `runBasicAttack(info)` / `endEnemyTurn()`을 지역 함수로 추출해 보스·일반이 공유. **턴 제한 검사를 `endEnemyTurn` 안에 넣은 게 핵심** — 스킬을 쓴 턴에도 빠짐없이 걸린다.
- 소환은 보스전에서 `log_warning` 후 건너뛴다(MonsterManager 미초기화라 슬롯이 없다).
- `healBlocked` 감소를 분기 밖으로 빼 보스전에도 적용. 예전 보스 분기는 이걸 건너뛰어 **보스가 `block_heal`을 걸면 영영 안 풀렸다.**
- `SeasonBossManager:GetActiveBossId()` 추가.

**검증**: 임시 `every_n_turns`(2턴) 광역 공격 스킬을 런타임 주입 —
턴 1·3 스킬 없음(기본 공격) · 턴 2·4 `[BM] 적 스킬 발동: [attack] 테스트광역 monsterId=seasonboss_1` · 턴 30에 스킬 발동 후 `[SeasonBoss] 제한 턴 소진 — 도전 종료`.
**일반 스테이지 회귀 확인**: 임시 스킬 제거 → `StartRun` → `isBossMode=false 몬스터수=4 enemySlots=4` → 적 턴 1회 → `현재턴=Player IsBattleOver=false`. 빌드·런타임 오류 0건.

⚠️ **보스 기본 공격의 데미지 분배 기준이 바뀌었다.** 예전 `ApplyDamageToHeroes`는 **전체 파티 인원수**로 나눠 죽은 영웅 몫이 허공에 날아갔다. 지금은 일반 몬스터와 같은 `ApplyDamageToTargetHeroes("all")`이라 **생존자 수**로 나눈다 — 영웅이 죽을수록 남은 영웅이 더 아프다. 일반 전투와 규칙이 통일된 것이지만 보스전 난이도는 후반부에 올라간다.

> 규칙: **특수 케이스를 전용 분기로 격리하면 그 아래 로직 전체가 조용히 스킵된다.** 데이터(`info` 레코드) 수준에서 일반화해 같은 경로로 흘려보내는 편이, 분기를 유지하며 기능을 하나씩 복사하는 것보다 싸고 안전하다. 이 파일에서 A6(이펙트)·F4(스킬)가 같은 뿌리의 증상이었다.

### 2026-09-06 시즌보스 첫 공격 스킬 — 파멸의 강타

F4로 파이프라인이 뚫린 뒤 넣은 **보스 첫 실전 스킬**. CSV 3개 + 코드 1곳.

| 파일 | 추가 |
|---|---|
| `EnemySkillSet` | `5,seasonboss_1,balrog_smash,파멸의 강타,attack,every_n_turns,3,all,,,…` |
| `EnemySkillEffectSet` | `balrog_smash,1,damage_all,2.0,0,0,` |
| `EnemyEffectSet` | `3,balrog_smash,<보스 이펙트>,1.0,1.0,<영웅 피격>,1.0,,,,,1.0,<사운드>,` |

3턴마다 기본 공격력 500 × 2.0 = **1000을 생존 영웅에게 균등 분배**.

**`MonsterEffectRUID`가 보스에서 안 먹던 문제도 같이 고쳤다.** 이 필드는 원래 **몬스터의 스프라이트를 이펙트 클립으로 갈아끼우는** 방식이다. 그런데 보스 몸통(`BossOverlay/BossSlot`)은 파츠 4개를 얹는 **컨테이너라 `SpriteGUIRendererComponent`가 없다**(실측: `UITransformComponent`만 보유). 그래서 `renderer ~= nil` 가드에 걸려 **오류 없이 아무 일도 안 났다.** 렌더러가 없으면 `_Effect:PlayEffect`로 슬롯 위에 이펙트를 얹도록 `else` 분기를 추가했다.

> 보스는 그림이 깨질 걱정이 없다 — 갈아끼울 렌더러 자체가 없으니 예전 코드는 그냥 무시했을 뿐이다.

**`PlaySkillEffects`에 진단 로그를 상설화했다.** "이펙트가 안 보인다"가 ① 데이터 미등록 ② 대상 슬롯 없음 ③ RUID 문제 중 무엇인지 한 줄로 갈린다:
```
[EnemySkillManager] 연출 재생 balrog_smash [attack] 적슬롯=true 영웅슬롯=5 monster=true hero=true sound=true
```

**검증**: 위 로그가 실제 전투 경로에서 그대로 출력 — 영웅슬롯 5개 확보, 3종 RUID 모두 등록. 보스 몸통 자식 **4 → 5 → 4**(이펙트 부착·소멸).

> ⚠️ **검증 함정**: `ExecuteEnemyAttack`은 `wait()`로 블로킹하므로, 호출 직후 건 타이머의 **첫 발화가 3초 뒤**다(실측 07:36:13 → 07:36:16). 그 사이에 끝나는 짧은 이펙트는 자식 수 폴링으로 못 잡는다 — 한 번 "히어로 이펙트가 안 붙는다"고 오판했다. **블로킹 메서드 안쪽 사건은 폴링 말고 그 메서드 안의 로그로 증명할 것.**

**남은 것**: `BossSet.AttackEffectRUID`(기본 공격 연출)는 여전히 비어 있다. 버프·디버프 스킬은 사용자 요청으로 보류.

⚠️ **기존 구조의 어색함 하나** — 히어로 피격 이펙트의 대상이 **데미지 적용 후 생존자**로 계산된다(`applySkillEffects`가 `ApplyDamageToTargetHeroes` 뒤에 `GetTargetHeroIds`를 다시 부른다). 그래서 **그 공격에 맞아 죽은 영웅에게는 피격 이펙트가 안 뜬다.** 일반 몬스터 전투도 동일. 대상 목록을 데미지 전에 한 번만 잡으면 해결된다.

### 2026-09-06 시즌보스 스킬 3종 + 기본 공격 연출 (파츠별 이펙트)

**새 요구: 스킬마다 이펙트가 붙는 보스 파츠가 다르다.** 기존 구조는 시전자 슬롯 하나만 쓸 수 있어서 `EnemyEffectSet`에 **`MonsterEffectTarget` 열을 신설**했다 — 슬롯의 직계 자식 이름(`BossPart0~3`)을 적으면 거기에 건다. 비면 종전대로 슬롯 전체(일반 몬스터는 항상 공란).

| SkillId | 카테고리 | 트리거 | 대상 | 효과 | 파츠 |
|---|---|---|---|---|---|
| `seasonboss_1_basic` | (기본 공격) | 매 턴 | all | 500 | BossPart0 |
| `balrog_smash` 파멸의 강타 | attack | 3턴 | all | `damage_all` ×2.0 | BossPart0 |
| `balrog_seal` 회복 봉인 | debuff | 5턴 | all | `block_heal` 2턴 | BossPart1 |
| `balrog_pierce` 심연의 일격 | attack | 7턴 | front | `damage_single` ×4.0 | BossPart2 |

겹치는 턴이 고비가 되도록 주기를 3·5·7로 잡았다 — 15턴(강타+봉인) · 21턴(강타+일격) · 30턴(강타+봉인, 마지막 턴).

**보스도 일반 몬스터와 같은 "스프라이트 교체" 방식이다.** `MonsterEffectRUID`는 대상의 스프라이트를 이펙트 클립으로 **갈아끼우는** 필드이고, 보스 파츠도 그대로 덮어쓴다 — 새 이펙트 엔티티를 만들지 않는다.

> 처음에는 "보스 그림이 사라진다"고 판단해 이펙트 엔티티를 위에 얹는 방식으로 만들었으나, **덮어쓰기가 의도된 연출**이었다(사용자 확인). 되돌리는 게 맞다.

다만 **보스는 원본으로 되돌려줄 주체가 없다.** 일반 몬스터는 `MonsterManager`의 공격·피격 애니가 끝나면서 `spriteRUID`로 복원해주지만, 보스 파츠는 `_SetupBossVisuals`가 전투 시작 때 한 번 칠할 뿐이다. 그래서 **파츠를 지정한 경우에만** 교체 직전 `renderer.ImageRUID`(와 `AnimClipPlayType`)를 캡처했다가 `MonsterDuration` 뒤에 `SetTimerOnce`로 직접 복원한다. 복원이 없으면 보스가 이펙트 그림인 채로 굳는다.

**기본 공격에도 시전자 연출을 붙였다.** 합성 `info`에 `effectSkillId = <bossId> .. "_basic"`을 넣고, `runBasicAttack`이 그 값이 있을 때만 `PlaySkillEffects`를 부른다(일반 몬스터는 nil이라 무영향). 영웅 피격은 기존 `BossSet.AttackEffectRUID`가 담당하므로 `_basic` 행의 `HeroEffectRUID`는 비워 **중복 재생을 피한다.**

**맞아 죽은 영웅에게 피격 이펙트가 안 뜨던 문제도 고쳤다.** `applySkillEffects`가 `ApplyDamageToTargetHeroes` **뒤에** `GetTargetHeroIds`를 다시 불렀는데, 이 함수는 **생존자만** 돌려준다. 그래서 그 공격에 쓰러진 영웅이 목록에서 빠졌다. 대상을 데미지 적용 **전에** `hitTargetIds`로 잡아두고 연출에서 재사용하도록 바꿨다. 일반 몬스터 전투에도 함께 적용된다.

**검증** (턴 1·3·5·7 순차 실행):
```
턴1 연출 재생 seasonboss_1_basic [attack] 영웅슬롯=5 monster=true hero=false sound=true
턴3 적 스킬 발동: [attack] 파멸의 강타 → 영웅슬롯=5 monster/hero/sound 전부 true
턴5 적 스킬 발동: [debuff] 회복 봉인 → 회복 봉인 적용: 2턴 · 영웅슬롯=0(디버프라 정상)
턴7 적 스킬 발동: [attack] 심연의 일격 → 영웅슬롯=1 (단일 대상)
턴7 healBlocked 감소 → 1        (F4에서 분기 밖으로 뺀 처리가 보스전에서 실동작)
```
파츠 교체·복원 (세 스킬 동시 재생, `ImageRUID.DataId`로 실측):
```
원본    P0=5876ee61 P1=cab5fec6 P2=53685aca
t=0.25s P0=0c53ca97 P1=20ab96eb P2=0b287faf  자식=000   ← 각자 자기 이펙트로 교체
t=1.00s P0=0c53ca97 P1=20ab96eb P2=0b287faf  자식=000
t=1.25s P0=5876ee61 P1=cab5fec6 P2=53685aca  자식=000   ← MonsterDuration(1.0) 뒤 원본 복귀
```
자식수가 내내 0이라 **새 엔티티를 만들지 않는 것**도 함께 확인됐다. 파츠 미탐색 경고 0건.

> ⚠️ **`DataRef`는 `tostring()`으로 비교하면 안 된다** — 전부 `"MOD.Core.MODDataRef"`(타입 이름)로 찍혀 서로 달라도 같아 보인다. RUID를 읽으려면 **`ImageRUID.DataId`**를 쓸 것. 한 번 이걸로 "교체가 안 된다"고 오판했다.
죽은 영웅 피격: 선두 HP를 1로 만들고 심연의 일격 → **`영웅슬롯=1`** 유지 + 생존 5→4명. 예전 코드였다면 0이 됐다.

**남은 것**: 버프·디버프 스킬의 화면(Screen) 연출은 여전히 `skillCategory == "attack"`에서만 재생된다. 회복 봉인은 보스 파츠 이펙트 + 사운드로 충분해 지금은 문제가 없지만, 화면 전체 연출이 필요해지면 그 조건을 풀어야 한다.

### 2026-09-06 보스 이펙트가 중간에 끊기던 문제 — 복원 시점과 연출 템포를 분리

**증상 (사용자)**: 보스 이펙트가 다 재생되지 않고 중간에 끊긴다.

**원인은 바로 앞 커밋에서 내가 넣은 복원 타이머다.** 파츠 스프라이트를 원래 그림으로 되돌리는 시각을 `MonsterDuration`으로 잡았는데, 이 필드는 원래 **연출 템포**(`GetSkillEffectDuration` → `BattleManager`의 `wait(dur)`)이지 클립 길이가 아니다. CSV 값이 전부 `1.0`이라 실제 클립을 1초에서 잘라버렸다.

실측 클립 길이 vs 당시 복원 시점:

| 이펙트 | 실제 길이 | 잘린 시점 | 재생된 비율 |
|---|---|---|---|
| 기본공격 보스 | 2.04초 | 1.0초 | 49% |
| 강타 보스 | 3.21초 | 1.0초 | 31% |
| 일격 보스 | 3.60초 | 1.0초 | 28% |
| 봉인 보스 | 5.55초 | 1.0초 | **18%** |

**수정**: 두 값을 분리했다.
- **복원 시점 = 클립의 실제 길이 ÷ PlayRate.** `_LoadData`가 로딩 때 `LoadAnimationClipAndWait`로 프레임 Delay를 합산해 `_T.clipLen`에 캐시한다(전투 중에 부르면 `...AndWait`가 프레임을 멈춘다). 클립을 못 읽으면 `MonsterDuration`으로 폴백.
- **`MonsterDuration`은 연출 템포 전용**으로 남긴다.

곁들여 **`monsterPlayRate`를 실제로 적용**했다 — 스프라이트 교체 경로가 CSV의 PlayRate를 읽고도 렌더러에 안 넣고 있었다(`Effect:SpawnEffect`는 넣는다). 복원 시 원래 PlayRate도 같이 되돌린다.

**검증**: 로딩 로그가 6개 클립 길이를 자동 출력(`0c53ca97 = 3.21초` 등). 세 스킬 동시 재생 후 원본 복귀 시점 —
```
P0 원본 복귀 t=3.25s  (클립 3.21s)
P2 원본 복귀 t=3.75s  (클립 3.60s)
P1 원본 복귀 t=5.50s  (클립 5.55s)
```
0.25초 샘플링 오차 범위. 셋 다 끝까지 재생 후 복귀.

> 규칙: **한 CSV 필드에 두 의미를 겹쳐 쓰지 말 것.** `MonsterDuration`은 "다음 동작까지 대기"였는데 "이펙트 길이"로도 재사용하자마자 어긋났다. 클립 길이처럼 **리소스에서 계산할 수 있는 값은 데이터로 받지 말고 직접 재는 편이** 틀리지 않는다.

⚠️ **남은 템포 이슈(버그 아님)**: `MonsterDuration`이 여전히 1.0이라, 공격 스킬 후 `wait(1.0)`만 하고 다음 동작이 이어진다. 이펙트는 끝까지 재생되지만 **다 보기 전에 턴이 넘어간다.** 이펙트를 끝까지 보여주고 싶으면 `EnemyEffectSet.MonsterDuration`을 클립 길이에 맞춰 올리면 된다(강타 3.2 / 일격 3.6 등). 전투 속도와의 취향 문제라 그대로 뒀다.

### 2026-09-06 이펙트 잔상 버그 (파츠 세대 관리) + PlayRate 2.0

**증상 (사용자)**: "기본 공격과 스킬이 둘 다 나오는 것 같다."

**코드상 턴은 이미 배타적이다.** `skipMids`가 스킬을 쓴 적을 기본 공격 목록에서 빼므로 한 턴에 둘 다 실행될 수 없다(턴 1~7 실측: 1·2·4는 basic만, 3·6 강타만, 5 봉인만, 7 일격만). 사용자가 본 건 **이전 턴 이펙트가 다음 턴까지 남아 겹친 것**이었다.

**진짜 버그 — 복원 원본이 '이전 이펙트'로 오염된다.**

파츠 스프라이트를 교체할 때 그 직전 `ImageRUID`를 원본으로 캡처했는데, 이전 이펙트가 **아직 재생 중이면 그 값이 이펙트 RUID**다. 클립 3.21초 vs 턴 간격 2초라 실제로 겹쳤다:

```
t=0      강타 재생.       prevRUID = 원본       타이머 3.21s 예약
t=2      기본공격 재생.   prevRUID = 강타 이펙트 ← 오염!   타이머 2.04s 예약
t=3.21   강타 타이머 발화 → 원본 복원 (기본공격 이펙트가 도중에 지워짐)
t=4.04   기본공격 타이머 → **강타 이펙트로 복원** ← 보스가 이펙트 그림으로 굳는다
```

**수정 2가지**
- `_T.partOrig[파츠명]` — 진짜 원본을 파츠별로 **딱 한 번만** 기억한다. 이후 교체는 이 값을 건드리지 않는다.
- `_T.fxGen[파츠명]` — 재생마다 증가하는 세대 카운터. 복원 타이머는 자기 세대가 최신일 때만 복원한다. 낡은 타이머가 새 이펙트를 지우지 못한다.

둘 다 `InitForBattle`에서 초기화한다(보스가 바뀌면 파츠 그림도 바뀌므로).

**PlayRate**: `EnemyEffectSet.MonsterPlayRate`를 보스 이펙트 4종만 **2.0**으로 올렸다(일반 몬스터는 1.0 유지). 복원 시점이 `클립 길이 ÷ rate`라 자동으로 따라간다 — 강타 3.21→1.6초, 봉인 5.55→2.8초. `MonsterDuration`도 그에 맞춰 조정(1.6 / 2.8 / 1.8 / 1.0).

**검증** — 겹침 재현 테스트(강타 재생 → 0.75초 뒤 기본공격 덮어쓰기, BossPart0 실측):
```
t=0.2~0.8s  P0=0c53ca97 (강타)
t=1.0~1.8s  P0=e775807e (기본공격이 덮음)   ← 1.6s 강타 타이머가 발화해도 안 지움
t=2.0s~     P0=5876ee61 (진짜 원본 복귀)     ← 예전엔 여기서 강타 이펙트로 굳었다
```
턴 배타(턴 1~7): 모든 턴에 basic 또는 스킬 **중 하나만** 출력.

> 규칙: **"직전 상태를 원본으로 캡처"는 재진입이 있으면 깨진다.** 같은 대상에 연출이 겹칠 수 있으면 ① 원본은 최초 1회만 기억하고 ② 세대 카운터로 낡은 콜백을 무효화할 것. `MonsterManager`의 `battleGen`과 같은 패턴이다.

⚠️ **3턴·5턴·7턴 주기가 겹치는 턴에는 스킬이 둘 이상 발동한다**(15턴 = 강타+봉인, 21턴 = 강타+일격, 30턴 = 강타+봉인). 파츠가 달라 화면은 겹치지 않지만, 한 턴에 하나만 원한다면 우선순위 규칙이 필요하다.

### 2026-09-06 같은 턴에 겹친 스킬의 연출 순차화

**증상 (사용자)**: 주기가 겹쳐 스킬이 동시에 나가면 이펙트가 겹쳐 사라진다.

**원인**: 4단계 파이프라인에서 **공격만 순차**였고 버프·디버프는 "전체 동시 발동 후 `wait(1.0)` 한 번"이었다. 봉인 연출이 2.8초인데 1초 뒤 다음 단계로 넘어가 겹쳤다.

**수정**
- 버프·디버프도 **하나씩 순차** 실행하고 각자의 연출 길이만큼 대기하도록 바꿨다(`waitForSkillEffect` 지역 함수로 세 단계가 공유).
- `GetSkillEffectDuration`이 **클립 실제 길이 ÷ PlayRate**를 폴백으로 쓰도록 했다. `MonsterDuration`이 있으면 그 값(명시적 오버라이드), 비었으면 자동 산출 — **CSV를 비워둬도 연출이 끝날 때까지 기다린다.**
- CSV `MonsterDuration`을 클립보다 약간 크게 잡아 연출 사이에 여유를 뒀다(강타 1.8 / 봉인 3.0 / 일격 2.0 / 기본 1.2 — 클립은 각 1.6 / 2.8 / 1.8 / 1.02초).

**검증** — 턴 15(강타 3턴 주기 + 봉인 5턴 주기가 겹치는 턴), 파츠 RUID 실측:
```
t=0.3s   P1=cab5fec6(원본)  → [debuff] 회복 봉인 발동
t=0.6s   P1=20ab96eb        ← 봉인 이펙트 시작
t=0.9~3.0s  P1=20ab96eb 유지 ← 3.0초 온전히 재생
t=3.3s   P1=cab5fec6(복귀)  → [attack] 파멸의 강타 발동   ← 봉인이 끝난 뒤
t=4.8~6.0s  P0=0c53ca97     ← 강타 이펙트
t=6.3s   P0=5876ee61(복귀)
```
두 스킬 모두 게임 효과는 적용되고(회복 봉인 2턴 + 강타 데미지), 연출만 순서대로 나온다.

> ⚠️ 부작용: 일반 몬스터 전투에서도 버프·디버프가 순차가 된다. 그런 스킬을 쓰는 몬스터가 여러 마리면 적 턴이 그만큼 길어진다(현재 데이터에는 `lantern_block_heal` 하나뿐이라 체감 영향 없음).

### 2026-09-06 공격 스킬 턴에 피격 이펙트가 두 개 겹치던 문제

**증상 (사용자)**: 전체 공격 스킬과 기본 공격의 이펙트가 겹쳐 보인다, 특히 피격 이펙트가 같이 나온다.

**원인**: `applySkillEffects`의 attack 분기가 **스킬 처리인데도** `info.attackEffect`(= 보스 **기본 공격**의 피격 이펙트 `BossSet.AttackEffectRUID`)를 재생하고, 그 아래 `PlaySkillEffects`가 스킬 자신의 `HeroEffectRUID`를 또 재생했다. 영웅 슬롯에 **피격 이펙트가 2개** 붙어 서로 뭉개졌다.

`info.attackEffect`는 원래 **`HeroEffectRUID`가 없는 스킬을 위한 폴백**이었는데, 있는 스킬에도 무조건 재생하고 있었다.

**수정**: `EnemySkillManager:HasHeroEffect(skillId)`를 추가하고, 스킬이 자기 피격 이펙트를 가지면 기본 공격 이펙트를 **생략**한다. 없는 스킬만 종전처럼 폴백한다.

**검증** (HeroSlot1 자식 수 최대값, 기준 4):
| 턴 | 결과 |
|---|---|
| 턴 3 파멸의 강타 (HeroEffectRUID 있음) | **증가분 1** — 강타 피격만 |
| 턴 1 기본 공격 (`_basic` 행은 HeroEffectRUID 공란) | **증가분 1** — `BossSet.AttackEffectRUID` 폴백 정상 |

수정 전이라면 강타 턴이 2였다.

> 규칙: **폴백 값은 "주 값이 없을 때"만 써야 한다.** 무조건 재생해두고 그 위에 진짜 연출을 덧그리면, 값이 둘 다 있는 경우에 조용히 겹친다. 일반 몬스터 전투도 같은 코드라 함께 고쳐졌다.

### 2026-09-06 페이즈 전환을 "턴 종료 시점"으로 옮김 + 무한 페이즈 표기

사용자 요청 3건.

**1. 페이즈 HP가 소진되면 그 턴의 남은 공격은 피해를 주지 않는다.**
예전에는 `ApplyBossDamage`가 HP ≤ 0인 순간 **즉시** 다음 페이즈로 넘기고 초과 딜을 이월했다. 그래서 한 방으로 페이즈를 건너뛸 수 있었고, 같은 턴의 남은 영웅 공격이 다음 페이즈 HP를 깎았다.
지금은 `_T.phaseDrained` 플래그를 세우고 HP를 0에 고정한다. 이후 공격은 **연출만 나가고 피해·딜량 0**(`ApplyBossDamage`가 맨 앞에서 return). **HP를 넘긴 초과분도 버린다** — `applied = min(finalDmg, currentHp)`.

> 랭킹 점수(누적 딜량)에 반영되는 건 **실제로 깎은 만큼**이다. 페이즈 HP 총합이 한 사이클의 상한이 되고, 그 이상 넣으려면 턴을 더 써야 한다.

**2. 적 턴까지 끝나면 경고 → 페이즈 전환 → HP 리필.**
`SeasonBossManager:OnEnemyTurnEnd()` 신설. `BattleManager`의 `endEnemyTurn`이 **턴 제한 검사 뒤에** 호출한다(제한에 걸려 끝나는 턴이면 전환하지 않는다). 경고 연출은 `_EnterPhase`가 이미 켜는 `BossOverlay/Warning` 엔티티가 담당한다(토스트 아님 — 사용자 지정).

**3. 무한 페이즈 표기.** 최종 페이즈를 한 번이라도 재충전하면 `_T.isEndless = true`가 되고 `_RefreshPhaseText`가 `"Phase ∞"`를 쓴다. 실기기 렌더링 확인 완료.

**검증** (P1 HP 10000 기준):
```
8000 타격      HP=2000  누적딜=8000
8000 더        HP=0 소진=true 누적딜=10000   ← 2000만 인정, 6000 버림
5000 ×2        HP=0 소진=true 누적딜=10000   ← 완전히 무시
적 턴 종료      P2 HP=20000/20000 Warning=true 텍스트='Phase 2'
… P3 소진 후 턴 종료 → 돌파=1 무한=true 텍스트='Phase ∞'
… 한 번 더    → 돌파=2, HP 다시 25000
```

⚠️ **`PhaseText` 폭이 좁아 "Phase ∞"가 두 줄로 줄바꿈된다.** 글리프는 정상 표시되므로 Maker에서 폭만 넓히면 된다(스크린샷 확인).
⚠️ 테스트 중 재확인된 것: **`shield_magic`의 `damageMult=0.5`는 한 번 걸리면 전투 끝까지 유지된다.** P2 진입 후에는 P3·무한 구간에서도 계속 절반이라, 페이즈 HP를 실제로 깎으려면 표기 수치의 2배가 필요하다.

### 2026-09-06 스테이지 무한 확장 대비 — B 블록 (B1·B2·B3·B4)

목표가 **25층 → 1000~2000 스테이지**로 바뀌면서(그룹 5개를 돌려쓰고 스탯만 올리는 방식), 그걸 막고 있던 구조 4건을 열었다.

**B1 그룹 순환 + 셔플.** `GameManager`의 `groupId=1` 하드코딩을 `StageManager:GetGroupIdForStage(stage)`로 교체. `GroupSet.csv`에 **`StageSpan`**(그룹이 담당할 스테이지 수) 열을 넣고, **그룹 수는 GroupSet 행 수를 그대로 쓴다** — 1행이면 항상 그 그룹, 5행이면 5그룹 순환. 첫 바퀴는 CSV 순서대로, 2바퀴째부터 **바퀴 번호를 시드로 하는 결정론적 Fisher-Yates**로 순서를 섞는다.
> 저장이 필요 없다 — 같은 스테이지는 언제 들어와도 같은 그룹이 나오고 모든 유저가 같은 순서를 본다. `math.random`은 전역 시드를 오염시키므로 LCG를 직접 돌렸다.

**B2 층수 하드코딩 제거.** `CompleteFloor`/`NextFloor`의 `>= 5` → `GetFloorCountForGroup(groupId)`(StageSet의 최대 FloorNum, 없으면 5).
⚠️ `absoluteFloor = (stage-1)*5 + floor`는 **아직 5층 고정 전제**다. 난이도 입력을 스테이지 번호로 바꾸면 해소된다.

**B3 스탯 공식 단일화 + CSV화.** `Init`·`SummonMinions`에 복붙돼 있던 공식을 `CalcBaseStats(floor)` 하나로. **`StageCurveSet` 신설**(공식 설정 1행): `HpBase/HpPerFloor/HpGrowth · Atk… · Def…`. `Growth`는 지수 항. 신설 시점엔 1.0(선형)이었고 **2026-09-06에 1.001605로 켜졌다** — 아래 "2026-09-06 밸런스" 절 참조.

**B4 저장 시점·빈도.** 🔴 **`OnEndPlay`에서의 DataStorage 저장을 3곳 모두 제거** — MSW 공식 가이드가 금지하는 패턴이었다(재입장 시 이전 저장이 제대로 로드되지 않을 수 있음). `UserLeaveEvent` 핸들러로 옮기고, 랭킹 제출은 **대기열 + 60초 주기 flush**로 묶었으며, 서버 측 최고점 캐시(`_KnownBest`)로 중복 제출 시 저장소 접근을 0으로 만들었다.
- 검증: `RequestSubmit` 3회 → 접근 0회 · flush 1회에 `갱신 없음 (기존 42 >= 3)` · stop 시 `퇴장 저장` 로그(`HandleUserLeaveEvent` 스택)

> ⚠️ 이 시점의 기록이다. 2026-09-06 성장 곡선이 **×1.5 → ×517**로 개편되고 기본 스탯도 ÷13 됐다 — 아래 "2026-09-06 밸런스" 절이 현행이다.

### 2026-09-06 다음 층 몬스터의 idle이 1회만 재생되던 문제

**증상 (사용자)**: 몬스터를 잡고 다음 층으로 넘어가면 애니메이션이 1회만 재생되고 멈춘다.

**원인**: `AnimClipPlayType`이 **슬롯에 남는다.** 사망 연출(`ImageRUID=dieRUID` + `Onetime`)과 공격 연출이 이 값을 `Onetime`으로 바꿔놓는데, 공격은 애니 종료 이벤트에서 `Loop`로 되돌리지만 **사망은 되돌리지 않는다**(죽었으니 되돌릴 이유가 없어 보였다). 그런데 **다음 층이 같은 슬롯 엔티티를 재사용**하고, `ApplySpriteToSlot`은 `ImageRUID`와 `Color`만 바꿀 뿐 `AnimClipPlayType`을 건드리지 않았다 → 새 몬스터의 idle이 `Onetime`으로 재생됐다.

**수정**: `ApplySpriteToSlot`이 `AnimClipPlayType = Loop`를 함께 설정한다. 호출처 3곳(스폰·부활·신규 소환)이 전부 idle 스프라이트(`md.spriteRUID`)를 넣는 자리라 항상 Loop가 맞다.

**검증**: 1층 진입 직후 `Loop` → 사망 연출을 흉내내 `Onetime` 강제 → 2층 진입 후 **`Loop` 복귀**.

> 규칙: **재사용되는 엔티티에 남는 렌더러 상태를 조심할 것.** `ImageRUID`만 갈아끼우면 `AnimClipPlayType`·`PlayRate`·`Color`·`FlipX` 같은 값은 이전 용도 그대로 남는다. 스프라이트를 바꾸는 함수는 **자기가 기대하는 재생 상태를 함께 세팅**해야 한다.

### 2026-09-06 츄릅나무 스킬 + 스킬 몬스터의 idle 복원

**그룹2 보스 츄릅나무에 스킬 1종 추가** — `churup_smash` 대지의 진노: 3턴마다 전체 `damage_all` ×2.5. 팩 `mob/8642016.img`의 `attack2` 세트를 그대로 씀(`attack2` / `attack2/info/hit` / `_audio/Attack2`). `MonsterDuration`을 **0으로 두어 클립 길이(2.16초)가 자동 산출**되게 했다.

**같이 고친 것 — 스킬만 쓴 턴에는 몬스터 스프라이트가 복원되지 않았다.**

일반 몬스터의 스킬 연출은 **자기 스프라이트를 갈아끼우는** 방식인데, 그 복원은 `PlayAttackAnimFiltered`의 애니 종료 콜백이 맡는다. 이 콜백의 복원 루프는 `skipMids` 필터 **없이 살아있는 몬스터 전부**를 되돌리므로, 다른 몬스터가 기본 공격만 하면 스킬을 쓴 몬스터도 함께 복원된다 — **아르마가 멀쩡했던 이유**다.

빈틈은 **`#targets == 0`인 턴**뿐이었다. 그러면 콜백 자체가 걸리지 않아 복원 루프가 돌지 않는다. **단독 보스가 스킬을 쓰는 턴**이 정확히 이 경우다(츄릅나무 · 3턴마다).

- 복원 루프를 `RestoreAllIdleSprites()`로 추출하고, `#targets == 0` 조기 반환 경로에서도 호출하게 했다. 스킬 연출은 호출부가 `wait`으로 끝까지 기다린 뒤라 여기서 되돌려도 잘리지 않는다.
- 검증: 보스층 idle `26cc1737/Loop` → 스킬 발동 → `2315670e/Onetime`(1.6~3.2s) → **`26cc1737/Loop` 복원**(3.6s).

> ⚠️ 처음엔 `BattleManager`에 별도 복원 타이머를 넣으려 했으나, **사용자가 "아르마는 멀쩡한데 기존 코드부터 확인하라"고 지적**해 다시 읽었더니 이미 복원 경로가 있었다. 기존 구조에 빠진 한 갈래만 메우는 편이 타이머를 새로 도는 것보다 안전하다(공격 애니와 경합하지 않는다).
> 규칙: **"복원이 안 된다"고 판단하기 전에 그 복원을 이미 누가 하고 있는지 먼저 찾을 것.** 비슷하게 동작하는 기존 사례(아르마)가 멀쩡하다면 경로가 이미 있다는 신호다.

### 2026-09-06 밸런스 — 기본 스탯 하향 + 난이도 곡선 재설계 (Phase 3 G3)

**사용자 지시**: "기본 공격력 및 체력을 낮춰서 500배가 되어도 딜이 너무 높지 않도록 / 몬스터 체력 및 공격력 성장 곡선도 이를 맞춰서 500스테이지에 풀강 영웅들과 맞먹을 수 있도록, 물론 500스테이지 이상도 갈 수 있어야."

**영웅 기본 스탯 ÷13** (역할 비중 유지) — 히어로 70/1,150 · 비숍 55/900 · 신궁 105/600 · 섀도어 100/600 · 바이퍼 80/750. 파티 합 ATK 410 / HP 4,000. ★5 25강 신궁 ATK **698,328 → 54,314**.

#### 진단이 두 번 뒤집혔다

**1차 — DEF를 빼고 셌다.** "`StageCurveSet`이 선형이라 500스테이지까지 ×215뿐 → 너무 쉬워진다"고 적었는데, 데미지 공식의 `10/(10+DEF)`는 곧 **유효 체력 배수**다:

```
옛 곡선 2500층: HP 300,800 · DEF 3,750  →  유효 113,100,800  (1층 대비 ×80,000)
```

`DefPerFloor`가 HP와 같은 자릿수로 커지면 곡선이 **2차식**이 된다. 500스테이지는 쉬운 게 아니라 **불가능**했다.
> 규칙: **난이도 곡선을 볼 때는 HP 단독이 아니라 `HP × (10+DEF)/10`으로 볼 것.** DEF는 조용한 2차항이다.

**2차 — 콤보를 눈대중으로 잡았다.** DEF를 눕히고 다시 짠 곡선도 틀렸다. **사용자 지적: "플레이어는 평균 한 턴에 7-10콤보까지 진행한다."**
`comboExp = 1 + (콤보-1)×0.25`가 곱으로 들어가고(8콤보 = ×2.75), 콤보가 많으면 `heroFiring[].atkMult`가 그룹마다 **누적**되며 5인 전원이 발동한다. 추정 500K/턴 → **실측 3,451,690/턴**, **7배** 오차.
> 규칙: **밸런스 곡선은 추정하지 말고 `ExecutePlayerAttack`을 실제로 돌려 실측할 것.** 곱연산이 4~5겹이라 눈대중은 배수 단위로 빗나간다.

#### 실측 방법

1. 클라이언트 `_HeroDataManager._T.inventoryData`를 `{heroes={id=14}, starforce={id=25}}`로 임시 덮어써 ★5 25강 파티를 만든다 (**표시용 캐시라 서버 데이터는 안 건드린다**. 끝나면 `RequestInventoryData()`로 원복).
2. `mm._T.monsters[i]`의 `hp/maxHp`를 1e12로, `def`를 목표 층 값으로 세팅 — **오버킬 손실 없이** 총 피해를 잰다.
3. 합성 `MatchResult`(`TotalComboCount=8`, `MatchGroups` = 6원소 8그룹)를 `bm:ExecutePlayerAttack`에 직접 넣는다.
4. 20초 뒤 `BIG - m.hp` 합계를 읽는다.

⚠️ **전투 시작 직후에 쏘면 안 된다.** `_Queue`가 전투 인트로 시퀀스와 겹쳐 **힐 스텝만 실행되고 공격 스텝이 통째로 잘린다**(피해 0으로 보임). 전투가 안정된 뒤에 쏠 것.

| 파티 | 8콤보 1턴 총 피해 |
|---|---:|
| ★1 0강 | **1,916** |
| ★5 25강 | **3,451,690** |

실전력 성장 **×1,801** = 스탯 ×517 × 성급 스킬 ×3.5. ★1은 tier 0 하나뿐이라 **단일 대상만** 때리고, ★5가 되어야 광역(콤보 데스폴트·제네시스·하울링 피스트)과 최종배율 2배(볼스 아이·쉐도우 파트너)가 열린다.

#### 새 곡선 (`StageCurveSet` 1행)

```
HP  = (1350 + 절대층×45)  × 1.002007^절대층
ATK = (  30 + 절대층×0.2) × 1.002007^절대층
DEF =    2 + 절대층×0.01
```

| 스테이지 | HP | ATK | DEF | 층 클리어 |
|---:|---:|---:|---:|---:|
| 1 | 1,590 | 31 | 2 | **3.3턴** (★1 0강) |
| 100 | 64,992 | 354 | 7 | |
| 300 | 1,393,276 | 6,678 | 17 | |
| **500** | **17,108,889** | **79,646** | **27** | **19.8턴** (★5 25강) |
| 600 | 55,836,949 | 257,992 | 32 | 64.7턴 |

- **"맞먹는다"의 근거**: 500스테이지 층 클리어 19.8턴인데 적 화력이 382,300/턴이라 파티 HP 2,069,120은 **5.4턴이면 소진**된다. 비숍 힐(417,005/턴)이 받쳐줘야 겨우 20턴 — 힐이 몇 번 빗나가면 진다.
- **지수항을 쓴 이유**: 선형이면 500 이후 난이도가 정체해 "500 이상도 갈 수 있어야"가 무의미해진다. 500→600 **×3.3**, 500→1000 **×299**.
- ⚠️ **`HpGrowth`는 2500제곱이라 소수점 4째 자리만 건드려도 종점이 배로 바뀐다.**
- 🔴 **`MonsterManager`에 같은 값의 폴백이 하드코딩돼 있다.** CSV만 고치면 로드 실패 시 조용히 옛 곡선으로 돌아간다.

**`BossSet` 시즌보스** — 300,000 / 600,000 / 900,000 · 공격력 3,000. 실측 딜 기준의 **잠정값**이고 확정은 A5.

**남은 것**
- **G4 큰 수 표기(K/M/B)** — 몬스터 HP 1,710만 · 1턴 피해 345만. 지금 표기로는 못 읽는다. 필수.
- **G5 중반 페이싱** — 10턴 클리어 기준 필요 전력배수 100스테이지 ×14 / 200 ×72 / 300 ×291 / 400 ×1,052. 300에서 ★5(15장)를 요구하는데 가챠 수급과 맞는지 미검증.
- **G6 보스층 HPMult** — 500스테이지 보스층이 **51턴**(일반 층의 2.6배). `HPMult 5.0`인데 보스는 1마리라 **광역 딜이 통째로 낭비**된다. 2.5면 25턴. 사용자 결정 필요.

### 2026-09-06 보스 난이도 — 스테이지 보스 HP 하향 + 시즌보스 무한 페이즈 가속 (Phase 3 G6·G7)

**사용자 지시**: "보스 체력을 낮추자, 그리고 시즌 보스는 3등급에 12성 이상이면 페이즈 3까지 잡을 수 있으면 좋겠어, 단 무한 페이즈에서는 턴 마다 보스의 공격력이 점점 올라가서 30턴을 모두 버티기 어렵도록 만들자, 30턴을 버티려면 5등급에 25강을 해야 버티도록"

#### 스테이지 보스층 (G6)

`MonsterSet`의 Boss 4행(`arma`·`churup_tree`·`weakened_harmony_spirit`·`adoratio`) **HPMult 5.0 → 2.5**.
500스테이지 보스층 **51.1턴 → 25.5턴**(일반 층 19.8턴의 1.3배).

> 🔴 보스는 **1마리**라 파티의 광역 딜이 통째로 낭비된다. ★5 파티는 전체 피해의 약 2/3이 광역이라, **같은 `HPMult`라도 성급이 오를수록 보스층이 상대적으로 길어진다**(1스테이지 2.1턴 ↔ 500스테이지 25.5턴). 보스층 길이를 볼 때는 파티 총 딜이 아니라 **단일 대상에 꽂히는 딜**로 계산할 것.

#### 시즌보스 (G7) — 2026-09-07 재설계

**측정** — 보스전 8콤보 1턴 딜 (보스 HP를 1e12로 올리고 `_T.totalDamage`를 읽는다. 둘 다 방어막이 안 걸린 값):

| 파티 | 1턴 딜 | 파티 HP | 힐/턴 |
|---|---:|---:|---:|
| ★3 12강 | 41,916 | 41,760 | 8,413 |
| ★5 25강 | 2,882,127 | 2,069,120 | 417,005 |

**구조** — 페이즈 3개 + 무한 페이즈 1개. 기믹은 한 번 켜지면 **전투 끝까지 유지**된다(사용자 확정).

| 구간 | 플레이어 딜 | 보스 공격력 | HP |
|---|---|---|---|
| 페이즈 1 | ×1.0 | ×1 | 60,000 |
| 페이즈 2~ | **×0.5** (`shield_magic`) | ×1 | 60,000 |
| 페이즈 3~ | ×0.5 | **×2** (`attack_all`) | 130,000 |
| **무한 페이즈** | ×0.5 | ×2 **× 1.194^턴** | **1억 (재충전 없음)** |

```
BossSet:  P1 60,000 · P2 60,000 · P3 130,000 · AttackDamage 2,000 · TurnLimit 30
          EndlessAtkGrowth 1.194 · EndlessHP 100,000,000   (뒤 둘은 신설 열)
```

🔴 **무한 페이즈는 페이즈 반복이 아니다.** 예전 구현은 `_EnterPhase(3)`을 다시 불러 **매 턴 재충전**하는 방식이었고, 그 때문에 세 가지가 한꺼번에 망가져 있었다:
1. 재충전마다 `NotifyPhaseChange`가 **기믹을 재무장**하고 `hp_threshold` 장부를 비웠다.
2. 매 턴 **초과 딜이 버려져** 턴당 인정 딜이 HP 한 칸으로 캡됐다 → **랭킹 붕괴**(전력 2.9배 차이가 점수 3.6% 차이).
3. `Phase ∞` 표기와 달리 실제로는 페이즈가 계속 "교환"되고 있었다.

**`_EnterEndless()`를 신설**해 HP를 한 번만 채우고 기믹은 켜진 채 둔다. `_EnterPhase`를 부르면 안 된다.

**설계 축 — 페이즈 HP는 낮게, 시간 압박은 공격력으로.** HP를 올려 시간을 벌면 약한 파티가 P3에 닿지 못하고, 시즌보스가 티켓 공급원이라 **성장 막다른 길**이 생긴다. 그래서 HP는 낮추고 압박을 전부 무한 페이즈 공격력 복리로 옮겼다.

> P2가 P1과 같은 60,000인 이유: 페이즈 2부터 딜이 반토막이라 **실효 체력은 60,000 → 120,000 → 260,000**으로 제대로 커진다.

**구현 주의**
- 🔴 `_EnterEndless`는 `_EnterPhase`를 부르지 않는다(기믹 재무장 방지).
- 🔴 무한 구간은 `ApplyBossDamage`에서 **초과 딜을 버리지 않는다**. 넘길 페이즈가 없으니 버릴 이유가 없고, 버리면 랭킹이 붕괴한다.
- 🔴 `GetExtraAtkMult`의 **1회성 소비를 제거**했다. 예전에는 읽는 순간 1.0으로 되돌려 ① 페이즈 3 진입 턴에만 강해지고 ② **진단용 조회가 다음 실제 공격의 배율을 훔쳐갔다**.
- ⚠️ `NotifyTurnEnd`는 **`OnEnemyTurnEnd`보다 먼저** 불러야 무한 진입 턴이 0턴으로 잡힌다.
- ⚠️ `EndlessAtkGrowth` 1.194는 지수라 무한 27턴째에 진입 시점의 ×70.5. 값을 만질 때는 30턴째를 계산해 볼 것.
- 🔴 **보스 공격력에 상한은 없다.** 1턴차 4,000(2,000 × 기믹 ×2)에서 복리로 계속 오른다 — 기믹 전 원값 `T5 4,776 · T10 11,590 · T20 68,254 · T30 401,950`. 30턴 제한이 사실상의 상한이고, 값 자체를 깎는 장치는 없다.
- 🔴 **`GetExtraAtkMult`(×2)는 지속형이다.** 시뮬레이션에서 매 턴 뒤 ×1로 되돌리면 실제보다 **보스가 절반으로 약하게** 나온다 — 이 실수로 가속값을 두 번 다시 잡았다(1.27 → 1.23은 틀린 값, → **1.194**가 최종).
- ⚠️ `BattleManager`에 새 메서드 호출을 추가하면 `LIA-1115`가 뜬다 — 호출부를 `fs.utimesSync`로 touch해 피호출자 뒤에 재임포트시키면 사라진다.
- 정리: `_PlayWarning()` 추출(`_EnterPhase`·`_EnterEndless` 공용), 의미를 잃은 `breakCount` 제거.

> 규칙: **보스 밸런스를 잴 때는 `EnemySkillSet`의 기믹 행을 먼저 읽을 것.** `shield_magic`(딜 0.5배)·`attack_all`(공격력 2배)은 파티 스탯만큼 결과를 흔든다 — 한 번 빼먹고 계산해 스펙이 통째로 뒤집힌 적이 있다(★3이 30턴 완주하고 ★5가 28턴에 전멸했다).

| 파티 | 무한 진입 | 생존 | 누적 딜 |
|---|---|---|---:|
| ★2 5강 | 실패(P3 진행 중) | 30턴 완주 | 148,910 |
| **★3 12강** | **12턴** | 26턴 전멸 | 543,412 |
| ★4 20강 | 4턴 | 25턴 전멸 | 2,896,000 |
| ★5 22강 | 3턴 | 29턴 전멸 | 13,302,000 |
| **★5 25강** | 3턴 | **30턴 완주** | 39,158,701 |

힐 발동률 **50~100% 전 구간**에서 같은 결론(★5 25강만 완주) — `balrog_seal`(5턴마다 2턴 회복 금지)이 흔들어도 안전하다.
`EndlessHP` 1억은 최강 파티가 30턴에 **39%**만 깎는 값이라 바가 눈에 띄게 줄되 절대 못 깬다.
★2 5강도 P1·P2는 깨므로 **티켓 2장은 확보**된다.

**런타임 실측** (★5 25강으로 4턴 실제 진행):
```
T1 후  페이즈2 · HP 60000/60000    · 누적딜    60000 · 딜배율 0.5 · 공배율 1.0
T2 후  페이즈3 · HP 130000/130000  · 누적딜   120000 · 딜배율 0.5 · 공배율 2.0
T3 후  무한 진입 · HP 1억/1억       · 누적딜   250000 · 무한턴 0 · 보스ATK 1000
T4 후  무한     · HP 98,558,938/1억 · 누적딜  1691062 (+1,441,062 전액 인정) · 무한턴 1 · 보스ATK 1270
```
빌드 0건. 화면 표기 `Phase ∞`.

#### G8·G9 해소 (2026-09-07)

- **G8 누적 딜량 랭킹** — G7 재설계로 자동 해소. 재설계 전 ★4 288만 / ★5 22강 467만 / ★5 25강 484만 → 지금 289만 / 1,330만 / 3,916만.
- **G9 기믹 지속 범위** — 사용자 확인 결과 **의도대로**. 페이즈 2부터 딜 0.5배, 페이즈 3부터 공격력 2배가 끝까지 유지되는 게 설계다.

### 2026-09-07 큰 수 표기 — 자릿수 확장 (Phase 3 G4)

**사용자 결정**: K/M/B 축약이 아니라 **"큰수를 갯수만 늘리는 방식으로, 지금 테스트에서 가장 높게 나오는 숫자보다 한 자릿수 높도록"**. 축약은 원래 값을 감추는데, 이 게임은 스타포스 한 강마다 딜이 얼마나 뛰는지 정확한 숫자로 보는 게 성장의 피드백이다.

🔴 **`UIDamageNumber`가 6자리 고정이라 이미 틀린 숫자를 찍고 있었다.** 스프라이트 숫자라 `D0`~`D5` 6칸이 하드 캡이고, `for i = 0, 5` + `s:sub(i,i)`로 **앞에서부터** 채워서 1,067,948이 "144106"으로 나왔다(자릿수가 통째로 줄어든, *그럴듯하게 틀린* 값).

**상한은 추정하지 않고 실측했다** — ★5 25강 · 500스테이지 DEF · 8콤보 1턴의 개별 피해:

```
427,175 (광역) / 150,089 / 194,538 / 30,237 / 1,067,948 (최대, 고정피해)
```

7자리가 실질 최대다 — 파티 1턴 총합이 345만이라 단일 타격이 8자리에 닿을 수 없다. 한 자릿수 올려 **8칸**.

**변경**
- 화면 파일의 데미지 숫자 풀 5개에 `D6`·`D7` 추가 (6칸 → 8칸)
- ⚠️ **컨테이너 `RectSize`는 건드리지 않는다** — 자릿수 칸이 부모의 **오른쪽 가장자리 기준**(anchor middle-right)이고 pivot이 0.5라, 폭을 바꾸면 오른쪽 가장자리가 밀려 **화면상 숫자 위치가 통째로 이동한다.** 런타임이 좌표를 다시 잡으므로 칸만 있으면 된다
- `UIDamageNumber`에서 자릿수 하드코딩 제거 — `D<n>` 자식을 있는 만큼 세어(`slotCount`) 쓴다. 이제 칸만 늘리면 코드 수정 없이 따라간다
- **넘침 가드** — 칸이 모자라면 `log_warning` + **9로 꽉 채워** 표시. 앞자리만 찍는 것보다 "한계 초과"가 드러난다
- **방치보상 숫자 폭 100 → 160** (`Stage/Box/reward/Gold|Stone/UIText`). 500스테이지 12시간 상한이 골드 731,520 / 결정석 243,840(6자리)인데 폭 100px·font 30은 6자가 경계였다. 정렬이 **Center**라 폭만 대칭으로 넓히면 숫자 위치는 그대로다

🔴 **부수 발견 — `HpText` 엔티티가 존재하지 않는다.** `MonsterManager` L422의 `hpText.Text = m.hp .. "/" .. m.maxHp`는 `isvalid` 가드에 막혀 **화면에 안 나오는 죽은 코드**다. G4의 원래 문제 진술("몬스터 HP 텍스트를 읽을 수 없다")은 실재하지 않았다 — HP는 `HpBar` 게이지로만 보인다. 나중에 `HpText`를 만들면 그때 폭을 함께 볼 것.

> 규칙: **UI 자릿수/폭 문제는 화면에 그 표시가 실제로 존재하는지부터 확인할 것.** 코드에 갱신 로직이 있다고 표시되는 게 아니다.

**검증**: 슬롯 5풀 전부 8칸 · `Play(33)`→`33______` · `Play(1067948)`→`1067948_` · `Play(87654321)`→`87654321` · `Play(123456789)`→ 경고 + `99999999` · 방치보상 폭 160 · 빌드 0건.

**폭이 충분해 손대지 않은 곳**: 랭킹 점수(280px/26 → 19자, 최대 39,158,701) · 강화 공격력·체력(250px/28 → 16자, 최대 594,872) · 인벤토리 상세(300px/24 → 22자) · 로비 헤더 재화(200px/30 → 12자).

### 2026-09-07 경제 밸런스 재설계 (Phase 3 G10)

**사용자 지시**: "확률은 건드리지 말자, 대신 강화 비용을 낮추고, 강화석 지급 비용도 낮추거나 강화 비용을 올리자. 난 22성까지 1달 안에 가기를 원하고 23성 이상은 운의 영역이라 생각해."

#### 먼저 잰 것 — 25강은 도달 불가였다

흡수 마르코프 체인으로 풀고 몬테카를로로 교차검증(0→22강 구간 75,617 vs 73,813회로 일치):

```
0강 → 25강   시도 31,063,404회 · 메소 512,044,418,173 · 결정석 97,285,085
             300스테이지 일일 수급 기준 1,565년
```

🔴 **비용의 87%가 9~13강에 몰려 있었다 — 고레벨이 아니다.** 11강부터 하락 50~70%가 붙는데 **파괴가 12강으로 되돌리니** 이 밴드를 끝없이 왕복한다.

| 현재 강화 | 9강 | 10강 | 11강 | 12강 | 13강 | 20강↑ |
|---|---:|---:|---:|---:|---:|---:|
| 기대 시도 | 16,551 | 18,204 | 16,547 | 12,409 | 6,993 | 14 |
| 비중 | 21.9% | 24.1% | 21.9% | 16.4% | 9.2% | 0.0% |

🔴 **결정석은 메소보다 26배 남아돌아 사실상 의미 없는 재화였다** (소요 비율 5,264:1, 수급 비율 3:1).

> 규칙: **강화 비용을 볼 때는 레벨별 단가가 아니라 `레벨별 기대 방문 횟수 × 단가`로 볼 것.** 하락이 붙은 구간은 방문 횟수가 폭증해 단가가 낮아도 비용을 다 먹는다.

#### 적용 — 확률·파괴 복구 지점 모두 그대로

```
강화 메소 비용 = 기존의 1/10        (1강 100 · 10강 1,200 · 20강 7,000 · 25강 15,000)
방치보상 메소  = 7,400 + (스테이지-1)×830   /h   (기존 1,080+120 의 6.8배)
방치보상 결정석 =    15 + (스테이지-1)×1.6  /h   (기존   360+40  의 1/26)
가챠 메소 보상 = 34,000 · 신규 지급 메소 = 34,000   (둘 다 기존 5,000)
```

**하이브리드를 택한 이유**(사용자 결정): 비용만 1/68로 낮추면 1강이 15메소라 숫자가 초라해지고, 수급만 올리면 강화 비용표를 손대지 못한다. 1/10 인하 + 6.8배 수급이 비율은 같으면서 규모가 게임 톤에 맞는다.

| 목표 | 기대 시도 | 기대 메소 | 기대 결정석 | 200스테이지 소요 |
|---|---:|---:|---:|---:|
| 20강 | 11,983 | 19,692,953 | 37,477 | 5일 |
| **22강** | **75,616** | **124,587,490** | **236,771** | **30일** *(⚠️ I4 이전 값 — 현재는 40,866 / 7,925만 / 19일)* |
| 23강 | 202,448 | 333,662,134 | 633,991 | 79일 |
| 24강 | 1,646,307 | 27억 | 5,155,908 | 646일 |

- **기준은 200스테이지**(일일 메소 4,202,880 · 결정석 8,019). 100스테이지면 57일, 300스테이지면 20일 — **기준 스테이지를 항상 같이 명시할 것.**
- 메소·결정석이 **동시에 병목**(둘 다 30일)이 되도록 맞췄다.
- 🔴 **`StarforceSet.GoldCost`와 `CurrencyManager:_RewardRateForStage`는 한 쌍이다.** 스타포스가 유일한 대형 소비처라 한쪽만 만지면 목표가 즉시 깨진다.

**검증**: 비용표 로드값 7개 · 시급 5개 스테이지 · **실제 강화 1회**(신궁 1→2강 성공, 메소 −150 · 결정석 −1 = 2강 행과 정확히 일치) · 테스트 상태 원복 · 빌드 0건.
부수 확인: 500스테이지 12시간 방치보상이 5,058,840(7자리)로 커졌지만 같은 날 G4에서 숫자 폭을 160(9자)으로 넓혀 둔 덕에 그대로 표시된다.

#### 🔴 남은 것 두 개

- **G11 일괄(자동) 강화** — 확률이 고정이면 **기대 시도 횟수도 고정**이다. 0→22강 75,616회 = 한 달이면 **하루 2,521회 클릭**. 비용을 0으로 만들어도 줄지 않으므로 이 기능 없이는 "22강 한 달"이 물리적으로 불가능하다. 사용자 결정으로 이번엔 등록만.
- **G12 스테이지 곡선 종점 재조정** — G3 곡선은 ★5 25강(전력 ×1,801)이 500스테이지와 맞먹는 전제인데, 현실적 상한이 22강(×632)이라 실제 도달선은 **350스테이지 부근**이다.

> **2026-09-07 2차 — 방치보상 기울기 완만화.** 위 표의 시급은 1차안(`7,400+(s-1)×830` / `15+(s-1)×1.6`)이다.
> 사용자 결정("굳이 스테이지를 더 높게 올려 시간을 쓰게 하고 싶지는 않아")으로 **`112,800+(s-1)×290` / `217+(s-1)×0.56`** 으로 바꿨다.
> 1차안은 1스테이지 **627일** / 100스테이지 57일 / 200스테이지 30일이라 **수급을 위해 200까지 미는 게 강제**였다.
> 2차안은 1스테이지 45일 · 50 40일 · 100 36일 · **200 30일**(앵커 유지) · 300 26일 · 500 20일 — 올리면 이득이지만 강제는 아니다.
> ⚠️ **기울기를 다시 세우면 그 강제가 되살아난다.**
>
> 🔴 **G12는 방치보상으로 못 푼다.** 확률이 고정이라 강수별 기대 비용의 **비율도 고정**이다 —
> 22강 대비 **23강 2.68배 · 24강 21.8배 · 25강 60.3배**. 방치보상은 전체 속도만 바꾸므로
> **"22강 30일"과 "25강 도달 가능"은 동시에 성립하지 않는다.**

### 2026-09-07 사용자 테스트 피드백 6건 (Phase 3 H블록)

사용자가 남은 테스트 8건을 전부 통과시키면서 지적한 6건. 테스트 항목은 `✅`로 닫고 발견 사항만 별도 처리했다.

**H1 쉴드 최대치 = 영웅 체력.** `GiveShieldToHero`가 `shieldMax += amount`라 **받은 양이 곧 최대치**였다 — 쉴드를 받을 때마다 게이지가 가득 찬 상태로 리셋돼 두께도 잔량도 알 수 없었다. `heroMaxHP`로 고정하고 현재값을 상한 절삭. 파티 초기화 전에는 상한을 못 잡으므로 옛 방식으로 폴백한다.
검증: 섀도어(13,056) `+100 → 100/13056` · `+500 → 600/13056` · `+39,168 → 13056/13056`.

**H2 로비 복귀 시 스테이지 진입 창이 남던 문제.** `UIStagePanelToggle`이 `/ui/Lobby/Stage`를 **열기만 하고 닫는 경로가 없었다.** `UIManager:ShowLobby()`에서 닫는다 — 클리어·패배·보스 복귀가 전부 이 메서드를 지나는 **단일 관문**이다. 새 프로퍼티 대신 `LobbyPanel:GetChildByName("Stage", false)`로 찾아 빌더 재실행에도 안 끊기게 했다.
> 규칙: **여는 코드를 쓸 때 닫는 경로도 같이 정할 것.** 특히 화면 전환을 넘어 살아남는 패널.

**H3 스테이지 보스층 20턴 제한.** `BattleManager.StageBossTurnLimit = 20`(0이면 무제한). 시즌보스의 `BossSet.TurnLimit`과 별개 — 그쪽은 소진 시 누적 딜량을 제출하지만 **이쪽은 남길 기록이 없어 패배**로 처리한다. `MonsterManager:IsBossFloor()` + `UpdateBossFloorTurnText` 신설, 판정은 `endEnemyTurn`의 비보스모드 분기.
⚠️ **제한만 걸고 표시가 없으면 패배가 예고 없이 닥친다** → `BossSlotPanel/TurnText` 신설(패널이 보스층에서만 켜지므로 표시도 자동으로 따라간다).
검증: 보스층 진입 "남은 턴 20/20" → 5턴 "15/20" → 19턴 "1/20".

**H4 가챠 ★5 중복 → 티켓 10장.** `MaxStarTicketReward = 10`. 성급 만렙이면 중복 장수가 아무 값도 없어 꽝만도 못했다.
🔴 **판정은 `AddHero` 전에** 해야 한다 — 뒤에 하면 방금 더한 장수가 섞여 오판한다.
결과 팝업에 `ticket` 종류 추가. **왜 티켓이 나왔는지 슬롯에서 읽히도록** 서버가 `maxStarHeroId`를 함께 보내고 부제에 `"섀도어 ★5 +10"`으로 적는다.
검증: 전 영웅 ★5 세팅 후 10연차 → `1) ticket SR hero_shadower amount=10` · 티켓 752 → 752(비용 −10 + 보상 +10). 상태 원복.

**H5 시즌보스 기본 공격 강화.** `AttackDamage` 1,000 → **2,000**(페이즈 3부터 ×2라 실제 4,000).
🔴 **공격력만 올리면 G7의 "★5 25강만 30턴 완주"가 깨진다** → `EndlessAtkGrowth`를 1.27 → **1.194**로 같이 낮췄다.
🔴 **1차로 잡은 1.23은 틀린 값이었다.** G9로 `GetExtraAtkMult`를 지속형으로 바꿔 놓고 **시뮬레이션 모델은 1회성인 채로 뒀다** — 실제 보스가 페이즈 3부터 매 턴 ×2를 적용하니 계산의 2배로 강했다. 런타임 확인(`endlessTurns=0` → 4,000 · `=10` → 31,702) 후 올바른 모델로 재탐색해 **1.194** 확정.
> 규칙: **밸런스 상수를 바꾸면 그 상수를 쓰는 시뮬레이션 모델도 같이 고칠 것.** 코드만 고치면 다음 튜닝이 통째로 어긋난다.
⚠️ **상한은 2,000.** 3,000이면 ★2 5강이 7턴에 전멸해 P1(60,000)도 못 깨고 **티켓이 0장** — 성장 막다른 길이 된다.
검증(힐 50/70/100% 전 구간): ★2 P1 돌파 턴8 · ★3 12강 P3 돌파 턴12 · ★5 22강 전멸 29턴 · ★5 25강만 30턴 완주.

**H6 퍼즐 드래그 10초 제한.** 손을 놓지 않으면 턴을 무한정 끌 수 있었다. `PuzzleLogic.DragTimeLimit = 10.0`, `@Logic`의 `OnUpdate`가 감소시키고 0이면 손을 뗀 것과 똑같이 처리.
🔴 종료 처리를 `PuzzleController:FinishDrag()` **한 곳으로 합쳤다** — 손 뗌과 시간 초과가 갈라지면 한쪽이 스냅을 빠뜨리거나 턴을 안 넘긴다.
⚠️ `FinishDrag`가 맨 앞에서 `EndDrag()`를 부르므로 **호출 전에 미리 정리하면 안 된다**(`wasDragging`이 false가 되어 턴이 안 넘어간다).
`Board_UI/DragTimer` 신설 — 드래그 중에만 켜지고 0.1초 단위로 표시.

**H6 후속 (같은 날) — 텍스트 타이머 → 드래그 중인 타일을 따라다니는 시계 게이지.** 사용자 요청("타이머는 마우스를 따라서 움직이면 좋겠어, 텍스트대신 시계 모양으로 게이지가 줄어드는"). 숫자를 읽으려면 시선이 손가락에서 보드 위쪽으로 떠나야 했다.
- `Board_UI/DragTimer`(텍스트) 제거 → **`Board_UI/PuzzleBoard/DragTimer`**(132×132 `empty`) + 링 스프라이트 2장: `Track`(검정 α0.5 · `Type=Simple`) · `Fill`(`Type=Filled` · `FillMethod=Radial360` · `FillOrigin=Top(2)` · `FillClockWise`).
- 🔴 **타일과 같은 부모(`PuzzleBoard`) 아래 둔 게 핵심이다.** 앵커·피벗이 같아 `_MoveDragTimer()`가 타일의 `anchoredPosition`을 그대로 복사하면 정확히 겹친다. 부모가 달라지면 좌표 변환이 필요해진다. 마지막 자식이라 타일 위에 그려지고, `raycast=false`라 드래그 입력을 안 가로챈다.
- `_ShowDragTimer(remain)`가 `FillAmount = remain / limit`을 쓰고 **3초 미만이면 적색**으로 바꾼다(링 길이만으로는 잔여 시간이 잘 안 읽힌다).
- 🔴 **계정 리소스 스토리지에 올린 스프라이트 RUID는 이 월드에서 아무 경고 없이 안 그려진다** — 아래 상시 이슈 참조. 공개 검색 RUID `5b53380da9064c3fb5e21a46592fffab`(136×136 흰 링, 실제 색 253·238·255이라 틴트가 깨끗하다)로 교체해 해결.
- 검증(플레이 실측): 타일 `(280,-160)` 이동 → 타이머 `(280,-160)` 추종 · `FillAmount 0.75` → 3/4 링 · `remain 2.0` → 짧은 적색 호 · 10초 소진 시 자동 종료되고 **턴이 넘어가 쉴드가 실제 적용** · 빌드 0건.

### 2026-09-07 계획 축소 2건 (사용자 결정)

- **G11 일괄(자동) 강화 — 만들지 않는다.** 대가는 문서에 박아 뒀다: 확률이 고정이라 0→22강 기대 시도가 **75,616회**이고 한 달이면 **하루 2,521회 클릭**이다. 그래서 **§4.7의 "22강 30일"은 재화 기준 상한**일 뿐 실제 도달 시점이 아니다 — 성장 속도를 말할 때 이 한계를 같이 말할 것.
- **미사용 맵(`Boss`/`Rest`/`Shop`/`UITest`) — 삭제하지 않고 남긴다.** ⚠️ 남기기로 한 이상 Phase 6 "전 맵 열거 후 샘플 엔티티 점검"의 **대상에는 포함**된다.

> ⚠️ 이번에도 `LIA-1115 UnresolvedFunction`이 떴다(`BattleManager` → `MonsterManager`의 새 메서드). **호출부를 `fs.utimesSync`로 touch해 피호출자 뒤에 재임포트**시키면 사라진다 — 이번 세션에서만 세 번째다.

### 2026-09-07 2차 사용자 테스트 피드백 4건 (Phase 3 I블록)

- ✅ I1. **최종 콤보를 공격 턴 끝까지 유지** (2026-09-07) — 콤보 이펙트가 `POP 0.1s + HOLD 0.8s + FADE 0.35s`로 바로 사라져, 캐스케이드가 끝난 **최종 콤보 수**를 읽을 새가 없었다
  - `UIComboCount:Hold()` / `Hide()` 신설. `Hold()`는 `_T.holding=true` + `elapsed`를 유지 구간에 고정해 **시간이 흐르지 않게** 만든다(알파 1.0 고정)
  - 🔴 **어디서 붙잡고 어디서 지우는지가 설계의 전부다.** 붙잡기 = `PuzzleLogic:OnResolveFinished`(캐스케이드가 끝난 지점 = 최종 콤보가 확정된 순간). 지우기 = `TurnManager:StartEnemyTurn`(몬스터 공격 시작)
  - ⚠️ **승리·패배는 적 턴이 오지 않는다** → `BattleManager:OnWin`/`OnLose`에서도 `HideCombo()`를 부른다. 안 그러면 결과 연출 내내 콤보가 남는다
  - ⚠️ `Play()`가 매번 `holding=false`로 되돌리므로 다음 턴의 콤보 팝업은 평소대로 동작한다
  - 검증(실제 턴 경로): `+0.7s Resolving 콤보ON holding=false` → `+1.4s holding=true` → `+2.8s 여전히 ON` → `+3.5s Enemy 콤보OFF`. 원래라면 1.25초에 사라졌을 것
- ✅ I2. **성급 필요 장수를 등급별로 분리** (2026-09-07 사용자 지적: "SSR과 SR의 최종 등급까지 필요한 영웅의 수가 같은데 확률로 따지면 SR이 더 많아야 한다")
  - **`StarLevelSet` 신설**(`.userdataset` + `.csv`) — `Rarity,Star1..Star5` = 각 성급에 필요한 **총 획득 장수**. `SSR 1/3/6/10/15` · `SR 1/8/15/25/38`
  - 🔴 **배율은 2.5배다(9배가 아니다).** 사용자가 말한 9배는 **배너 확률**(SR 10% vs SSR 1%) 기준인데, **SR 풀에는 영웅이 4명**이라 특정 1명이 나올 확률은 `10%÷4 = 2.5%` vs `1%÷1 = 1%` → **2.5배**다. 9배로 잡으면 SR ★5가 5,400뽑기(SSR의 3.6배)라 **등급의 의미가 뒤집힌다**
  - **2.5배의 좋은 성질**: SSR ★5 = 15÷1% = 1,500뽑기, SR ★5 = 38÷2.5% = 1,520뽑기. 게다가 SR 4명은 뽑기를 공유하므로 **파티 5인 전원 ★5가 ~1,500뽑기에 동시에 완성**된다
  - `CalcStarLevel(integer duplicates)` → **`CalcStarLevel(string heroId, integer duplicates)`**로 시그니처 변경. 호출부 9곳 전부 수정(`HeroDataManager` 3 · `HeroInventoryUI` 3 · `PlayerManager` 2 · `EnhanceUI` 1)
  - 등급 조회는 양쪽에 따로 있다 — 클라는 `heroBase`(`LoadHeroBase`), 서버는 `heroRarity`(`LoadHeroRarityIndex`에 역색인 추가). ⚠️ `LoadHeroBase`가 **ClientOnly**라 서버엔 영웅 표가 없다
  - ⚠️ **`StarLevelSet`에 등급 행이 없으면 조용히 만렙이 되지 않도록** 옛 기본값(1/3/6/10/15)으로 폴백하고 `log_warning`을 남긴다
  - ⚠️ **기존 세이브의 성급이 내려간다** — 비숍 14장이 ★4 → ★2가 되고 전투 스탯도 같이 떨어진다. 출시 전이라 보상 없이 수용
  - 검증(런타임): 임계값 로드 `SSR 1/3/6/10/15` · `SR 1/8/15/25/38` / 등급 조회 `warrior=SSR bishop=SR` / 성급 판정 `히어로 3장=★2·10장=★4` vs `비숍 8장=★2·25장=★4` / 빌드 0건
- ✅ I3. **영웅 상세에 "다음 성급까지" 게이지** (2026-09-07) — 영웅 인벤토리 화면의 `DetailPanel/StarProgress`(460×34, 패널 상단 여백)
  - 구성: `Label`(`★3까지 1장` — 남은 장수) + `Bar`(`SliderComponent`, 핸들 없음·`RaycastTarget=false`) + `Count`(`14 / 15`)
  - 상태 3가지: 미보유 `미보유`/`-`/0 · 진행 중 `★N까지 M장`/`현재 / 필요` · 만렙 `★5 MAX`/`N장`/게이지 가득
  - 채우기는 `HeroInventoryUI:_UpdateStarProgress`. 새 UUID 프로퍼티 없이 `panelDetail:GetChildByName("StarProgress", true)` 경로 조회를 쓴다(빌더 재실행에도 안 끊긴다)
  - 검증(런타임 3케이스): 비숍(SR 14장) `★3까지 1장 · 14/15 · 0.857` / 히어로(SSR 4장) `★3까지 2장 · 4/6 · 0.333` / 섀도어(SR 16장) `★4까지 9장 · 16/25 · 0.100` + 스크린샷
- ✅ I4. **스타포스 하락 경계를 11강 도전 → 13강 도전으로** (2026-09-07 사용자 지시: "12강 강화까지는 하락이 없도록, 12강에서 13강으로 도전할 때부터 하락, 12강 미만의 하락 확률을 실패(유지)로 옮기면 돼")
  - `StarforceSet`의 `Level` **11·12** 행 `DowngradeRate` 50/55 → **0**. 유지 확률은 CSV에 없고 `100 − 성공 − 하락 − 파괴`로 유도되므로 **하락을 0으로 두면 자동으로 유지에 합산**된다 — 다른 열은 건드릴 필요가 없다
  - 🔴 **`Level` 열은 "목표 강수"다.** 서버가 `GetStarforceData(currentSF + 1)`로 행을 고르므로 `Level=13` 행이 곧 **12강 → 13강 도전**이다. 여기를 헷갈리면 경계가 한 칸 밀린다
  - **왜 12강이 자연스러운 바닥인가**: 파괴가 스타포스를 **12강으로 되돌리는** 규칙이라, 12강 아래에도 하락이 있으면 복구선 아래에서 또 밀려나 왕복 구간이 9~13강까지 넓어진다. 12강을 바닥으로 만들면 **파괴 복구선과 하락 시작선이 같은 지점에서 맞물린다**
  - 🔴 **확률만 옮겼는데 경제가 반으로 줄었다** — 하락이 붙는 구간이 좁아지면서 기대 왕복이 급감한다:

| 목표 | 시도 (전 → 후) | 메소 (전 → 후) | 200스테이지 소요 |
|---|---|---|---|
| 20강 | 11,983 → **6,476** | 1,969만 → **1,251만** | 5일 → **3일** |
| **22강** | 75,616 → **40,866** | 1억2,459만 → **7,925만** | **30일 → 19일** |
| 23강 | 202,448 → **109,408** | 3억3,366만 → **2억1,227만** | 79일 → **52일** |
| 25강 | 3,106만 → **1,679만** | 512억 → **326억** | 사실상 불가(동일) |

  - ✅ **목표를 "22강 30일" → "22강 19일"로 교체했다**(2026-09-07 사용자 결정: "이대로 진행해보자"). 시급을 낮춰 30일로 되돌리면 **하락 완화의 체감이 사라지므로** 빨라진 페이스를 그대로 받는다. 30일 복원 계수(메소 ×0.65 · 결정석 ×0.56 = 200스테이지 `110,064`/h · `184`/h)는 **쓰지 않는 참고값**으로만 남긴다
  - 🔴 GDD가 경고한 "비용표와 시급은 한 쌍"이 **확률 쪽에서도 성립**한다는 사례다 — 확률 한 칸만 옮겨도 목표 일수가 통째로 움직인다
  - 병목도 바뀌었다: 전에는 메소·결정석이 둘 다 30일로 나란했는데, 지금은 **메소 19일 / 결정석 17일**로 메소가 병목이다
  - 검증(런타임): `11강 도전(10→11) 성공 50 유지 50 하락 0` · `12강 도전(11→12) 성공 45 유지 55 하락 0` · `13강 도전(12→13) 성공 40 유지 0 하락 60` · 16강부터 파괴 3% 유지 · 빌드 0건

### 2026-09-07 G5 중반 페이싱 검증 + Phase 5 전수 점검

**G5 결론: 스테이지 요구와 가챠 수급이 맞지 않는다.** 병목은 난이도 곡선이 아니라 **티켓**이다.

🔴 **기록돼 있던 요구 배수(100 ×14 / 200 ×72 / 300 ×291)는 재현되지 않았다** — 유도 과정이 문서에 없었다. 재현 가능한 모델로 다시 잡았다: `유효체력 = HP×(10+DEF)/10`, 몬스터 4마리, 목표 10턴.

**성급별 파티 1턴 딜(런타임 실측 · 0강 · 8콤보 3매치 합성)**

| ★1 | ★2 | ★3 | ★4 | ★5 |
|---:|---:|---:|---:|---:|
| 2,062 | 6,807 | 18,797 | 95,753 | 170,950 |
| ×1 | ×3.3 | ×9.1 | ×46.4 | ×82.9 |

**요구 vs 수급**: 스테이지 10→★1 · 50→★3 · **100→★4** · **150→★5** · 200→★5 10강 · 300→★5 22강.
가챠로는 ★2 240일 · ★3 360일 · ★4 480일 · ★5 **636일**. 스타포스 축(22강 19일)과 **30배** 벌어진다.

🔴 **피드백 루프**: 티켓은 시즌보스 페이즈 격파로 나오는데, 그 격파가 전력에 달려 있다. ★1은 P1만 깨서 **1.33장/일** → ★2까지만 240일. 전력이 있어야 티켓이 나오고 티켓이 있어야 전력이 는다.
⚠️ **곡선 재분배로는 안 풀린다** — 요구를 낮춰도 초반 240일은 그대로다. 대응은 티켓 축(G14).

> 실측 방법(재현용): 전투 진입 → `_HeroDataManager._T.inventoryData.heroes/starforce`를 목표 성급으로 덮어씀 → 몬스터 `hp/maxHp`를 1e12 → `BattleManager:ExecutePlayerAttack({TotalComboCount=8, MatchGroups=8개})` → HP 델타 합산.
> ⚠️ **`wait()`는 로컬 함수 안에서 죽는다** — 최상위(또는 최상위 `for`)에 둘 것.
> ⚠️ **측정 중 `maker_execute_script`로 폴링하면 실행 중인 스크립트가 중단된다.** 진행 확인은 `maker_logs`로만.
> ⚠️ 합성 매치를 `Count=3`으로 하면 `match_gte:5/7` 스킬이 빠져 **★4·★5가 과소평가**된다(위 수치는 하한).

**Phase 5 전수 점검 (화면 15개)**

- 🔴 **`Mail` L013 ERROR 678건 · `HeroInventory` 13건은 오탐이다.** 둘 다 부모가 `ScrollLayoutGroupComponent`인데 `ui_lint`가 스크롤 오프셋을 못 봐서 "화면 밖"으로 읽는다. **이 조합의 L013/L014는 앞으로 무시할 것.**
- ~~PC 예약영역(L012) 47건은 **전부 전체화면 배경/패널**이라 무해. 실제 조치 대상은 `BattleGroup/BossOverlay/TurnText` 1건.~~ → 🔴 **2026-09-07 오판으로 확인. 실제로는 7곳** — 아래 "PC 전용 전환" 절 참조.
- **모바일 SafeArea 레이어가 어느 화면에도 없다** → G15.
- **버튼 166개 중 클릭/호버 SFX가 붙은 것이 0개.** `SoundComponent` 3개는 전부 `BattleGroup`에 있고 `SoundRUID`가 비어 있다. 스크립트 `PlaySound`는 전투·퍼즐·강화 연출에만 있다 → G16.
- 빈 `ImageRUID` + alpha>0 22건(6건은 고아 파일). ~~대부분 버튼 히트영역용이라 `alpha=0`이 정답.~~ → 🔴 **틀린 판단이었다. 2026-09-07 정리했다가 시즌보스 화면이 깨져 전량 원복** — 아래 절 참조. 이 22건은 **손대지 않는다**
- 터치 타겟 88×88 미만 71건 — ① Mail 목록 행 40건(96×70) ② 로비 파티 슬롯 10건 ③ 높이만 올리면 되는 안전한 건 21건. → **2026-09-07 사용자 결정으로 계획에서 제외**(수치는 기준선으로 유지)
- **영웅 5종 격차는 없었다** — `HeroSkillSet` 5명 모두 tier 0~4 완비, 죽은 스킬 없음(비숍 `홀리 심볼`은 `passive`지만 `enhance_random_puzzle`이라 `ApplyPassiveSkills`가 정상 처리). 3매치에서 발동 수가 5/5/4/4/2로 갈리는 것은 `match_gte` 조건 차이이며 **설계 의도대로**다.

### 2026-09-07 빌드 에러 0건 + UI 클릭 SFX (G16) · 티켓은 과금으로 (G14)

🔴 **`LEA-1102/1103`은 stale 이 아니라 진짜 에러였다 — 앞선 판단이 틀렸다.**
`refresh` 가 항상 재진단하지는 않아서 "새 항목이 안 뜬다 = 해소됐다"로 오독했다. 실제로는 LSP 인덱스가 `CalcStarLevel` 의 옛 시그니처를 계속 들고 있었다.

> **해결법: `fs.utimesSync` touch 로는 안 된다. 파일 *내용*을 바꿔야 Maker 가 재임포트한다.**
> 끝 개행 토글(`\n` ↔ `\n\n`)만으로 충분하다. 이 방법으로 `LEA` 13건 + `LIA-1115` 3건이 **전부** 사라졌다.
> ⚠️ 상시 이슈 표의 "파일 내용 변경으로도 안 사라진다"는 **LIA-1115 에 한정된 옛 관찰**이었고, 실제로는 내용 변경이 듣는다. 새 메서드·시그니처 변경 뒤 경고가 남으면 **피호출자와 호출부 모두** 내용을 건드리고 refresh 할 것.

남은 `LIA-1114`(`BattleManager` 의 `_T`) 19건은 알려진 노이즈다(`_T` 는 선언 없이 쓰는 유일한 필드).
**`MapGroup` 화면의 L029**(루트가 아닌 `Map/NodeGroup` 에 `UIGroupComponent`) 1건 제거 → 화면 파일 15개 전체 **린트 ERROR 0건**(스크롤 하위 오탐 691건 제외).

**G16 — UI 클릭 SFX.** `UISoundManager`(`@Logic`) 신설. 화면마다 버튼에 컴포넌트를 붙이는 대신 **`/ui` 트리를 재귀로 훑어 `ButtonComponent` 보유 엔티티에 `ButtonClickEvent` 를 연결**한다. 클릭음 RUID `7eda004ded944e459f1fbb16c7f0f3f4`(사용자 지정 공용음).

- 🔴 **2초 주기 재스캔이 필수다.** `DefaultShow=false` 그룹의 자식은 `OnBeginPlay` 시점에 존재하지 않는다. 실측: 최초 **166개** → 2초 뒤 **5개 추가**(누적 171).
- 중복 연결은 `Entity.Id` 키로 막는다. `OnEndPlay` 에서 전부 `DisconnectEvent`.
- 기존 버튼 글루(`UIRanking` 등)와 **공존**한다 — 실측에서 클릭 한 번에 랭킹이 열리면서 사운드 핸들러도 같이 발화했다.
- 호버음은 `UITouchReceiveComponent` 가 있는 버튼이 거의 없어 제외.
- ⚠️ **소리가 실제로 들리는지는 AI 가 검증할 수 없다** — 사용자 실기 확인 필요(그래서 `🟡`).

**G14 — 티켓 수급은 과금으로 종결**(사용자 결정). G5 가 드러낸 무과금 격차(★4까지 480일)는 버그가 아니라 과금 설계의 여지로 수용한다. 무료 수급(시즌보스 일일 3장 + 랭킹 1장/3일)은 그대로.
⚠️ **무과금 페이싱 수치는 폐기가 아니라 기준선**이다 — 밸런스를 논할 때 "무과금 ★4 = 480일"에서 출발할 것.

### 2026-09-07 빈 `ImageRUID` 정리 → 시즌보스 화면이 깨져서 **전량 원복**

빈 `ImageRUID` + alpha>0 14건을 고쳤다가 **같은 날 전부 되돌렸다**(사용자 결정: "모두 굳이 필요없고 버그들만 키울 뿐이야"). 이 절은 **하지 말아야 할 일**의 기록이다.

🔴 **`.ui` 에 저장된 빈 `ImageRUID` 는 "누락"의 증거가 아니다.**
`BattleGroup` 의 `BossPart3` 는 `_SetupBossVisuals` 가 **런타임에 `BossPartSet` 의 RUID 를 넣는 슬롯**이라 저장값이 비어 있는 게 정상이다. 저장 상태만 보고 "안 그려지는 히트영역"으로 판단해 `Color.a=0` 을 걸었더니, **보스 파츠가 사라지고 뒤의 일반 몬스터 슬롯과 HP바가 드러났다.**

> **교훈**: 빈 RUID 를 정리 대상으로 삼기 전에 **런타임에 채워지는 슬롯인지** 먼저 확인할 것. 린트(L008)는 저장 파일만 보므로 이 구분을 못 한다 — 앞으로 **보스 파츠·영웅 슬롯·아이템 아이콘처럼 코드가 RUID 를 주입하는 자리의 L008 은 무시**한다.

**원복 방법** — 변경 전 조사 출력의 값을 그대로 되돌렸다. 두 형태가 섞여 있어서 구분이 필요했다:

| 대상 | 원본 상태 | 원복 방법 |
|---|---|---|
| `PopupGroup/PopupBack`, `.../deco_line` | `ImageRUID` 키가 **있고** `DataId = ""` | `patchComponent` 로 빈 값 재설정 |
| `SceneTransition/FadeOut`, 히트영역 11건 | 해당 키가 **아예 없음** | `getComponent` → 키 삭제 → `upsertComponent` |

⚠️ **`UIBuilder` 에는 값 키를 지우는 API 가 없다.** `patchComponent` 는 병합이라 키가 남는다 — **`upsertComponent` 로 컴포넌트를 통째 교체**하는 것이 유일한 방법이다. 14건 + `PopupBack.enable=false` 전수 대조 일치, 빌드 에러 0건.

🔴 **알려진 상태로 남긴 것 — `UIPopup`(공용 모달)은 화면에 뜨지 않는다.**
`PopupGroup/PopupBack` 이 `enable=false` 로 저장돼 있는데 `UIPopup:Open()` 은 **루트 `popupGroup` 만 켜고**, `PopupBack` 을 켜는 코드가 없다. 시즌보스 도전 횟수 소진 안내(호출 2곳)가 아무 로그 없이 사라진다. **켜 보면 로비 UI 뒤에 그려져** 딤이 안 보이고 캐릭터가 팝업 위로 뜨므로, 고치려면 z-order 까지 같이 손봐야 한다 → **M1 범위 밖**.

**터치 타겟 88×88 미만 71건은 계획에서 제외**(사용자 결정: "터치 타겟이 작은 건 어쩔 수 없다"). 50건이 `Mail` 행 높이·로비 파티 슬롯에 묶여 있어 버튼만 키울 수 없다. → **2026-09-07 PC 전용 확정으로 기준 자체가 적용 대상에서 빠졌다**(아래 절).

### 2026-09-07 PC 전용 전환 + Phase 6 착수

**타겟 플랫폼 = PC 전용**(사용자 결정). **화면 재작업은 없다** — UI 캔버스가 1920×1080 고정이라 그대로 쓴다. 바뀐 건 기준뿐:

- **빠진 것**: SafeArea(G15) · 터치 타겟 88×88 — 둘 다 모바일 기준.
- 🔴 **들어온 것: PC 시스템 UI 예약영역.** 좌상단 260×170(채팅) · 우상단 220×130(친구·메뉴)에 엔진이 **항상** UI를 그리고 **끌 공개 API가 없다**(캡처 옵션은 결과물에만 적용). 회피 설계가 유일한 대응.

🔴 **`.ui` 좌표를 잴 때 두 번 틀렸다 — 재사용할 교훈이 여기 있다.**

| 함정 | 증상 | 올바른 방법 |
|---|---|---|
| 필드명 | `AnchoredPosition`(대문자 A)로 읽으면 `undefined` → 전부 0으로 계산돼 **"다 안전"이라는 정반대 결론**이 나온다 | 실제 키는 **`anchoredPosition`**(소문자 a). `Position`도 같이 저장돼 있어 교차검증에 쓸 수 있다 |
| 조상 체인 | 부모 오프셋을 안 타면 화면 절대 좌표가 안 나온다 | 루트(1920×1080)부터 내려가며 `anchor → anchoredPosition → pivot`을 누적. `AnchorsMin ≠ AnchorsMax`면 크기도 부모 비율에서 계산 |

> `ui_lint`의 L012도 같은 이유로 신뢰할 수 없다. **예약영역 침범을 판단할 때는 직접 절대 좌표를 계산할 것.**

바로잡은 결과 — 컨테이너·전체화면 배경을 뺀 **실제 표시 요소 7곳**이 침범한다(고아 파일 2건 제외): `BattleGroup/BossOverlay/TurnText` · `Lobby/Header/Gold`(**골드가 채팅 버튼 밑**) · `Lobby/LeftAside/Profile` · `HeroInventory`의 `PartyName`·`partySlot1`·`DetailPanel/StarProgress` · `RobbyGroup/LobbyPanel/UISprite`. → G17. ⚠️ 예약영역 수치는 **보수적 추정치**라 실제 가림 여부는 플레이 화면에서 눈으로 볼 것.

**Phase 6 — 맵 전수 점검(6개) 결과**: 전부 `TileMapMode=0`. **템플릿 샘플 몬스터·NPC 0건** — 지울 게 없다. `Rest`의 `Shop` NPC와 `Shop`의 `npc-52`는 스크립트 참조가 없어 실행되지 않는다. `Boss`/`UITest`는 타일·풋홀드 0인 빈 껍데기.

**Phase 6 — 디버그 로그**: 108건 중 **11건만** 제거(매 턴·매 사망 반복분 — `[MM:Die]` 7 · `BuildHeroSkillSteps`/스타포스계수 2 · 힐 2). 🔴 **`OnUpdate`에 들어간 `log()`는 0건**이라 "과다 출력"의 실체는 이 11건이었고, 나머지 97건은 1회성 초기화·이벤트 로그라 진단 가치가 더 크다. 실패 경로 가드 로그는 정상 상태에선 안 찍히므로 유지.
