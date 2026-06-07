# 06. Conditional Processing 상세 정리

> Source: `06_Conditional Processing.pdf`  
> 주제: x86 assembly에서 CPU status flags와 conditional jump를 이용해 조건문, 반복문, table-driven selection, FSM, MASM high-level directives를 구현하는 방법

---

## 전체 Overview

- 이 강의자료의 핵심 주제
    - Assembly language에서 **조건 처리**를 어떻게 구현하는가?
    - 고급 언어에서는 다음과 같이 작성함

        ```c
        if (x > y) {
            ...
        } else {
            ...
        }
        ```

    - 하지만 x86 assembly에는 `if`, `else`, `while` 같은 고급 제어구조가 직접 존재하지 않음
    - 대신 다음 두 가지를 조합해서 조건 처리를 구현함
        - **CPU status flags**
            - 연산 결과가 0인지
            - carry가 발생했는지
            - signed overflow가 발생했는지
            - 결과가 음수처럼 보이는지
        - **conditional jump instructions**
            - `JZ`, `JNZ`, `JE`, `JNE`, `JG`, `JL`, `JA`, `JB` 등
            - flag 상태를 보고 특정 label로 이동함

- 이 장의 전체 흐름

    ```text
    Boolean / Comparison Instructions
    |
    |-- AND, OR, XOR, NOT, TEST
    |-- CMP
    |-- CPU status flags
        ↓
    Conditional Jumps
    |
    |-- JZ / JNZ
    |-- JE / JNE
    |-- unsigned jumps: JA, JAE, JB, JBE
    |-- signed jumps: JG, JGE, JL, JLE
        ↓
    Conditional Loop Instructions
    |
    |-- LOOPZ / LOOPE
    |-- LOOPNZ / LOOPNE
        ↓
    Conditional Structures
    |
    |-- IF
    |-- IF-ELSE
    |-- compound expression: AND / OR
    |-- WHILE
    |-- table-driven selection
        ↓
    Finite-State Machines
    |
    |-- input string validation
    |-- signed integer validation
        ↓
    MASM High-Level Directives
    |
    |-- .IF / .ELSE / .ELSEIF / .ENDIF
    |-- .WHILE
    |-- .REPEAT / .UNTIL
    ```

- 즉, 이 강의자료의 핵심은:

    ```text
    연산을 수행해서 flag를 만들고,
    conditional jump가 그 flag를 읽고,
    그 결과에 따라 프로그램의 실행 흐름을 바꾸는 것.
    ```

---

# 1. Boolean and Comparison Instructions

## 1.1 CPU Status Flags

- CPU status flags의 역할
    - x86 CPU는 연산을 수행한 뒤, 그 결과에 대한 정보를 flag에 저장함
    - 이 flag들은 이후 conditional jump instruction이 조건 판단에 사용함
    - 즉, assembly에서 조건문은 대부분 다음 흐름으로 만들어짐

        ```text
        연산 수행
        → CPU flag 변경
        → jump instruction이 flag 검사
        → 특정 label로 이동하거나 이동하지 않음
        ```

- 주요 flag
    - `ZF`, Zero Flag
        - 연산 결과가 0이면 set됨
        - 즉, `ZF = 1`
        - 예:

            ```asm
            cmp eax, 0
            ```

            - `eax == 0`이면 `eax - 0 = 0`
            - 따라서 `ZF = 1`

    - `CF`, Carry Flag
        - unsigned 연산에서 carry 또는 borrow가 발생하면 set됨
        - 예:

            ```text
            8-bit unsigned에서 255 + 1
            = 11111111b + 00000001b
            = 1 00000000b
            ```

            - 8-bit 공간에는 `00000000b`만 남고 carry가 밖으로 나감
            - 따라서 `CF = 1`

    - `SF`, Sign Flag
        - 결과의 최상위 bit, 즉 MSB를 복사한 flag
        - signed 관점에서 MSB가 1이면 음수로 해석됨
        - 따라서 `SF = 1`이면 결과가 음수처럼 보인다는 뜻

    - `OF`, Overflow Flag
        - signed 연산에서 표현 범위를 벗어나면 set됨
        - 예:

            ```text
            8-bit signed 범위: -128 ~ 127
            127 + 1 = 128
            ```

            - 128은 8-bit signed 범위 밖
            - 따라서 `OF = 1`

    - `PF`, Parity Flag
        - 결과의 low byte에 있는 1 bit의 개수가 짝수이면 set됨
        - 일반적인 if문보다는 parity check나 low-level error checking 쪽에서 의미가 있음

- 핵심 정리

    ```text
    Assembly에서 조건 판단은
    값을 직접 판단하는 것이 아니라,
    연산 결과로 설정된 flag를 보고 판단한다.
    ```

---

## 1.2 AND Instruction

- `AND` instruction의 기본 의미
    - 두 operand의 대응되는 bit끼리 AND 연산을 수행함
    - 결과는 destination operand에 저장됨

        ```asm
        AND destination, source
        ```

    - 진리표

        ```text
        0 AND 0 = 0
        0 AND 1 = 0
        1 AND 0 = 0
        1 AND 1 = 1
        ```

    - 둘 다 1일 때만 결과가 1

- `AND`의 대표적인 용도: bit masking
    - 특정 bit를 0으로 clear하고 싶을 때 사용함
    - 원리

        ```text
        x AND 1 = x
        x AND 0 = 0
        ```

    - 따라서 mask에서 1인 bit는 원래 값이 유지되고, 0인 bit는 강제로 0이 됨

- 예시: AL에서 bit 0과 bit 3만 clear하기

    ```asm
    and AL, 11110110b
    ```

    - `11110110b`에서 bit 0과 bit 3만 0임
    - 나머지는 1이므로 기존 값 유지
    - 결과적으로:

        ```text
        bit 0 → 0으로 clear
        bit 3 → 0으로 clear
        나머지 bit → 그대로 유지
        ```

- flag 변화
    - `AND`는 항상 `OF`와 `CF`를 clear함
    - 결과에 따라 `SF`, `ZF`, `PF`를 수정함
    - 예:

        ```asm
        and al, 00000000b
        ```

        - 결과가 0
        - 따라서 `ZF = 1`

- 핵심 정리

    ```text
    AND는 특정 bit를 0으로 지우는 데 매우 유용하다.
    또한 결과에 따라 ZF를 바꾸기 때문에,
    이후 JZ/JNZ와 함께 bit 검사에도 사용된다.
    ```

---

## 1.3 AND를 이용한 lowercase -> uppercase 변환

- ASCII 코드에서 대문자와 소문자의 관계
    - 예를 들어:

        ```text
        'a' = 0110 0001b = 61h
        'A' = 0100 0001b = 41h
        ```

    - 두 값의 차이는 bit 5 하나임
    - 소문자는 bit 5가 1
    - 대문자는 bit 5가 0

- 따라서 lowercase를 uppercase로 바꾸려면
    - bit 5를 0으로 clear하면 됨
    - mask는:

        ```text
        1101 1111b
        ```

    - bit 5만 0이고 나머지는 1임

- 예시

    ```asm
    and BYTE PTR [esi], 11011111b
    ```

    - `[esi]`가 가리키는 문자에서 bit 5를 clear
    - 만약 문자가 `'a'`라면:

        ```text
        0110 0001   ; 'a'
        1101 1111   ; mask
        ---------
        0100 0001   ; 'A'
        ```

- 배열 전체를 uppercase로 바꾸는 구조

    ```asm
    mov ecx, LENGTHOF array
    mov esi, OFFSET array

    L1:
        and BYTE PTR [esi], 11011111b
        inc esi
        loop L1
    ```

    - `ECX`
        - 반복 횟수
        - 배열 길이
    - `ESI`
        - 현재 문자 위치를 가리키는 pointer
    - `and BYTE PTR [esi], 11011111b`
        - 현재 문자의 bit 5 clear
    - `inc esi`
        - 다음 문자로 이동
    - `loop L1`
        - `ECX`를 감소시키고, 0이 아니면 반복

- 핵심 정리

    ```text
    ASCII 알파벳에서 대문자와 소문자는 bit 5만 다르다.
    따라서 AND 11011111b를 사용하면 lowercase를 uppercase로 바꿀 수 있다.
    ```

---

## 1.4 OR Instruction

- `OR` instruction의 기본 의미
    - 두 operand의 대응되는 bit끼리 OR 연산을 수행함

        ```asm
        OR destination, source
        ```

    - 진리표

        ```text
        0 OR 0 = 0
        0 OR 1 = 1
        1 OR 0 = 1
        1 OR 1 = 1
        ```

    - 둘 중 하나라도 1이면 결과가 1

- `OR`의 대표적인 용도
    - 특정 bit를 1로 set할 때 사용함
    - 원리

        ```text
        x OR 0 = x
        x OR 1 = 1
        ```

    - mask에서 1인 bit는 강제로 1이 되고, 0인 bit는 원래 값 유지

- 예시: AL의 bit 2를 set하기

    ```asm
    or AL, 00000100b
    ```

    - bit 2만 1인 mask
    - 결과:

        ```text
        bit 2 → 무조건 1
        나머지 bit → 그대로 유지
        ```

- 시스템 프로그래밍적 의미
    - device control byte
    - permission bit
    - status register
    - mode flag
    - 이런 값들은 각 bit가 특정 기능을 의미할 수 있음
    - 어떤 기능을 켤 때 `OR`를 사용함

- flag 변화
    - `OR`는 `CF`, `OF`를 clear함
    - 결과에 따라 `SF`, `ZF`, `PF`를 수정함

- 핵심 정리

    ```text
    AND가 특정 bit를 0으로 clear하는 명령이라면,
    OR은 특정 bit를 1로 set하는 명령이다.
    ```

---

## 1.5 Bit-Mapped Sets

- bit-mapped set의 개념
    - set membership을 bit로 표현하는 방식
    - 예를 들어 universal set이 다음과 같다고 하자

        ```text
        {0, 1, 2, ..., 31}
        ```

    - 32개의 원소는 32-bit integer 하나로 표현 가능함
    - 각 bit position이 원소 하나를 의미함

        ```text
        bit 0  → element 0
        bit 1  → element 1
        bit 2  → element 2
        ...
        bit 31 → element 31
        ```

- membership 표현
    - 어떤 bit가 1이면 해당 원소가 set에 포함됨
    - 어떤 bit가 0이면 해당 원소가 set에 포함되지 않음

- 예시
    - `SetX`에서 bit 0, 1, 2, 31이 1이면:

        ```text
        SetX = {0, 1, 2, 31}
        ```

- 특정 원소가 포함되어 있는지 검사하기
    - 예를 들어 element 4가 SetX에 있는지 보려면 bit 4를 검사해야 함
    - bit 4 mask:

        ```text
        10000b
        ```

    - 코드:

        ```asm
        mov eax, SetX
        and eax, 10000b
        ```

    - 결과 해석:

        ```text
        결과 == 0     → element 4 없음
        결과 != 0     → element 4 있음
        ```

- ZF와 연결
    - `AND` 결과가 0이면 `ZF = 1`
    - `AND` 결과가 0이 아니면 `ZF = 0`
    - 따라서 conditional jump와 연결 가능

        ```asm
        mov eax, SetX
        and eax, 10000b
        jnz ElementExists
        ```

- 핵심 정리

    ```text
    bit-mapped set은 각 bit를 하나의 원소 membership으로 사용하는 방식이다.
    특정 원소 검사에는 AND와 bit mask를 사용한다.
    ```

---

## 1.6 Set Complement, Intersection, Union

- Set Complement
    - 여집합은 `NOT`으로 만들 수 있음
    - `NOT`은 모든 bit를 반전함

        ```asm
        mov eax, SetX
        not eax
        ```

    - 결과:

        ```text
        SetX에 있던 원소 → 빠짐
        SetX에 없던 원소 → 들어옴
        ```

- Set Intersection
    - 교집합은 `AND`로 만들 수 있음

        ```asm
        mov eax, SetX
        and eax, SetY
        ```

    - 이유:

        ```text
        SetX bit = 1 and SetY bit = 1 → 결과 1
        그 외 경우 → 결과 0
        ```

    - 즉, 두 set 모두에 포함된 원소만 남음

- Set Union
    - 합집합은 `OR`로 만들 수 있음

        ```asm
        mov eax, SetX
        or eax, SetY
        ```

    - 이유:

        ```text
        SetX bit = 1 or SetY bit = 1 → 결과 1
        둘 다 0 → 결과 0
        ```

    - 즉, 둘 중 하나라도 포함된 원소는 결과 set에 포함됨

- 집합 연산과 boolean instruction의 대응

    ```text
    complement   → NOT
    intersection → AND
    union        → OR
    ```

- 핵심 정리

    ```text
    bit-mapped set을 사용하면
    집합 연산을 매우 빠른 bitwise instruction 하나로 처리할 수 있다.
    ```

---

## 1.7 XOR Instruction

- `XOR` instruction의 기본 의미
    - exclusive-OR 연산
    - 두 bit가 서로 다르면 1, 같으면 0

        ```text
        0 XOR 0 = 0
        0 XOR 1 = 1
        1 XOR 0 = 1
        1 XOR 1 = 0
        ```

- 중요한 성질 1: 0과 XOR하면 원래 값 유지

    ```text
    x XOR 0 = x
    ```

- 중요한 성질 2: 1과 XOR하면 bit 반전

    ```text
    x XOR 1 = NOT x
    ```

- 따라서 XOR는 특정 bit toggle에 유용함
    - 예:

        ```asm
        xor AL, 00001000b
        ```

    - bit 3만 toggle됨
        - 0이면 1
        - 1이면 0

- 중요한 성질 3: 같은 값으로 두 번 XOR하면 원래 값으로 돌아옴

    ```text
    X XOR Y XOR Y = X
    ```

    - 이유:

        ```text
        Y XOR Y = 0
        X XOR 0 = X
        ```

- 간단한 암호화에 사용 가능
    - plain text 문자 `X`
    - key `Y`
    - 암호화:

        ```text
        cipher = X XOR Y
        ```

    - 복호화:

        ```text
        plain = cipher XOR Y
        ```

    - 즉:

        ```text
        X XOR Y XOR Y = X
        ```

- flag 변화
    - `XOR`는 `CF`, `OF`를 clear함
    - 결과에 따라 `SF`, `ZF`, `PF`를 수정함

- 핵심 정리

    ```text
    XOR는 bit toggle에 유용하고,
    같은 key를 두 번 적용하면 원래 값으로 돌아오는 reversible property를 가진다.
    ```

---

## 1.8 Parity Flag 검사

- parity의 의미
    - 어떤 binary 값 안에 1 bit가 몇 개 있는지 보는 것
    - 1 bit 개수가 짝수이면 even parity
    - 1 bit 개수가 홀수이면 odd parity

- x86의 Parity Flag
    - 결과의 lowest byte에서 1 bit 개수가 짝수이면 `PF = 1`
    - 홀수이면 `PF = 0`

- 값을 바꾸지 않고 parity를 검사하는 방법
    - `xor`를 0과 수행

        ```asm
        xor al, 0
        ```

    - `AL XOR 0 = AL`
    - 값은 유지됨
    - 하지만 flag는 갱신됨

- 16-bit parity 검사
    - 16-bit 값을 upper byte와 lower byte로 나눔
    - 둘을 XOR하면 전체 parity 정보를 low byte로 압축할 수 있음
    - 이후 `PF`를 확인하면 됨

- 핵심 정리

    ```text
    XOR 0은 값을 바꾸지 않지만 flag를 갱신한다.
    이를 이용해 parity flag를 확인할 수 있다.
    ```

---

## 1.9 NOT Instruction

- `NOT` instruction의 기본 의미
    - operand의 모든 bit를 반전함

        ```asm
        not al
        ```

    - 예:

        ```text
        F0h = 1111 0000b
        NOT F0h = 0000 1111b = 0Fh
        ```

- 결과는 one's complement라고 부름
    - 모든 bit를 뒤집은 값

- flag 변화
    - `NOT`은 flag에 영향을 주지 않음
    - 이것이 `AND`, `OR`, `XOR`와 다른 점임

- 핵심 정리

    ```text
    NOT은 모든 bit를 반전하지만,
    CPU status flags는 변경하지 않는다.
    ```

---

## 1.10 TEST Instruction

- `TEST` instruction의 기본 의미
    - 내부적으로는 `AND` 연산을 수행함
    - 하지만 destination operand를 수정하지 않음
    - flag만 변경함

        ```asm
        test destination, source
        ```

- `AND`와의 차이
    - `AND`
        - 결과를 destination에 저장함
        - operand 값이 바뀜
    - `TEST`
        - 결과를 저장하지 않음
        - operand 값은 그대로
        - flag만 바뀜

- 특정 bit 검사에 유용함
    - 예:

        ```asm
        test AL, 00100000b
        ```

    - bit 5가 set되어 있는지 검사
    - 결과 해석:

        ```text
        AL의 bit 5 = 0 → TEST 결과 0 → ZF = 1
        AL의 bit 5 = 1 → TEST 결과 nonzero → ZF = 0
        ```

- 여러 bit 검사
    - 예:

        ```asm
        test AL, 00001001b
        ```

    - bit 0과 bit 3을 검사함
    - 결과:

        ```text
        bit 0과 bit 3이 모두 0 → ZF = 1
        둘 중 하나라도 1 → ZF = 0
        ```

- 조건 jump와 연결

    ```asm
    test AL, 00001001b
    jnz SomeBitSet
    ```

    - bit 0 또는 bit 3 중 하나라도 set되어 있으면 jump

- 핵심 정리

    ```text
    TEST는 값을 바꾸지 않고 bit 상태만 검사할 때 사용한다.
    보통 TEST + JZ/JNZ 조합으로 많이 사용한다.
    ```

---

## 1.11 CMP Instruction

- `CMP` instruction의 기본 의미
    - 두 값을 비교함
    - 내부적으로는 다음 연산을 수행한 것처럼 flag를 바꿈

        ```text
        destination - source
        ```

    - 하지만 실제 destination 값은 변경하지 않음

        ```asm
        cmp destination, source
        ```

- 예:

    ```asm
    cmp eax, ebx
    ```

    - 내부적으로 `eax - ebx`를 해본 것처럼 flag 설정
    - `eax`, `ebx` 값 자체는 바뀌지 않음

- 이후 conditional jump와 연결

    ```asm
    cmp eax, ebx
    je  EqualLabel
    ```

    - `eax == ebx`이면 subtraction 결과가 0
    - 따라서 `ZF = 1`
    - `JE`는 `ZF = 1`이면 jump

- unsigned 비교에서 중요한 flag
    - `ZF`
    - `CF`
    - 대략:

        ```text
        destination < source  → CF = 1
        destination == source → ZF = 1
        destination > source  → CF = 0 and ZF = 0
        ```

- signed 비교에서 중요한 flag
    - `SF`
    - `OF`
    - `ZF`
    - signed 비교는 단순히 CF만 보면 안 됨

- 예시 1

    ```asm
    mov ax, 5
    cmp ax, 10
    ```

    - 내부적으로 `5 - 10`
    - unsigned 관점에서는 borrow 필요
    - 따라서:

        ```text
        CF = 1
        ZF = 0
        ```

- 예시 2

    ```asm
    mov ax, 1000
    mov cx, 1000
    cmp cx, ax
    ```

    - 내부적으로 `1000 - 1000 = 0`
    - 따라서:

        ```text
        ZF = 1
        CF = 0
        ```

- 예시 3

    ```asm
    mov si, 105
    cmp si, 0
    ```

    - 내부적으로 `105 - 0 = 105`
    - 결과는 0이 아니고 borrow도 없음
    - 따라서:

        ```text
        ZF = 0
        CF = 0
        ```

- 핵심 정리

    ```text
    CMP는 값을 바꾸지 않고,
    subtraction 결과가 어떤 flag를 만들었을지만 기록한다.
    이후 JE/JNE/JG/JL/JA/JB 등이 그 flag를 보고 분기한다.
    ```

---

# 2. Conditional Jumps

## 2.1 Conditional Structures의 기본 원리

- x86에는 고급 언어의 `if`, `else`, `while`이 직접 없음
- 대신 다음 두 단계를 사용함
    - 1단계:
        - `CMP`, `AND`, `SUB`, `TEST` 같은 명령어로 CPU status flags를 설정함
    - 2단계:
        - conditional jump instruction이 flag를 검사해서 이동 여부를 결정함

- 기본 구조

    ```asm
    cmp eax, 0
    jz  L1
    ```

    - `cmp eax, 0`
        - `eax - 0`을 해본 것처럼 flag 설정
    - `jz L1`
        - `ZF = 1`이면 `L1`으로 jump
    - 의미:

        ```c
        if (eax == 0)
            goto L1;
        ```

- bit 검사 구조

    ```asm
    and dl, 10110000b
    jnz L2
    ```

    - `DL`의 특정 bit들을 검사
    - 결과가 0이 아니면 `ZF = 0`
    - `JNZ`는 `ZF = 0`이면 jump
    - 의미:

        ```text
        DL의 검사 대상 bit 중 하나라도 1이면 L2로 이동
        ```

- 핵심 정리

    ```text
    Conditional jump는 값을 직접 검사하지 않는다.
    바로 이전 연산이 만들어 놓은 flag를 검사한다.
    ```

---

## 2.2 Jcond Instruction

- conditional jump instruction의 일반 형식

    ```asm
    Jcond destination
    ```

    - `cond`
        - 검사할 flag 조건
    - `destination`
        - 조건이 true일 때 이동할 label

- 동작 방식
    - 조건이 true이면:

        ```text
        destination label로 jump
        ```

    - 조건이 false이면:

        ```text
        바로 다음 instruction 실행
        ```

- 대표적인 Jcond

    ```text
    JC   → Jump if Carry        → CF = 1
    JNC  → Jump if Not Carry    → CF = 0
    JZ   → Jump if Zero         → ZF = 1
    JNZ  → Jump if Not Zero     → ZF = 0
    ```

- 예시

    ```asm
    cmp eax, 0
    jz  L1
    mov ebx, 1
    L1:
    ```

    - `eax == 0`
        - `ZF = 1`
        - `jz L1` 실행
        - `mov ebx, 1` 건너뜀
    - `eax != 0`
        - `ZF = 0`
        - jump하지 않음
        - `mov ebx, 1` 실행

- 핵심 정리

    ```text
    Jcond는 계산하지 않는다.
    계산은 CMP/TEST/AND/SUB 등이 하고,
    Jcond는 flag만 읽는다.
    ```

---

## 2.3 CMP와 Conditional Jump의 연결

- `JE`
    - Jump if Equal
    - `ZF = 1`이면 jump
    - 보통 `CMP` 뒤에서 사용함

    ```asm
    cmp eax, 5
    je  L1
    ```

    - 의미:

        ```c
        if (eax == 5)
            goto L1;
        ```

- `JL`
    - Jump if Less
    - signed 비교에서 `<`

    ```asm
    mov ax, 5
    cmp ax, 6
    jl  L1
    ```

    - 의미:

        ```c
        if ((signed)ax < 6)
            goto L1;
        ```

- `JG`
    - Jump if Greater
    - signed 비교에서 `>`

    ```asm
    mov ax, 5
    cmp ax, 4
    jg  L1
    ```

    - 의미:

        ```c
        if ((signed)ax > 4)
            goto L1;
        ```

- 핵심 정리

    ```text
    CMP destination, source
    → destination - source를 해본 것처럼 flag 설정
    → conditional jump가 그 flag를 보고 이동
    ```

---

## 2.4 Conditional Jump의 네 가지 종류

- conditional jump instruction은 크게 네 그룹으로 나눌 수 있음
    - 특정 flag 값에 기반한 jump
    - equality 비교 jump
    - unsigned comparison jump
    - signed comparison jump

---

## 2.5 Specific Flag 기반 Jump

- 특정 flag를 직접 검사하는 jump

    ```text
    JZ   / JE   → ZF = 1
    JNZ  / JNE  → ZF = 0

    JC          → CF = 1
    JNC         → CF = 0

    JO          → OF = 1
    JNO         → OF = 0

    JS          → SF = 1
    JNS         → SF = 0

    JP          → PF = 1
    JNP         → PF = 0
    ```

- 예시

    ```asm
    add al, bl
    jc  CarryOccurred
    ```

    - `ADD` 결과 unsigned carry가 발생하면 `CF = 1`
    - `JC`가 `CarryOccurred`로 jump

- bit 검사 예시

    ```asm
    test al, 00001000b
    jnz BitIsSet
    ```

    - bit 3이 set되어 있으면 `TEST` 결과 nonzero
    - 따라서 `ZF = 0`
    - `JNZ` 실행

- 핵심 정리

    ```text
    특정 flag 기반 jump는
    비교 연산의 의미보다 flag 자체의 상태를 직접 본다.
    ```

---

## 2.6 Equality Comparisons

- 같음/다름 비교
    - `JE`
        - Jump if Equal
        - `ZF = 1`
    - `JNE`
        - Jump if Not Equal
        - `ZF = 0`
    - `JZ`
        - Jump if Zero
        - `ZF = 1`
    - `JNZ`
        - Jump if Not Zero
        - `ZF = 0`

- `JE`와 `JZ`의 관계
    - 실제 조건은 같음

        ```text
        JE == JZ
        ```

    - 둘 다 `ZF = 1`이면 jump
    - 하지만 의도상 구분해서 쓰는 것이 좋음
        - `CMP`로 두 값을 비교한 후 같음을 표현할 때:

            ```asm
            cmp eax, ebx
            je EqualLabel
            ```

        - 연산 결과가 0인지 보는 느낌이면:

            ```asm
            test al, 00010000b
            jz BitClear
            ```

- `JNE`와 `JNZ`의 관계

    ```text
    JNE == JNZ
    ```

    - 둘 다 `ZF = 0`이면 jump

- `(E)CX` 기반 jump
    - `JCXZ`
        - `CX = 0`이면 jump
    - `JECXZ`
        - `ECX = 0`이면 jump
    - `JRCXZ`
        - `RCX = 0`이면 jump, 64-bit mode

- 예시

    ```asm
    mov ecx, count
    jecxz Done

    L1:
        ; loop body
        loop L1

    Done:
    ```

    - `count == 0`이면 loop body를 아예 실행하지 않음

- 핵심 정리

    ```text
    JE/JZ는 같은 flag를 보지만,
    JE는 비교 결과의 같음을 표현할 때,
    JZ는 결과가 zero인지를 표현할 때 더 자연스럽다.
    ```

---

## 2.7 Unsigned Comparisons

- unsigned 비교에서 사용하는 jump
    - `JA`
        - Jump if Above
        - `leftOp > rightOp`
    - `JAE`
        - Jump if Above or Equal
        - `leftOp >= rightOp`
    - `JB`
        - Jump if Below
        - `leftOp < rightOp`
    - `JBE`
        - Jump if Below or Equal
        - `leftOp <= rightOp`

- 핵심 단어
    - `Above`
    - `Below`
    - 이 단어가 나오면 unsigned 비교라고 생각하면 됨

- 예시

    ```asm
    cmp eax, ebx
    ja  L1
    ```

    - 의미:

        ```c
        if ((unsigned)eax > (unsigned)ebx)
            goto L1;
        ```

- unsigned 비교에서 중요한 flag
    - `CF`
    - `ZF`

- flag 관계

    ```text
    leftOp < rightOp  → CF = 1
    leftOp == rightOp → ZF = 1
    leftOp > rightOp  → CF = 0 and ZF = 0
    ```

- jump 조건

    ```text
    JA   → CF = 0 and ZF = 0
    JAE  → CF = 0
    JB   → CF = 1
    JBE  → CF = 1 or ZF = 1
    ```

- 핵심 정리

    ```text
    unsigned 비교에서는 Above/Below 계열을 사용한다.
    JA, JAE, JB, JBE를 기억해야 한다.
    ```

---

## 2.8 Signed Comparisons

- signed 비교에서 사용하는 jump
    - `JG`
        - Jump if Greater
        - `leftOp > rightOp`
    - `JGE`
        - Jump if Greater or Equal
        - `leftOp >= rightOp`
    - `JL`
        - Jump if Less
        - `leftOp < rightOp`
    - `JLE`
        - Jump if Less or Equal
        - `leftOp <= rightOp`

- 핵심 단어
    - `Greater`
    - `Less`
    - 이 단어가 나오면 signed 비교라고 생각하면 됨

- 예시

    ```asm
    cmp eax, ebx
    jg  L1
    ```

    - 의미:

        ```c
        if ((signed)eax > (signed)ebx)
            goto L1;
        ```

- signed 비교에서 중요한 flag
    - `SF`
    - `OF`
    - `ZF`

- jump 조건

    ```text
    JL  → SF != OF
    JGE → SF == OF
    JG  → ZF = 0 and SF == OF
    JLE → ZF = 1 or SF != OF
    ```

- signed와 unsigned의 차이를 보여주는 예시
    - 8-bit 값:

        ```text
        AL = 11111111b
        BL = 00000001b
        ```

    - unsigned로 보면:

        ```text
        AL = 255
        BL = 1
        AL > BL
        ```

    - signed로 보면:

        ```text
        AL = -1
        BL = 1
        AL < BL
        ```

    - 따라서 같은 `cmp al, bl` 뒤에서도:

        ```asm
        ja UnsignedGreater
        ```

        - jump 가능
    - 하지만:

        ```asm
        jg SignedGreater
        ```

        - jump하지 않을 수 있음

- 핵심 정리

    ```text
    unsigned 비교: Above / Below
    signed 비교: Greater / Less
    ```

---

# 3. Conditional Jump Applications

## 3.1 Testing Status Bits

- 상황
    - 어떤 external device의 상태가 `status`라는 8-bit memory operand에 저장되어 있다고 하자
    - 각 bit는 특정 상태를 의미함
        - bit 5: device offline
        - bit 0, 1, 4: input 또는 error 관련 상태
        - bit 2, 3, 7: 특정 reset 조건

### bit 5가 set되어 있으면 jump

```asm
mov  al, status
test al, 00100000b
jnz  DeviceOffline
```

- `00100000b`
    - bit 5만 1인 mask
- `TEST`
    - AL 값은 바꾸지 않음
    - bit 5만 검사
- 결과:

    ```text
    bit 5 = 0 → TEST 결과 0 → ZF = 1 → JNZ 실행 안 됨
    bit 5 = 1 → TEST 결과 nonzero → ZF = 0 → JNZ 실행
    ```

- 의미:

    ```c
    if (status & 0b00100000)
        goto DeviceOffline;
    ```

### bit 0, 1, 4 중 하나라도 set되어 있으면 jump

```asm
mov  al, status
test al, 00010011b
jnz  InputDataByte
```

- `00010011b`
    - bit 0, bit 1, bit 4 검사
- 의미:

    ```text
    검사 대상 bit 중 하나라도 1이면 jump
    ```

- 핵심:

    ```text
    하나라도 set되어 있는지 검사할 때는 TEST + JNZ
    ```

### bit 2, 3, 7이 모두 set되어 있으면 jump

```asm
mov al, status
and al, 10001100b
cmp al, 10001100b
je  ResetMachine
```

- 왜 `TEST`만으로는 부족한가?
    - `TEST al, 10001100b`는 bit 2, 3, 7 중 하나라도 set되어 있으면 nonzero가 됨
    - 하지만 여기서는 세 bit가 모두 set되어 있어야 함

- 그래서:
    - 먼저 관심 bit만 남김

        ```asm
        and al, 10001100b
        ```

    - 그 결과가 mask와 같은지 확인

        ```asm
        cmp al, 10001100b
        je ResetMachine
        ```

- 의미:

    ```c
    if ((status & 0b10001100) == 0b10001100)
        goto ResetMachine;
    ```

- 핵심 정리

    ```text
    하나라도 set 검사 → TEST + JNZ
    모두 set 검사     → AND + CMP + JE
    ```

---

## 3.2 Larger of Two Integers

- 목표
    - unsigned integer인 `EAX`, `EBX` 중 큰 값을 `EDX`에 저장

- 고급 언어로 표현하면:

    ```c
    if (eax >= ebx)
        edx = eax;
    else
        edx = ebx;
    ```

- assembly 구조:

    ```asm
    mov edx, eax
    cmp eax, ebx
    jae L1
    mov edx, ebx
    L1:
    ```

- 흐름
    - 먼저 `EDX = EAX`라고 가정
    - `EAX >= EBX`이면 이미 정답이므로 그대로 둠
    - `EAX < EBX`이면 `EDX = EBX`로 변경

- `JAE`
    - unsigned comparison
    - above or equal
    - `eax >= ebx`

- 핵심 정리

    ```text
    일단 하나를 정답 후보로 넣고,
    비교 결과에 따라 필요할 때만 바꾼다.
    ```

---

## 3.3 Smallest of Three Integers

- 목표
    - unsigned 16-bit 값 `V1`, `V2`, `V3` 중 가장 작은 값을 `AX`에 저장

- 고급 언어로 표현하면:

    ```c
    ax = V1;

    if (V2 < ax)
        ax = V2;

    if (V3 < ax)
        ax = V3;
    ```

- assembly 구조:

    ```asm
    mov ax, V1

    cmp ax, V2
    jbe L1
    mov ax, V2

    L1:
    cmp ax, V3
    jbe L2
    mov ax, V3

    L2:
    ```

- 흐름
    - `AX`에 일단 `V1` 저장
    - `AX <= V2`이면 AX 유지
    - 아니라면 V2가 더 작으므로 AX 갱신
    - 같은 방식으로 V3와 비교

- 핵심 정리

    ```text
    최소값/최대값 탐색은
    후보값을 하나 정한 뒤,
    나머지와 비교하면서 갱신하는 방식으로 구현한다.
    ```

---

## 3.4 Loop until Key Pressed

- 목표
    - 사용자가 키를 누를 때까지 loop를 반복

- Irvine32 library의 `ReadKey`
    - input buffer에 key가 없으면 `ZF = 1`
    - key가 있으면 `ZF = 0`

- 코드 구조:

    ```asm
    L1:
        mov eax, 10
        call Delay
        call ReadKey
        jz   L1
    ```

- 흐름
    - 10ms delay
    - key 입력 확인
    - key가 없으면:

        ```text
        ZF = 1
        JZ L1 실행
        다시 loop
        ```

    - key가 있으면:

        ```text
        ZF = 0
        JZ 실행 안 됨
        loop 탈출
        ```

- delay를 넣는 이유
    - 너무 빠르게 busy waiting을 돌면 Windows가 event message를 처리할 시간이 부족할 수 있음
    - keystroke가 무시될 수 있음

- 핵심 정리

    ```text
    ReadKey가 ZF를 설정하고,
    JZ가 그 ZF를 보고 loop를 계속할지 결정한다.
    ```

---

## 3.5 Sequential Search of an Array

- 목표
    - 16-bit integer array에서 첫 번째 nonzero 값을 찾음
    - 찾으면 그 값을 출력
    - 못 찾으면 not found message 출력

- 고급 언어로 표현하면:

    ```c
    found = false;

    for (i = 0; i < ArraySize; i++) {
        if (array[i] != 0) {
            found = true;
            value = array[i];
            break;
        }
    }
    ```

- assembly 구조:

    ```asm
    mov esi, OFFSET array
    mov ecx, LENGTHOF array

    L1:
        cmp WORD PTR [esi], 0
        jne Found
        add esi, 2
        loop L1

    NotFound:
        ; not found message
        jmp Done

    Found:
        ; display value at [esi]

    Done:
    ```

- 흐름
    - `ESI`
        - 현재 배열 원소 주소
    - `ECX`
        - 남은 원소 수
    - `cmp WORD PTR [esi], 0`
        - 현재 원소가 0인지 비교
    - `jne Found`
        - 0이 아니면 찾은 것
    - `add esi, 2`
        - 16-bit array이므로 다음 원소로 이동하려면 2 증가
    - `loop L1`
        - `ECX` 감소 후 0이 아니면 반복

- 핵심 정리

    ```text
    배열 탐색은 pointer register로 위치를 이동하고,
    CMP + JNE로 조건을 검사하며,
    LOOP로 반복 횟수를 관리한다.
    ```

---

## 3.6 Simple String Encryption

- XOR의 reversible property

    ```text
    X XOR Y XOR Y = X
    ```

- 암호화 구조
    - plain text 문자에 key를 XOR

        ```text
        cipher = plain XOR key
        ```

    - cipher text에 같은 key를 다시 XOR

        ```text
        plain = cipher XOR key
        ```

- 프로그램 흐름
    - 사용자가 plain text 입력
    - 프로그램이 각 문자에 single-character key를 XOR
    - cipher text 출력
    - 같은 key로 다시 XOR
    - original plain text 출력

- 핵심 procedure 구조:

    ```asm
    TranslateBuffer PROC
        pushad
        mov ecx, bufSize
        mov esi, 0

    L1:
        xor buffer[esi], KEY
        inc esi
        loop L1

        popad
        ret
    TranslateBuffer ENDP
    ```

- 흐름
    - `ECX`
        - 문자열 길이
    - `ESI`
        - 현재 문자 index
    - `xor buffer[esi], KEY`
        - 현재 문자 암호화 또는 복호화
    - 같은 procedure를 두 번 호출하면:

        ```text
        첫 번째 호출 → 암호화
        두 번째 호출 → 복호화
        ```

- 핵심 정리

    ```text
    XOR encryption은 같은 key를 두 번 적용하면 원래 값으로 돌아온다는 성질을 이용한다.
    ```

---

# 4. Conditional Loop Instructions

## 4.1 LOOPZ / LOOPE

- `LOOPZ`
    - loop if zero
    - `LOOP`와 비슷하지만, `ZF = 1`이어야 반복함

- `LOOPE`
    - loop if equal
    - `LOOPZ`와 같은 opcode
    - 사실상 같은 명령어

- 동작

    ```text
    ECX = ECX - 1

    if ECX > 0 and ZF = 1:
        jump to destination
    else:
        다음 instruction 실행
    ```

- 중요한 점
    - `LOOPZ`, `LOOPE`는 flag를 변경하지 않음
    - 이전 instruction이 만들어 놓은 `ZF`를 사용함

- 예시:

    ```asm
    cmp ax, bx
    loope L1
    ```

    - `cmp ax, bx`가 ZF 설정
    - `loope L1`이 그 ZF를 검사

- 핵심 정리

    ```text
    LOOPZ/LOOPE는
    ECX가 아직 남아 있고,
    직전 비교 결과가 equal 또는 zero일 때 반복한다.
    ```

---

## 4.2 LOOPNZ / LOOPNE

- `LOOPNZ`
    - loop if not zero
    - `ZF = 0`이어야 반복

- `LOOPNE`
    - loop if not equal
    - `LOOPNZ`와 같은 opcode

- 동작

    ```text
    ECX = ECX - 1

    if ECX > 0 and ZF = 0:
        jump to destination
    else:
        다음 instruction 실행
    ```

- `LOOP`, `LOOPZ`, `LOOPNZ` 비교

    ```text
    LOOP      → ECX > 0이면 반복
    LOOPZ     → ECX > 0 and ZF = 1이면 반복
    LOOPNZ    → ECX > 0 and ZF = 0이면 반복
    ```

- 핵심 정리

    ```text
    LOOPNZ/LOOPNE는
    ECX가 남아 있고,
    직전 비교 결과가 not zero 또는 not equal일 때 반복한다.
    ```

---

## 4.3 LOOPNZ 예제: nonnegative number 찾기

- 목표
    - 배열을 앞에서부터 스캔하면서 처음으로 nonnegative number, 즉 0 이상인 값을 찾음

- 배열 예시

    ```asm
    array SWORD -3, -6, -1, -10, 10, 30, 40, 4
    ```

- 핵심 코드:

    ```asm
    mov esi, OFFSET array
    mov ecx, LENGTHOF array

    L1:
        test WORD PTR [esi], 8000h
        pushfd
        add esi, TYPE array
        popfd
        loopnz L1
        jnz quit
        sub esi, TYPE array

    quit:
    ```

- `test WORD PTR [esi], 8000h`
    - `8000h`는 16-bit 값의 sign bit mask

        ```text
        8000h = 1000 0000 0000 0000b
        ```

    - signed 16-bit integer에서 sign bit가 1이면 음수
    - 결과:

        ```text
        현재 값이 음수     → TEST 결과 nonzero → ZF = 0
        현재 값이 0 이상   → TEST 결과 0       → ZF = 1
        ```

- `loopnz L1`
    - `ZF = 0`이고 `ECX > 0`이면 반복
    - 즉, 현재 값이 음수이면 계속 반복
    - 0 이상인 값을 만나면 `ZF = 1`이 되어 반복 종료

- `pushfd` / `popfd`가 필요한 이유
    - 중간에:

        ```asm
        add esi, TYPE array
        ```

        가 있음
    - `ADD`는 flag를 변경함
    - 그런데 `LOOPNZ`는 `TEST`가 만든 ZF를 봐야 함
    - 따라서:

        ```asm
        test ...
        pushfd
        add ...
        popfd
        loopnz ...
        ```

    - `TEST` 직후의 flag를 stack에 저장했다가, `LOOPNZ` 직전에 복원하는 것

- 핵심 정리

    ```text
    LOOPNZ는 이전 flag를 사용한다.
    중간 instruction이 flag를 바꾸면 안 되므로,
    필요한 경우 PUSHFD/POPFD로 flag를 보존해야 한다.
    ```

---

# 5. Conditional Structures

## 5.1 Block-Structured IF Statements

- 고급 언어의 if 구조

    ```c
    if (condition) {
        statements;
    }
    ```

- assembly로 구현하는 기본 원리
    - 조건식을 평가해서 flag 변경
    - 조건이 false이면 body를 건너뜀
    - 조건이 true이면 body 실행

- 기본 패턴

    ```asm
    cmp op1, op2
    jne EndIf
        ; if body
    EndIf:
    ```

- 핵심 사고방식

    ```text
    if 조건이 true이면 body 실행
    ```

    보다는 assembly에서는 보통:

    ```text
    if 조건이 false이면 body를 skip
    ```

    으로 작성함

---

## 5.2 IF Example 1

- 고급 언어 코드

    ```c
    if (op1 == op2) {
        X = 1;
        Y = 2;
    }
    ```

- assembly 코드

    ```asm
    mov eax, op1
    cmp eax, op2
    jne L1
    mov X, 1
    mov Y, 2
    L1:
    ```

- 흐름
    - `op1`을 `EAX`로 이동
    - `op1`과 `op2` 비교
    - 같지 않으면 `L1`으로 jump
    - 같으면 `X = 1`, `Y = 2` 실행

- 왜 `JNE`를 쓰는가?
    - 원래 조건은 `op1 == op2`
    - 하지만 assembly에서는 반대 조건을 사용해서 body를 skip함

        ```text
        op1 != op2이면 if body를 건너뛰어라
        ```

- 핵심 정리

    ```text
    단순 IF문은 보통 조건을 반대로 뒤집어서,
    false일 때 body를 skip하도록 작성한다.
    ```

---

## 5.3 IF Example 2: NTFS Cluster Size

- pseudocode

    ```c
    clusterSize = 8192;

    if (terrabytes < 16)
        clusterSize = 4096;
    ```

- assembly 코드

    ```asm
    mov clusterSize, 8192
    cmp terrabytes, 16
    jae next
    mov clusterSize, 4096

    next:
    ```

- 흐름
    - 기본값으로 `8192` 설정
    - `terrabytes >= 16`이면 이미 기본값이 맞으므로 `next`로 이동
    - `terrabytes < 16`이면 jump하지 않고 `4096`으로 변경

- `JAE`
    - unsigned comparison
    - above or equal
    - `terrabytes >= 16`

- 핵심 정리

    ```text
    기본값을 먼저 넣고,
    조건에 해당할 때만 값을 수정하는 방식으로 코드를 줄일 수 있다.
    ```

---

## 5.4 IF-ELSE Example 3

- pseudocode

    ```c
    if (op1 > op2)
        call Routine1;
    else
        call Routine2;
    ```

- `op1`, `op2`
    - signed doubleword variables라고 가정
    - 따라서 signed comparison 필요

- assembly 코드

    ```asm
    mov eax, op1
    cmp eax, op2
    jg  A1

    call Routine2
    jmp A2

    A1:
        call Routine1

    A2:
    ```

- 흐름
    - `op1 > op2`이면 `A1`으로 jump해서 `Routine1`
    - 그렇지 않으면 아래로 내려와서 `Routine2`
    - `Routine2` 실행 후에는 `jmp A2`로 if-else 전체를 빠져나감
    - 그렇지 않으면 `Routine1`까지 이어서 실행되는 문제가 생김

- 왜 `op1`을 register로 옮기는가?
    - memory operand끼리 직접 비교가 제한될 수 있음
    - 따라서 하나를 register로 가져와 비교

- 핵심 정리

    ```text
    IF-ELSE 구조에서는
    한 branch를 실행한 뒤,
    다른 branch로 떨어지지 않도록 unconditional JMP가 필요하다.
    ```

---

## 5.5 White Box Testing

- white box testing의 의미
    - source code 내부 구조를 알고 있는 상태에서 테스트하는 방법
    - 각 조건문이 만드는 실행 경로를 직접 따라가며 검증함

- 왜 필요한가?
    - 복잡한 조건문은 여러 execution path를 만듦
    - 모든 path가 올바르게 동작하는지 확인해야 함

- 예시 pseudocode

    ```c
    if (op1 == op2) {
        if (X > Y)
            call Routine1;
        else
            call Routine2;
    } else {
        call Routine3;
    }
    ```

- 실행 경로
    - 경로 1:

        ```text
        op1 != op2
        → Routine3
        ```

    - 경로 2:

        ```text
        op1 == op2 and X <= Y
        → Routine2
        ```

    - 경로 3:

        ```text
        op1 == op2 and X > Y
        → Routine1
        ```

- assembly 코드 구조

    ```asm
    mov eax, op1
    cmp eax, op2
    jne L2

    mov eax, X
    cmp eax, Y
    jg  L1

    call Routine2
    jmp L3

    L1:
        call Routine1
        jmp L3

    L2:
        call Routine3

    L3:
    ```

- 흐름
    - 먼저 outer if 검사
    - `op1 != op2`이면 바로 `Routine3`
    - `op1 == op2`이면 inner if 검사
    - `X > Y`이면 `Routine1`
    - 아니면 `Routine2`

- 핵심 정리

    ```text
    White box testing은
    조건문이 만들 수 있는 모든 실행 경로를 입력값으로 확인하는 테스트 방식이다.
    ```

---

## 5.6 Compound Expressions: Logical AND

- 고급 언어 조건

    ```c
    if ((al > bl) && (bl > cl))
        X = 1;
    ```

- AND 조건의 특징
    - 두 조건이 모두 true여야 전체 true
    - 첫 번째 조건이 false이면 두 번째 조건은 검사할 필요 없음
    - 이것을 short-circuit evaluation이라고 함

- assembly 구현

    ```asm
    cmp al, bl
    jbe next

    cmp bl, cl
    jbe next

    mov X, 1

    next:
    ```

- 흐름
    - `al > bl`이 false이면 바로 `next`
        - `al <= bl`이면 false
        - 따라서 `JBE next`
    - 첫 번째 조건이 true이면 두 번째 조건 검사
    - `bl > cl`이 false이면 `next`
    - 둘 다 true일 때만 `mov X, 1`

- 핵심 정리

    ```text
    AND 조건에서는 하나라도 false이면 전체 false다.
    따라서 assembly에서는 false 조건을 만나면 바로 body를 skip한다.
    ```

---

## 5.7 Compound Expressions: Logical OR

- 고급 언어 조건

    ```c
    if ((al > bl) || (bl > cl))
        X = 1;
    ```

- OR 조건의 특징
    - 둘 중 하나라도 true이면 전체 true
    - 첫 번째 조건이 true이면 두 번째 조건은 검사할 필요 없음

- assembly 구현

    ```asm
    cmp al, bl
    ja  L1

    cmp bl, cl
    jbe next

    L1:
        mov X, 1

    next:
    ```

- 흐름
    - `al > bl`이면 바로 `L1`으로 jump
    - 첫 번째 조건이 false이면 두 번째 조건 검사
    - 두 번째 조건도 false이면 `next`
    - 하나라도 true이면 `X = 1`

- 핵심 정리

    ```text
    OR 조건에서는 하나라도 true이면 전체 true다.
    따라서 assembly에서는 true 조건을 만나면 바로 body로 jump한다.
    ```

---

## 5.8 WHILE Loops

- 고급 언어 while

    ```c
    while (val1 < val2) {
        val1++;
        val2--;
    }
    ```

- assembly 구현

    ```asm
    mov eax, val1

    beginwhile:
        cmp eax, val2
        jnl endwhile

        inc eax
        dec val2
        jmp beginwhile

    endwhile:
        mov val1, eax
    ```

- 흐름
    - `EAX`를 `val1`의 substitute로 사용
    - loop 시작에서 조건 검사
    - `val1 < val2`가 아니면 종료
    - 조건이 true이면 body 실행
    - 다시 loop 시작으로 jump

- `JNL`
    - Jump if Not Less
    - signed comparison
    - `val1 >= val2`이면 loop 종료

- 왜 `EAX`를 사용하는가?
    - `val1`을 매번 memory에서 읽고 쓰는 것보다 register 사용이 빠름
    - 마지막에만 `mov val1, eax`로 저장

- 핵심 정리

    ```text
    WHILE loop는 보통 조건을 반대로 검사해서,
    조건이 false가 되는 순간 loop 밖으로 jump하도록 만든다.
    ```

---

## 5.9 IF Statement Nested in a Loop

- 목표
    - 배열 원소 중 `sample`보다 큰 값만 더함

- 고급 언어 코드

    ```c
    while (index < ArraySize) {
        if (array[index] > sample) {
            sum += array[index];
        }
        index++;
    }
    ```

- register mapping
    - `EDX = sample`
    - `EAX = sum`
    - `ESI = index`
    - `ECX = ArraySize`

- assembly 구조

    ```asm
    mov eax, 0
    mov edx, sample
    mov esi, 0
    mov ecx, ArraySize

    L1:
        cmp esi, ecx
        jl  L2
        jmp L5

    L2:
        cmp array[esi*4], edx
        jg  L3
        jmp L4

    L3:
        add eax, array[esi*4]

    L4:
        inc esi
        jmp L1

    L5:
        mov sum, eax
    ```

- 흐름
    - `L1`
        - while 조건 검사
        - `index < ArraySize`이면 loop body
        - 아니면 종료
    - `L2`
        - if 조건 검사
        - `array[index] > sample`이면 `L3`
        - 아니면 `L4`
    - `L3`
        - sum에 현재 원소 추가
    - `L4`
        - index 증가 후 loop 반복
    - `L5`
        - 최종 sum 저장

- `array[esi*4]`
    - array가 DWORD 배열
    - 원소 하나가 4바이트
    - 따라서 index에 4를 곱해야 byte offset이 됨

- 핵심 정리

    ```text
    nested IF in WHILE 구조도 결국 label과 jump의 조합이다.
    while 조건, if 조건, update, 종료 위치를 각각 label로 나누면 된다.
    ```

---

# 6. Table-Driven Selection

## 6.1 Table-Driven Selection의 개념

- 목적
    - 긴 multiway selection 구조를 table lookup으로 대체
    - 고급 언어의 `switch-case`와 비슷한 상황에서 유용함

- 일반적인 if-else 구조

    ```c
    if (input == 'A')
        Process_A();
    else if (input == 'B')
        Process_B();
    else if (input == 'C')
        Process_C();
    else if (input == 'D')
        Process_D();
    ```

- table-driven 방식
    - lookup table을 만듦
    - table에는 다음이 들어감
        - lookup value
        - 해당 value와 연결된 procedure address

- table 예시

    ```asm
    CaseTable BYTE  'A'
              DWORD Process_A
              BYTE  'B'
              DWORD Process_B
              BYTE  'C'
              DWORD Process_C
              BYTE  'D'
              DWORD Process_D
    ```

- entry 구조

    ```text
    [lookup value 1 byte] + [procedure offset 4 bytes]
    ```

- 실행 흐름
    - 사용자 입력 문자를 받음
    - table의 각 entry와 비교
    - match가 발견되면 lookup value 바로 뒤의 procedure offset을 call
    - 각 procedure는 다른 string offset을 `EDX`에 넣고 출력

- 핵심 정리

    ```text
    Table-driven selection은
    비교 대상과 실행할 procedure 주소를 table에 저장하고,
    loop로 table을 검색해서 적절한 procedure를 호출하는 방식이다.
    ```

---

## 6.2 Table-Driven Selection의 장단점

- 단점
    - 초기 overhead가 있음
        - table을 만들어야 함
        - table 검색 loop가 필요함
    - case가 아주 적으면 오히려 직접 비교가 더 단순할 수 있음

- 장점
    - 코드 양을 줄일 수 있음
    - 많은 comparison을 깔끔하게 처리 가능
    - 긴 compare/jump/call 나열보다 수정하기 쉬움
    - table만 수정하면 새로운 case를 추가할 수 있음
    - runtime에 table을 재구성할 수도 있음

- 핵심 정리

    ```text
    case가 많을수록 table-driven selection이 유리하다.
    분기 로직을 코드가 아니라 데이터 table로 관리하는 방식이다.
    ```

---

# 7. Application: Finite-State Machines

## 7.1 FSM의 개념

- FSM, Finite-State Machine
    - 입력에 따라 상태가 바뀌는 machine 또는 program
    - graph로 표현 가능

- FSM graph 구성
    - node
        - program state
    - edge
        - transition
        - 한 state에서 다른 state로 이동하는 조건
    - initial state
        - 시작 state
    - terminal state
        - 정상 종료 가능한 state

- FSM의 기본 공식

    ```text
    현재 상태 + 입력 → 다음 상태
    ```

- 예시
    - 현재 state A
    - 입력이 `'x'`
    - state B로 이동

- assembly에서 FSM 구현
    - 각 state를 label로 표현
    - 각 transition은 `CMP + conditional jump`
    - error 상황은 `Error` label로 이동

- 핵심 정리

    ```text
    FSM은 입력을 하나씩 읽으면서 상태를 바꾸고,
    마지막에 terminal state에 도달했는지 검사하는 구조이다.
    ```

---

## 7.2 Validating an Input String

- 프로그램이 input stream을 읽을 때는 input이 유효한 형식인지 검증해야 함
- 예:
    - compiler
        - source code를 읽음
        - keyword, operator, identifier 등을 token으로 분리함
        - 이런 lexical analysis에 FSM이 사용될 수 있음

- FSM의 input validation 방식
    - 입력을 문자 하나씩 읽음
    - 현재 state에서 해당 문자 transition이 있는지 확인
    - 있으면 다음 state로 이동
    - 없으면 error
    - 입력이 끝났을 때 terminal state이면 valid
    - 입력이 끝났는데 nonterminal state이면 error

- FSM이 illegal input을 감지하는 방식
    - 현재 state에서 다음 입력 문자에 해당하는 transition이 없음
    - input이 끝났는데 현재 state가 terminal state가 아님

- 핵심 정리

    ```text
    FSM은 input sequence가 규칙을 따르는지 검사하는 데 적합하다.
    ```

---

## 7.3 Character String Example

- 규칙
    - 문자열은 반드시 `'x'`로 시작해야 함
    - 문자열은 반드시 `'z'`로 끝나야 함
    - 첫 문자와 마지막 문자 사이에는 `'a'`부터 `'y'`까지의 문자가 0개 이상 올 수 있음

- valid examples

    ```text
    xz
    xaz
    xabcdez
    xyyyyz
    ```

- state 구성
    - State A
        - 시작 상태
        - 아직 `'x'`를 읽지 않음
    - State B
        - `'x'`를 읽은 상태
        - 중간 문자 또는 마지막 `'z'`를 기다림
    - State C
        - 마지막 `'z'`를 읽은 상태
        - terminal state

- transition
    - A -> B
        - 입력이 `'x'`
    - B -> B
        - 입력이 `'a'`부터 `'y'`
    - B -> C
        - 입력이 `'z'`

- assembly 구조

    ```asm
    StateA:
        call ReadChar
        cmp al, 'x'
        je  StateB
        jmp Error

    StateB:
        call ReadChar

        cmp al, 'z'
        je  StateC

        cmp al, 'a'
        jb  Error

        cmp al, 'y'
        ja  Error

        jmp StateB

    StateC:
        ; valid terminal state
    ```

- 핵심 정리

    ```text
    FSM에서 각 state는 label,
    각 transition은 CMP + conditional jump로 구현할 수 있다.
    ```

---

## 7.4 Validating a Signed Integer

- signed integer 형식
    - optional leading sign
    - 그 뒤에 digit sequence

- valid examples

    ```text
    123
    +123
    -456
    0
    ```

- invalid examples

    ```text
    +
    -
    ++12
    12a3
    a123
    ```

- state 구성
    - State A
        - 시작 상태
        - sign 또는 digit을 기다림
    - State B
        - sign을 읽은 직후
        - 반드시 digit이 와야 함
    - State C
        - digit을 하나 이상 읽은 상태
        - digit이 계속 오거나 Enter로 종료 가능

- State A 동작
    - `'+'` 또는 `'-'`가 오면 State B
    - digit이 오면 State C
    - 그 외는 Error

- State B 동작
    - digit이 오면 State C
    - 그 외는 Error
    - 즉, sign만 있고 digit이 없으면 invalid

- State C 동작
    - digit이 오면 State C 유지
    - Enter가 오면 valid
    - 그 외는 Error

- assembly 구조

    ```asm
    StateA:
        call GetNextChar

        cmp al, '+'
        je  StateB

        cmp al, '-'
        je  StateB

        call IsDigit
        jz  StateC

        jmp Error

    StateB:
        call GetNextChar

        call IsDigit
        jz  StateC

        jmp Error

    StateC:
        call GetNextChar

        cmp al, ENTER_KEY
        je  ValidInput

        call IsDigit
        jz  StateC

        jmp Error
    ```

- 핵심 정리

    ```text
    signed integer FSM은
    sign이 optional이지만,
    digit은 반드시 하나 이상 있어야 한다는 규칙을 상태로 표현한다.
    ```

---

## 7.5 IsDigit Procedure

- `IsDigit`
    - Irvine32 library procedure
    - `AL` register를 input으로 받음
    - 결과는 `ZF`로 반환함

- 사용 예

    ```asm
    call IsDigit
    jz  DigitFound
    ```

- 의미
    - `AL`이 digit이면 `ZF = 1`
    - digit이 아니면 `ZF = 0`

- FSM에서의 활용

    ```asm
    call IsDigit
    jz StateC
    jmp Error
    ```

- 핵심 정리

    ```text
    조건 판단 결과가 반드시 register 값으로만 반환되는 것은 아니다.
    procedure가 flag를 설정하고,
    conditional jump가 그 flag를 바로 사용할 수도 있다.
    ```

---

# 8. Conditional Control Flow Directives

## 8.1 MASM High-Level Directives

- 지금까지 직접 작성한 조건 구조

    ```asm
    cmp eax, ebx
    jne ElsePart

    ; if body
    jmp EndIf

    ElsePart:
    ; else body

    EndIf:
    ```

- MASM은 이를 더 쉽게 쓰기 위한 directive를 제공함

    ```asm
    .IF eax == ebx
        ; if body
    .ELSE
        ; else body
    .ENDIF
    ```

- 중요한 점
    - `.IF`는 CPU instruction이 아님
    - assembler directive임
    - assembler가 내부적으로 `CMP`, `Jcc`, `JMP`, label을 생성함

- 핵심 정리

    ```text
    .IF, .WHILE 같은 directive는
    assembly source code를 읽기 쉽게 만들어주는 문법이다.
    실제 CPU는 여전히 CMP/Jcc/JMP를 실행한다.
    ```

---

## 8.2 Creating IF Statements

- MASM IF directive

    ```asm
    .IF condition
        statements
    .ELSEIF condition
        statements
    .ELSE
        statements
    .ENDIF
    ```

- 필수 / 선택
    - `.IF`
        - 필수
    - `.ENDIF`
        - 필수
    - `.ELSEIF`
        - 선택
    - `.ELSE`
        - 선택

- condition
    - C 언어와 유사한 operator 사용 가능
        - `<`
        - `>`
        - `==`
        - `!=`
        - `<=`
        - `>=`

- 예시

    ```asm
    .IF eax > ebx
        mov edx, eax
    .ELSE
        mov edx, ebx
    .ENDIF
    ```

- 내부적으로 생성될 수 있는 구조

    ```asm
    cmp eax, ebx
    jbe ElsePart

    mov edx, eax
    jmp EndIf

    ElsePart:
        mov edx, ebx

    EndIf:
    ```

- 핵심 정리

    ```text
    MASM .IF는 고급 언어의 if처럼 보이지만,
    assembler가 뒤에서 CMP와 conditional jump를 만들어준다.
    ```

---

## 8.3 Generating ASM Code

- `.IF` directive가 실제 어떤 instruction으로 바뀌는지 확인 가능
    - Visual Studio debugger 사용
    - debugging 중 right-click
    - `Go to Disassembly` 선택

- 확인할 수 있는 것
    - `.IF`가 어떤 `CMP`로 바뀌었는지
    - 어떤 conditional jump가 생성되었는지
    - signed comparison인지 unsigned comparison인지

- 왜 중요한가?
    - `.IF`를 쓰면 편하지만, 내부 jump가 무엇인지 모르면 signed/unsigned 비교에서 실수할 수 있음

- 핵심 정리

    ```text
    .IF directive를 사용해도,
    실제 생성된 assembly instruction을 이해할 수 있어야 한다.
    ```

---

## 8.4 Signed and Unsigned Comparisons in .IF

- `.IF`에서 비교할 때 주의할 점
    - MASM은 operand type을 보고 signed/unsigned jump를 생성함
    - unsigned variable이면 unsigned jump 사용
    - signed variable이면 signed jump 사용

- unsigned variable 예

    ```asm
    val1 DWORD 10
    val2 DWORD 20

    .IF val1 < val2
        ; ...
    .ENDIF
    ```

    - unsigned comparison이 생성될 수 있음
    - `JB`, `JBE`, `JA`, `JAE` 계열 사용

- signed variable 예

    ```asm
    sval1 SDWORD -10
    sval2 SDWORD 20

    .IF sval1 < sval2
        ; ...
    .ENDIF
    ```

    - signed comparison이 필요
    - `JL`, `JLE`, `JG`, `JGE` 계열 사용

- 핵심 정리

    ```text
    .IF를 써도 signed/unsigned 문제는 사라지지 않는다.
    변수 선언 type이 어떤 jump를 생성할지에 영향을 준다.
    ```

---

## 8.5 Comparing Registers

- register 비교의 문제
    - `EAX`, `EBX` 자체에는 signed/unsigned 정보가 없음
    - assembler는 register 값이 signed인지 unsigned인지 알 수 없음

- 예

    ```asm
    .IF eax <= ebx
        ; ...
    .ENDIF
    ```

- MASM의 기본 동작
    - register끼리 비교할 때 unsigned comparison을 기본으로 사용할 수 있음
    - 예를 들어 `JBE`가 생성될 수 있음

- 문제점
    - 프로그래머가 signed 비교를 의도했는데 unsigned jump가 생성될 수 있음

- 해결 감각
    - register 안의 값이 signed인지 unsigned인지 프로그래머가 명확히 알고 있어야 함
    - 필요하면 type override나 signed variable과의 비교로 의도를 분명히 해야 함

- 핵심 정리

    ```text
    register 자체에는 signed/unsigned type 정보가 없다.
    따라서 .IF로 register를 비교할 때 생성되는 jump를 주의해야 한다.
    ```

---

## 8.6 Compound Expressions in .IF

- `.IF` directive는 logical AND와 OR도 지원함

- AND 예

    ```asm
    .IF (eax > ebx) && (ebx > ecx)
        mov edx, 1
    .ENDIF
    ```

- OR 예

    ```asm
    .IF (eax == 0) || (ebx == 0)
        call Error
    .ENDIF
    ```

- 내부적으로는 short-circuit evaluation이 적용됨
    - AND
        - 첫 조건이 false이면 두 번째 조건 검사하지 않음
    - OR
        - 첫 조건이 true이면 두 번째 조건 검사하지 않음

- 핵심 정리

    ```text
    .IF의 compound expression도
    내부적으로는 CMP와 conditional jump의 조합으로 변환된다.
    ```

---

## 8.7 SetCursorPosition Example

- procedure 목적
    - cursor position을 설정하기 전 range checking 수행

- 입력 parameter
    - `DH`
        - Y-coordinate
        - 범위: 0 ~ 24
    - `DL`
        - X-coordinate
        - 범위: 0 ~ 79

- 조건
    - `DH > 24`이면 error
    - `DL > 79`이면 error
    - 둘 중 하나라도 out of range이면 error

- `.IF`로 표현하면:

    ```asm
    .IF (dh > 24) || (dl > 79)
        mov edx, OFFSET BadCoordMsg
        call WriteString
    .ELSE
        call Gotoxy
    .ENDIF
    ```

- OR 조건을 쓰는 이유
    - Y가 잘못되어도 error
    - X가 잘못되어도 error
    - 둘 중 하나라도 범위 밖이면 cursor position을 설정하면 안 됨

- 핵심 정리

    ```text
    parameter validation에서는
    여러 조건 중 하나라도 실패하면 error로 보내는 OR 조건이 자주 사용된다.
    ```

---

## 8.8 College Registration Example

- 상황
    - 학생이 course registration을 하려고 함
    - 등록 가능 여부를 두 기준으로 판단
        - grade average
        - 신청 credits

- grade average
    - 0 ~ 400 scale
    - 400이 최고 점수

- credits
    - 학생이 신청하려는 학점 수

- multiway branch 구조 사용 가능

    ```asm
    .IF condition1
        ; case 1
    .ELSEIF condition2
        ; case 2
    .ELSEIF condition3
        ; case 3
    .ELSE
        ; default case
    .ENDIF
    ```

- 예시적 구조

    ```asm
    .IF (gradeAverage >= 350) && (credits <= 18)
        ; registration accepted
    .ELSEIF (gradeAverage >= 250) && (credits <= 15)
        ; registration accepted with limit
    .ELSE
        ; registration denied
    .ENDIF
    ```

- 핵심 정리

    ```text
    .ELSEIF를 사용하면
    복잡한 multiway branch를 고급 언어처럼 읽기 쉽게 작성할 수 있다.
    ```

---

## 8.9 Creating Loops with .REPEAT and .WHILE

- MASM은 loop directive도 제공함
    - `.WHILE`
    - `.REPEAT`
    - `.UNTIL`

### .WHILE

- `.WHILE`의 특징
    - loop body를 실행하기 전에 조건을 먼저 검사함
    - C의 `while`과 유사함

- 예

    ```asm
    mov eax, 1

    .WHILE eax <= 10
        call WriteDec
        inc eax
    .ENDW
    ```

- 흐름

    ```text
    조건 검사
    → true이면 body 실행
    → 다시 조건 검사
    → false이면 종료
    ```

- body가 한 번도 실행되지 않을 수 있음
    - 처음부터 조건이 false이면 바로 종료

### .REPEAT / .UNTIL

- `.REPEAT`의 특징
    - loop body를 먼저 실행함
    - 그 다음 `.UNTIL` 조건을 검사함
    - C의 `do-while`과 유사함

- 예

    ```asm
    mov eax, 1

    .REPEAT
        call WriteDec
        inc eax
    .UNTIL eax > 10
    ```

- 흐름

    ```text
    body 실행
    → 조건 검사
    → 조건이 false이면 반복
    → 조건이 true이면 종료
    ```

- body는 최소 한 번 실행됨

- `.WHILE`과 `.REPEAT` 비교

    ```text
    .WHILE
    → 조건 먼저 검사
    → body가 0번 실행될 수도 있음

    .REPEAT
    → body 먼저 실행
    → body가 최소 1번 실행됨
    ```

---

## 8.10 Loop Containing an IF Statement

- 고급 언어 구조

    ```c
    while (condition) {
        if (another_condition) {
            statement;
        }
        update;
    }
    ```

- MASM directive로 표현

    ```asm
    .WHILE esi < ecx

        .IF array[esi*4] > edx
            add eax, array[esi*4]
        .ENDIF

        inc esi

    .ENDW
    ```

- 앞에서 본 배열 합산 예제와 연결
    - `ESI = index`
    - `ECX = ArraySize`
    - `EDX = sample`
    - `EAX = sum`
    - 조건:

        ```text
        while index < ArraySize
        if array[index] > sample
        sum += array[index]
        ```

- 직접 assembly와 비교
    - 직접 작성하면:

        ```asm
        L1:
            cmp esi, ecx
            jge L5

            cmp array[esi*4], edx
            jle L4

            add eax, array[esi*4]

        L4:
            inc esi
            jmp L1

        L5:
        ```

    - directive를 쓰면:

        ```asm
        .WHILE esi < ecx
            .IF array[esi*4] > edx
                add eax, array[esi*4]
            .ENDIF
            inc esi
        .ENDW
        ```

- 핵심 정리

    ```text
    MASM directive를 사용하면
    label과 jump가 많은 조건 구조를
    고급 언어처럼 읽기 쉽게 작성할 수 있다.

    하지만 내부적으로는 여전히 CMP, Jcc, JMP로 변환된다.
    ```

---

# 최종 핵심 정리

- 이 강의자료의 핵심은 다음과 같음

    ```text
    Assembly의 조건 처리는
    if문 자체가 아니라,
    flag와 jump의 조합이다.
    ```

- 전체 흐름

    ```text
    1. AND, OR, XOR, NOT, TEST, CMP를 배운다.
    2. 이 명령어들이 CPU status flags를 어떻게 바꾸는지 이해한다.
    3. JZ, JNZ, JE, JNE, JA, JB, JG, JL 등이 flag를 보고 jump한다.
    4. 이 jump들을 조합해서 IF, IF-ELSE, WHILE을 만든다.
    5. 더 복잡한 구조는 table-driven selection이나 FSM으로 만든다.
    6. MASM은 .IF, .WHILE 같은 directive로 이를 편하게 작성하게 해준다.
    ```

- 반드시 기억해야 할 비교 jump

    ```text
    Equality:
        JE / JZ     → ZF = 1
        JNE / JNZ   → ZF = 0

    Unsigned:
        JA   → >
        JAE  → >=
        JB   → <
        JBE  → <=

    Signed:
        JG   → >
        JGE  → >=
        JL   → <
        JLE  → <=
    ```

- bit 조작 핵심

    ```text
    AND  → 특정 bit clear
    OR   → 특정 bit set
    XOR  → 특정 bit toggle
    NOT  → 모든 bit invert
    TEST → 값을 바꾸지 않고 bit 검사
    CMP  → 값을 바꾸지 않고 비교 flag만 설정
    ```

- 조건 구조 핵심

    ```text
    IF:
        조건이 false이면 body skip

    IF-ELSE:
        한 branch 실행 후 다른 branch로 떨어지지 않게 JMP 필요

    AND condition:
        하나라도 false이면 바로 탈출

    OR condition:
        하나라도 true이면 바로 body 실행

    WHILE:
        조건이 false이면 loop 탈출
    ```

- MASM directive 핵심

    ```text
    .IF, .ELSE, .ELSEIF, .ENDIF
    .WHILE, .ENDW
    .REPEAT, .UNTIL

    이것들은 사람이 읽기 쉽게 해주는 assembler directive이고,
    실제 CPU는 assembler가 생성한 CMP/Jcc/JMP를 실행한다.
    ```

- 한 문장으로 정리하면:

    ```text
    Conditional Processing은
    CPU flag를 만들고,
    그 flag를 conditional jump가 읽게 하여,
    고급 언어의 if, while, switch, input validation 같은 논리를 assembly 수준에서 구현하는 방법이다.
    ```
