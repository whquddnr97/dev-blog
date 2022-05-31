---
title: "[V8] 배열 연산 최적화"
description: "Why certain code patterns are faster than others"
date: "2021-05-30"
update: "2021-05-31"
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
배열에 원소를 추가하면 각 원소는 0부터 시작하는 식별자를 가지게 되는데, `V8`은 이러한 것들을 따로 모아서 저장한다.  
또한, 내부적으로 `elements`라는 특별한 이름을 부여한다.  

## 2. Elements Kinds
---
`V8`은 `reduce`, `map`, `forEach`과 같은 연산을 최적화 하기 위해  
배열의 `elements kins`를 확인한다.  

```javascript
const arr = [1, 2, 3, 4, 5];
```

위와 같은 배열에서 `elements kind`는 `정수`가 아니다.  

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

### 2-1. PACKED vs. HOLEY
---
첫 번째 구분 기준은 배열이 채워져 있는지, 빈 공간이 있는지 여부이다.  

```javascript
let arr = [1, 2, 3, 4, 5]; // PACKED
arr.length = 6; // HOLEY
```

처음 배열을 생성할때, 배열에는 빈 공간이 없다. 이런 상태를 `PACKED`라고 한다.  
이후, 배열의 길이를 1 늘려주었다. 따라서 배열의 마지막 인덱스에는 `empty item`이 들어가게 되고, 이 상태를 `HOLEY`라고 한다.  
  
`DebugPrint()`를 이용하여 배열의 상태를 확인해보면 다음과 같다.  

```javascript
let arr = [1, 2, 3, 4, 5];
// elements kind: PACKED_SMI_ELEMENTS

arr.length = 6;
// elements kind: HOLEY_SMI_ELEMENTS
// - elements: 0x3a30a07cb2d9 <FixedArray[23]> {
//          0: 1
//          1: 2
//          2: 3
//          3: 4
//          4: 5
//          5-22: 0x133e03d41669 <the_hole>
//}
```

배열 인덱스 마지막 요소에 `<the_hole>`이 추가되어 있는 것을 확인할 수 있다.  
`<the_hole>`은 배열에 비어있는 값이 있거나, 삭제된 값이 있을 때 이를 확인하기 위해 추가된다.  

### 2-2. Three Basic Types
---
다음 구분 기준은, 배열 원소의 종류이다.  

```javascript
const arr1 = [1, 2, 3]; // PACKED_SMI_ELEMENTS
const arr2 = [4.5, 9.23, 1.82] // PACKED_DOUBLE_ELEMENTS
const arr3 = ['a', 'b', 'c'] // PACKED_ELEMENTS
```

- `SMI` : Small Integers의 약자로, 32비트의 부호있는 정수값
- `DOUBLE` : SMI로 표현할 수 없는 실수
- `NONE` : SMI와 DOUBLE로 표현할 수 없는 일반적인 원소

`SMI`는 `DOUBLE`의 부분집합이고, `DOUBLE`은 마찬가지로 `NONE`의 부분집합이다.  
따라서 `SMI` -> `DOUBLE` -> `NONE` 으로 이동하면서 점점 더 일반화된 형태를 띄게된다.  

## 3. Elements Kinds Transitions
---
배열에 값이 추가되거나, 삭제되면 `elements kind`는 변형된다. 

```javascript
let arr = [1, 2, 3]; // PACKED_SMI_ELEMENTS
arr.push(4.2); // PACKED_DOUBLE_ELEMENTS
arr.push('a'); // PACKED_ELEMENTS
```

값이 추가됨에 따라 `elements kind`가 변형되는 것을 확인할 수 있다.  

```javascript
let arr = [1, 2, 3]; // PACKED_SMI_ELEMENTS
arr.length = 4; // HOLEY_SMI_EMELENTS
```

마찬가지로 `PACKED`에서 `HOLEY`로도 변형된다.  

![transition_lattice](lattice.svg)
주의할 점은 `transition`은 한 방향으로만 이루어진다는 것이다.  
`SMI`에서 `DOUBLE`로, `PACKED`에서 `HOLEY`로의 변형은 가능하지만, 그 반대는 불가능하다.

```javascript
let arr = [1, 2, 3];
// arr = [1, 2, 3], elements kind : PACKED_SMI_ELEMENTS

arr.push(4.27);
// arr = [1, 2, 3, 4.27], elements kind : PACKED_DOUBLE_ELEMENTS

arr.pop();
// arr = [1, 2, 3], elements kind : PACKED_DOUBLE_ELEMENTS

arr.length = 4;
// arr = [1, 2, 3, <1 empty item>], elements kind : HOLEY_DOUBLE_ELEMENTS

arr[3] = 4;
// arr = [1, 2, 3, 4], elements kind : HOLEY_DOUBLE_ELEMENTS
```

또한 한번 변형된 `elements kind`는 더 `specific`한 형태로 변형이 되지 않는다.  
`SMI`에서 `DOUBLE`로 변형이 되었을 때, `DOUBLE`값을 가진 요소를 삭제해도 `SMI`로 돌아가지 않는다.  
마찬가지로, `HOLEY`로 변형이 되었을 때 비어있는 부분에 값을 추가해도 `PACKED`로 돌아가지 않는다.  

## 4. Performance Tips
---
작성중