---
layout: post
title: "LLVM Optimization Passes"
date: 2023-08-24
category: "Tech"
---

## LLVM 미들엔드 최적화
최근, LLVM에 존재하는 여러 최적화를 잘 검사하고, 자동으로 테스팅하는 연구를 시작하면서 먼저 LLVM에 존재하는 최적화들을 통째로 코드를 읽고 이해해야 하는 상황이 되었다. 항상 오픈소스 프로그램을 빌드하다 보면, 컴파일에 문제가 생기면 optimization pass를 조절하는 `-o1, -o2, -o3 ...`과 같은 최적화 강도를 조정했었는데, 이를 기회로 실제로 어떠한 최적화들이 일어나는지 알 수 있게 되었다.

## LLVM 최적화 pass 종류

### DCE (Dead Code Elimination)
가장 유명한, 데드 코드를 삭제하는 최적화이다. 데드 코드는 컴파일된 프로그램의 크기도 불필요하게 늘리고, 실제 프로그램에 영향이 없는 쓸모없는 코드를 실행하게 함으로써 프로그램의 성능을 저하시킨다. 또한 보안 취약점에 의해서 악용될 수 있는 여지도 있기 때문에 컴파일러는 컴파일 과정에서 데드 코드를 지우려고 한다.
```c
int foo (int x){
  int temp = x + 1;
  return x + 2;
}
```
위와 같은 코드에서 temp라는 변수는 아무 곳에서도 사용되지 않기에 삭제해도 프로그램의 동작에 아무런 문제가 없다. 그렇기에 컴파일러는 이러한 코드들을 분석하고 삭제하려고 한다.
컴파일러는 dead code를 어떻게 알아낼까? 이는 `Post-dominator Relation` 과 `Control Dependence`를 이용하여, 변수들을 사용하는 지점마다 변수들을 정의한 지점을 Backward로 따라가고, 끝까지 정의되지 않은 지점들을 삭제하는 Worklist 알고리즘을 사용하여 삭제하게 된다.

LLVM에는 DCE와 비슷한 `Aggressive Dead Code Elimination`, `Dead Argument Elimination`, `Dead Store Elimination`,`Dead Global Elimination` 와 같은 여러 최적화 pass를 가지고 있다.

### InstCombine
LLVM에서 가장 큰 [최적화 pass](https://github.com/llvm/llvm-project/tree/main/llvm/lib/Transforms/InstCombine)이다. 여러 instruction에 걸친 동작을 분석하고 하나로 결합함으로써 실질적으로 instruction의 개수를 줄여서 프로그램을 최적화 한다. 우리가 최적화라고 하면 가장 흔히 떠올리는 `X + 0`을 `X`로 최적화 하거나, `X * 2^N`을 `X << N` 으로 최적화 하는 등의 최적화가 이 pass에 해당된다. 그렇기 때문에 가장 많은 최적화를 추가되고 있고, 가장 큰 최적화 pass에 의해 여러가지 유지 보수 문제가 존재하고 있다.  

동작 과정은 다음과 같다. 기본적으로 worklist 알고리즘을 사용한다. 

1. 프로그램의 모든 instruction을 worklist에 추가한다.
2. worklist에서 한 instruction을 꺼내서, operator에 해당하는 최적화를 적용할 수 있는지 검사한다.
3. 최적화를 적용할 수 있다면 최적화를 적용하고, 최적화가 적용된 instruction을 추가로 worklist에 추가한다. 적용할 최적화가 없다면 그냥 넘어간다.
4. worklist에 instruction이 하나도 안 남을 때까지 2번으로 돌아간다.

### GVN (Global Value Numbering)
GVN도 많은 사람들이 아는 최적화일 것이다. 이는 basic block 단위에서 프로그램을 보아야 이 최적화가 무엇을 위한 것인지 알 수 있다.

**최적화 전 프로그램:**
<p align="center">
  <img width="460" height="200" src="/img/posts/before_gvn.png">
</p>

**최적화 후 프로그램:**
<p align="center">
  <img width="460" height="200" src="/img/posts/after_gvn.png">
</p>

최적화 전 프로그램을 보면 `bb1`, `bb2` 두 basic block 모두에 `%r = %y + 1;` 이라는 instruction이 똑같이 존재하는 것을 알 수 있다.
그렇다면 이 instruction을 굳이 두 basic block에 중복으로 아끼는 것보다, 공통된 `Entry` basic block으로 넘겨 주어서 instruction 개수를 줄이고, 더 많은 최적화를 일으킬 수 있도록 코드를 정리할 수 있다.
LLVM의 GVN 구현은 [링크](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/GVN.cpp)에서 확인할 수 있다. 

### SLP Vectorization (a.k.a. superword-level parallelism)
LLVM에는 [Loop vectorization](https://llvm.org/docs/Vectorizers.html#loop-vectorizer) 과 [SLP Vectorization](https://llvm.org/docs/Vectorizers.html#slp-vectorizer)이 존재한다. 이 중 더 활발하게 업데이트되고 버그도 많은 SLP-Vecorization만 소개하고자 한다.  [vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization)은 병렬 컴퓨팅에서 컴퓨팅 시간을 단축하기 위해서 중요하게 사용된다. SLP Vectorization은 여러 독립적인 instruction을 하나의 벡터로 묶는 최적화이다. 서로 독립적인 instruction을 병렬적으로 돌려도 프로그램 행동에 영향이 없을 때, 이를 vectoriztion을 한다. 추가로, vector로 들어가는 instruction이 비슷할 수록 더 효과가 좋기 때문에 비슷한 instruction을 탐색하는 과정도 구현되어 있다. 여담으로, LLVM의 SLP-Vectorization은 [구현](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Vectorize/SLPVectorizer.cpp)은 너무 더럽고 파일이 너무 커서 (21,000줄이 넘는다.), 언젠가는 모듈화가 잘 됐으면 하는 바람이 있다.

SLP 추출 알고리즘은 역시 함수에서 Basic Block을 순회하면서 동작하게 된다. 이 때의 직관은 인접한 메모리 주소를 참조하는 instruction은 vectorization을 하기에 적합하다는 직관을 가지고 동작하게 된다. 따라서 Basic Block을 순회하면서 메모리 참조하는 instruction이 두 개 있다면 주소가 인접한지 체크하고, 그들을 결합하여 vectorization을 하게 된다.

<p align="center">
  <img width="460" height="200" src="/img/posts/slp-algo.png">
</p>

### Correlated Value Propagation
이 최적화 pass는 CFG에서 가져올 수 있는 정보를 전파함으로써 코드를 최적화하기 위한 pass이다.
이 패스는 LLVM의 `DominatorTree`를 만드는 패스에 영향을 받는다.
아래와 같은 코드를 생각해보자.
```c
  int main() {
    int x = 1;
    int y = x + 1;
    int z = x + 4;
    return y + z;
  }
```
이 상황에서는 x는 1일 수밖에 없기 때문에 위의 코드에서 x라는 변수가 굳이 쓰일 필요가 없다. 그렇기에 아래와 같이 바꿀 수 있을 것이다. 
```c
  int main() {
    int y = 1 + 1;
    int z = 1 + 4;
    return y + z;
  }
```
위와 같은 코드는 메모리를 덜 쓸 수 있으며, 우리가 변하지 않는 값을 미리 계산해 넣어주었기에 후에 또 다른 최적화가 쉽게 일어날 수 있도록 도울 수 있다. 예를 들어, 이후에 `InstSimplify` 최적화가 일어나면 아래처럼 될 것이다.
```c
  int main() {
    return 7;
  }
```
이 처럼 컴파일러의 각 최적화 pass는 그 자체로도 최적화를 수행하지만, 다른 추후에 일어날 최적화를 위하여 미리 canonicalize를 하는 등 여러 동작을 수행하기도 한다.

### 다른 최적화들
LLVM는 20개가 넘는 최적화 pass가 존재하며, 위에 소개한 4개의 최적화 말고도 `VectorCombine`, `loop-idiom`, `indvars` 와 같은 여러 최적화들이 있다. 하지만 LLVM이 이들을 명확히 문서화를 안하기 때문에 (요즘 잘 업데이트가 안되는 것 같다.)
직접 구현코드를 읽으면서 최적화를 이해할 수밖에 없다. LLVM의 최적화 구현은 주로 [여기](https://github.com/llvm/llvm-project/tree/main/llvm/lib/Transforms)에 존재하며 읽으면 LLVM이 어떻게 동작하기를 원하고 개발되는지를 이해하는 데에 도움이 된다.