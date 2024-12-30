---
title: ref.current.contains(e.target) returns incorrect value if react element is removed on the next render
date: 2024-12-21
draft: false
tags:
  - bug
  - react
  - event
---

# 들어가며

근래 회사에서 큰 피쳐를 맡아 재밌게 개발하였고 성공적으로 배포까지 완료하였습니다.

개발하던 중 특정 요소의 외부 클릭을 감지해야 하는 기능이 필요했고, 사내 코드에 해당 기능을 제공하는 코드가 있어 활용하였습니다.

다만, 특정 상황에서 예상과 다르게 동작하여 난항을 겪었는데요, 오늘은 해당 문제를 해결하기 위해 고군분투한 내용을 작성해 보려고 합니다.

# 배경

회사에서 `onClickOutside` 유틸 훅을 사용하다가 버그를 마주했습니다.

외부 감지를 위해 `ref`를 부착한 `node`가 사라지는 경우 정상적으로 동작하지 않는 버그였습니다.

아래는 버그 재현 데모입니다.

<iframe src="https://codesandbox.io/embed/x37hmg?view=preview&module=%2Fsrc%2FuseOnClickOutside.ts&hidenavigation=1&expanddevtools=1"
     style="width:100%; height: 500px; border:0; border-radius: 4px; overflow:hidden;"
     title="useOnClickOutside-test"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

(가독성을 위해 실제 훅의 불필요한 부분은 모두 제거하였습니다.)

`onClickOutside`의 코드만 봤을 때는 왜 동작하지 않는지 쉽게 파악할 수 없었습니다.

디버깅 중 콘솔을 확인해 보니 더욱 혼란스러웠습니다.

위 데모의 `not working` 콘솔을 보시면, `e.target`은 정상적으로 출력되지만, `e.currentTarget?.contains(e.target)`은 `false`로 출력되는 것을 확인할 수 있습니다.

반대로 `working`의 콘솔은 예상하는 것과 동일하게 출력됩니다.

왜 이런 현상이 발생하는 것일까요?

# 해결 방법

디버깅 중 저와 같은 문제를 겪은 이슈를 발견할 수 있었습니다.

> [!Bug] [Bug: ref.current.contains(e.target) returns incorrect value if react element is removed on the next render #20325](https://github.com/facebook/react/issues/20325)

해당 이슈에서는 `event listener options`의 `capture` 속성을 `true`로 설정하여 문제를 해결할 수 있다고 설명합니다.

또 다른 해결책은 `event type`을 `mousedown`으로 설정하는 것입니다. 실제로 외부 클릭 감지 기능을 제공하는 많은 라이브러리들이 이와 같은 방식으로 문제를 해결합니다.

해결 방법은 매우 간단합니다. 다만, 왜 이것들이 해결 방법이 될 수 있는 것일까요?

해결 원리를 알면 실용적이진 않을 수 있지만 문제를 해결할 수 있는 또 다른 방법도 떠올릴 수 있습니다(후술)

# 근본적인 이유

앞선 방법들이 문제를 해결할 수 있는 이유는 다음 두 가지와 관련이 있습니다.

1. 브라우저의 이벤트 처리 방식

2. React의 이벤트 처리 방식

3. React의 렌더링 프로세스

## 브라우저의 이벤트 처리 방식

브라우저는 특정 요소에서 이벤트가 발생하였을 때 다음과 같은 단계를 거치며 이벤트를 처리합니다.

1. 캡쳐(capture): 트리 최상단 요소에서부터 시작해서 실제 이벤트가 발생한 타겟 요소까지 내려가는 것을 의미함.

2. 타겟(target): 타겟에 도달하는 단계. 이 단계에서 이벤트가 호출됨.

3. 버블링(bubbling): 이벤트가 발생한 요소에서부터 시작해 최상위 요소까지 다시 올라감.

즉, 캡쳐 단계에 등록된 이벤트는 우선적으로 실행될 것이며 버블링 단계에 등록된 이벤트는 후순위로 실행될 것입니다.

## React의 이벤트 처리 방식

기본적으로 React는 이벤트 핸들러를 각 요소에 부착하는 것이 아니라, [이벤트 위임](https://davidwalsh.name/event-delegate)을 통해 이벤트를 처리합니다.

실제로 위 데모의 `toggle work` 버튼의 이벤트 리스너를 확인해 보면 `noop(no operation)`이 등록된 것을 확인할 수 있습니다.

![toggle work 이벤트 리스너](./assets/Bug: ref.current.contains(e.target) returns incorrect value if react element is removed on the next render/noop.png)

그렇다면 각 이벤트 핸들러들은 어디에 등록될까요?

React 17 버전부터 이벤트 위임 대상이 `document`에서 **React 루트 요소**(일반적으로 div#root)로 변경되었습니다.

또한, [기본적으로 React는 이벤트 리스너를 버블링 단계에 호출합니다](https://ko.legacy.reactjs.org/docs/events.html#supported-events)

캡쳐 단계에서 이벤트 리스너를 호출하고 싶다면 `Capture` 키워드를 덧붙인 속성을 사용해야 합니다.(ex. `onClickCapture`)

점점 실마리가 보이는 것 같습니다.

## React의 렌더링 프로세스

React의 렌더링은 크게 다음 두 단계를 거칩니다.

1. 렌더 단계(Render Phase): 컴포넌트를 렌더링하고 변경 사항을 계산하는 단계.

2. 커밋 단계(Commit Phase): 실제 DOM 노드 및 인스턴스를 업데이트하는 단계. 해당 단계 이후 변경된 DOM에 접근 가능.

위 과정은 일반적으로 **동기적으로** 진행되며, 해당 과정을 거쳤다면 브라우저의 페인팅이 일어나진 않더라도(=UI 변경이 눈에 보이진 않더라도) DOM은 이미 업데이트가 된 상황입니다.

## 동작 흐름 및 검증

앞서 살펴본 개념들을 합쳐보면 왜 `useOnClickOutside`가 정상적으로 동작하지 않았는지 유추할 수 있습니다.

기존의 코드는 다음과 같은 단계로 진행되었고, 따라서 예상대로 동작하지 않습니다.

1. `click` 이벤트 발생

2. React root의 이벤트 핸들러 실행

3. React의 렌더링 프로세스 진행

4. DOM 업데이트 > `show` 버튼 제거

5. `document`의 이벤트 핸들러 실행

6. `ref?.current?.contains(e.target)`가 `false`로 판별(e.target === show button)됨에 따라 조건문 통과

7. `useOnClickOutside`의 콜백 함수(`setShow(true)`) 실행

8. React의 렌더링 프로세스 재진행

9. DOM 업데이트 > `show` 버튼 노출

앞선 과정을 모두 거친 후, 콜스택에 더 이상 처리할 프로세스가 없다면 브라우저는 페인팅을 시작합니다.

최종적으로 업데이트된 DOM은 `show` 버튼이 노출된 상태이기 때문에 UI 상으로는 아무 변화가 없는 것처럼 보이게 됩니다.

퍼포먼스 탭에 마커를 찍어보면 위 과정과 동일한 순서로 동작하는 것을 확인할 수 있습니다.

![브라우저 프로세스](./assets/Bug: ref.current.contains(e.target) returns incorrect value if react element is removed on the next render/퍼포먼스.png)

# 해결 방법이 해결 방법인 이유

이제 우리는 해결 방법이 어떻게 문제를 해결하는지 알 수 있습니다.

## `event listener options`의 `capture` 속성을 `true`로 설정

앞서 살펴본 동작 흐름에 따르면 React root의 이벤트 핸들러가 `document`에 등록된 이벤트 핸들러보다 먼저 실행되기 때문에 문제가 발생합니다.

따라서 `capture` 속성을 사용하여 `document`에 등록된 이벤트 핸들러를 캡쳐 단계에서 실행되도록 설정한다면 React root의 이벤트 핸들러보다 우선적으로 실행되어 앞선 문제를 해결할 수 있습니다.

## `event type`을 `mousedown`으로 설정

이 아이디어 또한 근본적인 해결 방식은 위와 같습니다.

마우스 클릭 이벤트는 `mousedown` > `mouseup` > `click` 순으로 발생합니다.

즉, React root의 `click` 이벤트 핸들러가 실행되기 전에 실행되는 `mousedown` 이벤트에 이밴트 핸들러를 등록하면 실행 우선권을 확보하여 문제를 해결할 수 있습니다.

## App Layout에 이벤트 핸들러 등록

이제 이벤트 핸들러 실행 순서를 조정하면 된다는 것을 알기에 이런 해결 방식도 떠올려 볼 수 있습니다.

기존에 이벤트 핸들러를 등록하던 타겟을 `document`가 아닌 [데모](#배경)의 `layout`으로 변경하는 것입니다.

이 방법이 문제를 어떻게 해결할까요?

React 루트 하위에 이벤트 핸들러를 등록하면 버블링 단계에서 루트에 위임된 이벤트 핸들러보다 먼저 실행되므로 앞서 언급한 방법들과 같이 문제를 해결할 수 있습니다.

# 맺으며

처음 이 문제를 마주했을 때는 혼란스러운 점들이 많았습니다. 디버깅 과정에서 알고 있던 것들을 되짚어 볼 수 있었고, 새로운 사실도 많이 배웠습니다.

오랜만에 재미있는 디버깅이었고, 사내 코드 리뷰 시간에 공유할 수 있어서 보람찼습니다.

간단한 생각거리를 남겨두고 글을 마무리하겠습니다.

## 생각해 보기

- 사용자가 React의 `onClick`이 아닌 `onClickCapture`를 사용한다면 위의 각 해결 방식이 정상적으로 작동할까?

# 참고

- 관련 이슈: https://github.com/facebook/react/issues/20325

- 이벤트 위임: https://davidwalsh.name/event-delegate

- React의 이벤트 리스너 호출 타이밍: https://ko.legacy.reactjs.org/docs/events.html#supported-events
