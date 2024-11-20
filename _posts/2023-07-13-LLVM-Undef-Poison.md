---
layout: post
title: "LLVM IR Undef & Poison"
date: 2023-07-13
category: "Tech"
---
## LLVM IR
LLVM은 계층적 구조를 가지고 있다. C/C++ 와 같은 고수준 언어를 입력으로 받아서 중간 표현(Intermediate Representation)으로 번역하고, LLVM IR 상에서 코드 최적화를 진행한 후, 다시 각 아키텍처의 기계어로 번역하는 과정을 거친다. 이렇게 LLVM IR은 여러 언어와 아키텍처를 통합해서 다뤄야 하기 때문에 여러 모호한 개념이 존재하며 이 중 대표적인 것은 `undef`, `poison`이다.

## Undef 이해하기
아래와 같은 C 프로그램을 생각해보자.

```c
int foo (int x){
  return x + 1;
}

int main(){
  int temp;
  return foo(temp);
}
```
`foo`의 인자로 `temp`를 넣었을 때, temp는 흔히 초기화되지 않은 변수, 쓰레기 값이 들어간다고 표현한다. 실제로 `temp`안에 어떤 값이 있는지는 정의되지 않았기 때문이다.

이럴 경우 LLVM IR은 이런 값을 `undef`로 표현할 수 있다.
그렇다면 위의 프로그램은 LLVM IR로 아래와 같이 표현될 수 있을 것이다.
```llvm
define i32 @main() {
  ret i32 undef
}
```
반환되는 값은 아무 값이나 될 수 있다. 
이 때, undef가 표현하는 값은 `- INT32_Max ... -1, 0 ,1 ... INT32_MAX` 와 같이 집합으로 나타낼 수 있다. 이 때, undef가 아무리 정의되지 않은 값이라고 하더라도, 그 값이 분명히 표현할 수 있는 범위는 존재하기 때문에, 이 값이 가질 수 있는 범위가 정확히 정해지는 것이 중요하다. 예를 들면 아래와 같은 프로그램들을 보자

```llvm
define i32 @src() {
  %even = mul i32 undef, 2
  ret i32 %even ; ... -2, 0, 2, ....
}

define i32 @tgt() {
  ret i32 undef ; ... -1, 0, 1, ...
}
```
두 프로그램 모두 정의되지 않은 값을 반환하지만, `tgt`는 모든 i32 값을 다 반환할 수 있지만, `src`는 오직 짝수만 반환할 수 있다. 이렇게 같은 `undef` 끼리도 값이 다를 수 있기 때문에, 만약 `src` 프로그램을 `tgt` 프로그램으로 번역했다면 이는 잘못된 번역 이다. 이처럼 LLVM은 최적화 과정에서 이를 잘 지키는 것이 중요하다.

## Poison 이해하기 
`poison` 은 LLVM IR 상에서 프로그램을 다룰 때, `undef`로 표현하기에는 좀 더 중요한 상황을 처리하고 싶을 때 사용한다. 정확히는, 어떠한 변수가 의도된 범위를 넘어갔을 때, 이 값이 잘못된 값이라는 것을 표현하고 싶지만, `undef`로 표현하면 초기화되지 않은 값과 동일한 의미를 가지기 때문에 특별한 표현을 쓴다.

아래와 같은 프로그램을 생각해보자
```c
int32_t i = 0;
while (i <= INT32_MAX){
  i++;
}
...
```
위 프로그램은 절대 끝나지 않을 것이다. 왜냐면 i가 `INT32_MAX`가 되는 순간, signed overflow가 나서 다시 루프를 돌 것이기 때문이다. 이런 경우를 방지하기 위해서 LLVM은 `i++`의 operation이 signed overflow가 나지 않도록 표시 한다. 
```llvm
  %i = add nsw %x, 1 ; poison when signed overflowed
```
`nsw (no signed overflow)`가 붙은 연산은 연산 결과가 signed overflow가 난 경우 `poison`값을 반환한다. `poison`역시 값을 읽으면 아무 값으로나 읽힐 수 있다.

## Undef 와 Poison의 차이점
`Undef` 의 값은 일반적인 수학적인 연산이 들어맞지 않는다.
```llvm
%y.1 = mul i32 undef, 2
%y.2 = add i32 undef, undef
```
`%y.1`와 `%y.2`는 얼핏 보면 같은 값으로 보인다. 하지만 `%y.1`은 항상 짝수지만 `%y.2`는 첫 `undef` 가 1이고, 두 번째 `undef`가 2인 경우 값이 3으로 홀 수도 될 수 있기 때문에 두 값은 같지 않다. 따라서 `X * 2 -> X + X` 라는 최적화는 X가 undef가 아니라는 가정하에만 올바른 최적화이다.

`Poison` 의 경우는 `poison` 값이 전파된다. 따라서 `poison * 2` 역시 `poison`이고 `poison + poison` 역시 `poison`이기 때문에 `X * 2 -> X + X`의 최적화가 `poison`에 대해서 올바르다

이렇게 `poison`이 `undef`보다 더 강력한 표현이기 때문에 최적화가 올바른지 따질 때에도 두 관계를 잘 따져봐야 한다.

```llvm
%y.3 = add i32 undef, poison ; poison
%y.4 = select i1 true, i32 undef, i32 poison ; undef
```

위와 같을 때, `%y.3`은 `poison` 이고, `%y.4` 는 `undef` 값을 가지게 된다. 하지만 아래와 같이 LLVM이 최적화 한다고 해보자. 이 때, `poison`을 `undef`로 바꾸는 것은 괜찮지만 `undef`를 `poison`으로 바꾸는 것은 안된다. `poison`이 더 강력한 표현이기 때문이다. 

```llvm
%y.3 = undef
%y.4 = poison
```

이렇게 LLVM에서 `undef`와 `poison`은 매우 복잡한 semantic을 가지고 있기에, LLVM 개발자들도 이를 모두 고려해서 컴파일러 최적화를 작성하기가 어렵다, 특히 `undef`의 경우, 대부분의 최적화에 적용될 수 없기에 매 번 이를 고려하는 것이 매우 어렵다. 그렇기에 최근에는 `undef`라는 표현을 없애려는 논의가 계속해서 진행 중이다. 그에 따라 최근에는 LLVM의 최적화 버그도 `undef`로 인해 유발된 경우, 잘 패치를 안하려는 경향을 보이고 있다. 실제로, 우리가 제보한 [버그](https://github.com/llvm/llvm-project/issues/96130) 에 대해서도 `undef`로 인해 유발된 버그의 경우, `undef`를 없애려는 쪽으로 논의를 하는 것을 볼 수 있다. 