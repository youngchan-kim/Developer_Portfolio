# Unity FPS 코드 리뷰 — 설계 근거·결함 분석·검증

> 프로젝트: [youngchan-kim/fps_project](https://github.com/youngchan-kim/fps_project)  
> 작성 기준: `Enemy_FSM_Development.md`, `Shooting_Damage_System_Development.md`, `Inventory_System_Development.md`, `architecture.md` 및 2026-10-19 GitHub `main`의 `EnemyFSM.cs`, `GunSystem.cs`, `HP_System.cs`, `Player.cs` 확인.  
> 검증 구분: **[코드 확인]** 소스에 존재하는 로직 / **[Unity 확인]** 개발자가 보고한 실행 테스트 / **[개선 제안]** 아직 적용·검증했다고 볼 수 없는 사항.

## 1. 리뷰 범위와 핵심 결과

| 영역 | 핵심 설계·문제 | 결과 | 검증 수준 |
|---|---|---|---|
| Enemy FSM | `Enemy`의 switch 기반 상태 관리 분리 | `IEnemyState` 및 상태 클래스 도입 | 코드 확인 + 핵심 행동 Unity 확인 |
| Shooting / Damage | 발사 입력과 총기·피해 처리 분리 | `Player → GunSystem → DamegeSystem → HP_System` | 코드 확인 |
| Reload | 인벤토리 탄약과 재장전, 애니메이션 조건 불일치 | 탄약 수량 연동 및 모션 조건 수정 | 개발자 Unity 확인; 최종 애니메이션 수정 커밋 재확인 필요 |
| Inventory | 기본 인벤토리의 FPS 확장 | Ground/일반/총기 슬롯 분기 및 의료품 사용 | 코드 경로 확인; 일부 실행 검증 미완료 |
| Healing | 회복 중 피해가 상쇄되는 문제 | `remainingHeal` 기반 코루틴으로 변경 | 코드 확인 + Unity 확인 |

## 2. Enemy FSM — 상태 전환과 행동 책임 분리

### 기존 문제와 변경

기존 `Enemy.cs`는 `switch`로 상태를 직접 분기했다. 이를 `EnemyFSM`의 현재 상태 보관과 `IEnemyState` 구현 클래스로 분리했다. 실제 이동·공격 행동은 여전히 `Enemy` 메서드가 수행하므로, 모든 행동을 별도 클래스로 옮겼다고 설명하지 않는다.

**[코드 확인]** `EnemyFSM.cs`의 핵심 구조:

```csharp
public interface IEnemyState
{
    void Enter();
    void Update();
    void Exit();
}

public void ChangeState(IEnemyState newState)
{
    currentState?.Exit();
    currentState = newState;
    currentState.Enter();
}
```

`EnemyFSM.Update()`는 `enemy.IsDead()`를 공통 검사해 `DieState`로 전환한다. `IdleState`는 장시간 정지하는 상태가 아니라 탐지 여부를 판단해 `RunState` 또는 `ReconnaissanceState`를 선택한다. `AttackState`는 타겟 상실·공격 거리 이탈 시 바로 Run으로 돌아가지 않고 Idle에서 다시 판단한다.

**설계상 장점:** 상태 전환 조건이 상태별로 분리되어 흐름을 추적하기 쉽다. `Enter/Exit` 계약을 통해 상태 진입·종료 시점을 명시한다.

**[Unity 확인]** 탐지·추적·공격·정찰 복귀·HP 감소 및 사망의 핵심 동작은 개발자가 확인했다.

**[개선 제안]** `Enemy.Update()`에서 FSM이 초기화되지 않았을 때의 null 경로, 초기화 순서, 상태 전환 로그를 계속 점검한다. 최근 개발 중 FSM null 오류가 보고되었으나, 해당 오류의 최종 수정 및 재검증 여부는 이 문서의 코드 확인 범위 밖이다. 적 HP UI와 오브젝트 풀링은 현재 구현 완료 기능으로 소개하지 않는다.

참고: [EnemyFSM.cs](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Script/Enemy/EnemyFSM.cs), [Enemy.cs](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Script/Enemy/Enemy.cs)

## 3. 사격 및 데미지 — 입력·발사·피해 책임 구분

**[코드 확인]** `Player.Update()`는 현재 활성 총기를 `GunSystem_chage()`로 탐색한 뒤 발사 입력과 `HasAmmo()`, `ReadyToShoot()`, `FiringIsPossible()` 조건을 검사한다. 조건을 통과하면 `GunSystem.Firing()`을 호출한다. `GunSystem.Shoot()`은 `GunMuzzle` 기준 `Physics.Raycast`로 명중을 검사하고, 대상에 `DamegeSystem`이 있으면 `TakeDamage()`를 호출한다.

```mermaid
flowchart LR
    P["Player: 입력·발사 조건"] --> G["GunSystem: 발사·Raycast"]
    G --> D["DamegeSystem: 피해 전달"]
    D --> H["HP_System: 체력 관리"]
```

**설계상 장점:** 입력 처리와 총기 로직을 분리하고, 피격 대상은 `DamegeSystem`을 통해 HP를 변경하도록 구성했다. 총기 속성은 `Gun_Scriptable`로 관리한다.

**[코드 확인]** 탄창은 `GunSystem.Awake()`에서 `Gun_property.magazineSize`로 초기화한다. 이전 테스트용 3000발 설정을 실제 탄창 크기로 바꾼 변경이다.

**[개선 제안]** 총구 Raycast와 화면 중앙 조준점이 정확히 일치하는지, 무기 교체·비활성화 시 발사 예약 처리가 안전한지 확인한다. 총기 참조를 프리팹 자식 인덱스로 탐색하는 부분은 계층 변경에 취약하다.

참고: [Player.cs](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Script/Player/FPSControl/Player.cs), [GunSystem.cs](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Script/GameSystem/GunSystem.cs)

## 4. 재장전 — 실제 상태와 애니메이션 조건의 불일치

### 문제

탄약이 없는 상태에서 R키를 누르면 실제 재장전은 불가능한데 플레이어 재장전 애니메이션은 실행되는 현상이 있었다.

### 원인

개발 기록에 따르면 `Player.cs`의 실제 재장전에는 `ReloadIsPossible()` 검사가 있었지만, `PlayerMovement.cs`의 애니메이션 경로에서는 R키 입력만 확인했다. **게임 상태 변경 조건과 시각적 피드백 조건이 서로 달랐다.**

### 해결

재장전 애니메이션 시작에도 현재 총기의 `ReloadIsPossible()` 조건을 적용하도록 수정했다. 탄창 보충은 `magazineSize - bulletsLeft`와 인벤토리 보유 탄약의 최솟값을 사용하고, 보충한 수량만큼 인벤토리 탄약을 차감한다.

**[Unity 확인]** 탄창 소모 후 발사 제한, 보유 탄약으로 재장전, 탄약이 없을 때 불필요한 재장전 모션 제한을 개발자가 확인했다.

**[추가 확인]** 애니메이션 조건 수정의 **최종 GitHub 반영 여부**와 재장전 중 무기 교체·연속 R키 입력은 별도 확인이 필요하다. 적 총기가 플레이어 인벤토리 조회에 의존할 수 있는 구조도 분리 검토 대상이다.

참고: [Shooting_Damage_System_Development.md](Shooting_Damage_System_Development.md)

## 5. 인벤토리 — 튜토리얼 기반 구조의 FPS 확장

### 참고 범위와 직접 확장

**[개발 기록 확인]** `InventoryObject`, `InventorySlot`, 기본 드래그 UI는 [sniffle6의 Scriptable-Object-Inventory](https://github.com/sniffle6/Scriptable-Object-Inventory/tree/master/Part%207%20Code%20Review)를 참고했다. 프로젝트에서는 `InventoryID`를 기준으로 Ground, 일반 인벤토리, 총기 슬롯의 이동 처리를 분기하고, 아이템 버리기·총기 장착·의료품 사용을 게임 로직과 연결했다.

**[코드 경로 확인]** `UserInterface.OnDragEnd()`가 출발·도착 인벤토리 종류에 따라 슬롯 이동 및 `GunSlot_swap()`·`ItemDropSystem()` 등의 처리를 연결한다. `UserInterface.OnUp()`은 의료품 사용 시 `HP_System.OnHpAnimtion()`을 호출한다.

**[Unity 확인]** 의료품 수량 감소, 마지막 1개 사용, HP 증가·UI 갱신을 확인했다.

**[미검증]** 바닥 아이템 이동 전체, 총기 장착·해제 모든 경우, 정렬 경계 상황, 저장 후 재실행 복원은 아직 검증 완료로 표시하지 않는다.

### 발견된 위험 요소

| 위치 | 문제·제약 | 개선 방향 |
|---|---|---|
| `InventorySlot.AddAmount` | `value` 대신 `_item.amount`를 더하는 구현 | 의도한 수량 누적 여부 확인 후 수정 |
| `InventoryObject.Sort` | 인접 슬롯 중심 단일 순회 | 여러 빈 슬롯이 있는 사례 테스트 |
| 저장/불러오기 | `BinaryFormatter`와 GameObject 참조 직렬화 | 직렬화 전용 데이터 모델로 분리 검토 |
| 드래그 이벤트 | Dictionary 키 및 null 참조 가능성 | 이벤트 전제 조건 검사 추가 |

참고: [Inventory_System_Development.md](Inventory_System_Development.md)

## 6. HP 회복 — 고정 목표 체력으로 인한 피격 상쇄 버그

### 문제 재현

- 초기 HP: **30**
- 의료품 회복량: **70**
- 회복 도중 피격: **10 데미지**
- 기대 최종 HP: **90**

초기 구현에서는 `CurrentHp` 자체가 증가하지 않는 문제가 있었다. 이후 회복 시작 시 계산한 **고정 목표 HP**를 향해 증가시키는 방식에서는 중간에 받은 피해가 최종 수치에 반영되지 않았다.

### 수정 코드

**[코드 확인]** GitHub `HP_System.cs`의 회복 코루틴 핵심 부분:

```csharp
float remainingHeal = heal;
float healSpeed = 30f;

while (remainingHeal > 0f)
{
    float healStep = Mathf.Min(healSpeed * Time.deltaTime, remainingHeal);
    float actualHeal = Mathf.Min(healStep, MaxHp - CurrentHp);

    if (actualHeal <= 0f)
        break;

    CurrentHp += actualHeal;
    remainingHeal -= actualHeal;
    yield return null;
}
```

**수정 의도:** 목표 HP를 고정하지 않고 **소비할 수 있는 회복량**을 별도로 관리한다. 회복 중 피해로 `CurrentHp`가 감소해도 `remainingHeal`은 늘어나지 않는다. `MaxHp - CurrentHp`로 최대 HP 초과도 제한한다.

**[코드 확인]** `OnHpAnimtion(InventorySlot slot)`은 빈 슬롯·수량 0·최대 HP·이미 회복 중인 상황을 검사하고, 회복 시작 시 슬롯 수량을 차감한다.

**[Unity 확인]** HP 30 + 회복 70 → 100, 회복 중 10 피해 → 최종 90, 마지막 의료품 사용 및 HPBar 반영을 개발자가 확인했다.

**[개선 제안]** 회복 중 사망, 게임 오브젝트 비활성화, 체력 데이터의 외부 직접 변경에 대한 경계 테스트가 추가로 필요하다.

참고: [HP_System.cs](https://github.com/youngchan-kim/fps_project/blob/main/Assets/Resources/Script/GameSystem/HP_System.cs)

## 7. HP UI 갱신과 초기화 책임

**[코드 확인]** `Player.Initialize()`에서 `hp.Initialize()` 후 `hpBar.SetMaxHealth()`와 `hpBar.SetHealth()`를 호출한다. `Player.Update()`는 `hp.GetCurHp()`를 읽어 `HPBar.SetHealth()`로 반영한다. 따라서 `HP_System`이 UI를 직접 조작하는 구조가 아니라 **Player가 체력값을 읽고 UI에 전달하는 구조**다.

**[개선 제안]** `Player.Start()`에서 `hpBar.SetMaxHealth(maxHealth)`가 null 검사 없이 호출되므로 Inspector 연결을 확인한다. UI 갱신을 이벤트 기반으로 바꾸는 것은 향후 최적화 과제이며 현재 구현 기능이 아니다.

## 8. 추가 정적 리뷰 — 코드에서 바로 확인되는 위험

아래는 실제 GitHub `main`에서 읽은 코드에 근거한 검토이며, **Unity에서 버그가 재현되었다는 뜻은 아니다.**

### 8.1 `Player.Start()`의 HPBar 참조

```csharp
hpBar.SetMaxHealth(maxHealth);
```

`Player.Initialize()`에는 `hpBar != null` 검사가 있지만 `Start()`의 호출에는 없다. Inspector에서 HPBar가 미할당이면 `NullReferenceException`이 발생할 수 있다. 두 진입점의 참조 검사를 일치시키거나 초기화 책임을 한곳으로 모으는 편이 안전하다.

### 8.2 `GunSystem.Awake()`의 총기 데이터 참조

```csharp
bulletsLeft = Gun_property.magazineSize;
```

`Gun_property`가 할당되지 않은 프리팹에서는 `Awake()`에서 예외가 날 수 있다. `Start()`의 `GunMuzzle`, `attackPoint` 검사보다 먼저 실행되므로, 데이터 에셋 연결 검사도 `Awake()`에서 수행해야 한다.

### 8.3 `GunSystem.Start()`의 플레이어 전역 참조

```csharp
player = GameMgr.Instance.player.GetComponent<Player>();
```

플레이어용 총기와 적용 총기가 `GunSystem`을 공유한다. 적 총기까지 `GameMgr.Instance.player`를 참조하면 재장전 정책이 플레이어 인벤토리와 결합된다. 총기 소유자나 탄약 공급자를 명시적으로 전달하는 구조가 장기적으로 안전하다. 단, **현재 적 재장전에서 실제 오류가 발생했다고 단정하지 않는다.**

### 8.4 `Player.Start()`의 계층 인덱스 탐색

```csharp
Gun = transform.GetChild(0).GetChild(0).GetChild(0)
    .GetChild(2).GetChild(0).GetChild(0)
    .GetChild(2).GetChild(0).GetChild(0)
    .GetChild(0).GetChild(4);
```

자식 계층을 변경하면 참조 대상이 달라지거나 인덱스 예외가 발생할 수 있다. 총기 장착 위치 `Transform`을 `[SerializeField]`로 직접 연결하는 방식을 검토한다.

### 8.5 `HP_System`의 체력 값 변경 경로

`CurrentHp`가 public이므로 외부 코드에서 최대 HP를 넘기거나 음수로 설정할 수 있다. 데미지·회복·초기화를 전용 메서드로 통일하면 범위 제한과 생존 판정을 한곳에서 유지하기 쉽다. 이는 **개선 제안**이며 현재 적용된 설계로 소개하지 않는다.

## 9. 개선 우선순위 및 포트폴리오 표현

| 우선순위 | 후속 점검 | 이유 |
|---|---|---|
| 높음 | 초기화 순서·null 참조 및 GameMgr 실행 경로 | Play 중 예외와 상태 불일치 예방 |
| 높음 | 인벤토리 저장/복원과 아이템 수량·정렬 | 데이터 유실·수량 오류 위험 |
| 중간 | 재장전 애니메이션 최종 코드·경계 테스트 | UI와 실제 게임 상태의 일치 확인 |
| 중간 | 적의 독립 탄약 관리 | 플레이어 인벤토리 의존 제거 |
| 중간 | 총기 슬롯 장착·해제 실기능 테스트 | UI와 실제 장비 상태 동기화 |
| 낮음 | HP UI 이벤트 기반 갱신 | 불필요한 매 프레임 호출 축소 |

### 포트폴리오용 핵심 설명

> Unity FPS 프로젝트에서 기존 적 AI의 switch 기반 FSM을 상태 인터페이스와 개별 상태 클래스로 리팩터링하고, 플레이어 입력·총기 발사·피해 전달·HP 관리를 분리해 연결했습니다. 인벤토리 탄약을 사용하는 재장전을 구현하면서 실제 재장전 조건과 애니메이션 조건이 달라 발생한 문제를 수정했습니다. 의료품 회복에서는 고정 목표 HP 방식이 회복 중 피격을 상쇄하는 버그를 발견해 남은 회복량 기반 코루틴으로 개선했으며, HP 수치·아이템 소모·UI 반영을 Unity에서 확인했습니다. 인벤토리 기본 구조는 공개 튜토리얼을 참고했고 FPS 전용 기능을 확장했습니다.

## 10. 관련 문서

- [Overview](overview.md)
- [Architecture](architecture.md)
- [Enemy FSM 개발 기록](Enemy_FSM_Development.md)
- [사격·데미지 개발 기록](Shooting_Damage_System_Development.md)
- [인벤토리 개발 기록](Inventory_System_Development.md)
