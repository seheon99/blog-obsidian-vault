---
title: JavaScript의 Iteration Protocol
topics:
  - JavaScript
description: JavaScript의 통일된 순회 가능한 데이터
---
## Iteration Protocol
- ES6에 등장한 순회 가능한 데이터 컬렉션을 만드는 규칙
- Iterable Protocol과 Iterator Protocol이 있음
- ES6 이전의 배열, 문자열, 유사 배열 객체 (Array-like Object), DOM 컬렉션 등을 통합하는 규칙
- ES6의 순회 가능한 데이터 컬렉션은 `for ... of`, spread, destructuring 할당의 대상이 될 수 있음
	```javascript
	const iterable = [1, 2, 3, 4, 5];
	for n of iterable {
		console.log(n);
	}
	const new_iterable = [0, ...iterable];
	const [n, ...others] = iterable;
```
### Iterable Protocol
- `Symbol.iterator` 을 프로퍼티 키로 갖는 메서드가 존재하는 객체
- 해당 메서드는 iterator protocol을 준수하는 객체를 반환해야 한다!
### Iterator Protocol
- `{ value, done }` 를 반환하는 `next` 메서드를 가짐
	- `value` 는 순회 중인 이터러블의 값
	- `done` 은 이터러블의 순회 완료 여부를 나타내는 값
## 빌드인 이터러블
- `Array`, `String`, `Map`, `Set`, `TypedArray`, `arguments`, DOM 컬렉션은 모두 이터러블
	- `for ... of`, spread, destructuring 할당 가능하다!
## 유사 배열 객체
- 배열처럼 인덱스로 프로퍼티 값에 접근할 수 있고 `length` 프로퍼티를 갖는 객체
- `Symbol.iterable` 메서드가 없다면 `for ... of` 로 접근 불가능
- `Array.from` 을 통해 이터러블 객체로 변환 가능
## 지연 평가
- 데이터가 필요한 시점 이전까지는 데이터를 생성하지 않다가 필요한 시점에 생성하는 기법
- 이터러블을 소비하는 대부분의 방법은 `next` 가 평가되기 전까지는 데이터를 생성하지 않는 지연 평가 사용
	- `for ... of`, destructuring 할당 등
## 커스텀 이터러블 구현
```javascript
const fibonacci = {
	[Symbol.iterator]: () => {
		let [pre, cur] = [0, 1];
		return {
			next: () => {
				[pre, cur] = [cur, pre + cur];
				return {
					value: cur,
					done: false,
				};
			},
		};
	},
};

const [n1, n2, n3] = fibonacci;
console.log(n1, n2, n3); // 1 2 3
```