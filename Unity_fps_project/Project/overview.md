# Unity FPS 프로젝트 개요 (Overview)

> 프로젝트: [fps_project](https://github.com/youngchan-kim/fps_project)  
> 개발 환경: Unity, C#  
> 문서 기준: 업로드된 3개 시스템 개발 기록 및 작성자가 보고한 Unity 실행 테스트

## 1. 프로젝트 소개

Unity를 이용해 FPS 게임의 적 AI, 총기 발사·피격, 인벤토리 및 회복 아이템 기능을 구현하고 연결한 개인 프로젝트다. 기능 구현뿐 아니라 기존 코드의 책임 분리, 오류 분석과 테스트를 통해 구조를 개선하는 데 중점을 뒀다.

## 2. 주요 기술과 적용 위치

| 기술 | 적용 내용 |
|---|---|
| C# / Unity | 플레이어 입력, 컴포넌트 간 연동, 게임 로직 |
| FSM / State Pattern | 적의 정찰·추적·공격·사망 상태 전환 |
| NavMeshAgent | 적 이동과 추적 |
| Physics.Raycast | 타겟 탐지 및 사격 명중 판정 |
| ScriptableObject | 아이템·총기 속성 데이터 |
| Coroutine | 회복량을 시간에 따라 HP에 적용 |
| Unity UI | 인벤토리 슬롯과 HP 표시 |

## 3. 핵심 기능

### 적 AI (Enemy FSM)
- `Idle`, `Reconnaissance`, `Run`, `Attack`, `Die` 상태 관리
- 시야 내 플레이어 탐지 후 추적·공격, 타겟 상실 시 마지막 위치 기반 정찰
- 기존 `switch` 기반 상태 관리를 `IEnemyState`와 상태 클래스로 분리

### 사격·피격·재장전
- 플레이어 입력에 따른 단발·연사, 총구 기준 Raycast 명중 처리
- `GunSystem → DamegeSystem → HP_System`을 통한 피해 전달
- 총기별 탄창 크기 적용, 인벤토리 보유 탄약을 사용한 재장전
- 탄약이 없는데 재장전 모션이 실행되는 문제 수정

### 인벤토리·의료품
- 일반·Ground·총기 슬롯 간 이동 처리 및 총기 장착 연동
- 의료품 사용 시 수량 감소와 HP 회복 처리
- 회복 도중 피해를 받아도 회복량이 늘어나지 않도록 `remainingHeal` 기반 코루틴 적용
- HPBar에 현재 HP 반영

## 4. 설계 및 개발 기여 범위

- **참고한 부분:** 공개 ScriptableObject 인벤토리 튜토리얼의 기본 데이터, 슬롯, 드래그 UI 구조.
- **확장·개선한 부분:** FPS에 필요한 Ground·총기 슬롯 처리, 의료품과 HP 연동, 기존 적 FSM의 State Pattern 리팩터링, 사격·탄약·재장전 흐름의 문제 수정.
- **표현 주의:** 기존 튜토리얼의 인벤토리 기초 기능 전체를 독자 개발했다고 소개하지 않는다.

## 5. 검증 현황

| 항목 | 현황 |
|---|---|
| 적 탐지·추적·공격·정찰 복귀 및 내부 HP·사망 | 개발자 Unity 동작 확인 |
| 탄창 소모, 보유 탄약 재장전, 탄약 없을 때 재장전 모션 제한 | 개발자 Unity 동작 확인 |
| 의료품 회복, 마지막 1개 사용, HP UI 갱신 | 개발자 Unity 동작 확인 |
| 회복 중 피해 10 반영: HP 30 + 회복 70 → 최종 90 | 개발자 Unity 동작 확인 |
| 인벤토리 저장/복원, 정렬 경계 사례, 총기 교체 중 재장전 | 추가 검증 필요 |

## 6. 대표 문제 해결 사례

1. **Enemy FSM 구조:** 상태 분기와 행동 구현이 한 클래스에 집중된 구조를 상태 인터페이스와 클래스 기반으로 재구성.
2. **재장전 불일치:** 실제 탄약 조건과 애니메이션 조건이 달라 발생한 문제를 재장전 가능 조건에 맞춰 수정.
3. **회복 중 피격:** 고정 목표 HP 방식이 회복 중 피해를 상쇄하는 문제를 남은 회복량 관리 방식으로 수정.

## 7. 상세 문서

- [Architecture](architecture.md) — 시스템 구조 및 의존 관계 *(작성 예정)*
- [Code Review](code-review.md) — 주요 코드 분석 *(작성 예정)*
- [Development Flow](development-flow.md) — 개발·검증 과정 *(작성 예정)*
- [Enemy FSM Development](Enemy_FSM_Development.md)
- [Shooting & Damage Development](Shooting_Damage_System_Development.md)
- [Inventory System Development](Inventory_System_Development.md)

## 8. 남은 작업

게임 실행 화면 및 핵심 코드 캡처 확보, 미검증 기능 점검, GitHub README 구성, 제출용 PDF 제작.
