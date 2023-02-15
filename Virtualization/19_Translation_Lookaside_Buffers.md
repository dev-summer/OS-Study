# 19 Translation Lookaside Buffers

페이징 기법을 사용해서 가상 메모리를 지원하면 퍼포먼스 오버헤드가 발생할 수 있다. 주소 공간을 작은&고정된 크기의 페이지로 쪼개기 때문에 매핑 정보가 많이 필요하다. 매핑 정보는 일반적으로 물리 메모리에 저장되기 때문에, 페이징은 프로그램에 의해 생성된 가상 주소를 찾아보기 위해서 추가적인 메모리를 필요로 한다. 매번 instruction이 있을 때마다 메모리에 접근해서 주소 변환을 하기 때문에 로드/저장이 느려지게 된다.

주소 변환을 더 빠르게 하고, 페이징 할 때 추가로 메모리를 참조하지 않을 수 없을까?(CPU가 메모리에 있는 페이지 테이블을 보기 위해 한 번, 알게 된 프레임에 접근하기 위해 한 번 → 총 2번의 메모리 접근이 필요하다.)

**Translation Lookaside Buffer(TLB)**
- 하드웨어 칩의 MMU(Memory Management Unit)의 일부로, 자주 변환되는 주소들에 대한 하드웨어 캐시이다.
- 가상 메모리 참조마다 하드웨어는 우선 TLB를 확인해서 원하는 변환된 주소가 있는지 확인한다. 만약 TLB에 있으면 페이지 테이블을 통해 주소를 변환하지 않아도 되기 대문에 주소 변환 과정이 빠르게 일어난다. 

## 19.1 TLB Basic Algorithm

1. 가상 메모리 주소로부터 가상 페이지 번호를 추출한다.
2. TLB가 이 페이지 번호에 대한 변환된 주소를 가지고 있는지 확인한다.
    - 만약 가지고 있다면 TLB hit -> TLB 엔트리에서 프레임 번호를 추출하고, offset을 붙여서 원하는 물리 주소를 얻는다. (메모리에 있는 페이지 테이블에 접근 안 함)
    - 만약 없다면 TLB miss -> 페이지 테이블에서 프레임 번호로 변환하고, TLB에 변환한 값을 업데이트 한다. TLB가 업데이트 된 후에 하드웨어는 다시 TLB에서 페이지 번호에 대한 변환된 주소를 가지고 있는지 확인한다.
3. 얻어낸 물리 메모리 주소에 접근한다.

## 19.2 Example: Accessing An Array

**spacial locality**: 같은 페이지 안의 주소값을 참조하는 경우 처음 접근했을 때만 miss, 그 다음부터는 hit
**temporal locality**: TLB가 충분히 커서 필요한 translation을 모두 캐싱할 수 있을 경우M 일정 시간 내에 재참조를 하면 hit

## 19.3 Who Handles the TLB Miss?

하드웨어 또는 소프트웨어(OS)가 처리하겠지?

하드웨어가 처리하는 방법
- 하드웨어는 **페이지 테이블 레지스터(PLTB)**를 통해 페이지 테이블이 메모리의 어디에 위치해 있는지 정확히 알아야 한다.
- TLB miss가 발생하면, 하드웨어는 페이지테이블에 가서 페이지 테이블 엔트리를 찾아서 원하는 주소 변환값을 얻어온 후 TLB에 그 값을 업데이트 한 후 TLB에 페이지 번호를 가지고 가서 원하는 주소 변환값이 있는지 찾는다.

소프트웨어(OS)가 처리하는 방법
- TLB miss가 발생하면 하드웨어는 exception을 발생시켜서 현재이ㅡ instruction stream을 일시 정지 시키고, previlege level을 커널 모드로 올리고, trap handler를 작동시킨다(jumps to)?
- trap handler는 OS내에 쓰여진 TLB miss를 처리하기 위한 코드일 것이다.
- trap handler가 실행되면, privileged instruction을 사용해서 TLB를 업데이트 한 후 trap에서 돌아온다. 그러면 하드웨어가 다시 instruction을 시도한다 (이번에는 TLB hit이 발생할 것)
    - TLB miss handling trap에서 return할 때에 하드웨어는 trap을 발생시킨 instruction을 다시 시작해야 한다(resume). instruction을 다시 실행하면 TLB hit이 발생한다.
    - TLB miss를 처리하는 코드를 실행할 때 OS는 TLB miss가 영원히 반복되지 않도록 주의해야 한다.
        - 해결방법1: TLB miss handler를 물리 메모리에 보관하기
        - 해결방법2: TLB의 몇몇 엔트리를 TLB에 보관하고 (영구적으로 유효한 translation의 경우), 이 엔트리들을 TLB miss handler에서 사용하기

## 19.4 TLB Contents: What's in There?

TLB는 32/64/128 개의 엔트리를 가지고 있고, fully associative할 수 있다.
- fully associative = translation은 TLB의 어디에나 들어갈 수 있다 (자리가 정해져 있지 않다는 느낌). -> 하드웨어는 원하는 translation을 찾기 위해 TLB 전체를 병렬적으로 검색한다.
- TLB엔트리는 VPN, PFN, 기타 비트들로 구성된다.
- 기타 비트들에는 일반적으로 다음과 같은 비트들이 있다.
    - **valid bit**: 해당 엔트리가 valid translation인지
    - **protection bit**: 해당 페이지에 접근할 수 있는지 (읽기/쓰기/실행)
    - **address space identifier**, **dirty bit** 등

## 19.5 TLB Issue: Context Switches

TLB를 사용하면 프로세스 간 문맥 교환이 일어날 때 추가적인 문제들이 생긴다.
TLB는 현재 실행 중인 프로세스에만 유효한 virtual-to-physical translation을 가지고 있기 때문에, 프로세스가 전환되면 이 translation들은 의미가 없게 된다. 따라서 프로세스를 변경하게 되면 하드웨어나 OS는 실행 예정인 프로세스가 이전 프로세스의 translation을 사용하지 않도록 주의해야 한다.

문맥 교환 시에 TLB를 어떻게 관리해야 하는가?
1. 문맥 교환 시 TLB를 비운다.
    - 프로세스를 실행할 때마다 반드시 TLB miss가 발생하게 된다. 프로세스 전환이 자주 일어날 경우 비용이 커진다.
    - 이 오버헤드를 줄이기 위해 문맥 교환시에도 TLB를 공유할 수 있는 하드웨어 기능을 제공하기도 한다.
2. 문맥 교환 시에도 TLB를 공유
    - TLB에 **address space identifier(ASID)** 항목을 제공한다.
        - **process identifier(PID)**와 유사하지만, 일반적으로 비트 수가 더 적다.
    - ASID로 프로세스를 구분할 수 있기 때문에 여러 프로세스의 translation을 동시에 가지고 있을 수 있다.
    ![](https://i.imgur.com/nliRscT.png)
3. 서로 다른 프로세스의 TLB 엔트리가 같은 물리 메모리 프레임을 가리킬 경우
    - 예를 들어 두 프로세스가 같은 물리 메모리 프레임을 공유할 경우 이런 상황이 발생할 수 있다.
    - 이런 경우, 사용되는 물리 프레임 갯수가 줄어들기 때문에 메모리 오버헤드가 줄어든다.
    ![](https://i.imgur.com/nxqUtgl.png)

## 19.6 Issue: Replacement Policy

TLB에 새로운 엔트리를 넣으려고 할 때, 기존 엔트리를 교체해야 한다면 어떤 엔트리를 교체할 것인가?
1. LRU(Least Recently Used)
    - 메모리의 참조 지역성을 이용한 전략
2. Random
    - 단순하고, 엣지케이스를 피할 수 있다는 장점이 있다.
    
## 19.8 Summary

하드웨어에(MMU에) 주소 변환용 캐시인 TLB를 둠으로써, 메인 메모리의 페이지 테이블에 접근하지 않고도 대부분의 메모리 참조를 처리할 수 있다. 결과적으로 프로그램의 성능이 메모리가 가상화되지 않은 것처럼 향상될 수 있다.
하지만 프로그램이 짧은 시간 내에 접근하는 페이지의 개수가 TLB에 들어갈 수 있는 페이지의 개수보다 많다면, 프로그램은 TLB miss를 많이 발생시키게 되어 더 느리게 동작할 수도 있다. 이 현상을 **TLB coverage**를 초과했다고 한다.
또, TLB는 CPU pipeline에 병목현상을 발생시킬 수 있는데, physically-index cache일 경우 그렇다. 이런 캐시에서는 주소 변환이 반드시 캐시에 접근하기 전에 일어나야 하기 때문에, 느려질 수 있다. 가상 주소로 캐시가 인덱싱 되어 있다면 이런 문제는 어느정도 해결된다.
