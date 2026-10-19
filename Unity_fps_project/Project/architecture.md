# Unity FPS 시스템 아키텍처 (Architecture)

> 프로젝트: [fps_project](https://github.com/youngchan-kim/fps_project)  
> 기준: `Enemy_FSM_Development.md`, `Shooting_Damage_System_Development.md`, `Inventory_System_Development.md`, `overview.md` 및 GitHub `main`의 `EnemyFSM.cs`, `GunSystem.cs`, `HP_System.cs`, `Player.cs` 확인(2026-10-19).  
> **범위:** 주요 게임플레이 시스템의 **논리적 구조**. Unity Scene/Prefab의 모든 Inspector 참조를 완전히 추출한 도면은 아니다. 화살표는 아래에서 정의한 호출·데이터 사용 관계를 의미한다.

## 1. 아키텍처 개요

프로젝트는 플레이어 입력, 총기·데미지, 적 AI, 인벤토리, HP/UI를 여러 Unity 컴포넌트와 ScriptableObject 데이터로 연결한다. 각 기능이 완전히 독립된 계층으로 구현된 것은 아니며, `Player`, `Enemy`, `GunSystem` 사이에 직접 참조가 존재한다.

```mermaid
flowchart TD
    Player["Player.cs<br/>입력 · 장착 총기 선택"]
    UI["UserInterface.cs<br/>드래그 · 아이템 사용"]
    Inv["InventoryObject / InventorySlot<br/>아이템·수량 데이터"]
    Pickup["PickUpController<br/>획득 · 장착 · 버리기"]
    Gun["GunSystem.cs<br/>사격 · 탄창 · 재장전"]
    GunData["Gun_Scriptable<br/>총기 속성"]
    ItemData["ItemObject<br/>아이템·탄약 속성"]
    Damage["DamegeSystem.cs<br/>피해 전달"]
    HP["HP_System.cs<br/>현재 체력 · 회복"]
    HPUI["HPBar.cs<br/>체력 UI"]
    Enemy["Enemy.cs<br/>이동 · 공격 행동"]
    FSM["EnemyFSM.cs / IEnemyState<br/>상태 전환"]
    Target["TargetCheck.cs<br/>대상 탐지"]
    Nav["NavMeshAgent<br/>이동"]

    Player -->|현재 활성 총기·발사 요청| Gun
    Player -->|상속·아이템 획득| Pickup
    Player -->|현재 체력 읽기·UI 갱신| HP
    Player --> HPUI
    UI -->|슬롯 이동| Inv
    UI -->|의료품 사용| HP
    UI -->|장착·버리기 처리 연결| Pickup
    Pickup --> Inv
    Gun -->|총기 속성 읽기| GunData
    Gun -->|탄약 종류 조회| ItemData
    Gun -->|재장전 보유 탄약 조회| Player
    Gun -->|명중 대상에 피해| Damage
    Damage -->|HP 감소| HP
    Enemy -->|상태 관리| FSM
    FSM -->|행동 메서드 호출| Enemy
    Enemy --> Target
    Enemy --> Nav
    Enemy -->|공격 시 사용| Gun
```

> **화살표 주의:** `Player → HPUI`는 `Player.Update()`의 표시 갱신 호출이다. `HP_System`이 `HPBar`를 직접 갱신하는 구조로 표현하지 않는다. `GunSystem → Player`는 재장전 시 플레이어 인벤토리 조회에 따른 현재 구현상의 의존 관계다.

## 2. 주요 클래스와 책임

| 영역 | 클래스·구성 요소 | 책임 |
|---|---|---|
| 플레이어 | `Player` | 입력, 현재 활성 총기 탐색, HP 초기화, HP UI 갱신 |
| 아이템 | `PickUpController` | 아이템 획득·버리기 및 총기 슬롯 연동 |
| 인벤토리 | `UserInterface`, `InventoryObject`, `InventorySlot` | UI 입력 분기, 슬롯 및 수량 관리 |
| 데이터 | `ItemObject`, `Gun_Scriptable` | 아이템/탄약 데이터, 총기별 성능 설정 |
| 전투 | `GunSystem` | 발사 조건, Raycast, 탄창, 재장전 |
| 피격 | `DamegeSystem`, `HP_System` | 피해 전달, 현재 HP, 생존 및 회복 |
| 표시 | `HPBar` | 체력 Slider 값 표시 |
| 적 AI | `EnemyFSM`, `IEnemyState`, 상태 클래스 | 상태 관리·전환 조건 |
| 적 행동 | `Enemy`, `TargetCheck`, `NavMeshAgent` | 탐지, 이동, 정찰, 공격 |

### 핵심 클래스 관계 (선별)

```mermaid
classDiagram
    class PickUpController
    class Player {
        +Initialize()
        +GunSystem_chage()
        +GetGunSystem()
    }
    class GunSystem {
        +Firing()
        +HasAmmo() bool
        +ReloadIsPossible() bool
        +Reload()
    }
    class DamegeSystem {
        +TakeDamage(damage)
    }
    class HP_System {
        +CurrentHp float
        +GetCurHp() float
        +OnHpAnimtion(slot)
    }
    class HPBar {
        +SetMaxHealth(value)
        +SetHealth(value)
    }
    class EnemyFSM {
        +ChangeState(newState)
        +Update()
    }
    class IEnemyState {
        <<interface>>
        +Enter()
        +Update()
        +Exit()
    }
    class IdleState
    class RunState
    class AttackState
    class ReconnaissanceState
    class DieState
    class Enemy

    PickUpController <|-- Player
    Player --> GunSystem : 활성 총기 사용
    Player --> HP_System : HP 조회·초기화
    Player --> HPBar : UI 갱신
    GunSystem --> DamegeSystem : 피격 대상 호출
    DamegeSystem --> HP_System : 피해 반영
    Enemy --> EnemyFSM : FSM 보유
    EnemyFSM --> IEnemyState : 현재 상태
    IEnemyState <|.. IdleState
    IEnemyState <|.. RunState
    IEnemyState <|.. AttackState
    IEnemyState <|.. ReconnaissanceState
    IEnemyState <|.. DieState
```

> `classDiagram`은 설명에 필요한 일부 관계만 표현한다. 전체 멤버 목록이나 UML의 엄격한 소유권·생명주기 관계를 뜻하지 않는다.

## 3. 적 AI 상태 전환

`EnemyFSM`은 현재 `IEnemyState`를 보관하고 `ChangeState()`에서 기존 상태 `Exit()` → 새 상태 `Enter()`를 실행한다. 실제 NavMesh 이동·공격은 `Enemy`의 메서드에서 처리한다.

```mermaid
flowchart TD
    I["Idle<br/>탐지 결과에 따른 분기"] -->|탐지 성공| R["Run<br/>추적"]
    I -->|탐지 실패| S["Reconnaissance<br/>마지막 위치 정찰"]
    S -->|재탐지| R
    R -->|공격 거리 진입| A["Attack<br/>사격"]
    R -->|타겟 상실 또는 탐지 거리 이탈| I
    A -->|타겟 상실 또는 공격 거리 이탈| I
    I -.->|HP 소진| D["Die"]
    S -.->|HP 소진| D
    R -.->|HP 소진| D
    A -.->|HP 소진| D
```

`Attack → Idle`은 현재 타겟을 다시 확인한 뒤 추적 또는 정찰을 결정하기 위한 설계다. `Die` 판정은 `EnemyFSM.Update()`의 공통 경로에서 수행한다.

## 4. 사격·피격 처리 흐름

```mermaid
sequenceDiagram
    actor User as 사용자
    participant P as Player
    participant G as GunSystem
    participant Ph as Physics.Raycast
    participant D as DamegeSystem
    participant H as HP_System
    User->>P: 마우스 발사 입력
    P->>G: HasAmmo / ReadyToShoot / FiringIsPossible
    alt 발사 가능
        P->>G: Firing()
        G->>Ph: 총구 기준 Raycast
        alt 피격 대상에 DamegeSystem 존재
            Ph-->>G: RaycastHit
            G->>D: TakeDamage(damage)
            D->>H: HP 감소 요청
        end
        G->>G: 탄창 탄약 감소
    else 발사 불가
        P-->>User: 발사하지 않음
    end
```

- `GunSystem`의 총구(`GunMuzzle`) 기준 Raycast와 화면 중앙 조준 목표는 별개이므로 총구·조준점 정렬은 프리팹별로 확인해야 한다.
- 재장전은 탄창 부족분과 인벤토리 보유 탄약을 비교하여 보충하는 구조다.

## 5. 인벤토리·회복 처리 흐름

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as UserInterface
    participant Slot as InventorySlot
    participant HP as HP_System
    participant P as Player
    participant Bar as HPBar
    User->>UI: 의료품 우클릭 사용
    UI->>HP: OnHpAnimtion(slot)
    HP->>HP: 슬롯·수량·HP·회복 중 여부 검사
    alt 사용 가능
        HP->>Slot: totalamount 감소
        HP->>HP: HpSliderAnimation(heal)
        loop 남은 회복량 > 0
            HP->>HP: CurrentHp 증가, remainingHeal 감소
            Note over HP: 피격은 별도로 CurrentHp 감소
            P->>HP: GetCurHp()
            P->>Bar: SetHealth(CurrentHp)
        end
    else 사용 불가
        HP-->>UI: 회복 시작하지 않음
    end
```

**검증 예:** HP 30에서 회복량 70인 의료품을 사용해 회복 중 10 피해를 받으면 최종 HP는 90으로 유지된다. 회복량을 고정 목표 HP가 아닌 `remainingHeal`로 관리하기 때문이다. 의료품 마지막 1개 소모와 HP UI 갱신도 개발자가 Unity에서 확인했다.

## 6. 인벤토리 이동과 총기 장착

```mermaid
flowchart TD
    A["UserInterface.OnDragEnd()"] --> B{"도착 InventoryID"}
    B -->|0: Ground| C["ItemDropSystem() 등<br/>바닥으로 버리기"]
    B -->|1: 일반 인벤토리| D["획득·추가·제거 처리"]
    B -->|3: 총기 슬롯| E["슬롯 허용 검사<br/>SwapItems / GunSlot_swap"]
    B -->|2| F["분기 존재<br/>핵심 처리 주석 상태"]
    C --> G["슬롯/월드 상태 연동"]
    D --> G
    E --> G
```

기본 슬롯·드래그 UI 구조는 공개 ScriptableObject 인벤토리 튜토리얼을 참고했다. FPS 전용 Ground 이동, 총기 슬롯 연동, 의료품 사용은 프로젝트에 맞춰 확장한 영역이다. **총기 장착·버리기의 모든 예외 시나리오가 검증되었다는 의미는 아니다.**

## 7. 현재 구조의 한계 및 개선 후보

| 항목 | 현재 구조 또는 확인된 한계 | 향후 방향 |
|---|---|---|
| 총기 참조 | `Player`가 깊은 `transform.GetChild()` 계층을 통해 총기 부모를 찾음 | Inspector 참조 등 명시적 연결 |
| 적 총기 재장전 | `GunSystem`이 `GameMgr.Instance.player`를 통해 플레이어를 참조 | 적·플레이어 탄약 정책 분리 |
| UI 갱신 | `Player.Update()`에서 HPBar 값을 매 프레임 갱신 | HP 변경 이벤트 방식 검토 |
| 저장·불러오기 | 인벤토리 `BinaryFormatter` 및 GameObject 참조 직렬화 문제 | 직렬화 가능한 데이터만 분리 |
| 슬롯 처리 | 수량 누적·정렬·드래그 예외 사례 추가 검증 필요 | 경계값 테스트 및 방어 코드 |
| 초기화 | `Enemy` 중복 초기화 호출은 수정 확인, `GameMgr` 경로는 추가 점검 | 초기화 책임·호출 시점 정리 |

위 표의 개선 후보는 **현재 구현 완료 기능이 아니다.**

## 8. 관련 문서

- [Overview](overview.md)
- [Enemy FSM 개발 기록](Enemy_FSM_Development.md)
- [Shooting & Damage 개발 기록](Shooting_Damage_System_Development.md)
- [Inventory 개발 기록](Inventory_System_Development.md)
- [Code Review](code-review.md) *(작성 예정)*
- [Development Flow](development-flow.md) *(작성 예정)*

---

**검증 상태:** 핵심 소스 구조 확인 + 개발자가 보고한 Unity 실행 결과. 전체 Scene/Prefab 자동 분석이나 모든 경계 조건의 테스트 완료를 의미하지 않는다.
