---
layout: post
title: "mobx-react v6 마이그레이션하기"
date: 2019-09-09 12:02:48 +0900
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
        static displayName = componentClass.displayName || componentClass.name;
        static contextTypes = componentClass.contextTypes;
        static propTypes = componentClass.propTypes;
        static defaultProps = componentClass.defaultProps;
        render() {
          return componentClass.call(this, this.props, this.context);
        }
      }
    );
    hoistStatics(observerComponent, componentClass);
    return observerComponent;
  }
  // 기타 내용들
}
```

리액트에 `hooks`이 들어온 지금 `Stateless Function Component`는 더이상 맞지 않는 주석이다. 함수 컴포넌트를 클래스의 랜더 함수에서 실행시켜 리턴하니 경고가 나오고 클래스의 안쪽이므로 자연스럽게 `hook`을 사용할 수 없게 된다.

이런 문제점은 `mobx`커뮤니티에서 문제가 되었고 이런 구조들을 개선하기 위해서 라이브러리 전체를 다시 쓰게 되는데, 그래서 나온 것이 `mobx-react-lite`이다.

# `mobx-react-lite`

이 라이브러리는 완전히 다시 쓰여진 모브엑스의 새 버전이다. 그렇다면 `mobx-react`는 어떻게 되는가? 다음 버전인 v6으로 업데이트 되었지만, 내부는 기존 라이브러리의 호환성을 유지하기 위해서 v5의 함수들을 이 라이브러리로 구현해 둔 것으로 앞으로 주력 개선사항은 `mobx-react-lite`에서 개발될 것으로 생각이 된다.

`mobx-react-lite`는 `hooks`의 도입과 `legacy context`의 제거 이 두가지를 가장 염두에 둔 것으로 보인다. 이 새 라이브러리가 문제를 어떻게 해결했는지 살펴보자.

## `legacy context`

`redux`의 철학은 `single source of truth`이다. 따라서 라이브러리에서 관리되는 모든 상태들이 하나의 스토어에 들어있어야 한다. 그렇기 때문에 라이브러리에서 관리되는 프로바이더가 필연적이였을 것이라고 생각한다. 하지만 `mobx`는 그렇지 않다. 관심사에 따라서 여러개의 스토어를 만들고 필요에 따라서 컴포넌트위에 프로바이더를 만들어서 사용할 수 있다.

새 버전에서는 레거시 컨텍스트를 들어내면서 이 철학에 맞는 가장 간단한 방법을 사용했다.

> 컨텍스트의 컨셉이 너무나도 간단해서 라이브러리에 추가로 필요한 값들이 없었습니다. 만약 모든 앱을 hook으로 관리하도록 마이그레이션 하는데 성공한다면, Mobx에서 제공하는 Provier가 필요하지 않고 심지어 더 많은 제어가 가능해집니다. [(원문)](https://mobx-react.js.org/recipes-migration)

필요하면 직접 만들어 쓰라는 말이다. 실제로 그 방법은 너무나도 간단하다.

```js
const UserContext = createContext(null);

const UserProvider = ({ children }) => {
  // class로 개발된 mobx 스토어든, mst로 작성된 스토어든, 여기서 초기화를 시켜준다.
  const [userStore] = useState(() => new UserStore());

  // 스토어 생성 비용이 크지 않다면 이렇게 사용해도 된다.
  const userStore = useRef(new UserStore()).current;

  // 공식 문서에서는 useMemo는 리액트에서 랜덤하게 값을 버려버리기 때문에 사용하지 말라고 권장하고 있다.
  // const userStore = useMemo(() => new UserStore(), []);

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

이렇게 함으로서 `mobx`는 `inject`를 사용할 필요가 없어졌다. 간결하게 작성할수 있으며 context를 이용하기 때문에 타입스크립트의 타입 추론도 더 잘 받을수 있게 되었다.

사실 `mobx-react-lite`가 훅을 어떻게 사용하는지 들여다본다면 `Provider`의 존재조차 필요가 없다. 자세한 내용은 `hook`을 설명하면서 상세히 들여다 보겠다.

## `hooks`

`mobx-react`에서 많은 훅이 추가가 되었지만 가장 눈여겨 볼 만한 훅은 `useObserver`이다. 사실상 이 훅이 `mobx`와 리액트를 이어주는 가장 핵심적인 `hook`이기 때문이다.

새로 추가된 `useLocalStore`같은 함수들은 공식 문서를 보면 사용법이 잘 나와있기 때문에 따로 확인하고 넘어가지 않겠다.

핵심적인 내용을 위해서 몇 가지 내용을 제거한 코드를 보자. `mobx`의 `reaction`을 알고 있다면 꽤 간단하게 짜여져 있다.

우선 `mobx`의 `Raction`에 대해서 짚어보고 넘어가자. 공식 문서에 있는 `reaction`과는 살짝 다르다. `reaction`은 외부로 노출된 함수이고 `Reaction`은 실제로 기능을 하는 클래스라고 생각하면 될 것 같다. 아래는 `Reaction`코드의 [주석](https://github.com/mobxjs/mobx/blob/master/src/core/reaction.ts#L35)에 쓰여져 있는 동작 과정이다.

```
 * The state machine of a Reaction is as follows:
 *
 * 1) 인스턴스가 생성된 뒤에는 reaction은 반드시 runReaction을 호출하거나 스케줄링함으로서 시작되어야 합니다.
 * 2) `onInvalidate`는 `this.track(someFunction)`를 호출하는 함수여야 합니다.
 * 3) `someFunction`에서 접근되는 모든 옵저버블은 이 reaction에 의해서 관찰되어집니다.
 * 4) Reaction의 someFunction의 디펜던시가 변경되게 되면 이 다음 실행때 리스케줄됩니다. 디펜던시가 변경되었을때 `isScheduled`가 ture로 변경됩니다.
 * 5) `onInvalidate`가 실행되고, 1번으로 되돌아갑니다.
```

```ts
// 원본 코드
// https://github.com/mobxjs/mobx-react-lite/blob/master/src/useObserver.ts
import { Reaction } from "mobx";
import { useDebugValue, useRef, useState } from "react";

import { printDebugValue } from "./printDebugValue";
import { isUsingStaticRendering } from "./staticRendering";

export function useObserver<T>(
  fn: () => T,
  baseComponentName: string = "observed"
): T {
  // 강제로 업데이트하는 setState를 만들어준다.
  // 업데이트의 주체가 리액트가 아닌 상태관리 컴포넌트에서 흔히 사용되는 패턴이다.
  const [, setTick] = useState(0);

  const forceUpdate = useCallback(() => {
    setTick(tick => tick + 1);
  }, []);

  // reaction을 만들어 준 다음, 초기화를 시켜준다.
  // 이때, reaction이 일어날때마다 강제로 리랜더를 시켜준다.
  const reaction = useRef<Reaction | null>(null);
  if (!reaction.current) {
    reaction.current = new Reaction(`observer(${baseComponentName})`, () => {
      forceUpdate(); // 리랜더가 되므로 아래 라인의 track을 다시 호출한다. `onInvalidate`의 조건을 충족시킴
    });
  }

  // cleanup 함수를 만들어주고 등록해준다.
  const dispose = () => {
    if (reaction.current && !reaction.current.isDisposed) {
      reaction.current.dispose();
    }
  };

  useEffect(() => dispose, []);

  // reaction의 안쪽에 인자로 받은 함수를 실행시켜주고, 스케줄링한다.
  // 따라서, 인자로 넘겨주는 함수의 안쪽에 observable이 존재하면 이 값이 변경될때마다 리랜더가 되는 것이다.
  // 그렇기 때문에 useObserver로 넘겨주는 값들이 비구조화 할당을 하여 주소가 아닌 값이 된다면 값을 추적하지 못하는 것이다.
  // render the original component, but have the
  // reaction track the observables, so that rendering
  // can be invalidated (see above) once a dependency changes
  let rendering!: T;
  let exception;
  reaction.current.track(() => {
    try {
      rendering = fn();
    } catch (e) {
      exception = e;
    }
  });
  if (exception) {
    dispose();
    throw exception; // re-throw any exceptions catched during rendering
  }
  return rendering;
}
```

눈치가 빠른 사람들은 왜 `Provider`가 필요없는지 알 수 있을 것이다. 업데이트 로직이 훅 안에 전부 존재함으로서 더이상 context를 통해서 상태를 전달받을 필요가 없어졌다. 단지 `useObserver`안에 `observable`한 값만 존재하면 된다. 이 `hook`이 알아서 변경을 추적해서 리랜더를 한다. 이렇게 된다면 `store`를 싱글톤으로 만들고, 컴포넌트에서 그냥 가져다가 쓰는 방식으로 사용할 수도 있다. 물론 `mobx` 공식 문서에서는 [테스트를 위해서라면 권장하지 않는다](https://mobx-react.js.org/recipes-context)고 쓰여져 있다.

# 마이그레이션하기

이제 어느정도 두 버전 사이의 차이를 알아보았으니, 점진적인 마이그레이션을 위해서 `mobx`가 어떤 방법을 내놓았는지 보자.

눈여겨볼 만한 변경점은 `mobx-react` v6의 `Provider`의 변화이다. 공식 문서에 써져있던대로 거의 손을 대지 않고 `react`의 `context`를 그대로 사용하고 있다. 역시 불필요한 코드는 제거했다.

```js
// 원본 코드
// https://github.com/mobxjs/mobx-react/blob/master/src/Provider.js
import React from "react";

// 컨텍스트를 만든다.
export const MobXProviderContext = React.createContext({});

export function Provider({ children, ...stores }) {
  // 상위 컨텍스트의 스토어들 가져온다.
  const parentValue = React.useContext(MobXProviderContext);
  // 부모의 컨텍스트 값과 새로운 스토어들을 가져와 합쳐준다.
  const value = React.useRef({
    ...parentValue,
    ...stores
  }).current;

  return (
    <MobXProviderContext.Provider value={value}>
      {children}
    </MobXProviderContext.Provider>
  );
}

Provider.displayName = "MobXProvider";
```

몇 줄만으로 `Provider`의 구현이 끝났다. 마찬가지로 사용도 간단하다. `MobxProviderContext`는 상위 컨텍스트의 값을 가져와서 사용하기 때문에 컴포넌트에서 호출만 하면 된다. Inject도, observe로 감싸줄 필요가 없다.

```js
const UserProfile = () => {
  // Inject에 대응하는 부분이다.
  const { userStore } = useContext(MobxProviderContext);

  // 이렇게 하면 타입스크립트의 느낌표(!) 지옥에서도 빠져나올 수 있다.
  if (!userStore) {
    throw "userStore가 없습니다";
  }

  // observe에 대응하는 부분이다.
  return useObserver(() => (
    <div>
      {userStore.name} - {userStore.age}
    </div>
  ));

  // 혹은, 랜더 시에 필요한 특정 값만 필요하다면 이렇게도 사용할 수 있다.
  const businessLogicProfile = useObserver(() => {
    if (!userStore.public) {
      return "조회할수 없는 프로필입니다";
    }
    return `${userStore.name} - ${userStore.age}`;
  });

  return <div>{businessLogicProfile}</div>;
};
```

`mobx`는 그 자유도 때문에 프로젝트 구조가 매우 다양할 것이기 때문에 적절한 hook을 작성하여 스토어를 가져다가 쓰면 될 것 같다. 기본 구현이 매우 간단하기 때문이다.

그러면 새로운 코드는 어떻게 짜면 좋을까? 간단하게나마 [예제](https://codesandbox.io/s/friendly-pine-x2fid?fontsize=14)를 작성해 보았다. 가장 좋았던 점은 더이상 타입스크립트의 타입 추론을 !를 사용하여 강제시킬 필요가 없다는 점인것같다.

`useObserver`의 사용이 살짝 불편한 감이 없지 않아 있는것 같지만 어쨋든 새로운 기능을 사용해보는 것은 언제나 즐거운 일인 것 같다.

---

지금까지 간략하게 `mobx-react`의 내부적인 변화를 살펴보고 어떻게 새 변화에 맞추어 기존 코드베이스에 적용할지를 작성해보았다. 마이그레이션 부분은 너무나도 많은 자유도가 있기 때문에 내용이 부족한 부분이 있다고 생각되지만 고민하고 있는 사람들에게 많은 도움이 되었으면 좋겟다.
