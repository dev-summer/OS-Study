# 28 Locks

## 28.1 Locks: The Basic Idea

임계 구역이 `balance = balance + 1`이라고 가정할 때, 락을 사용하기 위해서는 임계 구역 주변에 코드를 추가해야 한다.
```
lock_t mutex; // 전역 변수로 mutex 선언
...
lock(&mutex); // mutex 잠금
balance = balance + 1;
unlock(&mutex); // mutex 해제
```
- 락은 변수이기 때문에, 락을 사용하기 위해서는 락으로 사용할 변수를 반드시 선언해야 한다 (위 예제의 `mutex`).
    - 락은 두 가지 상태를 가질 수 있다(unlocked/locked).
    - unlocked(available/free) 상태일 때에는 임계 구역에 진입해 있는 스레드가 없다.
    - locked(unavailable/held) 상태일 때에는 정확히 하나의 스레드만이 임계 구역에 진입해 있다.
- 어떤 스레드가 락을 점유중인지, 락을 점유하기 위한 순서를 담고 있는 큐 등의 정보를 넣어서 락을 구현할 수도 있지만, 이런 정보들은 락의 사용자(스레드)는 알 수 없다(hidden).

`lock()`과 `unlock()`의 동작방식
- `lock()`을 호출하면 락을 점유하려고 시도한다. 만약 락을 점유한 스레드가 없다면, `lock()`을 호출한 스레드가 락을 점유하고 임계 구역에 진입한다. 이 스레드를 락의 소유자라고 부르기도 한다.
- 다른 스레드가 같은 변수(`mutex`)의 `lock()`을 호출하면, 락을 이미 사용하고 있는 스레드가 있는 동안은 return 하지 않는다. 이런 방식을 통해 락을 처음 점유한 스레드가 임계 구역에 진입해 있는 동안에 다른 스레드가 임계 구역을 진입하는 것을 막는다.
- 락을 점유 중인 스레드가 `unlock()`을 호출하면, 락이 다시 사용 가능해진다(free). 만약 락을 기다리고 있는 스레드가 없다면 락의 상태가 `free`로 바뀐다. 만약 락을 기다리고 있는 스레드가 있다면, 락의 상태가 변경된 것에 대해 noti를 받아서 락을 점유하고, 임계 구역에 진입한다.

## 28.2 Pthread Locks

POSIX 라이브러리에서는 락을 **mutex**라고 부른다. 스레드 간의 상호 배제를 제공하기 때문이다.

## 28.3 Building a Lock

락을 만들기 위해서 하드웨어, OS는 무엇을 지원해야 하는가?
어떻게 하면 효율적인 락을 만들 수 있을까?
- 효율적인 락: 낮은 비용으로 상호 배제를 지원하고, 몇몇 다른 프로퍼티들도 지원하면 좋겠다.

## 28.4 Evaluating Locks

락의 효율성을 평가하는 기준
- 락의 기본 기능: 상호 배제를 지원하는가? (여러 개의 스레드가 임계 구역에 진입하는 것을 막는가?) 
- 공정성: 락을 사용하기 위해 기다리는 각 스레드가 락을 점유할 기회를 공평하게 받는가? 기아 상태가 발생하는 스레드는 없는가?
    - 기아 상태: 무한 대기 & 끝까지 얻지 못함
- 성능: 락을 사용함으로써 추가되는 time overhead
    - 경쟁이 없는 경우(실행 중인 스레드가 하나 뿐인 경우)는 락을 점유하고 해제하는 오버헤드가 어떻게 될까?
    - 여러 스레드가 하나의 CPU에서 락을 점유하기 위해 경쟁하고 있는 경우에는 성능이 어떻게 될까?
    - 여러 스레드가 여러 개의 CPU에서 락을 점유하기 위해 경쟁하고 있는 경우 락은 어떻게 기능할까?

## 28.5 Controlling Interrupts

상호 배제를 지원하기 위한 방법 중 하나는 임계 구역에서는 인터럽트가 발생하지 못하도록 하는 것이다. 이 방법은 싱글 프로세서 시스템을 위해 고안되었다.
```C
void lock() {
    DisableInterrups();
}
void unlock() {
    EnableInterrupts();
}
```
- 임계 구역에 진입하기 전에 인터럽트를 꺼버리면(disable), 임계 구역 내의 코드가 atomic하게 실행되는 것을 보장할 수 있다.
- 임계 구역의 코드의 실행을 마치고 나면 인터럽트를 다시 켠다(enable).

이 방법의 장점은 간단하다는 것이다.
이 방법의 단점은 다음과 같다.
- `lock()`/`unlock()`을 호출하는 스레드가 previleged operation(인터럽트 껐/켰)을 수행하도록 허용해야 한다. 특정 프로그램이 이 권리를 남용할 수 있는 위험성이 있다 (프로그램 시작 시 `lock()`을 호출해서 인터럽트가 발생하지 못하도록 해버리기).
- 멀티 프로세서에서는 작동하지 않는다.
    - 여러 개의 스레드가 서로 다른 CPU에서 실행 되고 있고 각 스레드가 같은 임계 구역에 진입하려고 한다면, 인터럽트가 꺼져있든 켜져있든 스레드는 다른 프로세서에서 각각 실행될 수 있고, 임계 구역에 진입할 수 있다.
- 일정 기간동안 인터럽트를 끄면, 필요한 인터럽트를 못 받아서(miss) 심각한 시스템 문제를 일으킬 수 있다.
    - 예: 디스크가 읽기 작업을 끝냈다는 인터럽트를 CPU가 놓쳐서 그 작업을 기다리던 프로세스가 영영 깨어나지 못할 수 있다.
- 비효율적일 수 있다.
    - 현대의 CPU에서 인터럽트를 mask/unmask하는 코드는 느리게 수행되는 경향이 있다.

위의 이유로 인해 인터럽트를 끄고 켜는 방식은 제한된 문맥 하에서만 사용된다.

## 28.6 A Failed Attempt: Just Using Loads/Stores

하나의 플래그 변수만을 가진 간단한 락을 만들어 보자.
- 임계 구역에 진입해있는 스레드가 존재하면 플래그의 값은 `1`, 존재하지 않으면 `0`
- 임계 구역에 처음으로 들어가는 스레드는 `lock()`을 호출한다. `lock()`은 플래그의 값이 `1`인지를 확인한다.
- 플래그의 값이 `1`이 아니면 해당 스레드가 락을 점유했다는 것을 알리기 위해 플래그의 값을 `1`로 변경한다.
- 임계 구역의 실행을 끝내면, 해당 스레드는 `unlock()`을 호출하고 플래그를 비워서 락이 점유되지 않았다는 것을 알린다.
- 첫 번째 스레드가 임계 구역에 있는 동안 다른 스레드가 `lock()`을 호출하면, 그 스레드는 첫 번째 스레드가 `unlock()`을 호출해서 플래그를 비울 때까지 while 루프 안에서 spin-wait 한다.
- 첫 번째 스레드가 `unlock()`을 호출하면 대기 중인 스레드가 while 루프 바깥으로 빠져나와서 플래그를 `1`로 설정하고 임계 구역에 진입한다.

위 예시의 문제점은 다음과 같다.
- 정확성
    - 동시적인(concurrent) 환경에서 인터럽트가 특정한 시점에 발생해서 두 개의 스레드가 동시에 플래그를 `1`로 설정해서 임계 구역에 들어가는 경우.
- 성능
    - 이미 점유 중인 락을 기다리는 방법의 문제
    - 플래그의 값을 계속 확인하는 방법(spin-waiting)은 값을 계속 확인하는 작업으로 인해 락을 점유 중인 스레드가 작업할 시간을 낭비시킨다 (특히 uniprocessor인 경우 점유 중인 스레드가 아예 실행되지 못할 수도 있다).


## 28.7 Building Working Spin Locks with Test-And-Set

인터럽트를 끄는 방법은 멀티 프로세서에서는 작동하지 않고, 위의 방법(loads and stores)은 작동하지 않기 때문에, 하드웨어 지원이 필요하다.
가장 단순한 하드웨어 지원 방법은 **test-and-set**(**atomic change**)이다.
```C
int TestAndSet(int *old_ptr, int new) {
    int old = *old_ptr; // fetch old value at old_ptr
    *old_ptr = new; // store 'new' into old_ptr
    return old; // return the old value
}

typedef struct __lock_t {
    int flag;
} lock_t;

void init(lock_t *lock) {
    // 0: lock is available, 1: lock is held
    lock->flag = 0;
}

void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1)
        ; // spin-wait (do nothing)
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```
- `old_ptr`에서 가리키는 old value를 반환함과 동시에 `old_ptr`을 `new`로 업데이트 한다. (이 과정이 atomic하게 실행되어야 한다.)
    - test-and-set 이라고 불리는 이유는 old value를 test 함과 동시에 new value에 메모리 위치를 set하기 때문이다.
1. 어떤 스레드가 `lock()`을 호출하고, 현재 락을 점유하고 있는 스레드가 없을 경우 플래그는 0이다.
    - 해당 스레드가 `TestAndSet(flag, 1)`을 호출하면, 플래그의 old value 인 0을 반환한다. 이것을 플래그의 값을 테스트 한다고 하고, 플래그의 값을 테스트하는 스레드(`TestAndSet`을 호출한 스레드)는 while loop에 spin-wait하지 않고 락을 점유한다. 그리고 플래그 값을 1로 바꾼다.
    - 해당 스레드가 임계 구역의 코드를 모두 실행하고 나면 `unlock()`을 호출해서 플래그를 0으로 설정한다.
2. 락이 이미 점유된 상황(플래그 = 1)에서 어떤 스레드가 `lock()`을 호출하고 `TestAndSet(flag, 1)`을 호출하면 플래그의 old value인 `1`을 반환하는 동시에 플래그의 값을 `1`로 다시 설정한다.
    - 다른 스레드가 락을 점유하고 있는 한, `TestAndSet()`은 반복적으로 `1`을 반환한다. 그러므로 이 스레드는 락이 해제될 때까지 spin한다.
    - 락을 점유하고 있떤 스레드에 의해 플래그의 값이 0으로 설정되면 이 스레드는 `TestAndSet()`을 호출하고 플래그의 값 0을 반환받음과 동시에 플래그의 값을 1로 설정해서 락을 점유하고 임계 구역에 진입한다.

락의 old value에 대한 **test**와 락의 new value에 대한 **set**을 하나의 atomic operation으로 만듦으로써 단 하나의 스레드만이 락을 점유하도록 보장할 수 있다.
이런 종류의 락을 **spin lock**이라고 한다.
- 락이 사용 가능해 질 때까지 CPU cycle을 사용해서 spin하기만 한다.
- 싱글 프로세서에서 제대로 작동하기 위해서는 **선점 스케줄러**(**preemptive scheduler**)가 필요하다.
     - 선점 없이는 싱글 CPU에서는 spinning하는 스레드가 계속 CPU를 점유하기 때문에 제대로 작동할 수 없기 때문.


## 28.8 Evaluating Spin Locks

정확성: 상호 배제를 지원하는가? O
- spin lock은 한 번에 하나의 스레드만이 임계 구역에 들어갈 수 있다.

공정성: 대기 중인 스레드가 언젠가는 임계 구역에 들어간다는 것을 보장하는가? X
- spin lock은 경쟁 상태에서는 영원히 spin하는 스레드가 발생할 수도 있어 기아 상태가 발생할 수 있다.

성능: spin lock을 사용하는 비용
- 싱글 CPU의 경우 오버헤드가 매우 크다. 대기 중인 스레드들이 CPU cycle을 모두 한 번씩 잡아먹으면서 lock을 점유할 수 있는지 확인하기 때문이다.
- 멀티 CPU의 경우 스레드의 수가 CPU의 수와 엇비슷할 경우 잘 작동한다.
    - 스레드A는 CPU1에 있고 스레드B는 CPU2에 있으며, 둘이 경쟁 상태인 경우
    - 스레드A가 락을 점유한 후 스레드B가 락을 점유하려고 하면 스레드B는 CPU2에서 spin-wait한다. (스레드A는 CPU1에서 임계 구역을 실행)


## 28.9 Compare-And-Swap

= Compare-And-Exchange
```C
int CompareAndSwap(int *ptr, int expected, int new) {
    int original = *ptr;
    if (original == expected)
        *ptr = new;
    return original;
}
```
`ptr`에서 지정한 주소의 값이 `expected`와 일치하는지 테스트한다.
- 일치한다면 `ptr`이 가리키는 메모리 주소의 값을 새 값으로 업데이트 한다.
- 일치하지 않다면 아무것도 하지 않는다.
- 두 경우 모두 해당 메모리 위치의 original value를 반환하게 해서, compare-and-swap을 호출한 코드에서 compare-and-swap이 성공했는지 여부를 알 수 있게 해야 한다.

```C
void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1)
        ; // spin
}
```
- 플래그가 0인지 확인하고, 0이면 플래그를 1로 바꾸고 락을 점유한다.
- 락을 점유하려는 스레드는 락이 풀릴 때까지 spin-wait 한다.

Compare-And-Swap은 Test-And-Set보다 더 강력한 instruction이다. (왜????)
