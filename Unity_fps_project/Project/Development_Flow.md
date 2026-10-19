# Development Flow — 개발 절차와 실행 흐름

`flowchart TD`는 Mermaid에서 위→아래 방향의 흐름도 선언이다. 아래 **개발 절차**와 **게임 실행 흐름**은 서로 다르다.

## 개발·검증 절차
```mermaid
flowchart TD
    A[요구사항 정의] --> B[기존 코드·참고 자료 분석]
    B --> C[구현 및 연결]
    C --> D[Unity Play 테스트]
    D --> E{정상 작동?}
    E -->|아니오| F[로그·상태값·호출 순서 분석]
    F --> G[원인 수정]
    G --> D
    E -->|예| H[Git 커밋]
    H --> I[개발 문서 갱신]
```
이 그림은 반복 작업 절차의 개념도다. 실제 변경의 날짜순 기록은 Git 커밋 이력을 근거로 별도 작성해야 한다.

## 확인된 개선 과정
1. **Enemy:** switch FSM → `IEnemyState` 상태 클래스 분리 → 중복 Enemy 초기화 호출 제거 확인.
2. **Shooting:** 테스트용 탄약값 제거 → 탄창·인벤토리 재장전 연동 → 탄약이 없을 때 재장전 애니메이션만 실행되던 문제 수정.
3. **Inventory/HP:** 의료품 사용 연결 → `CurrentHp` 미증가 수정 → 고정 목표 HP가 피격을 상쇄하던 문제 수정 → `remainingHeal` 방식으로 검증.

## 런타임: 사격
```mermaid
flowchart TD
    A[마우스 입력] --> B[Player 발사 조건 검사]
    B --> C{탄약·발사 간격·재장전 조건 통과?}
    C -->|아니오| X[발사 취소]
    C -->|예| D[GunSystem.Firing / Shoot]
    D --> E[Physics.Raycast]
    E --> F{피격 대상에 DamegeSystem?}
    F -->|예| G[TakeDamage / HP 감소]
    D --> H[탄창 탄약 감소]
```

## 런타임: 회복 중 피격
```mermaid
flowchart TD
    A[의료품 사용] --> B[HP·수량·회복 중 여부 확인]
    B --> C{사용 가능?}
    C -->|아니오| X[취소]
    C -->|예| D[아이템 수량 감소]
    D --> E[remainingHeal 설정]
    E --> F[프레임별 실제 회복량 계산]
    F --> G[CurrentHp 증가 / remainingHeal 감소]
    G --> H{회복량 남음?}
    H -->|예| F
    H -->|아니오| I[종료]
    J[회복 중 총알 피격] --> K[CurrentHp 감소]
    K -.-> G
```

## Unity 테스트 확인
HP 30 + 회복 70 → 100; 회복 중 피해 10 → 90; 의료품 마지막 1개 사용 정상. 나머지 경계 상황은 시스템별 개발 문서 참조.
