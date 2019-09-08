---
layout: post
title: "mobx-react v6 마이그레이션하기"
date: 2019-09-08 19:38:15 +0900
categories: develop
---

`mobx-react`는 `observable`한 상태들, `computed`로 계산된 결과를 메모이즈 해둘 수 있는 기능들 등 `redux`와는 다른 매력을 가지고 있는 라이브러리이다. 하지만 `react`가 16.8버전에 도입된 `hooks`와 호환이 안되는 문제가 발생하게 되었는데, 이 글에서는 그 문제가 무엇인지, `mobx-react`는 어떻게 그 방법을 해결했는지를 작성해보고 실제 코드에서 어떻게 마이그레이션 할지를 고민해본 내용을 작성해 볼 생각이다.

# `mobx-react` v5의 문제점

이 버전이 가지고 있는 가장 큰 문제는 컴포넌트들의 상태 공유를 하기 위하여 `legacy context`를 사용한다는 점이다. 레거시 컨텍스트는 이후 `react`에서 제거될 기능이기 때문에 계속해서 사용한다면 리액트 버전을 최신으로 유지하는 데에 문제가 될 여지가 있다.

두번째 문제점은 `observer`함수 자체가 클래스를 위해서 만들어진 함수이기 때문에 현재 `hook`과는 호환되지 않는 문제점이다.

`hook`을 사용하고싶어서 함수 컴포넌트로 코드를 작성하고 난 다음, 코드를 실행시키면 클래스 컴포넌트에서 `hook`을 사용할 수 없다는 에러 메시지를 본 것은 필자 뿐만 아니라 많은 사람들이 겪었을 문제라고 생각한다. `mobx`에서는 아래의 코드 처럼 함수일 경우에는 클래스로 래핑하여 재귀적으로 호출함으로서 해결하였다.

```js
export function observer(arg1, arg2) {
    // ... 기타 내용들
    // Stateless function component:
    // If it is function but doesn't seem to be a react class constructor,
    // wrap it to a react class automatically
    if (
        typeof componentClass === "function" &&
        (!componentClass.prototype || !componentClass.prototype.render) &&
        !componentClass.isReactClass &&
        !Component.isPrototypeOf(componentClass)
    ) {
        const observerComponent = observer(
            class extends Component {
                static displayName = componentClass.displayName || componentClass.name
                static contextTypes = componentClass.contextTypes
                static propTypes = componentClass.propTypes
                static defaultProps = componentClass.defaultProps
                render() {
                    return componentClass.call(this, this.props, this.context)
                }
            }
        )
        hoistStatics(observerComponent, componentClass)
        return observerComponent
    }
    // 기타 내용들
```

당연히 리액트에 `hooks`이 들어온 지금 `Stateless Function Component`는 더이상 맞지 않는 주석이다.

이런 문제점은 `mobx`커뮤니티에서 당연히 문제가 되었고 이런 구조들을 개선하기 위해서 라이브러리 전체를 다시 쓰게 되는데, 그래서 나온 것이 `mobx-react-lite`이다.

# `mobx-react-lite`

이 라이브러리는 완전히 다시 쓰여진 모브엑스의 새 버전이다. 그렇다면 `mobx-react`는 어떻게 되는가? 다음 버전인 v6으로 업데이트 되었지만, 내부는 기존 라이브러리의 호환성을 유지하기 위해서 v5의 함수들을 이 라이브러리로 구현해 둔 것 뿐이다.

모브엑스 리액트 라이트는 `hooks`의 도입과 `legacy context`의 도입 이 두가지를 가장 염두에 둔 것으로 보인다. 이 새 라이브러리가 문제를 어떻게 해결했는지 살펴보자.

## `legacy context`

`redux`의 철학은 `single source of truth`이다. 따라서 라이브러리에서 관리되는 모든 상태들이 하나의 스토어에 들어있어야 한다. 그렇기 때문에 라이브러리에서 관리되는 프로바이더가 필연적이였을 것이라고 생각한다. 하지만 `mobx`는 그렇지 않다. 관심사에 따라서 여러개의 스토어를 만들고 필요에 따라서 컴포넌트위에 프로바이더를 만들어서 사용할 수 있다.

새 버전에서는 레거시 컨텍스트를 들어내면서 이 철학에 맞는 가장 간단한 방법을 사용했다.

> 컨텍스트의 컨셉이 너무나도 간단해서 라이브러리에 추가로 필요한 값들이 없었습니다. 만약 모든 앱을 hook으로 관리하도록 마이그레이션 하는데 성공한다면, Mobx에서 제공하는 Provier가 필요하지 않고 심지어 더 많은 제어가 가능해집니다. [(원문)](https://mobx-react.js.org/recipes-migration)

필요하면 직접 만들어 쓰라는 말이다. 실제로 그 방법은 너무나도 간단하다.

```js
const UserContext = createContext(null);

const UserProvider = ({ children }) => {
  const [userStore] = useState(() => new UserStore()); // class로 개발된 mobx 스토어든, mst로 작성된 스토어든, 여기서 초기화를 시켜준다.

  const userStore = useRef(new UserStore()).current; // 스토어 생성 비용이 크지 않다면 이렇게 사용해도 된다.

  // 공식 문서에서는 useMemo는 리액트에서 랜덤하게 값을 버려버리기 때문에 사용하지 말라고 권장되어 있다
  return (
    <UserContext.Provider value={userStore}>{children}</UserContext.Provider>
  );
};

// 값이 필요하다면 그냥 useContext를 사용하면 된다.
const UserProfile = () => {
  const userStore = useContext(UserContext);

  // useObserver에 대해서는 아래에 설명이 되어있다.
  return useObserver(() => (
    <div>
      {userStore.name} - {userStore.age}
    </div>
  ));
};
```

사실 지금 `mobx`가 훅을 어떻게 사용하는지 들여다본다면, `Provider`의 존재조차 필요가 없다. 자세한 내용은 `hook`을 설명하면서 상세히 들여다 보겠다.
