---
title: Animation 연속으로 적용하기
date: 2022-06-26
draft: false
tags:
  - css
---
 
최근 인터랙티브한 작업을 진행하던 중 새롭게 알게 된 지식이 생겨 잊지 않게 기록해 두고자 합니다.

### 기존 방식
여러가지 애니메이션을 순차적으로 적용하기 위해 제가 사용한 기존의 방식은 다음과 같습니다.
```css
.animation {
  animation-name: first, second;
  animation-delay: 0s, 5s;
  animation-duration: 5s, 12s;
}
```
duration과 delay 프로퍼티로 애니메이션 작동 타이밍을 적절하게 조절하는 것입니다. 다만 해당 방법은 일일이 타이밍을 고려해야 한다는 번거로움이 존재했습니다.

### 새롭게 알게 된 방식
이번에 알게 된 방식은 다음과 같습니다.
```javascript
element.getAnimations(options);

```
해당 메서드는 예약된 애니메이션에 관련된 정보가 담긴 배열을 반환합니다.
옵션 객체의 프로퍼티는 subtree가 존재합니다. {subtree: true} 인 경우, 해당 항목의 자손들의 애니메이션 정보도 포함합니다.

반환값의 프로퍼티는 다음과 같습니다.
![](https://velog.velcdn.com/images/n0eyes/post/2f22f976-038c-46a9-b950-bc078781e858/image.png)
다양한 정보들이 존재하지만 저희가 살펴볼 값은 effect.composite 입니다. 해당 프로퍼티는 애니메이션이 중첩 될 경우 처리하는 방식을 설정하는데 기본값은 'replace' 입니다. 말 그대로 나중에 적용된 애니메이션이 앞선 애니메이션을 대체합니다. 
```javascript
element.getAnimations().forEach((ani) => (ani.effect.composite = "add"));
```
값을 이렇게 'add'로 바꿔주면 애니메이션이 순차적으로 작동하게 됩니다.
다만, 아직 꽤나 experimental한 방식이라 사용할 땐 호환성을 꼭 체크하는 편이 좋을 것 같습니다.