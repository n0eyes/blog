---
title: 타입 호환성 (feat. 집합)
date: 2023-03-14
draft: false
tags:
  - typescript
---
타입스크립트는 유용하게 사용한다면 안정성 높은 코드를 보장해 주지만 타입 체커에 의존하여 작성하다 보면 타이핑에 많은 시간을 할애하는 모습을 발견할 수 있습니다. (특히 제가 그렇습니다)

~~(빠른 퇴근)~~ 생산성을 높히기 위해 학습한 타입 호환성에 대해 공유하고자 합니다.

## 타입을 집합으로 바라보기
결론부터 말씀드리면 타입은 할당 가능한 모든 값들의 집합입니다.
```ts
type number = 1 | 2 | 3 | 4 | 5 ...
```
타입을 집합으로 바라본다면 부분집합도 존재할 것입니다.
```ts
const subNum:1 = 1 
```
부분집합의 정의에 따라 아래 등식이 유효합니다
```ts
let supNum:number
let subNum:1 = 1 

supNum = subNum
subNum = supNum //Type 'number' is not assignable to type '1'.
```
부분 집합은 주어진 집합에 할당할 수 있지만, 반대는 허용되지 않습니다.
생각해보면 당연합니다. `number` 타입의 1을 제외한 무한대의 숫자들은 1에 할당될 수 없으니 말이죠.

`supNum`과 같이 다른 한 타입을 포함하는 타입을 슈퍼타입(supertype)이라고 하고, `subNum`슈퍼타입에 포함되는 타입을 서브타입(subtype)이라고 합니다.

따라서 위에서 확인한 등식을 다시 말하자면, supertype에서 subtype으로 downcasting은 가능하지만 subtype에서 supertype으로 upcasting은 허용되지 않는다고 말할 수 있습니다.

### 공집합
문자 그대로 어떠한 값도 할당될 수 없는 `never` 타입 입니다.
즉, 어떤 타입으로도 upcasting 할 수 없는 모든 타입의 subtype 입니다.

### 전체집합
문자 그대로 어떠한 값도 할당될 수 있는(따라서 어떤 값인지 알 수 없는) `unkown` 타입 입니다.
즉, 어떤 타입으로도 downcasting 할 수 있는 모든 타입의 supertype 입니다.


### any?
따라서 any는 매우 위험한 타입입니다.
어떤 타입으로도 upcasting 할 수 없는 모든 타입의 subtype 이면서
어떤 타입으로도 downcasting 할 수 있는 모든 타입의 supertype 이기 때문입니다.


### 교집합
공통된 값을 지닌 집합입니다.
```ts
interface Dog {
	kind: 'animal',
  	bark(): void  	
}

interface Cat {
	kind: 'animal',
  	meow(): void
}

const DogAndCat:Dog & Cat = {kind: 'animal', bark(){}, meow(){}}
```

교집합이면 아래가 맞는게 아닌가? 라는 의문을 가질 수 있습니다.
```ts
const DogAndCat:Dog & Cat = {kind: 'animal'}
```
공통 프로퍼티는 kind니까 말이죠.
하지만 타입스크립트는 속성(타입)의 집합이 아닌 값들의 집합으로 적용되기 때문입니다.

반대로 생각해보면 간단합니다. 교집합이라함은 Dog와 Cat 모두에 포함될 수 있어야 합니다. 즉 Dog와 Cat 모두의 subtype이어야 하고 `{kind: 'animal', bark(){}, meow(){}}`는 그 최소 조건을 만족하는 부분집합입니다.

따라서 부분집합의 관점에서 본다면 아래의 식도 만족합니다.
```ts
interface WoowahanAnimal {
  name:'woowahanAnimal' ,
  kind: 'animal',
  bark():void, 
  meow():void 
}

const woowahanAnimal: WoowahanAnimal = {
  name:'woowahanAnimal' ,
  kind: 'animal', 
  bark(){},
  meow(){}
}
const DogAndCat:Dog & Cat = woowahanAnimal
```

> (번외) 아래 코드는 왜 동작하지 않을까요?

```ts
const DogAndCat:Dog & Cat = {
  name:'woowahanAnimal' ,
  kind: 'animal', 
  bark(){},
  meow(){}
}
```

### 합집합
두 집합을 모두 포함하는 집합입니다.
```ts
interface Dog {
	kind: 'animal',
  	bark(): void  	
}

interface Cat {
	kind: 'animal',
  	meow(): void
}

const DogAndCat:Dog | Cat = {kind: 'animal'} // OK
const DogAndCat:Dog | Cat = {bark(){}} // Error
```

교집합 처럼 반대로 생각해본다면, 값들의 합집합은 Dog, Cat, Dog & Cat 모두를 할당할 수 있어야 합니다.
그렇다면 `{ bark(){} }`는 왜 할당할 수 없는 것일까요?
Dog의 부분집합이니 Dog | Cat (합집합)에 할당할 수 있어야 하는 것 아닐까요?

정답은 `{ bark(){} }` 가 Dog의 부분집합이 아닌 supertype이기 때문입니다.

```ts
interface DogCharacter {
	bark():void
}

interface Dog {
	kind: 'animal',
  	bark(): void  	
}

const dogCharacter:DogCharacter = {bark(){}} as Dog // OK
const dog:Dog = {kind:'animal', bark(){}}  as DogCharacter // Error

```

### 맺으며
타입스크립트를 집합으로 바라보면 많은 궁금증이 해소됩니다. 머릿속으로 벤 다이어그램을 그리며 원인 모를 타입 체킹을 쫓아가다 보면 그 끝에는 결국 집합의 형상이 자리 잡고 있었던 것 같습니다. 실제 수학적 집합과 느낌이 다른 부분도 있기에 저도 아직 아리송하는 경우가 많지만.. 집합으로 바라보는 방식도 있다는 것을 공유하고 싶었습니다!

글을 마무리하며 몇 가지 생각해 볼 만한 타이핑을 적어두고 이만 마무리하겠습니다!

```ts
interface Top {
  top: string
}

interface Super extends Top{
  _super: string
}

interface Sub extends Super {
  sub: string
}

interface FnA {
  (arg: Sub): Top
}

interface FnB {
  (arg: Super): Super
}

const fnA:FnA = {} as FnB // error?
const fnB:FnB = {} as FnA // error?

// hint
interface AnyFn {
  (...args:never[]) : unknown
}

let anyFn:AnyFn

anyFn = fnA // ok
anyFn = fnB // ok
```
```ts
interface FnA {
  (arg1:string, arg2: number): void
  
}

interface FnB {
  (arg1:string): void
}

const fnA:FnA = (() => {}) as FnB // error?
const fnB:FnB = (() => {}) as FnA // error?
```
```ts
interface OverloadingA {
  (arg:string): void
  
}

interface OverloadingB {
  (arg:string): void
  (arg:number): void
}

const ovA:OverloadingA = (() => {}) as OverloadingB // error?
const ovB:OverloadingB = (() => {}) as OverloadingA // error?
```


## 참고
-  이펙티브 타입스크립트