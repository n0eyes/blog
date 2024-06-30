---
title: Lazy initial state
date: 2022-05-15
draft: false
tags:
  - react
  - optimization
---
 
최근 코드 리뷰를 받고 성능적으로 자그마한 이득을 볼 수 있는 지식을 습득하게 되었습니다. 공식 문서에 기재되어 있는 정식 명칭은 'Lazy initial state'이며 '지연 초기화' 또는 '초기값 지연' 정도로 부르는 것 같습니다.

### 사용하지 않았을 때
제가 흔히 사용하는 useState의 방식입니다.
```javascript
const [count,setCount] = useState(1);
```
해당 방식은 (인위적이지만)아래와 같이도 많이 쓰입니다.
```javascript
const initalState = 1
const [count,setCount] = useState(initalState);
```
앞선 예시에서 초기값은 최초 선언시에만 등록됩니다. 다만 리렌더링이 발생할 경우 초기값 등록은 일어나지 않더라도 다른 명령들은 재실행됩니다. 위의 예시에선 initalState의 선언 및 할당이 재실행 될 것입니다.
### 문제점
위와 같이 간단한 명령은 괜찮습니다. 다만 리소스가 많이 사용되는 명령은 어떨까요? 예를 들어 대용량의 데이터에 복잡도가 큰 알고리즘을 적용한다면 분명 위 예시보단 리소스가 많이 사용될 것입니다. 이와 같은 코드가 리렌더링마다 실행된다면 성능적인 저하를 일으킬것이 분명합니다.
```javascript
// 매번 실행된다!
const result = someExpensiveComputation();
const [count,setCount] = useState(result);
```

### 사용하였을 때
위와 같은 문제점을 해결해줄 수 있는 방법이 바로 'Lazy initial state'입니다. 개념은 간단합니다. 매번 복잡한 연산을 실행하여 결과값을 도출하는 대신 상대적으로 리소스가 저렴한 함수 생성문을 실행하고 해당 함수를 최초에만 호출하도록 변경하는 것입니다.
```javascript
// 매번 선언만 된다!
const someExpensiveComputation = () => {...};
                                        
const [count,setCount] = useState(someExpensiveComputation);
//or
const [count,setCount] = useState(()=>someExpensiveComputation());
```
아래와 같이 변경해 줌으로써 비용이 큰 데이터를 초기값으로 등록할 때 조금이나마 성능을 향상시킬 수 있습니다.