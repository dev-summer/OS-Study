# 18 Introduction to Paging

OS는 공간 관리 문제를 해결하기 위해 주로 두 가지 접근방식 중 하나를 채택한다.
1. 각기 다른 크기의 조각들로 나누기: **segmentation**
    - 공간을 각각 다른 크기의 덩어리로 나누면, 공간 자체가 파편화 되어 할당이 더 어려워질 수 있다.
2. 같은(고정된) 크기의 크기의 조각들로 나누기: **paging**
    - **page**: 프로세스의 **주소 공간을** 일정한 크기로 나눈 것
    - **page frame**: **물리 메모리**를 일정한 크기로 나눈 것
    - 하나의 page frame은 하나의 가상 메모리 페이지를 가질 수 있다(contain).

## 18.1 A Simple Example and Overview

![](https://i.imgur.com/HCGdG1L.png)

페이징의 장점
- flexibility: 프로세스가 주소 공간을 어떻게 사용하는지와 관계 없이 주소 공간을 효율적으로 추상화 할 수 있다
    - 힙/스택이 어느 방향으로 커지는지, 어떻게 사용되는지를 신경쓰지 않아도 된다.
- simplicity: 빈 프레임 리스트를 가지고 있고, 페이지를 할당해야 할 때에는 빈 프레임에 할당하기만 하면 됨

**page table**: 페이지가 어떤 페이지 프레임(물리 메모리)에 있는지를 기록
- **각 프로세스마다** 존재한다.
- 페이지 테이블의 역할: 각 주소 공간의 가상 페이지의 address translation(물리 메모리의 어디에 페이지가 저장되는지)을 저장한다.

프로세스가 생성한 가상 주소를 translate하기 위해서 가상 주소를 두 가지 요소로 쪼갠다.
- **virtual page number(VPN)**
- **offset**
- 예: 가상 주소 공간의 크기가 64 byte, 페이지의 크기가 16 byte 경우
    ![](https://i.imgur.com/gflqJKl.png)
    - 가상 주소를 표현하기 위해서는 총 6bit가 필요하다 (2^6 = 64).
    - 가상 주소에서 가장 높은 비트는 Va5, 가장 낮은 비트는 Va0
    - 64 byte 주소공간을 16 byte짜리 페이지 4개로 나누었기 때문에, 4개의 페이지를 식별할 수 있어야 한다. 페이지를 식별하기 위해 가상 주소의 가장 위 2개의 비트를 사용한다 (4개를 식별하면 되니까 2비트면 충분하기 때문). -> 이 2개의 비트가 VPN
    - 나머지 비트(위의 경우 4bit)를 사용해서 페이지의 어떤 byte를 원하는지를 찾는다. -> 이 4개의 비트가 offset

프로세스가 가상 주소를 생성하면 OS와 하드웨어가 가상 주소를 유의미한 물리 주소로 변환해야 한다.
- 예: 가상주소 010101(21)의 경우
    - virtual page number: 1
    - offset: 5
    - 페이지가 어느 프레임에 있는지를 찾아야 한다 -> 페이지 테이블 활용
    - ![](https://i.imgur.com/n9T89Gs.png)
        - physical frame number(pfn) 또는 ppn(physical page number): 물리 메모리의 프레임 주소
        - virtual page number를 physical frame number로 바꾸고, offset은 그대로 활용해서 physical address로 변환한다.


## 18.2 Where Are Page Tables Stored?

페이지 테이블의 크기가 매우 커질 수도 있다.
- 예: 32 bit 주소 공간, 페이지 크기는 4KB인 경우
    - 가상 주소는 VPN 20 bit, offset 12 bit로 표현된다.
    - VPN이 20 bit라는 것은 OS가 각 프로세스마다 2^20개의 주소 변환을 관리해야 한다는 것을 의미한다.
    - 각 **page table entry(PTE)**가 4byte를 필요로 한다고 가정하면, 페이지테이블마다 4MB를 필요로 하게 된다.
    - 프로세스 100개를 실행하고 있다면, 주소 변환을 위해 메모리를 400MB를 써야 한다.
    - 주소 공간의 크기가 64 bit라면 더 커질 것이다.

페이지 테이블은 크기가 크기 때문에, 현재 실행 중인 프로세스의 페이지 테이블을 MMU에 저장하지 않는다. 대신 메모리 어딘가에 저장한다.
![](https://i.imgur.com/lXs6lad.png)

## 18.3 What's Actually In The Page Table?

페이지 테이블: 가상 주소(가상 페이지 번호)를 물리 주소(물리 프레임 주소)에 매핑하기 위한 데이터 구조
여러 가지 데이터 구조를 사용할 수 있다.
- linear page table: 배열을 사용
    - OS는 가상 페이지 번호로 배열을 인덱싱하고, 해당 인덱스의 값인 페이지 테이블 엔트리(PTE)를 활용해서 원하는 물리 프레임 번호(PFN)을 찾는다.

페이지 테이블 엔트리의 요소
- **valid bit(유효 비트)**: 특정 변환이 유효한지를 알려준다.
    - 예: 프로그램이 실행되기 시작했을 때, 그 프로그램의 주소 공간에서 사용ㅎ되지 않는 부분(힙~스택 사이의 공간)은 모두 유효하지 않은 것으로 마크된다. 프로세스가 유효하지 않은 메모리에 접근하려고 하면 trap을 발생시켜 OS가 프로세스를 종료시킨다.
    - invalid로 마크된 페이지에는 물리 프레임을 할당하지 않아도 되기 때문에 메모리를 절약할 수 있다.
- **protection bit(보호 비트)**: 페이지에 읽기/쓰기/실행이 가능한지를 알려준다.
    - valid vs protection
        - invalid인 경우는 메모리에 로드가 안 된 경우와 접근이 불가능한 경우를 모두 포함하고, protection bit는 로드가 되었는지 안 되었는지를 알려주는 비트(valid하지만 메모리에 로드가 안 되어있는 경우).
- **present bit**: 해당 페이지가 현재 물리 메모리에 있는지, 디스크에 있는지를 알려준다.
    - 디스크에 있는 상태를 **swapped out(스왑 영역에 있다)**고 표현한다.
- **dirty bit**: 페이지가 메모리에 로드된 후 수정된 적이 있는지를 알려준다.
    - = **modified bit(수정 비트)**
- **reference bit(참조 비트)**: 페이지가 참조되었는지(access 되었는지)를 알려준다.
    - = **accessed bit**
    - 페이지가 자주 쓰이는지 확인해서 메모리에 해당 페이지를 유지하는 게 좋을지를 판단할 때 쓰인다.

## 18.4 Paging: Also Too Slow

원하는 데이터를 가져오기 위해서 시스템은
1. 가상 주소를 물리 주소로 변환한다.
2. 물리 주소에서 데이터를 가져오기 전에, 프로세스의 페이지 테이블에서 알맞은 페이지 테이블 엔트리를 가져와야 한다.
3. 그 다음에 주소 변환을 수행하고
4. 물리 메모리에서 데이터를 로드한다.

이것을 수행하기 위해 하드웨어는 현재 실행 중인 프로세스의 페이지 테이블이 어디에 있는지를 알아야 한다.
- **page table base register(PTBR)**
    - 페이지 테이블의 시작 위치의 물리 주소를 가지고 있다고 가정
    - 원하는 페이지 테이블 엔트리의 위치를 찾기 위해 하드웨어는 다음과 같은 함수를 수행한다.
        ```
        VPN = Virtual Address & VPN_MASK >> SHIFT
        PTEAddr = PageTableBaseRegister + VPN * sizeOf(PTE)
        ```
    - 예: 가상 주소가 6bit / 페이지가 4개인 경우
        - `VPN_MASK`는 `110000` (앞의 2개 비트가 VPN이기 때문)
        - `SHIFT`의 값은 `4` (offset이 4bit이기 때문에 VPN만 얻기 위해서)
        - 가상 주소 010101을 넣으면 01이 가상 페이지 번호로 나온다.
        - 이 가상 페이지 번호 값을 PTE 배열의 인덱스로 사용해서, PTBR에 페이지 번호 값을 더해 PTE의 물리 주소 값을 얻는다.
    - PTE의 물리 주소 값을 알면, 하드웨어는 메모리에서 PTE를 찾아서 프레임 번호를 찾아낼 수 있다. 찾아낸 프레임 번호와 가상 주소의 offset을 연결해서 원하는 물리 주소를 만들 수 있다.
        ```
        offset = VirtualAddress & OFFSET_MASK
        PhysAddr = (PFN << SHIFt) | offset
        ```
    - 마지막으로 하드웨어는 메모리에서 원하는 데이터를 가져와서 레지스터 `eax`에 넣는다. -> 메모리에서 값을 로드하기 과정 끝!
    
요약
```
// Extract the VPN from the virtual address
VPN = (VirtualAddress & VPN_MASK) >> SHIFT

// Form the address of the page-table entry (PTE)
PTEAddr = PTBR + (VPN * sizeof(PTE))

// Fetch the PTE
PTE = AccessMemory(PTEAddr)

// Check if process can access the page
if (PTE.Valid == False)
RaiseException(SEGMENTATION_FAULT)
else if (CanAccess(PTE.ProtectBits) == False)
RaiseException(PROTECTION_FAULT)
else
// Access is OK: form physical address and fetch it
offset = VirtualAddress & OFFSET_MASK
PhysAddr = (PTE.PFN << PFN_SHIFT) | offset
Register = AccessMemory(PhysAddr)
```
1. 가상 주소에서 페이지 번호를 추출
2. PTE 주소를 만든다.
3. PTE를 가져온다.
4. 프로세스가 페이지에 접근할 수 있는지 확인한다(유효 비트, 보호 비트 확인)

-> 페이징 기법은 페이지 테이블에서 주소 변환을 하기 위해 추가적으로 메모리 참조를 한 번 더 해야 한다 (페이지 테이블에서 프레임 번호로 변환하는 과정을 말하는듯).

결론: 페이징 기법은 두 가지 문제점이 존재한다.
1. 페이지 테이블이 너무 커져서 메모리를 많이 차지해서 시스템이 느려질 수 있다.
2. 페이지 테이블에서 참조가 추가적으로 1번 더 발생하기 때문에 느려질 수 있다.

## 18.6 Summary

세그멘테이션 기법과 비교했을 때 페이징 기법의 장점
- 외부 단편화가 발생하지 않는다.
- flexible하다: 가상 주소 공간을 띄엄띄엄 쓸 수 있다.

페이징에서 발생할 수 있는 문제점
- 느려질 수 있다 (페이지 테이블에 접근하기 위해 추가적인 메모리 접근이 필요하기 때문)
- 메모리 낭비 발생 가능 (페이지 테이블이 메모리 공간을 차지하기 때문에)

## Homework
1.
-P 1k -a 1m -p 512m -v -n 0
: 페이지 크기 1k, 주소 공간 크기 1mb, 물리 메모리 사이즈 512mb
-> 페이지 개수는 1024개(2^10)

주소 공간의 크기를 키우면 어떻게 될까? -> 페이지 개수가 증가 -> 페이지 테이블 크기가 커진다
페이지 크기를 키우면 어떻게 될까? -> 페이지 개수가 감소 -> 페이지 테이블 크기가 작아진다
큰 페이지를 안 쓰는 이유: 페이지 내부 단편화가 발생해서?


2.
![](https://i.imgur.com/xUfpygp.png)
ARG address space size 16k
ARG phys mem size 32k
ARG page size 1k
-> 페이지는 16개 -> VPN이 4bit

0번째(페이지 번호 0번)의 PTE는 0x80000018 -> valid, 16+8 = 24 = 프레임 번호

VA 0x00003986 (decimal:    14726) --> PA or invalid address?
- 2진수로 변환하면 11 1001 1000 0110
- 앞에서부터 4개 비트가 VPN, 뒤의 10개 비트가 offset
- VPN = 8 + 4 + 2 = 14
- offset = 0x186
- 14번 페이지는 invalid 

VA 0x00002bc6 (decimal:    11206) --> PA or invalid address?
- 2진수로 변환하면 10 1011 1100 0110
- VPN = 8 + 2 = 10
- offset = 0x3C6
- 10번 페이지는 0x80000013 valid, physical page = 0x13 = 19
- 1k씩 19번째 페이지 + 0x3C6 = physical address?
- 1024 * 19 = 19456, 0x3C6 = 966 > 20422(decimal)
- 페이지 크기(1024)안에 있으니까 PA~
VA 0x00001e37 (decimal:     7735) --> PA or invalid address?
VA 0x00000671 (decimal:     1649) --> PA or invalid address?
VA 0x00001bc9 (decimal:     7113) --> PA or invalid address?


3.
첫번째, 두번째: virtual page 4개, physical frame 128개
세번째: virtual page 256개, physical frame 512개
