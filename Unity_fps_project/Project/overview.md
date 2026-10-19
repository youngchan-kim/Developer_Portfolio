# Unity FPS — Overview

[소스 저장소](https://github.com/youngchan-kim/fps_project) · Unity / C# / NavMeshAgent / Raycast / ScriptableObject

## 목표와 구현 범위
플레이어의 사격·피격, 적 AI, 인벤토리와 회복 아이템을 하나의 FPS 게임 플레이로 연결한 개인 프로젝트다.

| 시스템 | 핵심 기능 | 검증 상태 |
|---|---|---|
| Enemy FSM | 탐지, 정찰, 추적, 공격, 사망 | 핵심 행동 Unity 동작 확인(개발자) |
| Shooting/Damage | 단발·연사, Raycast, 데미지, 탄창·재장전 | 주요 재장전 시나리오 확인(개발자) |
| Inventory | 아이템 획득·이동·버리기, 총기 슬롯, 의료품 사용 | 의료품과 HP 연동 확인, 기타 일부 미검증 |
| HP | 체력 감소, HPBar 갱신, 점진 회복, 회복 중 피격 반영 | Unity 확인(개발자) |

## 직접 확장한 범위
인벤토리의 기초 ScriptableObject/슬롯/드래그 UI는 공개 튜토리얼을 참고했다. Ground 인벤토리, 총기 슬롯 연동, 의료품 사용을 FPS 구조에 맞게 확장했다. Enemy는 기존 switch 기반 FSM을 상태 클래스 기반으로 리팩터링했다.

## 남은 과제
인벤토리 저장·복원, 정렬 경계 상황, 적의 독립 탄약 관리, GameMgr 초기화 순서, 재장전 중 무기 교체 등은 추가 검증이 필요하다.

자세한 내용은 [Architecture](Architecture.md), [Development Flow](Development_Flow.md), [Code Review](Code_Review.md) 참고.
