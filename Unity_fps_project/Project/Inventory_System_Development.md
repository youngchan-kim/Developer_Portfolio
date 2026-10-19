# ScriptableObject 기반 FPS 인벤토리 시스템 확장 개발

## 1. 프로젝트 개요

- 프로젝트: Unity C# FPS 개인 프로젝트
- 개발 방식: 공개 튜토리얼의 인벤토리 구조를 학습한 뒤 FPS 게임에 맞춰 확장
- 기반 자료: [sniffle6 / Scriptable-Object-Inventory — Part 7 Code Review](https://github.com/sniffle6/Scriptable-Object-Inventory/tree/master/Part%207%20Code%20Review)
- 기능 참고: PUBG(배틀그라운드)의 인벤토리 사용 방식
- 구현 코드: [youngchan-kim / fps_project](https://github.com/youngchan-kim/fps_project)

> **기여 범위:** 기본 ScriptableObject 인벤토리와 슬롯·드래그 UI 구조는 튜토리얼을 참고했다. 원본 Part 7에 없는 Ground 인벤토리, FPS 총기 슬롯 연동, 인벤토리 유형별 아이템 이동 및 사용 처리 등을 프로젝트에 추가했다. 이 문서는 소스 코드와 Unity 실행 테스트를 기반으로 작성했다. 회복 아이템 및 HP 연동은 실행 검증을 완료했으며, 그 외 기능의 검증 상태는 아래 체크리스트에 구분해 기록한다.

## 2. 개발 배경과 목표

기본 인벤토리 예제는 아이템 데이터와 슬롯 교환을 중심으로 구성되어 있었다. FPS 게임에서는 바닥에 놓인 아이템을 획득하거나 다시 버리고, 일반 소지품과 장착 총기를 구분하며, 의료품을 사용하는 기능이 필요했다. 이를 위해 PUBG의 인벤토리 조작 방식을 참고하여 기존 구조를 게임에 맞게 확장했다.

## 3. 원본과 확장 코드 비교

| 항목 | Part 7 원본 | FPS 프로젝트 확장 |
|---|---|---|
| 데이터 구조 | `InventoryObject`, `Inventory`, `InventorySlot` | 기본 구조 활용, `InventoryID`, 슬롯 수량·월드 오브젝트 참조 등 추가 |
| 아이템 분류 | Food, Helmet, Weapon 등 | Gun, Bullet, Medical, Bag, Armor 등 FPS 중심 분류 |
| 드래그 앤 드롭 | `SwapItems()` 중심 | 이동 대상 `InventoryID`별 분기 처리 |
| 바닥에 버리기 | UI 밖으로 드래그하면 슬롯 데이터 제거 | `ItemDropSystem()`을 호출하여 월드 아이템 처리와 연결 |
| 바닥 아이템 획득 | 원본 예제의 충돌 기반 아이템 획득 | Ground/일반 인벤토리 간 이동 처리 추가 |
| 총기 장착 | Part 7에 FPS 전용 총기 장착 연동 없음 | `GunSlot_swap()`, `RemoveGun()` 호출 연결 |
| 아이템 사용 | 일반 슬롯 이벤트 | 우클릭 의료품 사용과 체력 시스템 호출 연결 |
| 정렬 | 별도 정렬 메서드 없음 | `InventoryObject.Sort()` 추가 |
| 저장·불러오기 | `BinaryFormatter` 사용 | 기반 방식 유지, 추가 객체 직렬화 문제 확인 |

**주의:** 원본에도 기본 아이템 획득 및 드래그·슬롯 교환 기능이 존재한다. 따라서 아이템 획득과 드래그 기능 자체를 처음부터 독자 개발했다고 표현하지 않는다.

## 4. 주요 구조 및 구현

### 4.1 인벤토리 유형에 따른 아이템 이동

`UserInterface.OnDragEnd()`에서 드래그 출발 슬롯의 `InventoryID`와 마우스가 위치한 목적지 인벤토리의 ID를 확인한다. 목적지 ID에 따라 다음과 같이 처리한다.

- `0` (Ground): 아이템을 바닥으로 내리는 `ItemDropSystem()` 및 총기 해제 처리 연결
- `1` (일반 인벤토리): 아이템 획득·추가·제거 관련 플레이어 메서드 호출
- `2`: 코드상 분기가 있으나 핵심 처리 주석 상태
- `3` (총기 슬롯): 슬롯 허용 아이템 검사, 아이템 교환 및 `GunSlot_swap()` 연결

UI 이벤트에서 단순 슬롯 교환뿐 아니라 실제 게임 내 아이템 상태를 변경하는 흐름으로 확장했다.

### 4.2 FPS에 맞춘 아이템 데이터

`ItemObject.cs`의 `ItemType`을 총기·탄약·의료품·가방 등으로 구성했다. `Item`에는 `addHeal`, `amount`, `item_object`, `item_object_Prefab`, `inven_in_active` 등 게임 내 사용과 월드 오브젝트 연결을 위한 필드가 추가되어 있다.

### 4.3 아이템 사용과 체력 시스템 연결

`UserInterface.OnUp()`에서 우클릭과 아이템 사용 가능 여부를 확인하고, 특정 의료품(`Id == 4`, `addHeal != 0`)이면 `player.hp.OnHpAnimtion()`을 호출한다. 이 호출을 통해 의료품 사용을 `HP_System`에 전달한다. Unity 실행 테스트에서 의료품 수량 감소, 마지막 1개 사용, 실제 HP 증가와 HP UI 갱신이 정상 작동하는 것을 확인했다.

### 4.4 의료품 회복 처리 및 HP UI 동기화 (실행 검증 완료)

`Player.Initialize()`에서 `HP_System.Initialize()`로 체력을 초기화한 뒤 `HPBar.SetMaxHealth()`와 `HPBar.SetHealth()`로 UI Slider의 최대값 및 현재값을 설정했다. 이후 `Player.Update()`에서 `hp.GetCurHp()`를 읽어 HP UI에 반영한다.

`HP_System.HpSliderAnimation()`은 코루틴으로 매 프레임 일정량의 체력을 회복한다. 초기 구현에서는 회복량을 계산해도 `CurrentHp`가 증가하지 않았고, 수정 과정에서 회복 시작 시점의 고정 목표 HP를 사용하여 회복 중 피격 데미지가 최종 체력에 반영되지 않는 문제가 발생했다.

이를 해결하기 위해 **고정 목표 HP 대신 남은 회복량(`remainingHeal`)을 관리**하도록 변경했다. 각 프레임에서 `healStep`을 계산하고 `MaxHp - CurrentHp` 범위 안에서 실제 회복량을 적용한다. 회복한 양만큼 `remainingHeal`을 차감하므로 회복 중 데미지를 받아도 소모 가능한 회복량이 늘어나지 않는다.

```csharp
float remainingHeal = heal;
float healSpeed = 30f;
while (remainingHeal > 0f)
{
    float healStep = Mathf.Min(healSpeed * Time.deltaTime, remainingHeal);
    float actualHeal = Mathf.Min(healStep, MaxHp - CurrentHp);
    if (actualHeal <= 0f) break;

    CurrentHp += actualHeal;
    remainingHeal -= actualHeal;
    yield return null;
}
```

**검증 시나리오:** HP 30에서 회복량 70인 아이템을 사용하면 최종 HP 100이 된다. 회복 도중 데미지 10을 받으면 최종 HP는 90으로 유지된다. 아이템을 2개 보유한 경우 연속 사용과 마지막 1개 사용도 정상 작동함을 확인했다.

### 4.5 아이템 정렬

`InventoryObject.Sort()`는 수량이 0인 슬롯을 비우고, 비어 있는 슬롯 뒤의 아이템을 앞쪽으로 이동시키려는 로직을 포함한다. 현재 구현은 인접 슬롯 위주로 처리되므로 전체 정렬이 항상 완성되는지는 테스트가 필요하다.

## 5. 처리 흐름 요약

```text
사용자 드래그 시작
    ↓
UserInterface.OnDragStart()
    ↓
임시 드래그 아이콘 생성
    ↓
UserInterface.OnDragEnd()
    ↓
출발/도착 인벤토리 확인
    ├─ Ground → ItemDropSystem() 등
    ├─ 일반 인벤토리 → 아이템 획득·추가·제거 처리
    └─ 총기 슬롯 → SwapItems(), GunSlot_swap() 등
    ↓
슬롯 UI 갱신(UpdateSlotDisplay)
```

## 6. 코드 확인 과정에서 발견한 제한 사항

1. **저장·불러오기:** `BinaryFormatter`로 `Inventory` 전체를 직렬화하는 방식이다. 추가된 `GameObject` 참조 때문에 직렬화 오류가 발생했다는 주석이 있으며, 정상 저장·복원 완료 여부는 확인되지 않았다. 또한 `BinaryFormatter`는 신규 개발에 권장되지 않는다.
2. **수량 누적:** `InventorySlot.AddAmount(Item _item, int value)`가 인자로 전달된 `value` 대신 `_item.amount`를 더한다. 의도한 수량과 실제 누적 수량이 다른 경우가 생길 수 있다.
3. **정렬:** `Sort()`는 단일 순회에서 인접 슬롯을 이동시키므로 중간 빈 슬롯이 여러 개일 때 원하는 정렬 결과가 나오는지 검증이 필요하다.
4. **드래그 예외 처리:** `MouseData.slotHoveredOver`의 Dictionary 키 존재 여부 등 방어 코드가 부족할 수 있다.
5. **의료품 회복:** 기존 회복량 미반영 및 회복 중 피격 시 고정 목표 HP로 복귀하는 문제를 수정하고 Unity 실행 검증을 완료했다.
6. **총기 장착·해제:** 호출 연결은 확인했지만 Unity에서 슬롯 이동, 무기 활성화, 월드 오브젝트 상태가 일관되게 바뀌는지는 별도 실행 검증이 필요하다.

## 7. Unity 테스트 체크리스트

- [ ] 바닥 아이템을 일반 인벤토리로 드래그해 획득
- [ ] 일반 인벤토리 아이템을 바닥에 버렸을 때 월드 오브젝트 생성 및 슬롯 정리
- [ ] 총기를 전용 슬롯에 장착하고 실제 손에 든 무기와 동기화
- [ ] 장착 총기 해제 및 다른 총기로 교체
- [ ] 같은 아이템 여러 개 획득 시 수량 정상 증가
- [ ] 빈 슬롯이 여러 개일 때 정렬 정상 동작
- [x] 의료품 사용 시 실제 HP 증가 및 아이템 수량 감소
- [x] 마지막 의료품 1개 사용 및 슬롯 처리
- [x] HP UI와 현재 HP 연동, 회복 중 피격 데미지 반영, 최대 HP 제한
- [ ] 저장 후 재실행 시 아이템 데이터 정상 복원

## 8. 포트폴리오에 사용할 핵심 설명

> 공개 튜토리얼의 ScriptableObject 기반 인벤토리 구조를 학습한 뒤, PUBG의 아이템 관리 방식을 참고하여 Unity FPS 프로젝트에 맞게 확장했습니다. 기본 슬롯 교환 기능에 Ground 인벤토리와 일반 인벤토리 간 이동, 총기 슬롯 장착·해제, 아이템 버리기 및 의료품 사용 처리 등을 연결했습니다. 특히 드래그 종료 시 인벤토리 종류에 따라 서로 다른 게임 로직을 실행하도록 UI 이벤트를 확장했습니다. 의료품 회복 과정에서는 회복량을 프레임별로 적용하도록 개선해 회복 중 피격 데미지가 정상 반영되도록 했으며, HP UI 연동과 마지막 아이템 사용까지 Unity에서 검증했습니다. 저장·불러오기 및 일부 일반 수량·정렬 처리는 추가 검증이 필요합니다.

## 9. 관련 코드

**원본 (Part 7)**
- [`InventoryObject.cs`](https://github.com/sniffle6/Scriptable-Object-Inventory/blob/master/Part%207%20Code%20Review/Assets/Scriptable%20Objects/Inventory/Scripts/InventoryObject.cs)
- [`UserInterface.cs`](https://github.com/sniffle6/Scriptable-Object-Inventory/blob/master/Part%207%20Code%20Review/Assets/UserInterface.cs)
- [`ItemObject.cs`](https://github.com/sniffle6/Scriptable-Object-Inventory/blob/master/Part%207%20Code%20Review/Assets/Scriptable%20Objects/Items/Scripts/ItemObject.cs)

**확장 프로젝트**
- [`InventoryObject.cs`](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Scriptable%20Object/Inventory/Script/InventoryObject.cs)
- [`UserInterface.cs`](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Scriptable%20Object/Interface/Script/UserInterface.cs)
- [`ItemObject.cs`](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Scriptable%20Object/items/Script/ItemObject.cs)
- [`PickUpController.cs`](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Script/Item/PickUpController.cs)

---

**작업 재개 키워드:** `인벤토리 시스템`

**문서 상태:** 의료품 회복·HP UI 연동·마지막 아이템 사용 Unity 실행 검증 완료. 나머지 기능은 체크리스트 기준으로 검증 진행 중.
