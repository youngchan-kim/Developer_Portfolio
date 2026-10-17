# FPS 사격 및 데미지 처리 시스템 개발 과정

> 프로젝트: Unity FPS / 언어: C#  
> 근거: [fps_project GitHub](https://github.com/youngchan-kim/fps_project), main 브랜치 (2026-10-18 확인)  
> 문서 상태: 소스코드 기반 초안. Unity 플레이 모드에서 모든 기능을 재검증한 문서는 아님.

## 1. 구현 목적
플레이어의 발사 입력에서 총기 발사, Raycast 명중 판정, 피격 대상의 체력 감소까지 여러 컴포넌트가 연결되는 FPS 전투 기능을 구성했다. 본 문서는 코드로 확인 가능한 구현 범위를 설명한다. 구체적인 최초 설계 동기, 참고 자료, 구현 과정의 문제 해결 경험은 개발자 확인 후 보완한다.

## 2. 주요 클래스 및 책임

| 파일 | 책임 |
|---|---|
| `Assets/Resources/Script/Player/FPSControl/Player.cs` | 마우스 입력, 단발/연사 조건 확인, `GunSystem.Firing()` 호출, R키 재장전 요청 |
| `Assets/Resources/Script/GameSystem/GunSystem.cs` | 총기 속성 참조, 발사 가능 여부, 탄약 카운트, 발사 간격, 총구 이펙트, Raycast, 데미지 전달 |
| `Assets/Resources/Script/Cam/Cam_ScreenCenter.cs` | 카메라 화면 중앙을 월드 좌표로 변환해 조준 목표 지점 계산 |
| `Assets/Resources/Script/Enemy/Findattackpoint.cs` | 적 AI의 공격 목표를 플레이어 머리 위치 또는 전방 지점으로 설정 |
| `Assets/Resources/Script/GameSystem/DamegeSystem.cs` | `TakeDamage()`에서 HP 감소 후 생존 판정 요청 |
| `Assets/Resources/Script/GameSystem/HP_System.cs` | 최대/현재 체력, 생존 플래그 관리 |
| `Assets/Resources/Script/Enemy/EnemyFSM.cs` | `Enemy.IsDead()` 결과를 이용한 `DieState` 전환 |

## 3. 실제 호출 흐름

```text
Player.Update()
  └─ 마우스 입력 / 발사 조건 검사
      └─ GunSystem.Firing()
          ├─ 조준 흔들림 · 총구 파티클
          └─ GunSystem.Shoot()
              ├─ GunMuzzle 기준 Physics.Raycast
              ├─ 충돌 Collider에 DamegeSystem이 있으면 TakeDamage(damage)
              │   └─ HP_System.SetCurHp() → Lifecheck()
              ├─ 충돌 위치에 피격 흔적 오브젝트 생성
              └─ 탄약 차감 / Invoke를 통한 발사 간격 처리
```

적 AI는 `Enemy.UpdateAttack()`에서 발사 조건에 따라 같은 `GunSystem`을 호출한다. 적 사망은 HP의 생존 플래그를 `EnemyFSM.Update()`가 확인하는 별도 흐름이다.

## 4. 구현에서 드러나는 설계 요소

### 입력과 총기 처리의 분리
`Player.cs`는 단발/연사 입력과 발사 조건을 확인하고, `GunSystem.cs`는 실제 발사 및 명중 판정을 담당한다. 총기 속성은 `Gun_Scriptable`에서 읽는다.

### 화면 중앙 조준과 총구 발사
`Cam_ScreenCenter.cs`는 카메라 뷰포트 중앙을 월드 좌표로 변환해 조준 위치를 갱신한다. `GunSystem.cs`는 총구 위치와 전방 벡터로 Raycast를 수행한다. 화면 중앙 조준점과 총구에서 발사하는 광선은 별도 처리이므로 정렬 여부는 Unity에서 검증해야 한다.

### 피격 처리의 재사용
`GunSystem`은 충돌한 Collider에 `DamegeSystem`이 있는지 검사한 뒤 `TakeDamage()`를 호출한다. HP 감소와 생존 판정은 별도 컴포넌트가 수행한다.

## 5. 현재 코드의 한계 및 추가 검증

- `GunSystem.Awake()`에서 `bulletsLeft = 3000`으로 초기화하므로 정상적인 탄창 크기 제한 구현으로 설명할 수 없다.
- `ReloadFinished()`에서 실제 탄약 보충 코드가 주석 처리되어 있다. 재장전 완료 기능은 미완성이다.
- `GunSystem.BulletIsEmpty()`는 이름과 달리 **탄약이 남아 있을 때** `true`를 반환한다. 명명 개선이 필요하다.
- 발사 처리에 `Invoke("Shoot", ...)`를 사용한다. 연속 발사와 상태 변경 시 중복 호출·취소 처리 검증이 필요하다.
- `Cam_ScreenCenter`의 조준 목표와 총구 방향이 항상 일치하는지 플레이 모드 검증이 필요하다.
- `HP_System.Lifecheck()`는 현재 체력이 0 이하일 때 생존 플래그를 false로 바꾼다. 실제 사망 애니메이션과 오브젝트 비활성화는 별도 확인이 필요하다.
- 컴포넌트를 다단계 `transform.GetChild()`로 탐색하는 부분은 프리팹 계층 구조 변경에 취약하다.

## 6. 실행 증빙 준비

- [ ] 플레이어 단발 발사 영상 또는 스크린샷
- [ ] 연사 모드에서 연속 발사 확인
- [ ] Raycast 명중과 피격 대상 HP 감소 확인
- [ ] 적의 사격과 플레이어 피격 확인
- [ ] 사망 전환 확인
- [ ] 재장전의 실제 작동 여부 확인 및 미완성 범위 기록

## 7. 개발자 확인 후 작성할 개발 과정

다음은 코드만으로 추정하지 않고 개발자의 실제 경험을 확인한 후 보완한다.

1. 조준점과 총구 발사를 별도로 설계한 이유
2. `GunSystem`과 `DamegeSystem`을 분리한 계기
3. 구현 중 가장 오래 걸렸던 문제와 실제 해결 방법
4. 본인이 직접 설계한 부분과 참고한 코드/자료의 구분

## 8. 포트폴리오 요약 초안

Unity C# FPS 프로젝트에서 입력, 총기 발사, Raycast 명중 판정, 데미지 전달, 체력 관리를 여러 컴포넌트로 연결했다. 플레이어 입력과 발사 처리, 피격 처리와 HP 관리를 분리해 기능별 책임을 구분했다. 단, 탄약 보충과 재장전 등 일부 기능은 미완성이며, 각 전투 시나리오의 실행 검증 결과는 추가 기록이 필요하다.
