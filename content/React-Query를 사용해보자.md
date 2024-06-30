---
title: React-Query를 사용해보자
date: 2022-04-11
draft: false
tags:
  - react
  - react-query
---
### React-Query란?
react-query는 swr과 함께 API 통신 과정을 간결하면서도 강력하게 사용할 수 있게 해주는 라이브러리입니다(실제로 둘 다 http 캐시 무효 전략인 stale-while-revalidate에서 착안했다고 합니다). 공식 문서의 말을 빌리자면 선언적이면서도 간단하며 zeroconf가 react-query의 장점인데요. 실제로 비동기 로직을 꼼꼼하게 구현하려면 세심한 관리와 노력이 필요합니다. 이와 같은 수고를 덜어줄 수 있는 react-query를 공부해보고 한번 사용해보며 알아보았습니다.

### 상태란?
프론트엔드 개발을 하다보면 상태를 관리하는 일이 정말 많은데요. 저희가 관리하는 상태란 무엇일까요?
제가 생각하는 상태는 이렇습니다.

- 특정한 때의 시스템을 구성하는 것으로 언제든지 변경될 여지가 있는 데이터
- 문자열, 배열, 객체 등의 형태로 프로그램에 저장된 데이터

세련된 UI, 뛰어난 UX가 대두됨에 따라 서비스의 규모가 커지고 코드의 양도 많아졌습니다.
자연스레 프론트엔드에서 관리하는 상태도 많아졌고 이로 인해 발생하는 다양한 이슈들을 해결하기 위한 방안들도 강구되고 있는 상황입니다(Props Drilling을 막기 위한 Redux, Mobx 등).

특히 백엔드로부터 받아오는 데이터는 매번 요청부터 응답까지 성공과 실패를 고려한 로직을 짜느라 바빴고, 응답 이후 클라이언트의 상태와 동기화를 시켜주어야 했습니다. 그럼에도 불구하고, 백엔드에서 받아오는 데이터들은 언제든 조작이 가능한 데이터들이기 때문에 클라이언트의 의도와 달리 데이터가 변경될 수 있고 잠재적으로 out of date가 될 가능성을 지녔습니다.

> 이러한 관점에서 react-query는 클라이언트에서 관리할 데이터와 서버에서 관리할 데이터를 분류해서 바라보게 해줍니다.

- Client State : 클라이언트에서 완벽하게 조작 가능한 동기적이면서도 최신 상태의 데이터입니다.
	
    - ex) useState
- Server State : 비동기 API를 통해 받아오는 데이터를 또 다른 상태로 바라봅니다.
	
    - 클라이언트에서 제어하지 못하는 데이터
    - 나만 조작 가능한 것이 아니기 때문에 최신화를 보장하지 못한다. => out of date 고려
    - ex) DB의 데이터
	
실제로 클라이언트에서 Server State도 함께 관리하다보면 store가 매우 혼잡해집니다(얼핏 보면 상태 관리 코드가 아니라 비동기 통신 코드같기도 합니다). 제가 사용해본 방법 중 가장 복잡했던 방식은 saga였는데요. 

```javascript
function* deletePost({id}) {
  try {
    yield put({ type: DELETE_POST_SUCCESS, deletePostId: id });
  } catch (err) {
    yield put({ type: DELETE_POST_ERROR, payload: err });
  }
}

function* watchAddComment() {
  yield takeLatest(DELETE_POST_REQUEST, addComment);
}
```
강력한 기능들을 제공하는 미들웨어임은 틀림없지만 간단한 비동기 로직에도 코드 양이 상당하고 초기 세팅, 러닝 커브 등 다양한 불편한 점들이 있었습니다. react-query를 사용해보니 쉬운 난이도와 가독성 대비 제공하는 기능들도 강력해서 충분히 만족할 수 있었습니다.

### Installation
```javascript
 $ npm i react-query
 # or
 $ yarn add react-query
```

### Before Start
```javascript
import { QueryClient, QueryClientProvider, useQuery } from 'react-query'
 
const queryClient = new QueryClient()

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
		<Todos />
    </QueryClientProvider>
  )
}
```
react-query를 사용하다보면 Server State가 전역적으로 관리된다는 느낌을 받는데요. Provider에서 눈치채신 분들도 계시겠지만 실제로 react-query는 react가 제공하는 context를 기반으로 만들어졌고 데이터를 관리한다고 합니다.

### react-query 사용해보기
#### Queries
```javascript
import { useQuery } from 'react-query'
 
function App() {
  const info = useQuery('todos', fetchTodoList, options)
}
```
- useQuery의 인자로 key, promise를 반환하는 함수, 옵션을 필요로한다.
- useQuery는  data, error, isError, isLoading, isSuccess, refetch, status 등등을 반환한다.
- options
  - enable : false로 설정하면 쿼리가 자동으로 호출되지 않는다. 즉 최초 선언시 호출을 막을 수 있다.
  - retry : 요청이 실패했을시 재요청을 실행한다. 기본값은 3이다.
  - refetchInterval : 일정한 간격으로 refetching이 가능하다.
  - refetchOnMount : mount되었을 때 refetch 여부를 결정한다. true라면 stale 상태일 때 refetch 한다. (booelan | "always")
  - staleTime : 데이터가 stale한 시점을 결정한다. 기본값은 0이다.
  - cacheTime : 캐싱된 데이터가 메모리에 남아있는 시간을 결정한다. 기본값은 5분(5 \* 60 \* 1000)이다.
  - 이 외에도 다양한 옵션들이 존재한다. 공식 문서를 확인해보자!
  
 
위 옵션들을 보면 낯선 옵션들이 존재하는 것을 확인할 수 있습니다(stale은 뭐고 cacheTime은 뭐지? refetchOnMount는 true 아니면 false지 always는 또 뭐지?). 원활한 이해를 위해 react-query의 핵심 사항들을 짚어보고 가겠습니다.
 
> Query의 라이프 사이클 

query는 크게 5가지의 상태를 가집니다.
차례대로 fetching > fresh > stale > inactive > deleted
fresh < -- > stale : 신선하거나 오래된 상태의 컨셉으로 진행됩니다.
 
- 최초 query가 선언되면 fetching 상태가 됩니다.
- 이후 데이터를 받아온다면 위 옵션에서 정해준 staleTime이 만료되기 전까지는 fresh 상태로 간주합니다. 다만 따로 설정을 하지 않는다면 staleTime은 0이기 때문에 바로 stale상태가 됩니다.
- fresh 상태에서 staleTime이 만료된다면 stale상태가 됩니다.
- 만약 query가 unmount된다면 inactive 상태가 됩니다.
- 이후 cacheTime이(디폴트 5분) 만료된다면 GC가 메모리에서 데이터를 삭제하고 deleted 상태가 됩니다.

query는 이러한 흐름을 갖고 진행됩니다.
예를 들어 query 옵션을 { refetchOnMount : "always", staleTime: 5000 } 로 설정한다면 fetching 이후 5초간은 fresh한 상태가 됩니다. 5초뒤에는 stale 상태가 됩니다. 만약 5초 이내 query가 unmount > mount 되었다면 refetchOnMount 옵션이 "always" 인 관계로 fresh한 상태지만 refetch가 일어나게 됩니다.

#### Query Keys
useQuery에 사용되는 key는 query를 식별하는 unique한 값으로 string과 array 두 가지 타입만 사용이 가능합니다.

##### string
```javascript
useQuery('todos', ...) // queryKey === ['todos']
```
다만, string도 내부적으로 단일 배열로 변환됩니다.

##### array
쿼리 데이터 식별에 추가적인 정보가 필요하다면 array 형태로 key를 제공할 수 있습니다.
```javascript
useQuery(['todo', 5], ...)
 
useQuery(['todo', 5, { preview: true }], ...)
```

다만 배열의 순서에 따라 모두 개별 key로 인식합니다.
```javascript
useQuery(['todos', status, page], ...)
useQuery(['todos', page, status], ...)
//not equal
```

#### Parallel Queries
쿼리를 여러개 선언해야 하는 상황에서도 별다른 스킬 없이 선언하면 됩니다. react-query가 내부적으로 병렬적으로 처리해준다고 합니다.
```javascript
function App () {
   const usersQuery = useQuery('users', fetchUsers)
   const teamsQuery = useQuery('teams', fetchTeams)
   const projectsQuery = useQuery('projects', fetchProjects)
   ...
 }
```

#### Queries Syncronization
예를 들어 A 라는 컴포넌트에서 todo query를 호출한 뒤 B 라는 컴포넌트에서 또 다시 todo query를 호출한다면 어떻게 될까요? API 호출이 한번 더 일어날까요? 정답은 "query가 stale한 경우만 API 호출이 발생한다" 입니다. 즉 fresh한 상태일 경우 API 호출 자체가 일어나지 않습니다. stale한 query는 refetch 된 후 A 컴포넌트와 B 컴포넌트에 최신화된 상태로 전달됩니다.


#### Mutations
앞서 살펴본 query는 데이터를 fetching 하는 것이 주목적이었다면 mutation은 create/update/delete에 사용을 권장합니다.

```javascript
function App() {
   const mutation = useMutation(newTodo => {
     return axios.post('/todos', newTodo)
   })
 
   return (
     <div>
       {mutation.isLoading ? (
         'Adding todo...'
       ) : (
         <>
           {mutation.isError ? (
             <div>An error occurred: {mutation.error.message}</div>
           ) : null}
 
           {mutation.isSuccess ? <div>Todo added!</div> : null}
 
           <button
             onClick={() => {
               mutation.mutate({ id: new Date(), title: 'Do Laundry' })
             }}
           >
             Create Todo
           </button>
         </>
       )}
     </div>
   )
 }
```
사용법도 매우 직관적이며 useQuery와 마찬가지로 다양한 프로퍼티들을 반환하기 때문에 상황에 맞게 사용이 용이합니다.

#### Mutation Side Effects
mutation은 라이프 사이클 로직 옵션도 제공합니다. axios의 인터셉터와 유사한 컨셉인 것 같습니다.
```javascript
useMutation(addTodo, {
   onMutate: variables => {
     // ... //
     return { id: 1 }
   },
   onError: (error, variables, context) => {
     // ... //
     console.log(`rolling back optimistic update with id ${context.id}`)
   },
   onSuccess: (data, variables, context) => {
     // ... //
   },
   onSettled: (data, error, variables, context) => {
     // ... //
   },
 })
```
특히 onMutate같은 경우 로직이 시작되자 마자 실행이 됩니다. 따라서 좋아요와 같은 Optimistic update를 적용할 때 유용해 보입니다.

#### Invalidation
invaliation은 특정 query가 stale 상태 혹은 개발자가 원하는 특수한 상태일 때 query를 처분(무효화) 하기 위해 사용합니다. 대개는 mutation이 일어난 후 기존 데이터가 stale 상태일 확률이 높기 때문에 mutation의 onSuccess 옵션과 함께 사용합니다.
```javascript
import { useMutation, useQueryClient } from 'react-query'
 
const queryClient = useQueryClient()
 
const mutation = useMutation(addTodo, {
  onSuccess: () => {
    queryClient.invalidateQueries('todos')
    queryClient.invalidateQueries('reminders')
  },
})
```
이 과정을 통해 특정 query는 stale 상태로 전환되고, 현재 rendering 되고 있는 query 들은 백그라운드에서 refetch 과정을 거친 후 최신 데이터를 유지하게 됩니다.

#### Devtools
react-query는 데브툴을 제공합니다.
```javascript
import { ReactQueryDevtools } from 'react-query/devtools'
 
function App() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* The rest of your application */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```
key별 query들의 실시간 상태 정보를 제공하는 등 디버깅에 도움이 되는 기능들이 많습니다.

간단한 공식문서 예시입니다.
https://codesandbox.io/s/github/tannerlinsley/react-query/tree/master/examples/simple

#### 좋은 점
- 비동기 로직이 매우 간결해진다. 즉, 가독성이 정말 좋아진다.
- Client에는 Client State만 남아 구조가 가벼워진다.
- useInfiniteQuery, ssr 및 graphQL 대응 등 강력한 기능들이 제공된다.

#### 불편한 점
- 사실 react-query나 swr과 같은 컨셉의 라이브러리들을 사용한지 얼마 되지 않았고 큰 프로젝트에 적용해보지 않아서 아직..간결한 코드가 너무 편하게 다가온다..
- saga의 특정 기능들의 편리함에서 오는 불편( ex. takeLatest )
- 컴포넌트별로 query를 흩뿌리는 방식이 불편해서 한 곳에 모아두고 사용하니 조금 개선되었다.
```javascript
// /feedQuery

import { useQuery } from "react-query";
import { fetchComments } from "../api/fetchComments";
import { fetchFeeds } from "../api/fetchFeeds";

export const useFetchFeeds = (category, page) =>
  useQuery(category, () => fetchFeeds(category, page));

export const useFetchComments = (id) =>
  useQuery(["comments", id], () => fetchComments(id));
```

### 참고

react-query 공식 문서
https://react-query.tanstack.com/overview