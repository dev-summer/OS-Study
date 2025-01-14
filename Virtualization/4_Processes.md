# 4 Processes

프로세스: 실행중인 프로그램(running program)
- 프로그램 자체는 디스크에 가만히 있는 애일 뿐이고, OS가 프로그램을 실행시킨다.

물리적인 CPU의 개수는 정해져있는데, 어떻게 수십 개의 프로그램을 동시에 실행할 수 있는 것일까? (마치 CPU가 무한대로 있는 것처럼) → **OS가 CPU를 가상화**해서 마치 CPU가 무한대로 있는 것 같이 작동한다.
- **Time Sharing** 기법: 실제로는 제한된 개수의 물리적 CPU만을 가지고 있지만, 한 프로세스를 실행하고, 그 프로세스를 멈추고 다른 프로세스를 실행하는 방식으로 여러 개의 프로세스를 동시에 실행

CPU 가상화를 잘 구현하기 위해서 OS는 low-level machinery, high-level intelligence가 필요하다.
- low-level machinary = **mechanism**
  - 하나의 기능을 구현하는 low-level 메서드 또는 프로토콜
  - 예: 문맥 교환(context switching)을 어떻게 구현(수행)하는가?
- high-level intellignce = **policies**
  - OS내에서 결정을 내리는 알고리즘
  - 예: 스케줄링 정책(scheduling policy): 실행을 기다리고 있는 여러 개의 프로세스 중 OS는 어떤 프로세스을 실행할 것인가?
- low-level mechanism과 high-level policies를 분리하는게 일반적인 OS 디자인 패러다임이다. 둘을 분리하면 mechanism에 대해 고민할 필요 없이 policy를 변경할 수 있음 → 모듈화

## 4.1 The Abstraction: A Process

프로세스를 구성하는 것이 무엇인지 알려면, 프로세스의 **machine state**를 이해해야 한다.
- machine state: 프로그램이 실행되고 있을 때 읽거나(read), 업데이트(update)할 수 있는 것

machine state의 구성 요소
- 메모리: 명령(instruction), 프로그램을 실행할 때 읽고 쓰는 데이터는 메모리에 있음. 그러므로 프로세스가 주소를 지정할 수 있는 메모리(프로세스의 **address space**)는 프로세스의 일부이다.
- 레지스터: 많은 명령은 명시적으로 레지스터를 읽거나 업데이트하기 때문
  - **Program Counter(PC)**: **Instruction Pointer (IP)**라고도 부른다. 프로그램이 다음에 어떤 명령을 실행할 지를 알려준다.
  - **Stack Pointer**, **Frame Pointer**: 함수의 매개변수, 지역 변수, 반환 주소를 위한 스택을 관리한다.

## 4.2 Process API

모던 OS는 아래의 API 기능을 제공함.
- Create: 새로운 프로세스 생성
- Destroy: 프로세스 강제 종료
- Wait: 프로세스 대기시키기
- Status: 프로세스의 상태 정보를 가져오기 (얼마나 오래 실행되었는지, 어떤 상태에 있는지 등등)

## 4.3 Process Creation

OS는 어떻게 프로그램을 불러와서 시작할까?(프로세스를 어떻게 만들까?)
1. 프로그램의 코드와 정적 데이터(예: 초기화 된 변수)를 메모리에 로드 (프로세스의 address space에 로드)
    - 초기 OS는 로드 과정이 **eager**(프로그램 실행 전에 한 번에 다 로드함), 모던 OS는 **lazy**(프로그램 실행 시에 필요할 때 필요한 데이터를 로드) → 페이징이랑 스와핑을 통해서 함
2. 프로그램에게 런타임 **스택** 메모리를 할당함
    - C 프로그램은 스택을 지역 변수, 함수 매개변수, 반환 주소값을 저장하는 데 씀
    - OS는 스택으로 쓸 메모리를 프로세스한테 할당해준다.
3. 프로그램에게 **힙** 메모리를 할당함.
    - C 프로그램은 힙을 동적으로 할당하는 데이터를 저장하는 데 씀
    - 프로그램은 `malloc()`을 호출해서 공간을 달라고 요청하고, `free()`를 명시적으로 호출해서 사용했던 공간을 비운다.
    - 연결 리스트, 해시 테이블, 트리 등의 데이터 구조에서 힙을 씀
4. 다른 초기화 작업을 함 (특히 I/O 관련된 것들)
    - 유닉스 시스템에서 각 프로세스는 기본적으로 세 개의 open **file descriptor**를 가지고 있다(표준 input, output, error).
      - descriptor는 프로그램이 터미널로부터 쉽게 input을 읽고, output을 화면에 출력하도록 해준다.

코드와 정적 데이터를 메모리에 로드하고, 스택을 생성하고 초기화하고, I/O 설정 관련된 작업을 하고 나면 OS는 프로그램을 실행할 준비가 됐다. 마지막으로 프로그램 실행을 시작하면 된다 (보통 `main()`을 호출).

## 4.4 Process States

Initial/New(생성): 프로세스를 생성 중인 상태.

Running(실행): 프로세서에서 프로세스가 실행중인 상태. 명령을 수행하고 있다는 뜻.

Ready(준비): 프로세스는 실행될 준비가 되었지만, OS가 얘를 실행하지 않고 있음.

Blocked(대기): 다른 이벤트가 일어나기 전까지는 프로세스가 실행 될 준비가 안 되는 작업을 수행했을 경우. 예: I/O 작업을 요청해서 I/O 작업이 끝나야만 작업을 수행할 수 있는 경우

Final/Terminated(종료): 프로세스가 종료되었지만 아직 메모리에서 해제(정리)되지 않은 상태. 유닉스 시스템에서는 zombie state라고 한다.
  - 다른 프로세스(주로 부모 프로세스)에서 막 종료된 프로세스의 return code를 분석해서 성공적으로 실행 되었는지 알기 위해 사용할 수 있다. 유닉스 기반 시스템에서는 성공적으로 작업을 완료 한 경우 0을 반환함.

Ready → Running: 프로세스가 스케줄링 되었다.

Running → Ready: 프로세스가 디스케줄링 되었다.

프로세스가 대기 상태가 되면, OS는 특정 이벤트가 발생할 때까지 해당 프로세스를 보관한다. 특정 이벤트가 발생하면 프로세스가 다시 준비 상태가 된다.

## 4.5 Data Structures

OS는 중요한 데이터 구조들을 가지고 있다.
- **process list(task list)**: 시스템에서 실행 중인 모든 프로그램을 추적
- **Process Control Block(PCB)**: 하나의 프로세스에 대한 정보(**process descriptor**)를 저장.

OS는 각 프로세스의 상태를 추적해야 한다. 프로세스에 관한 정보 중 다음과 같은 정보들을 OS는 추적한다.
- 프로세스 메모리의 시작(start)
- 프로세스 메모리의 크기
- 이 프로세스의 커널 스택의 끝(bottom)
- 프로세스 상태
- 프로세스 ID (PID)
- 부모 프로세스
- **register context**: 중지된 프로세스와 그 프로세스의 레지스터의 내용을 가지고 있다.
  - 어떤 프로세스가 중지되면 그 프로세스의 레지스터는 이 메모리 위치(register context)에 저장된다.
  - 레지스터를 복구해서 OS는 그 프로세스를 재개할 수 있다.
  - context switch 관련 내용임
- 등등등
