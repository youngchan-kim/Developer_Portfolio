# Architecture — 시스템 구조

> 아래는 핵심 소스와 기존 개발 문서를 기반으로 한 논리적 연결도이며, 전체 프리팹의 모든 참조를 추출한 것은 아니다.

## 시스템 연결
```mermaid
flowchart TD
    P[Player 입력·HP UI 갱신] --> G[GunSystem 발사·재장전]
    P --> I[PickUpController / Inventory]
    UI[UserInterface 아이템 사용] --> I
    UI --> H[HP_System]
    I --> G
    G --> D[DamegeSystem]
    D --> H
    H --> B[HPBar Slider]
    E[Enemy 이동·공격] --> F[EnemyFSM / IEnemyState]
    E --> T[TargetCheck]
    E --> G
```

## Enemy 상태 전환
```mermaid
flowchart TD
    I[Idle 판단] -->|탐지| R[Run 추적]
    I -->|미탐지| S[Reconnaissance 정찰]
    S -->|재탐지| R
    R -->|공격 거리| A[Attack]
    R -->|타겟 상실| I
    A -->|상실·거리 이탈| I
    I -.-> D[Die]
    S -.-> D
    R -.-> D
    A -.-> D
```

## 의료품 사용 시퀀스
```mermaid
sequenceDiagram
    actor U as 사용자
    participant UI as UserInterface
    participant HP as HP_System
    participant P as Player
    participant B as HPBar
    U->>UI: 의료품 사용
    UI->>HP: OnHpAnimtion(slot)
    HP->>HP: 조건 확인, 수량 차감, 코루틴 시작
    loop 회복량 남음
        HP->>HP: CurrentHp 증가, remainingHeal 감소
        Note over HP: 중간 피격은 CurrentHp 감소
        P->>HP: GetCurHp()
        P->>B: SetHealth()
    end
```

## 클래스 책임
- `Player`: 입력, 활성 총기, HP 초기화 및 UI 반영
- `GunSystem`: 사격, Raycast, 탄약, 재장전
- `DamegeSystem`: 피격 대상 HP 감소 연결
- `HP_System`: 현재 HP 및 코루틴 회복
- `EnemyFSM`, `IEnemyState`: 상태 전환
- `Enemy`, `TargetCheck`: 이동·공격과 타겟 탐지
- `UserInterface`, `InventoryObject`, `InventorySlot`: 슬롯 이동 및 아이템 사용

## 설계상 주의
`transform.GetChild()` 계층 의존, GameMgr 초기화 경로, 인벤토리 직렬화와 적의 탄약 관리 분리는 후속 점검 대상이다.
