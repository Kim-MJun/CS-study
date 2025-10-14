# Javascript에서 var와 let의 핵심 차이

### 1. 오해하기 쉬운 var, let 차이

- 전역/지역 변수 차이가 아닌 스코프 범위의 차이

---

### 2. 핵심 차이점

#### a. 스코프 범위

1. var -> 함수 스코프 (function scope)

```js
function test() {
  if (true) {
    var x = 10;
  }
  console.log(x); // 10 출력 (접근 가능!)
}
```

2. let -> 블록 스코프 (block scope)

```js
function test() {
  if (true) {
    let x = 10;
  }
  console.log(x); // ReferenceError! (접근 불가)
}
```

\*\*\* `var`는 `{}`를 무시하고 함수 전체에서 접근 가능해서 `의도치 않은 버그`가 발생하기 쉬움

#### b. 호이스팅 동작

1. var -> `undefined`로 초기화

```js
console.log(a); // undefined
var a = 5;
```

2. let -> TDZ(Temporal DeadZone)

```js
console.log(b); // ReferenceError!
let b = 5;
```

\*\*\* `var`는 선언 전에 사용해서 에러가 안 나서 버그를 찾기 어려움

#### c. 재선언 가능 여부

1. var -> 재선언 가능

```js
var x = 1;
var x = 2; // 에러 없음 (위험!)
```

2. let -> 재선언 불가

```js
let y = 1;
let y = 2; // SyntaxError!
```

#### d. 전역 객체 오염

1. var -> window 객체에 추가됨

```js
var globalVar = 'test';
console.log(window.globalVar); // 'test'
```

2. let -> window 객체에 추가 안됨

```js
let globalLet = 'test';
console.log(window.globalLet); // undefined
```

\*\*\* `var`는 javascript 전역 객체 오염이 가능

### 3. 메모리/가비지 컬렉션 관점

- `let`은 블록 스코프라 블록을 벗어나면 바로 가비지 컬렉션 대상이 됨
- `var`는 함수 전체에서 살아있어서 불필요하게 메모리를 더 오래 차지할 수 있음

```js
function process() {
  // var는 함수 끝까지 메모리에 존재
  for (var i = 0; i < 1000000; i++) {
    var temp = heavyObject();
  }
  // temp가 여기서도 접근 가능 (메모리 낭비)

  // let은 블록 끝나면 즉시 해제
  for (let j = 0; j < 1000000; j++) {
    let temp2 = heavyObject();
  }
  // temp2는 여기서 접근 불가 (이미 해제됨)
}
```

### 결론

`let` 사용 권장되는 이유는

1. 예측 가능한 스코프 (블록 단위)
2. 버그 방지 (재선언 불가, TDZ)
3. 더 나은 메모리 관리
4. 전역 오염 방지

현대 Javascript에서는 `var`를 사용할 이유가 없다.
`let`, `const`의 사용이 베스트이다.
