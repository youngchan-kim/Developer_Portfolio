# Code Review — 핵심 코드 검토

> 기준: 3개 시스템 개발 문서 및 GitHub `main`의 `EnemyFSM.cs`, `GunSystem.cs`, `HP_System.cs`, `Player.cs` 일부. Unity 검증은 개발자 테스트 보고와 구분한다.

## 1. Enemy FSM
**확인:** `EnemyFSM.ChangeState()`가 기존 상태 `Exit()` 후 새 상태 `Enter()`를 실행한다. `Update()`에서 공통 사망 상태 전환을 검사한다.

**장점:** 상태 전환 조건을 상태 클래스로 분리하고 이동·공격 구현은 `Enemy`에 유지한다. `Attack → Idle`은 타겟 탐지 재판단을 위한 설계다.

**후속 검토:** FSM null·초기화 순서, 사망 상태 반복 처리.

## 2. Shooting / Damage
**확인:** `GunSystem.Awake()`에서 `Gun_property.magazineSize`로 탄창 초기화. `HasAmmo()`로 탄약 검사. `Shoot()`은 총구 기준 Raycast로 `DamegeSystem.TakeDamage()`를 호출한다.

**장점:** 입력(`Player`), 발사(`GunSystem`), 피해(`DamegeSystem`)의 책임 구분.

**후속 검토:** 적 총기에서 플레이어 인벤토리를 참조하는 재장전 구조, 총기·탄약 에셋 null, 재장전 중 교체.

## 3. HP 회복 버그 분석
**초기 결함:** 회복량을 계산했지만 `CurrentHp`를 증가시키지 않음.

**중간 결함:** 시작 시 고정한 `targetHp`까지 회복하여 중간 피격이 상쇄됨.

**현재 확인 코드:**
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
**검증:** HP 30에서 70 회복 도중 10 피해 → 최종 HP 90(개발자 Unity 확인).

**후속 검토:** 사망 중 회복, 오브젝트 비활성화 시 코루틴 정리.

## 4. Player / HPBar
**확인:** `Player.Initialize()`에서 `hp.Initialize()` 후 HPBar 초기값 설정. `Player.Update()`에서 `GetCurHp()`를 Slider에 전달한다.

**후속 검토:** `Player.Start()`의 `hpBar.SetMaxHealth()`는 null 검사 없이 호출되므로 Inspector 미할당 시 예외 가능. 매 프레임 갱신은 추후 변경 이벤트 방식 고려.

## 5. Inventory
**확인:** 기본 ScriptableObject 인벤토리 구조는 공개 튜토리얼 참고, Ground·총기 슬롯·의료품 연결은 확장. 의료품 수량 감소와 마지막 1개 사용은 Unity 검증.

**후속 검토:** `InventorySlot.AddAmount()` 인자 사용, `Sort()` 다중 빈 슬롯, `BinaryFormatter` 저장, 드래그 Dictionary 키 검사.

## 6. 우선순위
| 우선순위 | 개선 대상 |
|---|---|
| 높음 | 초기화 순서·null 참조, 저장/복원 검증 |
| 중간 | 적 전용 탄약 정책, 인벤토리 수량·정렬 경계값 |
| 낮음 | HP UI 변경 이벤트 기반 갱신 |

**구분:** 코드 확인 ≠ Unity 전체 실행 검증. 개선 제안 ≠ 이미 구현한 기능.
