---
title: "[V8] 배열 연산 최적화"
description: "Why certain code patterns are faster than others"
date: "2021-05-30"
update: "2021-05-30"
series: "V8 Internals"
tags:
  - V8
---
## 1. Elements
---
***JavaScript***의 객체는 `key-value`로 구성된 `property`의 집합이다.    
특정 `property`에 접근하기 위해 사용하는 `식별자`는  `property key`또는 `property name`으로 불리며, 모든 문자열이 올 수 있다.  
`V8`은 이 중 식별자가 `정수`인 `property`를 특별하게 관리한다. 

`정수`로 되어있는 식별자는 `배열`을 생성할 때 가장 흔하게 볼 수 있다.  
`배열`에 원소를 추가하면 각 원소는 0부터 시작하는 식별자를 가지게 되는데, `V8`은 이러한 것들을 따로 모아서 저장한다.  
또한, 내부적으로 `elements`라는 특별한 이름을 부여한다.  

## 2. Elements Kinds
---
`V8`은 `reduce`, `map`, `forEach`과 같은 연산을 최적화 하기 위해  
배열에 어떤 종류의`elements`가 들어있는지 확인한다.  

```javascript
const arr = [1, 2, 3, 4, 5];
```

위와 같은 배열에서, `elements kind`는 `정수`가 아니다.  

**JavaScript**는 숫자를 `정수`, `실수`와 같은 타입으로 표현하지 않기 때문이다.  
ES2020에 추가된 `BigInt`형식을 제외하면 모든 숫자는 `number`형식으로 표현한다.  

```javascript
const [int, float, bigInt] = [3, 4.1, 9n];

[int, float, bigInt].map((v) => typeof(v));
// ['number', 'number', 'bigint']
```

`elements kind`가 `number`인 것도 아니다.  

엔진 레벨에서는 더 정밀하게 구분하는데,  
터미널에 `node --allow-natives-syntax`를 입력하고, `%DebugPrint()`함수를 이용하면 `elements kinds`를 확인할 수 있다.

```
❯ node --allow-natives-syntax
Welcome to Node.js v17.6.0.
Type ".help" for more information.
> const arr = [1, 2, 3, 4, 5]; %DebugPrint(arr);
DebugPrint: 0x3eba93e87dc9: [JSArray]
 - map: 0x07fe76e03d89 <Map(PACKED_SMI_ELEMENTS)> [FastProperties]
 - prototype: 0x025d1c1a4b29 <JSArray[0]>
 - elements: 0x2daabc5cb651 <FixedArray[5]> [PACKED_SMI_ELEMENTS (COW)]
```

위 배열의 `elements kind`는 `PACKED_SMI_ELEMENTS`이다.