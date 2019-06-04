---
layout: post
title: 'redux-saga와 typescript 편하게 사용하기'
date: 2019-06-04 15:02:15 +0900
categories: develop
---

나는 리덕스 사가의 팬이다. 작은 단위로 코드를 쪼개놓아 재사용성이 늘어나는 것도 좋고, 개발자의 코드가 개발자가 실행 시점을 정의하는게 아닌 **사가가 실행의 주체**가 된다는 점도 좋다. 작년 말부터 사가를 쓰기 시작해서 어느정도 비동기 호출이 있는 프로젝트에는 항상 사가를 넣어두고 시작할정도로 지금은 사가 없이는 살수 없는 사람이 되어버렸다.

그러던 중 `TypeScript`를 알게 되었고, `redux-saga`와 `TypeScript`를 함께 쓰면서 꽤 만족스러운 패턴을 작성하게 되어 이것을 소개하고자 글을 작성하게 되었다.

## `FetchEntity`

사가를 쓰면서 가장 즐겨 썼던 패턴은 `redux-saga`의 [real-world-example](https://github.com/redux-saga/redux-saga/blob/master/examples/real-world/sagas/index.js)에서의 `fetchEntity`함수였다. 나는 내가 사용하기 편하게 조금 다른 방식으로 작성했다.

```js
const fetchEntity = (entitiy, apiFn) =>
  function*(...params) {
    yield put(entitiy.request());
    try {
      const data = yield call(apiFN, ...params);
      yield put(entity.success(data));
    } catch (err) {
      yield put(entitiy.failure());
    }
  };
```

비동기 호출의 단위를 `entity`로 관리하고 이 호출의 루틴을 함수로 만들어 재사용성을 높인 함수이다.

여기서 안보이는 `entity`는 이렇게 생겼다. 역시 개인적으로 사용하기 편하게 간략하게 작성했다.

```js
export const user = {
  request: () => ({ type: 'USER_REQUEST' }),
  success: user => ({ type: 'USER_SUCCESS', payload: { user } }),
  failure: () => ({ type: 'USER_FAILURE' }),
};
```

정리하자면 각 비동기 단계의 `action`을 정의한 것이 `entitiy`인 것이다. 매번 이렇게 세가지 액션을 만들어주는 것이 귀찮으므로 함수를 작성해서 사용하자.

```js
const createEntityAction = entity => ({
  request: () => ({ type: entity.REQUEST }),
  success: data => ({ type: entitiy.SUCCESS, payload: data }),
  failure: () => ({ type: entity.FAILURE }),
});
```

이 패턴을 사용하면 간단한 비동기 호출은 정말 편하게 작성할 수 있다. `user`의 정보를 가져온다고 생각해보자.

```js
const fetchUserAPI = async userId => (await axios.get(`/user/${userId}`)).data;

// Action Types
const FETCH_USER = 'FETCH_USER';
const USER = {
  REQUEST: 'USER_REQUEST',
  SUCCESS: 'USER_SUCCESS',
  FAILURE: 'USER_FAILURE',
};

// action. 컴포넌트에 커넥트 시켜서 사용
const fetchUser = id => ({ type: FETCH_USER, payload: { id } });

// Entity
export const user = createEntityAction(USER);

const fetchUserSaga = fetchEntity(user, fetchUserAPI);

function* fetchUserWatcher() {
  const {
    payload: { id },
  } = yield take(FETCH_USER);
  yield call(fetchUserSaga, id);
}
```

`fetchEntity`패턴을 사용하면 기본적인 비동기 호출은 대부분 대응이 가능하다. `fetchUserWatcher`에서는 `call`을 사용했지만, 상황에 따라서 `fork`나, 혹은 `watcher`자체를 `takeLatest`나 `takeLeading`으로 사용해도 무방하다.

## TypeScript

> 이하는 타입스크립트 3.4버전 이상을 기준으로 작성되었습니다.

위의 패턴에서 타입스크립트를 붙여보자. 개인적으로는 **사용하는 쪽에서 제네릭을 최대한으로 사용하지 않는** 형태로 작성하는 것을 선호한다. 개발자가 통제할 수 없는 부분에서만 제네릭을 넣는 것이 사용성과 생산성을 높이는 것이라고 생각하기 때문이다.

우선 `API`호출을 하는 함수의 타입 지정을 해보자. 크게 복잡한 타입은 아니지만, 코드 길이를 줄이는데 도움을 준다. 비동기 데이터를 리턴하는 함수의 타입이다.

```ts
type APIEndpoint<P extends any[], R> = (...p: P) => Promise<R>;
```

그 다음은 api 호출을 하는 함수를 작성 해보자.

```ts
const fetchUserAPI = async (userID: number) =>
  (await axios.get<IUser>(`/user/${id}`)).data;
```

인터페이스 `IUser`는 여기서 중요한 부분이 아니라고 생각되어 작성하지 않았다. 함수는 간단하다. `userID`를 받아서 `IUser`를 서버에서 받아온 후 그것을 리턴 하는 함수이다.

이제 액션 타입을 작성 해보자.

```ts
// Action Type
const GET_USER = 'GET_USER' as const;
const USER = {
  REQUEST: 'GET_USER_REQUEST',
  SUCCESS: 'GET_USER_SUCCESS',
  FAILURE: 'GET_USER_FAILURE',
} as const;
```

주목해 보아야 할 부분은 `as const`의 사용이다. `as const`구문은 타입스크립트 3.4버전에서 추가된 기능이다. 상세한 문법의 스펙은 다음과 같다.

- 리터럴의 타입은 확장될 수 없다(`'hello'`에서 `string`으로 확장되는 행위 불가능)
- 오브젝트 리터럴은 `readonly`속성을 가지게 된다.
- 배열 리터럴은 `readonly` 튜플이 된다.

따라서, 이 객체나 액션 타입의 선언은 그 자체로 타입과 비슷하게 동작하게 된다. 이렇게 처리하게 되면 리듀서에서 스위치케이스 문을 작성하는 데에 아주 유용하게 사용할 수 있다.

이제 액션 크리에이이터를 작성해 보자. 우선 `getUser`를 먼저 작성했다.

```ts
const getUser = (userId: string) => ({ type: GET_USER, payload: { userId });
type GetUser = ReturnType<typeof getUser>;
```

`ReturnType`은 타입스크립트 2.8버전에서 추가된 타입으로, 함수의 반환값을 잡아준다.

3.4버전 `as const`가 없었을 때에는 액션크리에이터에 사용이 불가능했지만, `as const`로 액션크리에이터의 타입을 잡아주는데 굉장히 편한 타입이 되었다.

```ts
// GET_USER에 as const 가 없을떄
type GetUser = {
  type: string;
  payload: {
    userId: string;
  };
};
// GET_USER에 as const가 있을때
type GetUser = {
  type: 'GET_USER';
  payload: {
    userId: string;
  };
};
```

그 다음은 `entity`를 만들어주는 함수를 작성해보자. 이번에는 자바스크립트로 작성했을때와는 모양이 조금 다르다.

```ts
// 엔티티 타입을 위한 인터페이스
interface IEntity<R, S, F> {
  REQUEST: R;
  SUCCESS: S;
  FAILURE: F;
}

// 액션을 만들어 내는 데에 필요한 것은 DATA타입이다.
// SUCCESS액션의 리턴타입에 타입 지정을 해주기 해주기 위해서 반드시 필요하다.
// 이때, api를 모르는 상태에서는 반드시 제네릭으로 넣어주는 것이 필요한데,
// 제네릭을 넣어주는 방법을 우회하기 위해서 api와 액션을 함께 묶어 객체로 만들어주는 방법을 택했다.
const createEntityAction = <R, S, F, PARAM extends any[], DATA>(
  entitiy: IEntity<R, S, F>,
  api: ApiEndpoint<DATA, PARAM>
) => ({
  ACTION: {
    REQUEST: () => ({ type: entitiy.REQUEST }),
    SUCCESS: (data: DATA) => ({ type: entitiy.SUCCESS, payload: data }),
    FAILURE: () => ({ type: entitiy.FAILURE })
  },
  API: api
})

// 타입을 끄집어 내기 위한 헬퍼 타입
interface IEntityAction {
  ACTION: {
    REQUEST: (...p: any[]) => any;
    SUCCESS: (...p: any[]) => any;
    FAILURE: (...p: any[]) => any;
    [key: string]: (...p: any[]) => any;
  };
  API: ApiEndpoint<any, any>;
}

// 액션 타입 추출을 위한 타입
// ACTION의 각 단계의 리턴타입을 가져왔다.
EntityAction<T extends IEntityAction> = ReturnType<T['ACTION'][keyof T['ACTION']]>;

const userEntity = createEntityAction(USER, getUserApi);
type UserEntity = EntityAction<typeof userEntity>;
```

액션이 api에서 리턴하는 타입을 반드시 알고 있어야 하기 때문에 api를 인자로 넣어 주었다. 여러 가지 방법을 시도해 보았지만, 저 세가지 함수를 자동으로 만들어 주기 위해서는 어떤 부분에서 api에서 리턴하는 타입을 반드시 제네릭으로 넣어주어야 했기 때문에 다른 방법을 찾았던 것이 이 방법 이였다.

`Entity`가 api 앤드포인트 정보를 포함함에 따라서 `fetchEntity`의 모양도 달라지게 된다.

```ts
// 두 가지를 인자로 받는 대신에 객체 하나를 받는것으로 변경했다.
function fetchEntity<T extends IEntityAction>({ ACTION, API }: T) {
  return function*(...p: Parameters<T['API']>) {
    try {
      yield put(ACTION.REQUEST());
      const data = yield call(API, ...p);
      yield put(ACTION.SUCCESS(data));
    } catch {
      yield put(ACTION.FAILURE());
    }
  };
}
```

`Parameters<T>`는 `ReturnType<T>`와 비슷하게 동작한다. 함수에서 인자의 타입을 추출하는 유틸리티 타입인데, `API`함수에서 받는 인자를 그대로 리턴되는 함수에서 받기 위해서 사용했다.

이렇게 작성했으면 이제 사가를 작성하면 된다.

```ts
const getUserSaga = fetchEntity(userEntity);

function* getUserWatcher() {
  const {
    payload: { userId },
  }: GetUser = yield take(GET_USER);
  // fetchEntity가 타입을 제대로 지정해주기 때문에 userId인자의 타입까지 확인해준다.
  yield call(getUserSaga, userId);
}
```

여기까지 자바스크립트로 작성된 패턴을 타입스크립트로 다시 작성해 보았다. 부족한 실력이지만 이 글을 통해서 사가와 타입스크립트로 프로젝트를 작성하는 사람들에게 도움이 되었으면 좋겠다.

---

혹시 오타나 잘못된 내용이 있다면 피드백 주시면 감사하겠습니다.

전체 코드는 [이곳](https://gist.github.com/Jonir227/b7fc8b5b0646b7a90c26bd73a70c12b9)에서 확인할 수 있습니다.
