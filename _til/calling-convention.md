---
layout: post
title: "함수 호출 규약(Calling Convention)"
date: 2026-09-14
---

## 레지스터

들어가기에 앞서 레지스터에 대한 설명 없이 구조를 이해할 수가 없다고 생각해 자주 등장하는 범용 레지스터와 명령어 포인터에 대해서만 x86-64 기준으로 간략히 설명하겠다. 내가 아는 수준으로 예제 어셈블리 코드를 이해하는 데 문제없는 정도로만 다루겠다.

명령어 포인터 `rip`는 다음 실행할 코드의 주소를 담고 있다.

범용 레지스터는 `rax`, `rbx`, `rcx`, `rdx`, `rbp`, `rsp`, `rdi`, `rsi`, `r8`~`r15` 총 16개가 있다.

- `rax`는 반환값을 저장하는 데 쓰인다.
- `rbp`는 스택 프레임의 베이스 포인터, 밑바닥이 되는 주소를 담는다.
- `rsp`는 스택의 top의 주소를 담는다. (push, pop, top으로 다루던 그 스택과 같다.)
- `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`는 매개변수를 담는 데 사용된다.
- `rbx`, `r10`~`r15`는 이 글의 예제에 등장하지 않으니 아래 표로 대신한다.

그리고 어셈블리 코드를 보면 `rax`, `eax` 이렇게 r로 시작하는 게 있고 e로 시작하는 게 있어서 종류가 되게 많아 보이는데, 이 둘은 물리적 레지스터는 하나고 접근 폭만 다르게 한 것이다. 16비트에서 `ax`였던 것이 32비트에서 `eax`로 확장되고 64비트에서 `rax`로 확장된 것이다. `ax`를 상위 8비트는 `ah`, 하위 8비트는 `al`로 세분화할 수도 있다. 32비트 레지스터를 쓴 것은 입력값의 크기가 크지 않은 경우 상위 32비트가 0으로 초기화되고 결과는 같은데 인코딩이 짧아 컴파일러가 선호하기 때문이다. 범용 레지스터뿐 아니라 명령어 포인터인 `rip`도 32비트에서는 `eip`로 쓰인다.

레지스터에 대한 설명이 너무 미흡한 것 같으면 아래 표를 참조하기 바란다. 개인적으로는 이 표보다는 내 설명이 낫다고 본다.

| 레지스터 | 유래 | 명령어가 암묵적으로 쓰는 경우 | ABI 역할 (SysV) |
|---|---|---|---|
| rax | Accumulator | mul/div(rdx:rax), cpuid, syscall 번호·반환값 | 반환값. 가변인자 호출 시 al = 사용한 xmm 개수 |
| rbx | Base | cpuid 출력 | callee-saved |
| rcx | Counter | loop, `rep movs/stos` 반복 횟수, shift 횟수(cl) | 4번째 인자 |
| rdx | Data | mul 결과 상위, div 피제수 상위·나머지, in/out 포트 번호 | 3번째 인자 |
| rsi | Source Index | 문자열 명령(movs lods cmps)의 원본 주소 | 2번째 인자 |
| rdi | Destination Index | 문자열 명령(movs stos scas)의 대상 주소 | 1번째 인자 |
| rbp | Base Pointer | 없음 | 프레임 포인터, callee-saved. `-fomit-frame-pointer`면 범용으로 씀 |
| rsp | Stack Pointer | push/pop/call/ret가 자동 증감 | 스택 top. 다른 용도 불가 |
| r8, r9 | 없음 (x86-64 추가) | 없음 | 5, 6번째 인자 |
| r10 | 없음 (x86-64 추가) | 없음 | syscall의 4번째 인자(rcx 대신), 중첩 함수 static chain |
| r11 | 없음 (x86-64 추가) | syscall이 rflags 보관 | 임시(caller-saved) |
| r12~r15 | 없음 (x86-64 추가) | 없음 | callee-saved |

**레지스터를 다른 폭으로 부르는 이름**

| 64 | 32 | 16 | 8 (high) | 8 (low) |
|---|---|---|---|---|
| rax rbx rcx rdx | eax ebx ecx edx | ax bx cx dx | ah bh ch dh | al bl cl dl |
| rsi rdi rbp rsp | esi edi ebp esp | si di bp sp | 없음 | sil dil bpl spl (64비트 모드 전용) |
| r8~r15 | r8d~r15d | r8w~r15w | 없음 | r8b~r15b |

이제 레지스터에 대해서도 알았고 본격적으로 함수 호출 규약에 대한 설명을 시작해보겠다.

## 함수 호출 규약이란

### 정의

함수 호출 규약은 함수를 호출하고 반환하는 방법에 대한 약속이다. 구체적으로는 다음을 정한다.

- 인자를 어떤 순서로, 레지스터와 스택 중 어디에 넣어 전달하는가
- 반환값을 어디에(보통 특정 레지스터) 담아 돌려주는가
- 인자 전달에 쓴 스택 공간을 호출자(caller)와 피호출자(callee) 중 누가 정리하는가
- 어떤 레지스터를 피호출자가 반드시 보존해야 하고, 어떤 레지스터는 호출자가 필요하면 스스로 지켜야 하는가
- 호출 시점의 스택 정렬 조건

### 필요한 이유

한 함수가 다른 함수를 호출하면 실행 흐름은 피호출자로 넘어가고, 피호출자가 반환하면 다시 호출자로 돌아와 원래 흐름을 이어 간다. 이것이 가능하려면 호출 시점에 두 가지가 보존되어야 한다.

- 돌아올 위치: 반환 주소(return address)
- 피호출자가 반환한 뒤에도 호출자가 계속 쓸 값: 레지스터 값과 스택 프레임

이 보존 책임을 누가 지는지도 규약의 일부이다. x86/x86-64에서 `call` 명령은 반환 주소를 스택에 push하고, 피호출자는 보존해야 하는 레지스터(RBP 등) 중 자신이 쓸 것의 기존 값을 스택에 저장했다가 반환 직전에 복원한다. 반면 규약이 피호출자가 마음껏 써도 상관없다고 정한 레지스터(인자 레지스터 RDI, RSI 등, 반환값 레지스터 RAX, 그리고 R10, R11)는 호출 과정에서 값이 바뀔 수 있다. 따라서 해당 레지스터의 값들이 호출 후에도 필요하다면 호출자가 호출 전에 직접 저장해야 한다.

또한 호출자는 피호출자가 요구하는 인자를 약속된 위치에 넣어 주어야 하고, 피호출자는 실행이 끝날 때 반환값을 약속된 위치에 넣어 돌려주어야 한다. 두 함수가 따로 컴파일되었더라도 같은 규약을 따르면 올바르게 연결된다.

## 함수 호출 규약의 종류

### 아키텍처에 따른 차이

기본 규약은 아키텍처가 제공하는 레지스터 개수에 크게 영향을 받는다.

- **x86(32비트)**: 범용 레지스터가 8개뿐이라 인자를 레지스터로 넘길 여유가 적다. 그래서 인자를 주로 스택으로 전달한다. (fastcall처럼 처음 몇 개만 레지스터로 넘기는 규약도 있다.)
- **x86-64**: 범용 레지스터가 16개로 늘어나, 처음 몇 개의 인자는 레지스터로 전달하고 그보다 많은 인자만 스택으로 전달한다.

### 운영체제·컴파일러에 따른 차이

아키텍처가 같아도 규약은 다를 수 있다. 같은 x86-64에서 Windows는 Microsoft x64 규약을, Linux·macOS 등 Unix 계열은 System V AMD64 ABI를 사용한다. 이 차이를 결정하는 것은 컴파일러가 아니라 대상 운영체제의 ABI다. MSVC는 Windows를 대상으로 하므로 MS x64를 따르고, Linux용 gcc는 System V를 따르며, Windows용 gcc(MinGW-w64)는 MS x64를 따른다.

※ ABI: 컴파일된 바이너리끼리 맞물리기 위한 약속. 소스 코드 수준의 API에 대응하는 바이너리 수준의 규격이라고 생각하면 된다.

※ 같은 이름의 규약을 컴파일러마다 다르게 구현하기도 한다. x86의 fastcall이 대표적인데, MSVC는 처음 두 인자를 ECX, EDX로 전달하지만 Borland 계열 컴파일러는 EAX, EDX, ECX를 사용한다.

### 주요 함수 호출 규약

**x86**

- cdecl
- stdcall
- fastcall
- thiscall

**x86-64**

- System V AMD64 ABI
- Microsoft x64

예시 코드를 작성하고 이를 어셈블리어로 컴파일해 동작을 보며 각각의 함수 호출 규약에 대해 알아보자.

## x86

### cdecl

**cdecl.c**

```c
int __attribute__((cdecl)) callee(int a1, int a2, int a3, int a4, int a5, int a6, int a7, int a8){
  int localCallee = 100;
  int result = a1+a2+a3+a4+a5+a6+a7+a8;
  return result;
}

void caller(){
  int localCaller1 = 99;
  int localCaller2 = 999;
  int result = callee(1, 2, 3, 4, 5, 6, 7, 8);
}
```

gcc 명령어는 클로드 형님이 뽑아주셨다.

```sh
gcc -m32 -O0 -fno-pic -fno-omit-frame-pointer -fno-asynchronous-unwind-tables -masm=intel -S cdecl.c -o cdecl.S
```

각 옵션에 대해서는 다음 의미가 있다고 한다. (이후부터 옵션에 대한 설명은 생략하겠다.)

- `-m32`: 32비트 x86 타깃. 이것만으로 C 함수는 cdecl이 된다.
- `-O0`: 최적화 끔. 지역변수가 레지스터로 올라가지 않고 스택에 남아 프레임 구조가 보인다.
- `-fno-pic`, `-fno-asynchronous-unwind-tables`: PIC용 thunk 호출과 `.cfi` 지시어를 빼서 출력을 읽기 쉽게 한다.
- `-fno-omit-frame-pointer`: EBP 기반 프레임 유지.

**caller**

```nasm
caller:
    push    ebp                     ; caller의 호출자 EBP 보존
    mov     ebp, esp                ; 새 프레임의 기준점
    sub     esp, 16                 ; 지역변수 공간 (12바이트를 16으로 올림)
    mov     DWORD PTR [ebp-12], 99  ; localCaller1
    mov     DWORD PTR [ebp-8], 999  ; localCaller2
    push    8                       ; 인자를 오른쪽(a8)부터 push
    push    7
    push    6
    push    5
    push    4
    push    3
    push    2
    push    1                       ; a1이 마지막 → 가장 낮은 주소
    call    callee                  ; 반환 주소를 push하고 점프
    add     esp, 32                 ; 인자 8개 × 4바이트를 호출자가 정리 → cdecl의 표식
    mov     DWORD PTR [ebp-4], eax  ; 반환값(EAX)을 result에 저장
    nop
    leave                           ; mov esp, ebp / pop ebp
    ret                             ; pop eip
```

우선 호출자에 해당하는 caller를 보면 자신이 호출됐을 때(caller도 누군가에게 호출당할 거니까) 기존의 프레임 포인터 레지스터(스택 프레임의 밑바닥으로 `ebp`를 말한다.)를 자신이 끝날 때 복구시키기 위해 스택에 저장해두고, 새로운 프레임 포인터 레지스터를 만들기 위해 현재 스택의 top 위치로 `ebp`를 이동시킨다. `mov` 명령어의 동작은 뒤의 값을 앞에 저장한다.

그리고 `esp`에서 16바이트를 빼서 (메모리 구조에서 스택은 낮은 주소로 자란다고 한 것을 기억하자.) 스택 프레임에 지역변수들을 저장할 공간을 할당해준다. int 자료형이 4바이트 크기이므로 해당 간격으로 지역 변수들을 저장한다. 지역 변수들을 위한 공간을 16바이트만큼 할당했는데, i386 ABI가 `call` 시점의 스택 16바이트 정렬을 요구하기 때문이라고 한다.

※ padding의 크기도 제각각인데 이건 규약에 의한 것은 아니고 컴파일러에 따라, 옵션에 따라 달라지는 것으로 여기서 자세하게 설명하지는 않겠다.

그렇게 지역변수들을 스택에 저장하고 callee를 호출하기 전에 스택의 top 쪽으로 callee에게 넘겨줄 매개변수들을 뒤에서부터 push한다.

`call`을 하게 되면 반환 주소(`add esp, 32`)의 주소를 스택에 push하고 callee의 주소로 이동한다.

이후 callee가 끝난 뒤 `add esp, 32`로 실행 흐름이 돌아오고, `esp` 주소를 옮기며 callee를 호출하기 위해 할당한 공간을 정리하고 callee가 `eax`에 저장한 반환값을 스택에 저장한다.

저장한 result에 대해 다른 동작은 없었으므로 그대로 `leave` 명령어의 실행과 함께 스택 프레임을 비우고 'caller를 호출한 함수'의 스택 프레임으로 이동한다.

이후 `ret`으로 반환 주소로 `eip`를 옮겨 실행 흐름을 넘겨준다.

아래 스택의 상황을 그려뒀지만 스택의 상황이 머릿속에 그려지기를 바란다.

**callee**

```nasm
callee:
    push    ebp                     ; caller의 EBP 저장 — callee가 EBP를 쓸 것이므로
    mov     ebp, esp
    sub     esp, 16                 ; 지역변수 공간
    mov     DWORD PTR [ebp-8], 100  ; localCallee
    mov     edx, DWORD PTR [ebp+8]  ; a1 (EDX, EAX는 caller-saved라 저장 없이 그냥 씀)
    mov     eax, DWORD PTR [ebp+12] ; a2
    add     edx, eax
    ...                             ; a3=[ebp+16] … a7=[ebp+32] 같은 방식으로 누적
    mov     eax, DWORD PTR [ebp+36] ; a8
    add     eax, edx
    mov     DWORD PTR [ebp-4], eax  ; result
    mov     eax, DWORD PTR [ebp-4]  ; 반환값은 EAX에
    leave                           ; esp를 ebp로 되돌리고 caller의 EBP 복원
    ret                             ; 반환 주소 pop. 인자 32바이트는 아직 스택에 남아 있음
```

caller에서 설명한 것과 동일하게 callee를 호출한 caller의 프레임 포인터 레지스터(`ebp`)를 스택에 저장하고 `ebp`를 해당 주소로 옮긴 뒤 지역변수 공간을 할당하고 지역변수들을 저장한다. 그리고 caller가 전달한 매개변수들을 레지스터로 옮겨 담고 연산을 마친 뒤 `eax`에 저장하고, callee의 스택 프레임을 정리하고 caller의 스택 프레임으로 돌아간다(`leave`). 마지막으로 `ret`으로 실행 흐름까지 넘겨주면 끝이다.

**스택** (callee 호출 직후)

```text
높은 주소
  +--------------------+
  | result             |  [caller ebp-4]   caller의 프레임 (result는 아직 미정)
  | localCaller2 = 999 |  [caller ebp-8]
  | localCaller1 = 99  |  [caller ebp-12]
  | (padding)          |  [caller ebp-16]
  +--------------------+
  | 8   (a8)           |  [ebp+36]         가장 먼저 push
  | 7   (a7)           |  [ebp+32]
  | ...                |                   인자: caller가 push, caller가 정리
  | 2   (a2)           |  [ebp+12]
  | 1   (a1)           |  [ebp+8]          마지막 push
  +--------------------+
  | return address     |  [ebp+4]          call이 push, ret이 pop
  | saved ebp          |  [ebp]            <- EBP. push ebp가 저장, leave가 복원
  +--------------------+
  | result             |  [ebp-4]          callee의 지역변수
  | localCallee = 100  |  [ebp-8]
  | (padding)          |  [ebp-12]
  | (padding)          |  [ebp-16]         <- ESP
  +--------------------+
낮은 주소
```

callee가 호출된 직후의 스택의 모습은 이러하다.

cdecl은 매개변수를 전부 스택으로 오른쪽→왼쪽 순서로 전달하고 스택 정리를 호출자가 한다는 것이 특징이다. (caller의 `add esp, 32`가 스택 정리에 해당한다.)

### stdcall

**stdcall.c**

```c
int __attribute__((stdcall)) callee(int a1, int a2, int a3, int a4, int a5, int a6, int a7, int a8){
  int localCallee = 100;
  int result = a1+a2+a3+a4+a5+a6+a7+a8;
  return result;
}

void caller(){
  int localCaller1 = 99;
  int localCaller2 = 999;
  int result = callee(1, 2, 3, 4, 5, 6, 7, 8);
}
```

```sh
gcc -m32 -O0 -fno-pic -fno-omit-frame-pointer -fno-asynchronous-unwind-tables -masm=intel -S stdcall.c -o stdcall.S
```

**caller**

```nasm
caller:
    push    ebp
    mov     ebp, esp
    sub     esp, 16
    mov     DWORD PTR [ebp-12], 99  ; localCaller1
    mov     DWORD PTR [ebp-8], 999  ; localCaller2
    push    8                       ; 인자 전달은 cdecl과 동일: 전부 스택, 오른쪽부터
    push    7
    push    6
    push    5
    push    4
    push    3
    push    2
    push    1
    call    callee
    mov     DWORD PTR [ebp-4], eax  ; 반환값 저장. cdecl에 있던 add esp, 32 가 없음
    nop
    leave
    ret
```

stdcall의 경우 cdecl과 전부 동일하지만, 한 가지 차이는 cdecl에서는 호출자가 스택을 정리한 것과 다르게 피호출자가 스택을 정리하고 실행 흐름을 넘겨준다.

**callee**

```nasm
callee:
    push    ebp
    mov     ebp, esp
    sub     esp, 16
    mov     DWORD PTR [ebp-8], 100  ; localCallee
    mov     edx, DWORD PTR [ebp+8]  ; a1 — 인자 위치도 cdecl과 동일
    mov     eax, DWORD PTR [ebp+12] ; a2
    add     edx, eax
    ...                             ; a3=[ebp+16] … a7=[ebp+32]
    mov     eax, DWORD PTR [ebp+36] ; a8
    add     eax, edx
    mov     DWORD PTR [ebp-4], eax  ; result
    mov     eax, DWORD PTR [ebp-4]  ; 반환값은 EAX
    leave
    ret     32                      ; 반환 주소를 pop한 뒤 ESP += 32 → 인자를 피호출자가 정리
```

`ret` 과정에서 스택을 정리하는 것을 callee에서 확인할 수 있다.

### fastcall

fastcall은 이름부터 fast답게 인자 전달 시 레지스터를 사용해 속도를 향상시키기 위해 도입된 규약이다.

**fastcall.c**

```c
int __attribute__((fastcall)) callee(int a1, int a2, int a3, int a4, int a5, int a6, int a7, int a8){
  int localCallee = 100;
  int result = a1+a2+a3+a4+a5+a6+a7+a8;
  return result;
}

void caller(){
  int localCaller1 = 99;
  int localCaller2 = 999;
  int result = callee(1, 2, 3, 4, 5, 6, 7, 8);
}
```

```sh
gcc -m32 -O0 -fno-pic -fno-omit-frame-pointer -fno-asynchronous-unwind-tables -masm=intel -S fastcall.c -o fastcall.S
```

**caller**

```nasm
caller:
    push    ebp
    mov     ebp, esp
    sub     esp, 16
    mov     DWORD PTR [ebp-12], 99  ; localCaller1
    mov     DWORD PTR [ebp-8], 999  ; localCaller2
    push    8                       ; 3번째 이후 인자만 스택에, 오른쪽부터
    push    7
    push    6
    push    5
    push    4
    push    3
    mov     edx, 2                  ; a2는 EDX로
    mov     ecx, 1                  ; a1은 ECX로
    call    callee
    mov     DWORD PTR [ebp-4], eax  ; 반환값 저장. 스택 정리 코드 없음
    nop
    leave
    ret
```

fastcall은 처음 두 인자(4바이트 이하의 정수·포인터)를 ECX, EDX로 넘기고 나머지만 스택에 push한다. 그래서 caller의 push가 6개로 줄었고, 스택 정리는 stdcall처럼 피호출자가 하므로 `call` 뒤에 정리 코드가 없다.

**callee**

```nasm
callee:
    push    ebp
    mov     ebp, esp
    sub     esp, 24                 ; 지역변수 + 레지스터 인자를 내려놓을 공간
    mov     DWORD PTR [ebp-20], ecx ; a1을 스택에 내려놓음 (-O0라서)
    mov     DWORD PTR [ebp-24], edx ; a2
    mov     DWORD PTR [ebp-8], 100  ; localCallee
    mov     edx, DWORD PTR [ebp-20] ; a1
    mov     eax, DWORD PTR [ebp-24] ; a2
    add     edx, eax
    mov     eax, DWORD PTR [ebp+8]  ; a3 — 스택 인자는 여기서부터
    add     edx, eax
    ...                             ; a4=[ebp+12] … a7=[ebp+24]
    mov     eax, DWORD PTR [ebp+28] ; a8
    add     eax, edx
    mov     DWORD PTR [ebp-4], eax  ; result
    mov     eax, DWORD PTR [ebp-4]  ; 반환값은 EAX
    leave
    ret     24                      ; 스택 인자 6개 × 4바이트를 피호출자가 정리
```

callee는 ECX, EDX로 받은 a1, a2를 자기 프레임(`[ebp-20]`, `[ebp-24]`)에 내려놓은 뒤 다시 읽어서 쓴다. 이 부분은 컴파일 옵션의 `-O0` 때문이고, 최적화하면 이 부분은 사라지고 ECX, EDX 레지스터를 그대로 쓴다. 스택으로 온 인자는 a3부터 `[ebp+8]`에서 시작하고, `ret 24`는 스택 인자 6개분만 정리한다. 레지스터로 받은 인자는 정리할 것이 없다.

### thiscall

**thiscall.c**

```c
int __attribute__((thiscall)) callee(int a1, int a2, int a3, int a4, int a5, int a6, int a7, int a8){
  int localCallee = 100;
  int result = a1+a2+a3+a4+a5+a6+a7+a8;
  return result;
}

void caller(){
  int localCaller1 = 99;
  int localCaller2 = 999;
  int result = callee(1, 2, 3, 4, 5, 6, 7, 8);
}
```

```sh
gcc -m32 -O0 -fno-pic -fno-omit-frame-pointer -fno-asynchronous-unwind-tables -masm=intel -S thiscall.c -o thiscall.S
```

**caller**

```nasm
caller:
    push    ebp
    mov     ebp, esp
    sub     esp, 16
    mov     DWORD PTR [ebp-12], 99  ; localCaller1
    mov     DWORD PTR [ebp-8], 999  ; localCaller2
    push    8                       ; 2번째 이후 인자는 스택에, 오른쪽부터
    push    7
    push    6
    push    5
    push    4
    push    3
    push    2
    mov     ecx, 1                  ; 첫 인자만 ECX로 (C++이라면 this)
    call    callee
    mov     DWORD PTR [ebp-4], eax  ; 반환값 저장. 스택 정리 코드 없음
    nop
    leave
    ret
```

thiscall은 첫 인자만 ECX로 넘기고 나머지는 스택으로 넘기며, 정리는 피호출자가 한다. C++ 클래스 멤버 함수를 위한 규약으로, 첫 인자가 곧 `this`다. gcc는 확장으로 C 함수에도 이 속성을 허용하기 때문에 같은 예제로 볼 수 있다. 가변 인자 멤버 함수라면 전부 스택으로 넘기고 호출자가 정리한다(cdecl과 동일).

**callee**

```nasm
callee:
    push    ebp
    mov     ebp, esp
    sub     esp, 20
    mov     DWORD PTR [ebp-20], ecx ; a1을 스택에 내려놓음 (-O0라서)
    mov     DWORD PTR [ebp-8], 100  ; localCallee
    mov     edx, DWORD PTR [ebp-20] ; a1
    mov     eax, DWORD PTR [ebp+8]  ; a2 — 스택 인자는 여기서부터
    add     edx, eax
    ...                             ; a3=[ebp+12] … a7=[ebp+28]
    mov     eax, DWORD PTR [ebp+32] ; a8
    add     eax, edx
    mov     DWORD PTR [ebp-4], eax  ; result
    mov     eax, DWORD PTR [ebp-4]  ; 반환값은 EAX
    leave
    ret     28                      ; 스택 인자 7개 × 4바이트를 피호출자가 정리
```

fastcall에서 레지스터 인자가 하나로 줄어든 것 외에는 같다. 스택 인자는 a2부터 `[ebp+8]`에서 시작하고, `ret 28`로 7개분을 정리한다.

위 동작은 MSVC의 thiscall이다. Linux의 g++는 멤버 함수에 thiscall을 쓰지 않고, `this`를 첫 번째 스택 인자로 넘기며 호출자가 정리한다. 실제 멤버 함수를 g++로 컴파일해 확인해 보면 다음과 같다.

```cpp
struct S {
  int v;
  int add(int a, int b);
};

int S::add(int a, int b){ return v + a + b; }

int use(S* s){ return s->add(1, 2); }
```

```nasm
_ZN1S3addEii:                       ; S::add(int, int) — 맹글링된 이름
    push    ebp
    mov     ebp, esp
    mov     eax, DWORD PTR [ebp+8]  ; this — 첫 번째 스택 인자
    mov     edx, DWORD PTR [eax]    ; this->v
    mov     eax, DWORD PTR [ebp+12] ; a
    add     edx, eax
    mov     eax, DWORD PTR [ebp+16] ; b
    add     eax, edx
    pop     ebp
    ret                             ; 피연산자 없음 → 호출자가 정리

_Z3useP1S:
    push    ebp
    mov     ebp, esp
    push    2                       ; b
    push    1                       ; a
    push    DWORD PTR [ebp+8]       ; this를 마지막에 push → 첫 번째 인자 자리
    call    _ZN1S3addEii
    add     esp, 12                 ; 호출자가 12바이트 정리 → cdecl과 동일
    leave
    ret
```

※ 맹글링(Name Mangling): C++에서 오버로딩·네임스페이스·템플릿으로 같은 이름의 함수가 여럿 생기므로, 컴파일러가 소속과 매개변수 타입을 이름에 인코딩해 링커가 구별할 수 있는 고유한 심볼로 바꾸는 과정. `_ZN1S3addEii`는 "S::add(int, int)"를 뜻하며 `c++filt`로 되돌릴 수 있다.

이 차이 때문에 x86 Windows 바이너리를 분석할 때는 ECX에 `this`가 들어오는 것을 전제로 보고, Linux 바이너리에서는 `[ebp+8]`이 `this`라고 보면 된다.

## x86-64

### System V AMD64 ABI

- Linux, macOS, BSD 등 Unix 계열 운영체제가 쓰는 x86-64 규약이다.
- 정수·포인터 인자는 RDI, RSI, RDX, RCX, R8, R9 순서로 6개까지 레지스터로 전달하고, 실수 인자는 XMM0~XMM7로 8개까지 전달한다. 그보다 많은 인자는 스택에 오른쪽부터 push한다.
- 인자 레지스터 여섯 개(RDI, RSI, RDX, RCX, R8, R9)와 RAX, R10, R11은 호출된 함수가 마음대로 덮어써도 되는 레지스터다. 호출 뒤에도 이 레지스터의 값이 필요하다면 호출하는 쪽이 호출 전에 따로 저장해 두어야 한다. 반대로 RBX, RBP, R12~R15는 호출된 함수가 원래 값을 보존해야 하는 레지스터라, 쓰려면 먼저 저장해 두었다가 반환하기 전에 복원해야 한다.
- 인자 전달에 쓴 스택 공간은 호출자가 정리한다.
- `call` 시점에 RSP가 16바이트 정렬이어야 한다. caller에서 `sub rsp, 16` 뒤에 push 두 번(16바이트)을 하고 호출하므로 정렬이 유지된다. (정렬이 어긋날 상황이면 컴파일러가 `sub rsp, 8` 같은 패딩을 넣어 맞춘다.)
- 64비트를 넘는 정수 반환값은 RDX:RAX를, 실수 반환값은 XMM0를 쓴다.

**sysv.c**

```c
int callee(int a1, int a2, int a3, int a4, int a5, int a6, int a7, int a8){
  int localCallee = 100;
  int result = a1+a2+a3+a4+a5+a6+a7+a8;
  return result;
}

void caller(){
  int localCaller1 = 99;
  int localCaller2 = 999;
  int result = callee(1, 2, 3, 4, 5, 6, 7, 8);
}
```

x86-64 Linux에서는 이것이 기본 규약이라 속성 없이 `-m32`만 빼면 된다.

```sh
gcc -O0 -fno-pic -fno-omit-frame-pointer -fno-asynchronous-unwind-tables -masm=intel -S sysv.c -o sysv.S
```

**caller**

```nasm
caller:
    endbr64
    push    rbp                     ; 프레임 구조는 32비트와 같고 레지스터만 8바이트
    mov     rbp, rsp
    sub     rsp, 16                 ; 지역변수 공간 (12바이트를 16으로 올림)
    mov     DWORD PTR [rbp-12], 99  ; localCaller1
    mov     DWORD PTR [rbp-8], 999  ; localCaller2
    push    8                       ; 7번째 이후 인자만 스택에, 오른쪽부터. push는 8바이트 단위
    push    7
    mov     r9d, 6                  ; a6
    mov     r8d, 5                  ; a5
    mov     ecx, 4                  ; a4
    mov     edx, 3                  ; a3
    mov     esi, 2                  ; a2
    mov     edi, 1                  ; a1 — int라 64비트 레지스터의 하위 32비트만 씀
    call    callee                  ; 반환 주소(8바이트) push
    add     rsp, 16                 ; 스택 인자 2개 × 8바이트를 호출자가 정리
    mov     DWORD PTR [rbp-4], eax  ; 반환값(RAX의 하위 32비트)을 result에 저장
    nop
    leave
    ret
```

인자 8개 중 6개가 레지스터로 가고 a7, a8만 스택에 push된다. 스택 인자는 int라도 8바이트 칸을 차지하므로 정리도 `add rsp, 16`이다. 인자 레지스터는 64비트지만 값이 int라 `edi`, `esi`처럼 하위 32비트 이름으로 쓴다. `endbr64`는 Ubuntu gcc가 기본으로 켜는 `-fcf-protection` 때문에 붙는 것으로 호출 규약과 관계없고, `-fcf-protection=none`을 주면 사라진다.

**callee**

```nasm
callee:
    endbr64
    push    rbp
    mov     rbp, rsp                ; sub rsp 가 없음 — 아래 red zone 설명
    mov     DWORD PTR [rbp-20], edi ; 레지스터로 받은 a1~a6을 스택에 내려놓음 (-O0라서)
    mov     DWORD PTR [rbp-24], esi
    mov     DWORD PTR [rbp-28], edx
    mov     DWORD PTR [rbp-32], ecx
    mov     DWORD PTR [rbp-36], r8d
    mov     DWORD PTR [rbp-40], r9d
    mov     DWORD PTR [rbp-8], 100  ; localCallee
    mov     edx, DWORD PTR [rbp-20] ; a1
    mov     eax, DWORD PTR [rbp-24] ; a2
    add     edx, eax
    ...                             ; a3=[rbp-28] … a6=[rbp-40] 같은 방식으로 누적
    mov     eax, DWORD PTR [rbp+16] ; a7 — 스택 인자는 여기서부터
    add     edx, eax
    mov     eax, DWORD PTR [rbp+24] ; a8
    add     eax, edx
    mov     DWORD PTR [rbp-4], eax  ; result
    mov     eax, DWORD PTR [rbp-4]  ; 반환값은 RAX(EAX)
    pop     rbp                     ; rsp를 움직이지 않았으니 leave 대신 pop 하나로 충분
    ret                             ; 피연산자 없음 → 호출자가 정리
```

스택 인자가 `[rbp+16]`에서 시작하는 것은 반환 주소와 saved rbp가 각각 8바이트이기 때문이다. callee에 `sub rsp`가 없는 것은 red zone 때문이다. System V ABI는 RSP 아래 128바이트를 시그널 핸들러 등이 건드리지 않는다고 보장하므로, 다른 함수를 호출하지 않는 leaf 함수는 RSP를 내리지 않고 그 영역을 지역변수로 쓸 수 있다. callee는 leaf라 지역변수와 인자 슬롯 40바이트를 전부 red zone에 두었고, 그래서 스택 프레임을 정리하는 과정도 `leave`가 아니라 `pop rbp`다. caller는 `callee`를 호출하므로 leaf가 아니고, 정상적으로 `sub rsp, 16`을 한다.

**스택** (callee 호출 직후)

```text
높은 주소
  +--------------------+
  | result             |  [caller rbp-4]   caller의 프레임
  | localCaller2 = 999 |  [caller rbp-8]
  | localCaller1 = 99  |  [caller rbp-12]
  | (padding)          |  [caller rbp-16]
  +--------------------+
  | 8   (a8)           |  [rbp+24]         가장 먼저 push. 칸이 8바이트
  | 7   (a7)           |  [rbp+16]         스택 인자 2개(16바이트): caller가 push, caller가 정리
  +--------------------+
  | return address     |  [rbp+8]          8바이트
  | saved rbp          |  [rbp]            <- RBP = RSP (sub rsp 가 없음)
  +--------------------+
  | result             |  [rbp-4]          여기부터 RSP 아래 = red zone. callee의 지역변수
  | localCallee = 100  |  [rbp-8]
  | (padding)          |  [rbp-12]
  | (padding)          |  [rbp-16]
  | 1   (a1)           |  [rbp-20]         RDI 로 받은 a1 을 내려놓은 자리
  | 2   (a2)           |  [rbp-24]         RSI
  | 3   (a3)           |  [rbp-28]         RDX
  | 4   (a4)           |  [rbp-32]         RCX
  | 5   (a5)           |  [rbp-36]         R8
  | 6   (a6)           |  [rbp-40]         R9
  +--------------------+
낮은 주소
```

fastcall과 같은 구조가 더 크게 나타난 것이다. 레지스터로 온 인자 6개는 caller가 push한 영역에 없고 callee의 지역 영역에 복사본으로만 있으며, 최적화하면 이 복사본은 사라진다.

MSVC가 사용하는 Microsoft x64에 대한 설명은 생략하겠다.

## 마치며

함수 호출 규약들에 대해 알아봤는데 필자도 검색하면서 작성한 내용이 많고 평소 알고 있던 것은 System V나 cdecl 정도가 전부이다. 다른 함수 호출 규약에 대해 알아야 할 일이 생기면 보려고 다 정리하긴 했지만, System V, cdecl과 스택 프레임이라는 것이 뭔지, 함수가 어떤 식으로 호출되고 실행되는지, 지역 변수는 왜 함수가 끝나면 사라지는지 정도만 알아도 좋을 것 같다.