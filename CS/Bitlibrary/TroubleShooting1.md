# 자기참조 Category 리팩토링

- 해결

## 이슈

Thymeleaf로 리팩토링을 진행하던 중, JSP 기반의 레거시 코드에서 부모-자식 관계의 카테고리를 탐색하기 위해 3중 루프를 사용하는 구조를 발견하였다. 이러한 구조는 유지보수성이 낮고, 예외 상황 발생 시 특정 템플릿으로 처리하도록 되어 있어 다른 페이지에서 재사용하기 어려운 한계가 있다고 판단해 리팩토링을 진행하게 되었다.

## 문제 해결 과정
1. 루프 최적화 및 DTO 반환

서버 내에서 불필요한 3중 루프를 제거하고, 필요한 데이터를 한 번의 로직으로 추출하여 DTO로 반환하는 방식으로 개선하였다. 이를 통해 처리 속도가 향상되었으며, 결과 값을 다른 페이지에서도 활용할 수 있는 구조를 만들었다.

2. 조건문 단순화 문제

기존 코드를 리팩토링하면서 카테고리 항목이 많아져 if-else/switch 지옥이 생기는 문제가 발생하였다.
이는 가독성을 나쁘게 만들고 기능 확장에 큰 문제가 생길 것 같아 리팩토링이 필요하다고 생각하여 팀원들과 논의 끝에, 명확한 카테고리 관리를 위해 Enum 클래스를 활용하기로 결정하였다.

3. Enum 클래스 한계점

하지만 생각해보니 Enum을 도입하더라도 조건문 자체를 대체할 수는 없었기 때문에, 조건문이 길어지는 문제를 근본적으로 해결하지 못한다는 한계가 있다고 생각이 되었다. 어떻게 하면 좋을지 이틀정도 생각해보며, 여러가지 best practice들을 보며 최적의 방안을 생각했고, 그 중 최선으로 생각되는 factory startegy 패턴을 이용해 리팩토링을 하였다.

4. Factory 및 Strategy 패턴 도입

3개의 Depth로 구성된 카테고리를 관리하기 위해 Factory와 Strategy 패턴을 적용하였다. Factory 패턴을 사용해 부모 카테고리(Level2)에 따라 적절한 전략을 선택하도록 설계하였다. Strategy 패턴을 통해 자식 카테고리(Level3)의 처리 로직을 각 전략 클래스에 위임함으로써 조건문을 대폭 줄이고, 확장 가능성과 유지보수성을 높였다.


## 의문점

1. strategy pattern에서 실행클래스는 필요한가?

특별한 로직이 없는 실행 클래스라면 단순한 메모리 낭비라고 볼 수도 있다는 생각이 들었다. 하지만 실제 서비스라 확장을 해야되는 상황이 생긴다면, 합리적인 선택이라고 볼 수 있을 것 같다.