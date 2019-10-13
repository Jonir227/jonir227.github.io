---
layout: post
title: 'Typescript에서 redux-actions대체하기'
date: 2019-10-13 19:03:12 +0900
categories: develop
---

# `Redux-Actions`

리덕스 액션즈는 리덕스에서 귀찮은 일들을 상당부분 줄여주는 유틸리티 라이브러리이다. 많은 기능들이 있지만 필자가 가장 즐겨 쓰는 (사실은 그것만 사용하는)함수는 두개. `createAction`과 `handleActions`이다.

이 두가지 함수는 리덕스에서 상당히 귀찮은 작업인 `ActionCreator`를 만드는 작업과 `Reducer`를 만드는 작업의 보일러플레이트를 굉장히 편하게 줄여준다. 그래서 개인적으로 작업을 하든, 업무를 할때든 리덕스를 사용할때면 거의 항상 사용하곤 했다.

아래는 라이브러리 사용을 간단하게 작성해 보았다. 하기의 사용 예 말고 다양한 사용 사례가 많지만 개인적으로는 대부분 이 사용 방법에서 벗어나지 않게 사용했던것 같다.

```js
// 액션
const CHANGE_NAME = 'CHANGE_NAME';
const CHANGE_AGE = 'CHANGE_AGE';

const defaultStae = {
  name: 'john',
  age: 27
};

// ------ Before ----------

// 액션 크리에이터
const changeName = name => ({
  type: CHANGE_NAME,
  payload: { name }
});
const changeAge = age => ({
  type: CHANGE_NAME,
  payload: { age }
});

// 리듀셔
const user = (state = defaultState, action) => {
  switch (action.type) {
    case CHNAGE_NAME: {
      return {
        ...state,
        name: action.payload.name
      };
    }
    case CHANGE_AGE: {
      return {
        ...state,
        age: action.payload.age
      };
    }
    default:
      return state;
  }
};

// -------- after -------

// 액션 크리에이터
const changeName = createAction(CHANGE_NAME, (name = { name }));
const changeAge = createAction(CHANGE_AGE, (age = { age }));

// 리듀서
const user = handleActions({
  [CHANGE_NAME]: (state, { payload }) => ({
    ...state,
    name: payload.name
  }),
  [CHANGE_AGE]: (state, action) => {
    ...state,
    age: payload.age
  }
});
```

확실히 구문이 읽기 편하고 깔끔해졌다. 여기까지 봐서는 안쓸 이유가 없어보인다. 하지만 여기에는 문제가 있다. 이 라이브러리는 타입스크립트 지원이 굉장히 허술하게 되어있다.

# 무엇이 문제인가?

여러가지 문제가 있지만, 가장 큰 문제는 `createAction`이 뱉어내는 액션 크리에이터의 타입은 무조건 `string`으로 고정되어 있다는 사실이다. [타입 정의](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/redux-actions/index.d.ts)를 한번 보자.

```ts
// FSA-compliant action.
// See: https://github.com/acdlite/flux-standard-action
export interface BaseAction {
  type: string;
}

export interface Action<Payload> extends BaseAction {
  payload: Payload;
  error?: boolean;
}
```

이 라이브러리를 타입스크립트에서 사용하려고 시도해 보았던 사람들은 알 것이다. `string` 타입으로 고정해서 내놓는 것이 얼마나 끔찍한 일인지.. 우선 이 액션 크리에이터가 뱉어내는 액션은 기본적으로 `string`타입이기 때문에 액션 크리에이터가 리턴하는 인터페이스를 따로 짜주어야 한다. 이렇게 말이다.

```ts
const changeName = createAction(CHANGE_NAME, name => ({ name }));
interface ChangeName {
  type: typeof CHANGE_NAME;
  payload: {
    name: string;
  };
}
```

[이전 글](https://jonir227.github.io/develop/2019/06/04/redux-saga%EC%99%80-typescript-%ED%8E%B8%ED%95%98%EA%B2%8C-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0.html)에서 소개했던 `ReturnType`을 이용하는 방법도 어림없다. 왜냐하면 `changeName`이 리턴하는 함수는 무조건 `type`이 `string`이기 때문에 올바르게 액션을 만들어내지 못하기 때문이다.

또 다른 문제는 `handleActions`에도 있다. 이 함수의 문제 역시 꽤 많다. 우선 가장 기본이 되는 타입 정의부터 살펴보자.

```ts
export type Reducer<State, Payload> = (
  state: State,
  action: Action<Payload>
) => State;

export type ReducerMeta<State, Payload, Meta> = (
  state: State,
  action: ActionMeta<Payload, Meta>
) => State;

export type ReducerMapValue<State, Payload> =
  | Reducer<State, Payload>
  | ReducerNextThrow<State, Payload>
  | ReducerMap<State, Payload>;

export interface ReducerMap<State, Payload> {
  [actionType: string]: ReducerMapValue<State, Payload>;
}

export function handleActions<State, Payload>(
  reducerMap: ReducerMap<State, Payload>,
  initialState: State,
  options?: Options
): ReduxCompatibleReducer<State, Payload>;
```

여기서도 보이는 문제는 액션이 `type`을 `string`으로 간주한다는 점이다. 액션 타입에 대하여 제네릭이든지 어떤 타입 추론에 도움이 될만한 정보가 존재하지 않는다. 이 끔찍한 가정은 `handleActions`에서 두번째 제네릭의 인자로 "액션"이 아닌 "페이로드"만 받는것과도 상통한다. 그러면 이 함수를 어떻게 써야할까?

```ts
// 페이로드를 가져오기 위한 헬퍼 타입
type GetPayload<A extends (...args: any[]) => any> = ReturnType<A> extends Action<infer P>
  ? P
  : never;

const changeName = createAction(CHANGE_NAME, (name: string) => ({ name }));
const changeAge = createAction(CHANGE_AGE, (age: number) => ({ age }));

type UserPayloads = GetPayload<typeof changeAge | typeof changeName>;

interface User {
  age: number;
  name: string;
}

const defaultState = {
  age: 27,
  name: 'john'
};

const user = handleActions<User, UserPayloads>(
  {
    [CHANGE_NAME]: (state, action) => {
      // 으윽.. 머리가...!
      if(!action.payload.name) {
        return state;
      }
      return {
        ...state,
        name:
      };
    }
  },
  defaultState
);
```

무언가 머리가 아픈 부분이 발견되었는가? `CHANGE_NAME`이 받는 함수의 `action`은 `Typescript`의 `switch/case`문 안에서의 `Type narrowing`의 효과를 전혀 받지 못한다. 모든 액션의 페이로드가 섞여있기 때문에 각 액션 타입에 맞는 페이로드를 걸러낸 다음 사용해야 한다. 이렇게 되면 사실상 `Typescript`를 사용하는 이점이 많이 사라지기 때문에 타입스크립트를 사용하면서 이 라이브러리를 사용하기 않게 되었다.

# 대안?

물론 대안을 찾아보지 않은 것은 아니다. 문제를 해결하기 위한 유명한 라이브러리가 있는데 [`typesafe-actions`](https://github.com/piotrwitek/typesafe-actions)라는 라이브러리이다. 정말 좋은 라이브러리이지만 이 라이브러리의 사용법이 익숙하지 않고, `redux-acton`의 액션-핸들액션 맵핑 방식이 가독성이 매우 뛰어나다고 생각하여 직접 만들기로 했다.

# 직접 만들기

앞서서 `redux-actions`의 타입 정의가 엉망이라고 했지만 사실 과거에는 어쩔 수 없었다고 생각한다. `string`타입으로만 문자열을 사용할 수 있었으니까. 하지만 우리에게는 `3.4`버전부터 추가된 `as const`가 있다. 그걸 사용해서 비슷하지만, 타입정의를 정확하게 해주는 대체제를 만들어보자

## `handleActions`

코드와 함께 보자.

```ts
// 리덕스이 표준 액션 객체를 사용한다.
import { Action } from 'redux';

// `type`를 `string`이 아닌 제네릭으로 두어서 `type`의 타입을 유지하여 리턴한다.
// 정확한 액션 크리에이터 함수를 만들기 위해서 함수의 시그니쳐를 오버로딩한다.
// 1. 페이로드 크리에이터 함수가 정의되어 있는 경우
// 2. 페이로드 크리에이터 함수가 정의되어있지 않아 페이로드가 없는 경우
function createAction<T, P extends (...args: any) => any>(
  type: T,
  payloadCreator: P
): (...args: Parameters<P>) => Action<T> & { payload: ReturnType<P> };
function createAction<T>(type: T): () => Action<T>;
function createAction(type: any, payloadCreator?: any) {
  return (...args: any[]) => ({
    type,
    ...(payloadCreator && { payload: payloadCreator(...args) })
  });
}

export default createAction;
```

자 이렇게 하고 사용해보자. 사용법은 매우 유사하다.

```ts
const changeName = createAction(CHANGE_NAME, (name: string) => ({ name }));
const changeAge = createAction(CHANGE_AGE, (age: number) => ({ age }));

type UserActions = ReturnType<typeof changeName | typeof changeAge>;
```

한결 간결해지고, 단순히 페이로드만의 타입이 아닌, 액션의 타입 전부를 기억하고있는 액션 타입이 만들어졌다. 그 다음은 이것을 처리하는 `handleActions`의 차례이다.

## `handleActions`

처음 타입스크립트를 사용하려고 했을때 이 함수를 어떻게든 만들어보려고 했었다. 결과는 실패로 돌아갔는데 `as const`의 은총으로 액션 타입에 대한 정확한 정의가 가능해진 지금은 정복이 가능해졌다.

```ts
// 역시 리덕스의 표준 액션 인터페이스를 사용한다.
import { Action } from 'redux';

// 인자로 받을 리듀서의 키 밸류 맵이다.
// 리덕스의 액션 타입의 기본은 any로 되어있기 때문에 string으로 만든 액션을 상속받아서 사용한다.
type ReducerMap<A extends Action<string>, S> = {
  // 액션을 상속받은 A의 `type`의 타입들을 키로 사용한다.
  // 리듀서 함수의 인자로 넘어가는 액션은 그 키 타입에 맞는 액선을 끄집어내서(MatchedAction) 사용한다.
  [AT in A['type']]?: (state: S, action: MatchedAction<A, AT>) => S;
} & { [key: string]: (state: S, action: Action) => S }; // 객체의 키 접근을 위한 인덱스 시그니쳐

// 인자로 받은 액션들(A) 중에 액션타입(T)이 일치하는 타입만 내보내고 나머지는 지운다.
type MatchedAction<A, T> = A extends Action<T> ? A : never;

const handleActions = <S, A extends Action<string>>(
  reducerMap: ReducerMap<A, S>,
  defaultState: S
) => (state = defaultState, action?: A): S =>
  action && reducerMap[action.type]
    ? reducerMap[action.type](state, action)
    : state;

export default matchAction;
```

다음은 사용 예이다. 역시 원본과 유사하게 사용하면 된다. 단, 지금은 액션에 맞추어서 필터해두었기 때문에 해당하는 액션 타입에 맞는 액션이 함수의 인자로 들어온다.

```ts
const user = handleActions<UserActions, User>(
  {
    // 정의된 액션들에 맞는 타입만 객체 키값으로 사용가능하고, 인자로 넘겨지는 액션도
    // 그 키에 맞는 액션이 전달되기때문에 타입에 안전하다.
    [CHANGE_NAME]: (state, { payload }) => ({
      ...state,
      name: payload.name
    })
  },
  defaultState
);
```

여기까지 간단하게 `redux-actions`의 대체 함수를 만들어보았다. 사실 구현되지 않은 기능들이 리덕스 액션에 많이 존재하지만 간단한 사용 용도로는 이정도 구현도 적절하리라 생각한다. 필요에 따라서 추가로 구현하거나 하면 될 것 같다. 타입스크립트를 사용하면서 `redux-actions`를 그리워 하는 사람들에게 도움이 되었으면 좋겠다.
