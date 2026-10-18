# FPS 사격 및 데미지 처리 시스템 개발 과정

> 프로젝트: Unity FPS / 언어: C#  
> 기준 저장소: [fps_project GitHub](https://github.com/youngchan-kim/fps_project), `main` 브랜치  
> 갱신일: 2026-10-18  
> 검증 구분: 소스코드에서 확인한 구현과 개발자가 Unity에서 직접 확인했다고 설명한 동작을 구분하여 기록함. 전체 시나리오의 자동화 테스트 결과는 없음.

## 1. 구현 목적

플레이어의 발사 입력부터 총기 발사, Raycast 명중 판정, 피격 대상 체력 감소까지 연결되는 FPS 전투 시스템을 구현했다. 이후 탄창 크기와 인벤토리의 탄약 수량을 연동해 재장전할 수 있도록 수정했다. 본 문서는 현재 구현한 범위와 문제 해결 과정을 기록하며, 새로운 대규모 기능은 추가하지 않고 향후 개선 항목으로 분리한다.

## 2. 주요 클래스 및 책임

| 파일 | 책임 |
|---|---|
| `Assets/Resources/Script/Player/FPSControl/Player.cs` | 마우스 발사 입력, 단발/연사 조건 확인, 현재 활성 총기 탐색, R키 재장전 요청 |
| `Assets/Resources/Script/GameSystem/GunSystem.cs` | 총기 속성, 발사 간격, 탄창 탄약, Raycast 명중 판정, 이펙트, 인벤토리 탄약을 사용한 재장전 |
| `Assets/Resources/Script/Player/PlayerMovement.cs` | 플레이어 이동 및 발사·재장전 등 애니메이션 입력 처리 |
| `Assets/Resources/Script/Player/Player_Action.cs` | Animator의 재장전 상태를 포함한 캐릭터 애니메이션 제어 |
| `Assets/Resources/Script/Item/Player_recognizes_Item.cs` | 인벤토리의 탄약 아이템 수량 조회·변경 및 빈 슬롯 정리 |
| `Assets/Resources/Scriptable Object/items/Script/ItemObject.cs` | 탄약 등 인벤토리 아이템 데이터의 ScriptableObject 정의 |
| `Assets/Resources/Scriptable Object/items/gun/Gun_Scriptable.cs` | 탄창 크기, 발사 속도, 데미지, 재장전 시간 등 총기 속성 정의 |
| `Assets/Resources/Script/Cam/Cam_ScreenCenter.cs` | 화면 중앙을 기준으로 조준 목표 지점 계산 |
| `Assets/Resources/Script/Enemy/Findattackpoint.cs` | 적 AI의 공격 목표 설정 |
| `Assets/Resources/Script/GameSystem/DamegeSystem.cs` | 피격 시 `TakeDamage()`를 통한 HP 감소 요청 |
| `Assets/Resources/Script/GameSystem/HP_System.cs` | 최대/현재 체력 및 생존 여부 관리 |
| `Assets/Resources/Script/Enemy/EnemyFSM.cs` | 사망 상태 전환 |

## 3. 사격 및 데미지 처리 흐름

```text
Player.Update()
  ├─ GunSystem_chage() : 현재 활성화된 총기 탐색
  └─ 마우스 입력 / 단발·연사 / HasAmmo() / ReadyToShoot() 조건 검사
      └─ GunSystem.Firing()
          ├─ 조준 흔들림 및 총구 파티클
          └─ GunSystem.Shoot()
              ├─ GunMuzzle 기준 Physics.Raycast
              ├─ 충돌 대상의 DamegeSystem.TakeDamage(damage)
              │   └─ HP_System : 현재 HP 감소 및 생존 판정
              ├─ 피격 위치에 흔적 오브젝트 생성
              └─ bulletsLeft 감소 / Invoke를 통한 발사 간격 처리
```

`Player.cs`는 입력 및 발사 가능 조건을 확인하고, `GunSystem.cs`는 발사·명중·탄약 소모를 처리한다. 적 AI도 `Enemy.UpdateAttack()`에서 `GunSystem`을 호출하지만, 적의 재장전 로직은 플레이어와 동일하게 정상 동작하는지 별도 검증이 필요하다.

## 4. 탄약 및 재장전 구현

### 4.1 탄창 탄약 초기화

이전 테스트용 `bulletsLeft = 3000` 대신 `GunSystem.Awake()`에서 `Gun_property.magazineSize`로 초기화하도록 변경했다. 확인한 M4 총기 속성 에셋의 탄창 크기는 30발이다.

### 4.2 발사 가능 조건

`HasAmmo()`는 `bulletsLeft > 0`일 때 참을 반환한다. 이전에 의미가 불명확했던 `BulletIsEmpty()` 호출은 플레이어 및 적의 실제 발사 분기에서 `HasAmmo()`로 교체했다. 과거 메서드 이름이 주석 코드에 남아 있을 수 있으나 실행 경로와는 구분한다.

### 4.3 인벤토리 연동 재장전

```text
R키 입력
  └─ GunSystem.ReloadIsPossible()
      ├─ 이미 재장전 중인가?
      ├─ 탄창이 가득 찼는가?
      └─ 인벤토리에 해당 탄약 아이템이 있는가?
          └─ 가능하면 GunSystem.Reload()
              ├─ reloading = true
              └─ reloadTime 후 ReloadFinished()
                  ├─ 필요한 탄약 = magazineSize - bulletsLeft
                  ├─ 보충 탄약 = Min(보유 탄약, 필요한 탄약)
                  ├─ 탄창 수량 증가
                  ├─ 인벤토리 수량 차감
                  ├─ 인벤토리 UI/슬롯 갱신 요청
                  └─ reloading = false
```

`GunSystem`의 `bullet` 필드에는 탄약용 `ItemObject` 에셋이 연결되어야 한다. 저장소에는 `AR_Bullet.asset`(아이템 ID 2)과 `SR_Bullet.asset`(아이템 ID 3)이 존재한다. 에셋 파일의 존재와 각 총기 프리팹 Inspector에서의 실제 연결 여부는 별개다.

### 4.4 실제 동작 확인 범위

개발자가 Unity 플레이 중 다음 동작을 확인했다고 설명했다.

- 탄창의 총알을 소모하면 발사할 수 없음
- 인벤토리에 탄약이 있을 때 재장전하면 다시 발사할 수 있음
- 인벤토리 탄약이 없어도 R키 입력 시 플레이어 재장전 모션이 실행되는 문제를 발견함
- 해당 모션의 실행 조건을 수정한 뒤 문제가 해결되었다고 확인함

위 내용은 개발자의 플레이 테스트 보고에 근거한다. 재장전 도중 무기 교체, 연속 R키 입력, 장착 총기별 탄약 구분 등의 경계 상황은 별도 검증이 필요하다.

## 5. 문제 발견 및 수정 과정

### 문제 A. 테스트용 탄약 수량과 메서드 명칭

- **기존:** `bulletsLeft = 3000`으로 테스트했고, `BulletIsEmpty()`는 이름과 달리 탄약이 남았는지를 반환했다.
- **수정:** 총기 ScriptableObject의 `magazineSize`로 탄창 수량을 초기화하고, 발사 분기에서 `HasAmmo()`를 사용하도록 변경했다.
- **의의:** 테스트용 설정을 실제 탄창 제한에 맞추고, 조건문의 의미를 읽기 쉽게 정리했다.

### 문제 B. 재장전 시 인벤토리 탄약 반영

- **기존 초안:** 재장전 완료 시 탄약 보충 코드가 비활성화된 상태였다.
- **수정:** `ReloadFinished()`에서 부족한 탄약과 보유 수량을 비교해 탄창에 보충하고, 인벤토리 탄약을 차감하도록 연결했다.
- **의의:** 총기 탄창과 인벤토리 데이터를 분리해 관리하면서 재장전 시 연동하도록 했다.

### 문제 C. 탄약이 없어도 재장전 모션 실행

- **현상:** 탄약이 없는 상태에서 R키를 눌러도 플레이어 캐릭터가 재장전 모션을 실행했다.
- **원인:** `Player.cs`의 실제 재장전은 `ReloadIsPossible()`로 제한했지만, `PlayerMovement.cs`에서는 R키 입력만 확인하고 `anim.Reloading()`과 `ReloadTimer()`를 실행했다. 실제 탄약 처리와 애니메이션 시작 조건이 분리되어 있었다.
- **수정 방향 및 개발자 확인:** 재장전 애니메이션 시작에도 현재 총기의 `ReloadIsPossible()` 조건을 적용하도록 수정했으며, 개발자는 문제 해결을 확인했다. **이 최종 수정 코드가 GitHub에 반영되었는지는 별도로 재확인해야 한다.**
- **의의:** 실제 게임 상태와 시각적 피드백의 실행 조건을 일치시켰다.

## 6. 구현에서 드러나는 설계 요소

### 입력과 총기 처리의 분리

`Player.cs`는 사용자 입력 및 조건 검사를, `GunSystem.cs`는 실제 발사와 명중 판정을 담당한다. 현재 활성화된 총기 오브젝트를 탐색해 `GunSystem` 참조를 갱신하는 코드도 추가되었다.

### 총기 데이터와 동작의 분리

총기별 데미지, 탄창 크기, 재장전 시간 등의 속성을 `Gun_Scriptable`로 관리한다. 인벤토리의 탄약 종류는 별도 `ItemObject`로 구분한다.

### 피격 처리의 재사용

`GunSystem`은 충돌한 Collider의 `DamegeSystem`에 데미지를 전달하고, 체력 감소와 생존 판정은 `HP_System`이 담당한다.

### 화면 중앙 조준과 총구 발사

`Cam_ScreenCenter`가 계산하는 화면 중앙 조준 위치와 `GunMuzzle` 기준 Raycast는 별도 처리다. 실제 조준점과 탄착점의 일치 여부는 카메라·총기 배치에 따라 검증이 필요하다.

## 7. 남은 한계 및 향후 개선 방향

이번 포트폴리오 정리 단계에서는 큰 기능을 새로 구현하지 않고 아래 내용을 후속 개선 과제로 기록한다.

1. **애니메이션과 재장전의 통합:** `Player.cs`의 재장전 성공 처리에서 모션을 함께 시작하도록 구조를 단순화하고, 재장전 시간과 애니메이션 타이머를 일치시킨다. 현재 `PlayerMovement.ReloadTimer()`는 고정 3초를 사용한다.
2. **null 및 슬롯 안전성:** 총기의 `bullet` 미연결, 인벤토리의 빈 슬롯, `ItemObject` 누락 등에 대한 방어 코드를 보완한다. 관련 메서드의 최종 반영 상태는 확인이 필요하다.
3. **적 AI 재장전:** 적 총기에도 플레이어 인벤토리를 참조하는 코드가 있으므로, 적 전용 탄약·재장전 정책을 분리할 필요가 있다.
4. **계층 구조 의존성:** 다단계 `transform.GetChild()` 탐색은 프리팹 구조 변경에 취약하므로 Inspector 참조 또는 명시적 컴포넌트 연결을 검토한다.
5. **발사 예약 관리:** `Invoke()` 기반 연속 발사 시 무기 교체·비활성화·재장전과의 상호작용을 검증한다.
6. **명중 및 사망 처리:** 조준점·총구 정렬, 적과 플레이어의 HP 감소, 사망 상태 전환을 플레이 모드에서 추가 확인한다.

## 8. 실행 증빙 체크리스트

- [ ] 단발 사격 및 연사 모드 영상 또는 스크린샷
- [ ] Raycast 명중 및 적 HP 감소 화면
- [x] 탄창 탄약 소모 후 발사 제한 — 개발자 동작 확인
- [x] 인벤토리 탄약으로 재장전 후 재발사 — 개발자 동작 확인
- [x] 탄약이 없을 때 재장전 모션이 실행되는 문제 수정 — 개발자 해결 확인
- [ ] 탄창이 가득 찼을 때 재장전·모션 제한 재확인
- [ ] 재장전 시 인벤토리 수량 차감 화면 캡처
- [ ] 적의 사격·플레이어 피격·사망 전환 검증
- [ ] 최신 GitHub에 애니메이션 조건 수정 코드 반영 여부 확인

## 9. 개발자 확인 후 보완할 사항

1. 화면 중앙 조준과 총구 Raycast를 별도로 구성한 이유
2. `GunSystem`, `DamegeSystem`, `HP_System`을 분리한 과정
3. 직접 작성한 코드와 참고 자료를 바탕으로 수정·확장한 부분
4. 재장전 버그 수정 전후 화면 또는 코드 비교 자료

## 10. 포트폴리오 요약

Unity C# FPS 프로젝트에서 플레이어 입력, 총기 발사, Raycast 명중 판정, 데미지 전달, 체력 관리를 분리된 컴포넌트로 연결했다. 총기 속성의 탄창 크기를 적용하고 인벤토리의 탄약 아이템 수량을 조회·차감하는 재장전을 구현했다. 테스트 과정에서 실제 재장전 조건과 플레이어 애니메이션 실행 조건이 달라 탄약 없이도 재장전 모션이 나오는 문제를 발견했고, 애니메이션 시작 조건을 재장전 가능 여부와 연결해 수정했다. 향후 개선 항목으로 재장전·애니메이션 처리 통합, 참조 안전성, 적 AI의 독립적인 탄약 관리 등을 정리했다.
