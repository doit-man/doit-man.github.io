---
layout: post
title: "OCaml LLVM 바인딩 만들기"
date: 2024-11-16 10:45:13
category: "Tech"
---

## OCaml LLVM 바인딩
LLVM은 기본적으로 LLVM IR을 다른 C/C++이 아닌 다른 단어로 다루기 위한 바인딩(Bingdings)를 기본적으로 제공한다.
예전에는 Python와 같은 다른 언어들도 지원했지만 현재는 OCaml만 지원하고 있다.
OCaml을 이용해서 LLVM IR을 다루다 보면 LLVM이 기본적으로 제공하는 바인딩이 너무 부족한 경우가 많다.
이 경우에는 바인딩 함수들을 조합해서 기능을 만들어도 되지만 바인딩을 직접 추가하는게 훨씬 깔끔하고 편한 경우가 있다. 기본적으로 C/C++ 로 강력하고 다양한 API들을 제공하기 때문에 불필요한 구현이 필요없으며, OCaml만 작성하다가 C++을 짧지만 사용하는 짜릿함을 느낄 수 있다. 내가 구현한 OCaml LLVM 바인딩은 [여기](https://github.com/doit-man/ocaml-llvm-binding)서 확인할 수 있다.

## OCaml LLVM 바인딩 이해하기  
LLVM이 제공하는 기본 바인딩은 [LLVM Binding](https://github.com/llvm/llvm-project/tree/main/llvm/bindings) 에서 볼 수 있다. 이 중 우리가 바인딩을 추가하기 위해서 주의깊게 봐야 하는 파일은 [llvm_ocaml.c](https://github.com/llvm/llvm-project/blob/main/llvm/bindings/ocaml/llvm/llvm_ocaml.c) 파일이다. 다행히 코드가 굉장히 짧고 쉽다. 예를 들어 LLVM IR 프로그램에 [Add instruction](https://llvm.org/docs/LangRef.html#add-instruction)을 추가하는 바인딩은 아래와 같다.

```c
value llvm_const_add(value LHS, value RHS) {
  LLVMValueRef Value = LLVMConstAdd(Value_val(LHS), Value_val(RHS));
  return to_val(Value);
}
```
이는 OCaml 상에서 Add 할 두 값을 넣어주면 두 값을 더하는 instruction을 반환해주는 함수이다. Binding을 위해서는 단순히 LLVM에 있는 `LLVMConstADD`라는 API를 호출해주고, 해당 값을 OCaml에 존재하는 타입과 LLVM 라이브러리의 타입으로 오고가기 위한 `LLVMValueRef`, `Value_val`, `to_val` 으로 각 값을 감싸주면 구현이 간단하다. 이러한 API들은 [LLVM CORE](https://llvm.org/doxygen/group__LLVMCCore.html) 에 많이 존재하며, 이 중 대부분은 이미 OCaml으로 바인딩 되어 있다.

## OCaml LLVM 바인딩 만들어보기
우리는 LLVM이 이미 구현한 코드를 참고해서 우리만의 바인딩을 만들 수 있다. 기억해야 할 것은, OCaml 상에서 어떤 변수를 인자로 넣을 것인가, 어떤 api를 호출할 것인가, api 호출 결과를 어떻게 가져올 것인가, 이 3가지만 기억하면 된다.

### 예시 1
LLVM IR 상에는 아래와 같이 `nsw` 라는 플래그가 존재한다
```llvm
<result> = add nsw <ty> <op1>, <op2>      ; yields ty:result
``` 
이는 `<op1>` + `<op2>` 값이 signed overflow가 발생하면, instruction의 결과가 `poison`이 된다는 것을 말하는 flag이다. LLVM IR을 다루다 보면, 해당 instruction 이 `nsw` flag를 갖고 있는지 여부를 boolean 값으로 알고 싶은 경우가 있다. 따라서 이러한 바인딩을 추가해보자. 바인딩 코드는 아래와 같다.
```cpp
  value llvm_is_nuw(value instr)
  {
    CAMLparam1(instr);
    LLVMValueRef value_ref = Value_val(instr);
    Value *v = unwrap<Value>(value_ref);
    bool ret = cast<Instruction>(v)->hasNoUnsignedWrap();
    CAMLreturn(Val_bool(ret));
  }
```
우리가 원하는 것은 instruction을 입력해서 해당 instruction이 `nsw` flag를 가지고 있는지를 boolean으로 가져오는 것이다 그렇기에 instr을 입력 받아서 CAMLparam1 으로 인자를 처리해주고, LLVM api에 존재하는 값으로 타입을 바꾸기 위해서 ` LLVMValueRef value_ref = Value_val(instr); Value *v = unwrap<Value>(value_ref);` 으로 바꾸어 준다. 그 후는 LLVM에 이미 존재하는 api인 `hasNoUnsignedWrap()`을 호출해준다. 그 후 그 값을 다시 OCaml 상의 타입으로 반환해주면 끝난다.

### 예시 2
가끔은 api 함수 하나로만 구현할 수 없는 경우도 존재한다. 이럴 경우에도 그냥 cpp 상에서 여러 api 함수를 엮어서 바인딩을 구현하고, 결과만 받아온다면 OCaml 상에서는 굉장히 편하게 기능을 쓸 수 있다. (OCaml의 코드가 깔끔해진다.)

아래는 LLVM IR 상의 함수를 return type만 다른 함수로 그대로 복사하는 기능을 구현한 바인딩이다.
```cpp 
value llvm_transforms_utils_clone_function_with_retty(value F, value RetTy) {
    CAMLparam2(F, RetTy);
    Function *OldFunc = unwrap<Function>(Value_val(F));
    Type *RT = unwrap<Type>(Type_val(RetTy));

    std::vector<Type *> ArgTypes = OldFunc->getFunctionType()->params();
    bool IsVarArg = OldFunc->getFunctionType()->isVarArg();
    FunctionType *NewFTy = FunctionType::get(RT, ArgTypes, IsVarArg);

    Module *M = OldFunc->getParent();
    Function *NewFunc =
        Function::Create(NewFTy, OldFunc->getLinkage(), OldFunc->getName(), M);

    ValueToValueMapTy VMap;
    Function::arg_iterator DestI = NewFunc->arg_begin();
    for (const Argument &I : OldFunc->args()) {
      DestI->setName(I.getName());
      VMap[&I] = &*DestI++;
    }

    SmallVector<ReturnInst *, 8> Returns;
    CloneFunctionInto(NewFunc, OldFunc, VMap,
                      CloneFunctionChangeType::LocalChangesOnly, Returns, "",
                      nullptr);
    CAMLreturn(to_val(NewFunc));
  }
```
이 경우에도 복사할 함수와, 복사된 함수의 return type만을 입력받고, LLVM api인 `Function::Create`, `CloneFunctionChangeType` 새로운 함수로 복사해줄 수 있다. 이러한 api들은 [LLVM 문서](https://llvm.org/doxygen/Cloning_8h.html)를 읽어보면 쉽게 찾을 수 있다. 

## LLVM 바인딩을 OCaml 프로젝트에 추가하기
바인딩을 구현 했으면, OCaml 프로젝트에 추가하면 완료된다.

프로젝트의 dune에 아래와 같이 추가하자
```
 (foreign_stubs
  (language cxx)
  (flags
   :standard
   (-I/usr/lib/llvm-16/include
    -D_GNU_SOURCE
    -D__STDC_CONSTANT_MACROS
    -D__STDC_FORMAT_MACROS
    -D__STDC_LIMIT_MACROS
    -std=c++17)))
 (c_library_flags (-lLLVM-16) (-L/usr/lib/llvm-16/lib) -lstdc++)
 ```
여기서 사용한 llvm 버전만 사용자 환경에 맞추어 변경한다면 이제 프로젝트를 빌드할 때 우리가 작성한 바인딩 cpp 파일도 빌드가 될 것이다.
 
이제, 우리가 만든 바인딩 함수를 가져다 써보자
```ocaml
external is_nsw_raw : llvalue -> bool = "llvm_is_nsw"

(** [clone_function_with_fnty func fn_ty] returns
      a copy of [func] of function type [fn_ty] and add it to [func]'s module *)
external clone_function_with_fnty : llvalue -> lltype -> llvalue
  = "llvm_transforms_utils_clone_function_with_fnty"

```
각 바인딩 함수의 타입을 지정해주고, cpp로 작성한 바인딩 함수는 `external` 로 지정해준다면 이제 `is_nsw_raw`,  `clone_function_with_fnty` 와 같이 OCaml 프로젝트에서 우리가 직접 작성한 바인딩 함수를 사용할 수 있다.

내가 작성한 OCaml LLVM 바인딩은 [여기](https://github.com/doit-man/ocaml-llvm-binding)서 볼 수 있다.