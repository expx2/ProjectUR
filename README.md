# Unreal RPG Project

언리얼 엔진 5.7로 개발한 멀티플레이 RPG 게임입니다.

GAS(Gameplay Ability System)를 사용한 스킬 및 효과, 상태 관리 시스템을 가지고 있습니다.

데디케이티드 서버 환경에서의 전투 로직과 동기화, 지연 보정을 구현했습니다.

[PDF](https://github.com/expx2/ProjectUR/blob/main/Docs/ProjectUR.pdf)에도 같은 내용이 작성되어 있습니다.

## 핵심 구현

- [타겟 선택과 자동 이동 후 어빌리티 활성화](#활성화-구조의-개선-과정)

  플레이어는 일부 액티브 어빌리티를 사용할 때 타겟을 선택하고 타겟이 사거리 밖일 경우 자동 이동 후 활성화합니다. 초기에는 어빌리티 내부에서 실행 단계를 나눠 구현했지만 활성화 상태의 모호성을 이유로 플레이어 컨트롤러에서 타겟 선택과 자동 이동을 담당하도록 개선했습니다.

- [데이터 조합을 통한 스킬 구현](#액션과-타임라인을-통한-데이터-기반-시스템)

  공통 액션을 데이터로 조합하여 스킬마다 별도의 로직을 작성하지 않고도 다양한 스킬을 제작할 수 있도록 한 데이터 기반 시스템을 설계했습니다.

- [액티브 어빌리티의 네트워크 및 프레임 지연 보정](#네트워크-및-프레임-지연-보정)

  액티브 어빌리티는 `LocalPredicted` 정책을 사용하여 클라이언트가 서버의 응답을 기다리지 않고 활성화되며, 이로 인한 클라이언트와 서버의 실행 시점에 차이가 발생합니다. 클라이언트의 실행 시간을 기준으로 지연 시간을 계산해 보정하여 클라이언트와 서버의 액션 실행 시점의 차이를 줄이도록 구현했습니다.

- [다양한 공격 범위의 충돌 판정](#충돌-판정과-타격-대상-선정)

  RPG 게임에서의 다양한 형태의 공격 범위 판정을 위해 Collider로 후보를 검출한 뒤 수학적으로 범위를 필터링하는 방식의 충돌 판정을 구현했습니다.

## 플레이 영상

<div>
  <a href="https://youtu.be/U3yrNprBXRI">
    <img src="https://img.youtube.com/vi/U3yrNprBXRI/sddefault.jpg" title="플레이 영상(유튜브)" width="480">
  </a>
  <p><i>이미지를 클릭하면 유튜브로 연결됩니다.</i></p>
</div>

서버와 클라이언트 패키징 후, 서버는 AWS에서 구동하고 두 PC에서 플레이했습니다.

***게임 흐름***

1. 로비 화면 - 캐릭터 선택 후 게임 입장
2. 자코 전투
3. 보스 전투
4. 승리 후 로비로 복귀

## 상세 기술

### 액티브 어빌리티

액티브 어빌리티는 사용자가 직접 사용하는 어빌리티로 &lt;아이온2&gt;의 전투 시스템을 참고하여 구현했다. 레퍼런스 게임과 유사한 조작성과 액션성을 구현하기 위해 GAS의 `LocalPredicted` 실행 정책을 사용해, 클라이언트에서 입력에 즉시 반응하도록 했다.

- ### 어빌리티 선택

  ***기능 요구사항***

  레퍼런스 게임에서는 하나의 스킬 슬롯에 여러 스킬을 등록하고, 순서대로 사용할 수 있다. 해당 기능 후속 스킬이 있는 연계 스킬을 구현하기 위해 하나의 입력에 여러 어빌리티의 등록을 허용하며, 입력 시 등록된 어빌리티를 모두 활성화하지 않고 하나의 어빌리티를 선택하여 활성화를 시도해야 한다.

  > 연계 스킬의 후속 스킬은 별도의 어빌리티로 분리했다. 후속 스킬은 설명 및 아이콘 등의 정보가 다르고, 연계 사이에 다른 스킬을 실행하는 플레이를 지원하기 위해서다.

  ***어빌리티 선택 조건***

  프로젝트의 어빌리티는 `Lyra 프로젝트`를 참고한 어빌리티 태그 관계 매핑(AbilityTagRelationshipMapping)으로 태그 기반의 공통 활성화 조건과 어빌리티 간 차단/취소 관계를 관리한다.

  이 조건은 실제 어빌리티 활성화에는 필요하지만, 선택에도 그대로 사용하면 다른 어빌리티의 활성화처럼 스킬 슬롯의 선택 결과를 변경할 의도가 없는 공통 조건까지 선택에 영향을 준다. 이에 실제 활성화 조건과 별도로 선택 조건 검사 함수를 구현했다.

  선택 시에는 쿨다운이나 연계에 필요한 태그 등 개별 어빌리티의 조건을 검사하고, 태그 관계 매핑에 따른 공통 차단 조건은 제외했다. 또한 레퍼런스를 참고해 자원(마나) 부족 검사도 제외했다. 자원이 부족하더라도 선택된 어빌리티를 유지하고, 실제 사용 시 실패 사유를 안내한다.

  ***우선순위를 통한 선택***

  각 액티브 어빌리티에 우선순위를 설정하고, 선택 조건을 만족하는 어빌리티 중 가장 높은 우선순위를 가진 하나를 선택한다. 조건을 만족하는 어빌리티가 없으면 Fallback으로 우선순위가 가장 높은 어빌리티를 선택한다.

  _관련 코드_
  - [URGameplayAbility_ActiveAbility](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/GameplayAbility/URGameplayAbility_ActiveAbility.h): 액티브 어빌리티 클래스
  - [UURGameplayAbility_ActiveAbility::DoesAbilitySatisfyTagRequirementsToSelect()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/GameplayAbility/URGameplayAbility_ActiveAbility.cpp#L321): 어빌리티 선택 조건 검사 함수
  - [UURAbilitySystemComponent::ResolveCurrentAbilityForInput()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/URAbilitySystemComponent.cpp#L233): 어빌리티 선택 함수

- ### 액티브 어빌리티의 활성화

  ***타겟팅 정책***

  레퍼런스 게임의 액티브 스킬은 타겟을 지정해야 사용할 수 있는 타겟팅 스킬과 타겟 없이 사용할 수 있는 논타겟팅 스킬로 나뉜다. 다만 일부 논타겟팅 스킬은 지정된 타겟이 있으면 해당 타겟을 향해 사용한다. 이러한 동작을 구현하기 위해 타겟팅 정책을 구분하고 활성화 조건을 정리했다.

  | 정책 | 설명 | 타겟 있음 | 타겟 없음 |
  | --- | --- | --- | --- |
  | Targeting | 타겟이 있어야 사용 가능 | 사거리 밖이면 자동 이동 후 활성화 | 활성화 불가능 |
  | NonTargetingWithTarget | 타겟이 없어도 사용 가능하지만, 있을 경우 Targeting처럼 동작 | 사거리 밖이면 자동 이동 후 활성화 | 즉시 활성화 |
  | NonTargeting | 타겟이 없어도 사용 가능 | 즉시 활성화 | 즉시 활성화 |
  | Range | 활성화 후 위치를 지정해 사용 | 즉시 활성화 | 즉시 활성화 |

  > Range 정책은 활성화 후 `UAbilityTask_WaitTargetData`를 이용해 별도의 위치 데이터 지정 및 검증 과정을 거친다.

  ***타겟 선택 방법***

  타겟 후보로는 명시적으로 지정하는 `MainTarget`과 유저의 조준 방향에 따른 `AimTarget`을 사용한다.

  후보를 우선순위대로 검사하면서 피해 수신 가능 여부, 태그, 진영 등 어빌리티의 대상 조건을 확인하고, 처음 조건을 만족한 대상을 선택한다.

  > 단 레퍼런스 게임의 동작을 재현하여 `Targeting` 정책에서는 첫 번째 후보가 유효하지 않으면 후순위 후보를 검사하지 않고 사용을 실패 처리한다.

  #### ***활성화 구조의 개선 과정***

  액티브 어빌리티는 정책에 따라 타겟이 반드시 필요할 수 있으며, 아니더라도 지정된 타겟을 타격 대상 선정의 우선순위에 활용하므로 활성화 시 타겟 정보를 가지고 있어야 했다.

  초기에는 어빌리티를 활성화한 후, 어빌리티 내부에서 실행 단계를 나눠 타겟 선택과 자동 이동을 진행했다. 타겟 관리를 담당하는 클라이언트에서 타겟을 선택하고 `TargetData`를 생성해 서버에 전달하도록 구성했다.

  하지만 이 방식은 실제 스킬 사용 이전인 타겟 선택과 자동 이동 과정에서도 어빌리티는 활성화된 상태가 되면서, 활성화 태그가 부여되고 다른 어빌리티를 차단하거나 취소하는 동작이 실행되는 문제가 있었다.

  이를 해결하기 위해 타겟 선택 및 자동 이동은 `PlayerController`가 담당하고 어빌리티는 실제 스킬 실행 시점에 활성화되도록 책임을 분리했다.

  컨트롤러에서 선택한 타겟을 어빌리티에 전달하기 위해 `TryActivateAbility()` 대신 `FGameplayEventData`에 타겟 액터를 담아 `TriggerAbilityFromGameplayEvent()`를 통해 어빌리티를 활성화하도록 구현했다.

  이를 통해 어빌리티의 활성화 구간을 실제 스킬 사용 시점에 맞출 수 있었다.

  _관련 코드_
  - [AURGamePlayerController::TryActivateAbility()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Player/URGamePlayerController_Ability.cpp#L17): 플레이어 컨트롤러에서 타겟 선택과 자동 이동 요청/어빌리티 트리거 함수

- ### 자동 이동과 서버 검증

  플레이어는 일부 타겟팅 정책에서 타겟이 사거리 밖에 있을 때 어빌리티와 타겟 정보를 저장하고 `UPathFollowingComponent`로 자동 이동을 시작해 사용 가능한 거리까지 이동한 후 어빌리티를 활성화한다.

  이동 중에는 타겟의 유효성과 어빌리티 사용 조건을 검사하며, 사용 가능한 거리에 도달하면 자동 이동을 종료하고 저장된 어빌리티를 활성화한다.

  ***클라이언트에서 자동 이동 수행***

  액티브 어빌리티는 빠른 조작 반응성을 위해 `LocalPredicted`를 사용한다.

  사거리 밖의 타겟에게 스킬을 사용할 때 서버에서 자동 이동을 시작하도록 하면, 이동 시작까지 네트워크 지연이 발생하여 어빌리티의 실행 정책을 설정한 의미가 퇴색된다고 판단했다.

  이에 `bAllowClientSideNavigation=True` 설정하여 클라이언트가 즉시 자동 이동을 수행하도록 했다.

  ***서버 검증***

  자동 이동을 클라이언트에서 시작하므로 서버에서도 요청의 유효성을 확인하도록 구성했다.

  이동 시작 시 서버 RPC로 어빌리티와 타겟 정보를 전달하고, 서버에서 해당 어빌리티의 사용 가능 여부와 타겟 유효성을 검증한다. 이후 이동 중에 클라이언트에서는 유효성 검사를 반복한다.

  서버에서는 자동 이동 요청을 처음 수신할 때 유효성을 검증하고, 이동 중에는 검사를 반복하지 않는다. 어빌리티 활성화 시점에 사용 조건과 타겟 유효성을 다시 검증하므로, 이동 중 반복 검사를 생략해 서버의 검사 비용을 줄일 수 있다고 판단했다.

  이는 유효하지 않은 어빌리티 활성화는 막더라도 유효하지 않은 대상을 향한 접근은 허용할 수 있다. 최적화와 신뢰성의 트레이드오프로, 필요할 경우 서버에서 일정 간격으로 재검증하는 방식으로 신뢰성을 조금 더 확보할 수 있을 것이라 생각한다.

  ***서버 검증 실패에 따른 이동 취소***

  서버에서 자동 이동 요청의 검증에 실패하면 클라이언트 RPC를 통해 클라이언트의 자동 이동을 취소한다.

  이때 요청별 RequestID를 통한 동일 요청에 대한 취소를 검토했다. 하지만 클라이언트가 서버의 실패 응답을 받기 전 이동 요청을 반복 갱신하면, 이전 요청의 실패 응답이 무시되어 유효하지 않은 동작이 계속 유지될 가능성이 있다고 판단했다.

  이에 RequestID 대신 동일 어빌리티 조건만으로 취소를 하도록 했다. 이 방식은 이전 요청의 실패가 이후 새로 시작한 정상 요청을 취소할 수 있다는 문제가 있다. 하지만 정상 요청이 불필요하게 취소되는 문제보다 반복적인 요청 갱신으로 유효하지 않은 동작이 유지되는 상황의 위험이 더 크다고 판단해 이 방식을 선택했다.

  _관련 코드_
  - [AURGamePlayerController::SetPendingAbilityAndMoveToTarget()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Player/URGamePlayerController_Ability.cpp#L237): 자동 이동 시작 함수
  - [AURGamePlayerController::ServerSetPendingAbility()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Player/URGamePlayerController_Ability.cpp#L296): 서버에서의 자동 이동 검증 함수
  - [AURGamePlayerController::ClientKillPendingAbility()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Player/URGamePlayerController_Ability.cpp#L345): 서버에서의 검증 실패에 따른 이동 취소 함수

- ### 액션과 타임라인을 통한 데이터 기반 시스템

  MMORPG 게임에서 스킬의 동작은 상대적으로 정형화되어 있다고 생각한다. 또한 언리얼 엔진은 Blueprint라는 강력한 도구를 제공하므로, 데이터를 조합해 새로운 스킬을 빠르게 제작하고 테스트할 수 있는 데이터 기반 시스템이 장르와 엔진 특성에 잘 맞는다고 생각했다.

  이에 액션 데이터를 조합하여 사용하는 어빌리티 시스템을 설계했다.

  어빌리티는 활성화와 종료를, 개별 액션은 구체적인 동작을, 액션 타임라인은 액션의 실행 흐름을 관리하도록 책임을 분리했다.

  <img src="https://github.com/user-attachments/assets/4eff0fa5-58a2-44be-a840-ffe0fd34ef90" width="800">

  *액션 조합을 통한 액티브 어빌리티 구성 예시*

  이를 통해 개발 과정에서 플레이어 캐릭터를 추가하며 액티브 어빌리티 5종을 별도의 C++ 추가 작업 없이 기존 액션 데이터의 조합만으로 구현했다.

  ***액션***

  액션은 어빌리티의 개별 동작과 효과를 구성하는 데이터다.

  각 액션은 `FURAbilityActionBase`를 상속받는 구조체로 구현하고, `TInstancedStruct` 배열을 통해 액티브 어빌리티의 BP에서 조합할 수 있도록 했다.

  이를 통해 동일한 실행 구조 안에서 액션의 종류, 파라미터를 변경하는 것으로 서로 다른 스킬을 구성할 수 있다.

  | 액션 | 설명 |
  | --- | --- |
  | MeleeTrace | 사용자 위치를 기준으로 Trace를 진행해 타격한다 |
  | FindTargetAndTrace | 사용자 기준 일정 범위 내에 대상을 찾고, 찾은 대상 위치에서 Trace를 진행해 타격한다 |
  | MoveTo | 사용 캐릭터를 지정한 위치로 이동시킨다 |
  | PlayMontageAndWait | AnimMontage를 실행한다 |

  *주요 액션 예시*

  ***액션 타임라인***

  여러 액션의 실행 시점과 진행 상태를 어빌리티에서 직접 관리하면 어빌리티의 생명주기와 개별 액션 관리 책임이 하나의 클래스에 집중된다. 이를 분리하기 위해 액션 타임라인 어빌리티 태스크를 구현하여 액션 실행, 진행 상태 및 결과 관리를 담당하도록 했다.

  타임라인은 시작 시간을 기준으로 경과 시간을 계산하고, 각 액션에 설정된 시작 시점이 되면 해당 액션을 실행한다.

  액션은 시작 결과를 반환하며, 비동기 액션일 경우 어빌리티 태스크를 함께 반환한다.

  타임라인은 반환된 결과와 태스크의 완료/실패 이벤트를 이용해 각 액션의 진행 상태를 Pending, Running, Completed 등으로 관리한다.

  또한, 액션 실행 실패 시 타임라인 전체를 중단하거나, 나머지 액션을 계속 진행하도록 하는 실패 정책을 두어 중요도에 따라 처리할 수 있도록 했다.

  _관련 코드_
  - [URAbilityActionBase](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Combat/AbilityAction/URAbilityActionBase.h): 액션 베이스 구조체
  - [AbilityActionTask](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/AbilityTask/AbilityActionTask/AbilityActionTask.h): 비동기 액션의 어빌리티 태스크 베이스 클래스
  - [UAbilityTask_AbilityActionTimeline](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/AbilityTask/AbilityTask_AbilityActionTimeline.h): 액션 타임라인 어빌리티 태스크 클래스

- ### 네트워크 및 프레임 지연 보정

  <img src="https://github.com/user-attachments/assets/31364b43-73a6-4d98-9944-d13ab527ad79" width="800">

  *PktLag=500 환경 좌: 보정 없음 / 우: 보정 적용*

  ***배경***

  액티브 어빌리티는 `LocalPredicted` 정책을 사용하므로 클라이언트가 서버의 응답을 기다리지 않고 액션을 실행할 수 있다.

  서버에서는 네트워크 지연으로 인해 클라이언트보다 늦게 액션을 실행하게 되며, 클라이언트와 서버의 애니메이션 진행 상태 및 타격 판정 시점에 차이가 발생한다.

  이러한 차이를 줄이기 위해 클라이언트의 액션 실행 시점을 기준으로 서버의 액션 타임라인을 보정하는 방식을 구현했다.

  ***클라이언트 실행 시간 전달과 서버 보정***

  클라이언트는 어빌리티 활성화 후 사용 조건을 검증하고 액션 타임라인을 시작한다. 이때 실행 시점의 서버 시간(`GetServerWorldTimeSeconds()`)을 `TargetData`에 담아 서버에 전달한다.

  서버는 해당 데이터를 수신한 후 현재 서버 시간과 비교하여 보정할 지연 시간을 구한다. 다만 클라이언트가 보낸 시간을 그대로 신뢰하지 않고 평균 RTT와 비교하며, 최대 보정 시간을 제한한다.

  계산한 보정 시간을 타임라인의 시작 시간에 반영하여, 서버에서 타임라인이 늦게 시작되더라도 클라이언트와의 경과 시간 차이를 줄이고, 액션 실행 시점을 최대한 일치시키도록 했다.

  ***액션별 지연 보정***

  타임라인은 각 액션의 시작 시간에 맞춰 액션을 실행하지만, 네트워크 지연과 프레임 지연으로 실제 실행 시점이 의도한 시간보다 늦어질 수 있다.

  이를 보완하기 위해 타임라인에서 액션을 시작할 때 경과 시간을 액션에 전달하고, 각 액션에서 고유의 방식으로 보정하도록 했다.
  
  | 액션 | 설명 |
  | --- | --- |
  | MeleeTrace | 밀린 타격 회차를 한 프레임에 처리한다 |
  | PlayMontageAndWait | 정책에 따라 AnimMontage의 실행 속도를 조절하거나, 시작 시간을 설정해 중간 부분부터 실행한다 |

  *보정 예시*

  ***설계상 한계***

  현재 방식은 클라이언트가 전달한 실행 시간을 기준으로 서버의 액션 타임라인을 보정하지만, 과거 시점의 캐릭터 위치나 상태를 복원하는 방식(Server-side Rewind)은 아니다.

  따라서 다단히트 공격의 경우 지연으로 누적된 여러 타격 회차가 한 프레임에 처리될 수 있으며, 의도한 결과와 차이가 발생할 수 있다.

  또한 서버의 판정 결과가 클라이언트에 전달되기까지 추가적인 지연이 발생하므로, 타격과 피드백의 시점이 일치하지 않을 수 있다.

  _관련 코드_
  - [URGameplayAbility_ActiveAbility::ActionAbility()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/GameplayAbility/URGameplayAbility_ActiveAbility.cpp#L951): 클라이어트에서의 시간 전송과 서버에서의 수신 대기 함수
  - [URGameplayAbility_ActiveAbility::ValidateActionTimeTargetData()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/GameplayAbility/URGameplayAbility_ActiveAbility.cpp#L994): 서버에서 시간 수신 후 지연 시간 설정 및 액션 타임라인 실행 함수


### 전투

멀티플레이 환경에서 전투 결과의 신뢰성을 위해 실제 충돌 판정과 피해 효과(GameplayEffect) 적용은 권한이 있는 서버에서만 수행한다.

캐릭터뿐 아니라 프랍 등 ASC가 없는 액터도 타격할 수 있도록 `IURDamageableInterface`를 전투 대상으로 하며, ASC가 있는 대상은 GE를 전달하고, 없는 대상은 인터페이스의 `NativeTakeDamage()` 함수를 구현해 피해를 처리한다.

- ### 충돌 판정과 타격 대상 선정

  <a id="다양한-공격-범위-지원"></a>

  ***다양한 공격 범위 지원***

  다양한 공격 범위를 지원하기 위해 원, 부채꼴, 도넛, 도넛 부채꼴, 사각형 형태의 충돌 판정을 구현했다.

  `OverlapMultiByChannel()`로 1차로 후보를 검출한 뒤, 각 모양에 필요한 추가 조건을 검사해 대상을 검출한다. 이때 Overlap을 통한 검출과 일관성이 있도록 대상의 원점뿐 아니라 충돌 크기를 고려해 범위에 일부라도 걸치면 포함하도록 했다.

  충돌 모양은 `FURColliderShape`를 상속한 구조체로 정의하고, `TInstancedStruct`를 통해 액션 데이터에서 설정하도록 했다.

  ***Cylinder 형태의 충돌 판정***

  `OverlapMultiByChannel()`에서 사각형을 제외한 모양들은 1차적으로 반지름 내에 있는 캐릭터들을 후보로 얻는다.

  여기서 Sphere를 사용하면 중심과 수평 거리가 멀어질수록 허용 높이가 좁아져, 같은 높이에 있더라도 검출 결과에 차이가 생긴다. 이는 정상 동작일 수도 있지만 프로젝트에서 의도한 형태는 아니었다.

  이를 해결하기 위해 허용 높이를 따로 지정하도록 하고, Capsule로 Overlap을 진행한다. 검출된 후보들을 높이 검사를 통해 위아래 반구를 제외한 실린더 형태의 범위로 제한했다.

  이때도 마찬가지로 대상의 충돌체를 고려해 일부라도 걸치면 포함시켜 일관성 있게 처리했다.

  <img src="https://github.com/user-attachments/assets/4345af5f-2f58-4ae2-9493-c11ec5ddd9ba" width="800">

  *Sphere에서의 문제와 Cylinder 형태의 판정 설명 이미지*

  ***타격 대상 수 제한과 우선순위***

  레퍼런스 게임과 같이 범위 공격에 최대 타격 대상 수를 설정할 수 있도록 구현했다. 단순히 검출 순서, 거리 순서대로 대상을 선택하면 사용자가 지정한 타겟이 제외되거나, 다단히트 공격에서 매 회차마다 타격 대상이 달라질 수 있다. 이를 방지하기 위해 다음 순서대로 타격 대상을 선정한다.

  | 우선순위 | 대상 | 의도 |
  | --- | --- | --- |
  | 1 | 어빌리티 활성화 시 지정한 타겟 | 사용자가 지정한 대상을 우선 타격한다 |
  | 2 | 이전 타격 회차에서 처리한 대상 | 다단히트 중 타격 대상이 매번 바뀌지 않도록 한다 |
  | 3 | 그 외 범위 내 대상 | 남은 타격 대상 수를 채운다 |

  > 우선순위가 높더라도 현재 타격에 검출된 대상만 선정한다.

  _관련 코드_
  - [URColliderShape](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Collisions/URColliderShape.h): 공격 범위 정의
  - [UURCollisionBlueprintLibrary::GetDamageableActors()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/BlueprintLibrary/URCollisionBlueprintLibrary.cpp#L152): Shape을 통한 공격 대상 검출 함수
  - [UURCombatBlueprintLibrary::ResolveTargets()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/BlueprintLibrary/URCombatBlueprintLibrary.cpp#L52): 우선순위를 통한 타격 대상 선정 함수

- ### GameplayEffect 적용과 피격 처리

  ***SetByCaller를 이용한 GE 적용***

  하나의 효과에 수치별로 별도의 GE를 생성하지 않고 사용할 수 있도록 GE 클래스와 `SetByCaller` 태그 및 값을 관리하는 구조체를 만들었다. 이를 하나의 액션에서 대미지와 함께 여러 부가 효과를 부여할 수 있도록 배열로 관리했다.

  타격하여 GE를 적용할 배열을 순회하며 설정한 `SetByCaller` 태그에 값을 설정하도록 했다.

  ***GameplayEffectContext 확장***

  피격 Gameplay Cue 태그와 대미지 표시에서 치명타 여부를 구분하기 위해 피격자에게 추가적인 값을 전달해야 했다. 이를 위해 `FGameplayEffectContext`를 상속해 추가 정보를 담는 `FURHitGameplayEffectContext`를 만들어 사용했다.

  또한 다수의 대상에게 공격을 할 때 치명타 여부 등의 정보가 대상마다 달라질 수 있도록 타격 처리 시 Context를 타격 대상별로 복제하도록 했다.

  ***피해 계산***

  공격자와 피격자의 스탯(Attribute)을 기반으로 대미지 계산을 하기 위해 `ExecutionCalculation`을 이용했다. 공격의 대미지 계수는 공격 GE에 `SetByCaller`로 전달하며, 피해량의 적용은 `Lyra 프로젝트`를 참고해 피해량을 `HealthSet`의 메타 어트리뷰트 `DamageTaken`에 저장하고 `PostGameplayEffectExecute()`에서 실제 체력을 변경하도록 했다.

  _관련 코드_
  - [UURCombatBlueprintLibrary::Hit()](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/BlueprintLibrary/URCombatBlueprintLibrary.cpp#L167): FURHitEffect 이용한 GE 부여 등 타격 처리
  - [URDamageExecutionCalculation](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/AbilitySystem/GameplayEffect/Calculation/URDamageExecutionCalculation.h): 대미지 계산 클래스

- ### 적 AI와 어그로 관리

  ***Behavior Tree***

  <img src="https://github.com/user-attachments/assets/5311561e-8646-4f55-b329-1fdeb05f1888" width="800">

  *Behavior Tree 구성 이미지*

  순찰, 탐지(선공), 추적, 전투, 귀환 Task를 BP로 구현했다.

  전투에서 기본적으로 액티브 어빌리티를 우선순위에 따라 사용하도록 하고, 보스처럼 일정 공격 패턴이 있는 적을 위해 어빌리티의 사용 순서를 명시적으로 지정할 수 있는 `UAIAbilityOrderSet`을 구현했다.

  ***어그로 관리***

  적의 공격 대상을 관리하기 위해 피격 이벤트를 받아 공격자별 어그로를 누적하는 컴포넌트를 구현했다.

  기본적으로 피해량을 기반으로 누적하는데, 탱커 역할의 캐릭터를 더 공격할 수 있도록 캐릭터 스탯으로 `AggroMultiplier`를 두어 피해량에 곱해 어그로를 누적했다.

  _관련 코드_
  - [URAggroComponent](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Character/URAggroComponent.h): 어그로 관리 컴포넌트 클래스

### 아키텍처

- ### 플레이어 캐릭터 선택과 생성

  로비에서 선택한 캐릭터 정보를 게임 레벨로 전달하고, 서버에서 해당 캐릭터를 생성하도록 구현했다.

  <img src="https://github.com/user-attachments/assets/58f0c2b0-ac0e-41ac-a67a-78df919f7707" width="800">

  *시퀀스 다이어그램*

  선택한 캐릭터 ID는 레벨 이동 후에도 유지할 수 있도록 클라이언트의 GameInstance에 저장한다. 게임 서버의 GameMode는 `OnPostLogin()`에서 PlayerController를 통해 클라이언트에 캐릭터 ID를 요청한다. 클라이언트는 GameInstance에 저장된 ID를 서버 RPC로 전달하며, 서버는 유효성을 검사한 후 PlayerState에 저장한다. 이후 GameMode가 `RestartPlayer()`를 호출하여 PlayerState에 저장된 ID에 대응하는 캐릭터를 생성한다.

- ### PlayerState에서 플레이어의 AbilitySystemComponent 소유

  플레이어 캐릭터가 사망하여 액터가 사라져도 사망 상태를 유지할 수 있어야 한다. 또한 플레이어 캐릭터가 리스폰되더라도 쿨다운 등의 상태를 유지할 수 있어야 한다.

  이를 위해 ASC를 PlayerState에서 소유하도록 구성했다. 적 캐릭터는 이 기능이 필요 없으므로 캐릭터가 직접 ASC를 소유한다. 소유 주체는 다르더라도 공통 기능을 함께 사용할 수 있도록 플레이어 캐릭터와 적 캐릭터는 `URASCCharacter`를 상속받는다.

  ***해당 구조로 생기는 공격자 문제***

  GE 적용 시 Context는 공격 주체인 `Instigator`와 실제 효과를 발생시킨 객체인 `EffectCauser`를 제공한다. 여기서 공격자 캐릭터가 직접 공격하지 않고 투사체같이 별도의 액터를 통해 공격할 경우 `EffectCauser`는 액터가 된다.

  공격자 캐릭터를 구하기 위해 `Instigator`를 사용할 경우 플레이어는 `PlayerState`, 적은 `Character`가 되어 사용할 때 분기 처리가 필요하다.

  적의 어그로 관리 컴포넌트에선 이 문제로 인해 `Instigator`가 ASC를 가지고 있을 경우, `AvatarActor`를 가져와 공격자로 설정한다.

  _관련 코드_
  - [URGamePlayerState](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Player/URGamePlayerState.h): 플레이어의 ASC를 관리하는 플레이어 스테이트 클래스
  - [URASCCharacter](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/Character/URASCCharacter.h): ASC를 가진 캐릭터의 공통 기능을 담당하는 캐릭터 클래스

- ### LocalPlayerSubsystem과 AHUD를 통한 UI 관리

  LocalPlayerSubsystem을 상속한 `UURUISubsystem`을 구현해 UI 매니저의 역할로 사용한다. RootWidget과 UI 레이어를 관리하며, 토스트 메시지, 팝업 생성/제거 등 공통 UI 기능을 담당한다. 이는 어느 레벨에서 UI를 사용하든 일관된 처리를 보장하고 레이어의 관리를 수월하게 하기 위함이다.

  `UURUISubsystem`의 책임이 거대해지는 것을 막기 위해 AHUD를 해당 레벨의 전용 UI 매니저 역할로 사용한다. 프로젝트에서는 게임 레벨에서 `AURGameHUD`가 인게임 HUD, 타겟 인디케이터, 대미지 표시 위젯 등의 생성과 관리를 담당한다.

  _관련 코드_
  - [URUISubsystem](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/UI/URUISubsystem.h): UI 관리 로컬 플레이어 서브 시스템 클래스
  - [URGameHUD](https://github.com/expx2/ProjectUR/blob/main/Source/URGame/UI/HUD/URGameHUD.h): 게임 레벨 전용 UI 관리 클래스

## 포스트모템

언리얼 엔진의 GAS와 네트워크 기능을 활용하여, 클라이언트의 빠른 조작 반응성과 서버 권위의 전투를 함께 지원하는 멀티플레이 RPG를 구현하는 것을 목표로 프로젝트를 진행했습니다.

클라이언트 예측과 네트워크 지연 보정을 구현하는 과정에서, 반응성과 신뢰성 사이에서 균형을 잡고 서버와 클라이언트의 책임을 어떻게 구분하는지 고민하며 멀티플레이 구조 설계 경험을 쌓았습니다.

***설계상 타협***

전투 판정은 권위 있는 서버에서만 수행하지만, 빠른 조작 반응성을 위해 자동 이동과 액션 실행을 클라이언트에서 먼저 처리하고, 서버에서 검증 및 보정하는 방식을 택했습니다.

이 과정에서 어디까지 클라이언트를 신뢰할 수 있는지 그 범위를 결정하는 것이 어려웠습니다.

현재는 PvE 중심의 프로젝트이므로 반응성과 구현 복잡도를 고려해 클라이언트를 비교적 폭넓게 신뢰하는 방식으로 타협했지만, PvP 같은 경쟁 요소가 있는 환경에서는 서버 검증을 강화하고 클라이언트에 대한 신뢰를 더욱 엄격히 제한해야 한다고 생각합니다.
