# 32 Concurrency Bugs

## 32.2 Non-Deadlock Bugs
### Atomicity Violation
복수개의 메모리 접근이 원하는 대로 순차적으로 작동하지 않는 것.
```c
Thread 1::
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}

Thread 2::
thd->proc_info == NULL;
```
위 예시의 경우, 두 개의 스레드에서 `thd` 구조체 안의 `proc_info`에 접근하고 있다.
- 스레드1은 `proc_info`의 값이 NULL인지 확인하고, 아닐 경우 값을 출력한다.
- 스레드2는 `proc_info`의 값을 NULL로 설정한다.
- 스레드1이 호출되어 `proc_info`의 값을 확인한 후, `fputs`를 호출하기 전에 인터럽트가 발생하면 중간에 스레드2가 실행되어 값이 NULL이 되고, 다시 스레드1이 재개되면 NULL pointer가 발생할 수 있다.

위 예시가 올바르게 동작하도록 수정하려면, 공유 자원인 `proc_info`을 참조하는 코드 주변에 락을 설정하면 된다.
```c
pthread_mutex_t proc_info_lock = PTHREAD_MUTEX_INITIALIZER;

Thread 1::
pthread_mutex_lock(&proc_info_lock);
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}
pthread_mutex_unlock(*prock_info_lock);

Thread 2::
pthread_mutex_lock(&proc_info_lock);
thd->proc_info == NULL;
pthread_mutex_unlock(*prock_info_lock);
```

### Order Violation
두 개의 메모리 접근이 원하는 순서와 반대로 작동하는 것
```c
Thread 1::
void init() {
    mThread = PR_CreateThread(mMain, ...);
}

Thread 2::
void mMain(...) {
    mState = mThread ->State;
}
```
위 예시의 경우, 스레드2는 변수 `mThread`가 초기화 되어 있고, NULL이 아니라고 가정하고 있음을 알 수 있다.
- 만약 스레드2가 스레드1보다 먼저 호출되어 실행된다면 `mMain`을 실행할 때 `mThread`의 설정되어 있지 않기 때문에 NULL pointer가 발생한다.

위 문제를 해결하기 위해서는 순서를 강제하면 된다. 조건 변수를 사용해서 해결할 수 있다.
```c
pthread_mutex_t mtLock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t mtCond = PTHREAD_COND_INITIALIZER;
int mtInit = 0;

Thread 1::
void init() {
    mThread = PR_CreateThread(mMain, ...);
    
    pthread_mutex_lock(&mtLock);
    mtInit = 1;
    pthread_cond_signal(&mtCond);
    pthread_mutex_unlock(&mtLock);
}

Thread 2::
void mMain(...) {
    pthread_mutex_lock(&mtLock);
    while (mtInit == 0)
        pthread_cond_wait(&mtCond, &mtLock);
    pthread_mutex_unlock(&mtLock);
    
    mState = mThread ->State;
}
```
- `mtCond`: 조건 변수
- `mtLock`: 조건 변수 `mtCond`를 잠그는 락
- `mtInit`: 상태를 나타내는 변수

## 32.3 Deadlock Bugs

데드락 = 교착 상태

### 데드락의 발생 조건
아래 4가지 조건 중 하나라도 만족하지 않을 경우 데드락은 발생하지 않는다.
1. **상호 배제(Mutual Exclusion)**: 스레드는 자신이 요구하는 리소스를 상호 배제적으로 사용해야 한다(스레드가 락을 점유한다).
2. **점유와 대기(Hold-and-Wait)**: 스레드는 다른 리소스를 기다리는 동안 자신에게 할당된 리소스를 가지고 있다.
3. **비선점(No Preemption)**: 스레드로부터 강제로 리소스를 빼앗을 수 없다.
4. **원형 대기(Circular Wait)**: 앞 스레드가 점유 중인 자원을 다음 스레드에서 기다리는 형태로 스레드간에 원형 체인이 존재한다.

### 원형 대기 예방
락을 획득하는 순서를 모두 정해줘서 예방할 수 있다.
- 예: 락이 두 개가 존재할 경우, 항상 락1을 락2보다 먼저 획득하게 함으로써 데드락을 예방할 수 있다.
- 락이 여러 개가 존재할 경우, 일부에만 순서를 부여해도 데드락을 예방할 수 있다.

### 점유와 대기 예방
모든 락을 한 번에 atomic하게 점유함으로써 예방할 수 있다.
```c
pthread_mutex_lock(prevention); // begin acquisition
pthread_mutex_lock(L1);
pthread_mutex_lock(L2);
pthread_mutex_unlock(prevention); // end acquisition
```
- `prevention` 락을 점유함으로써 원하지 않는 스레드 문맥 교환이 일어나는 것을 방지한다.

### 비선점 예방

```c
top:
    pthread_mutex_lock(L1);
    if (pthread_mutex_trylock(L2) != 0) {
        pthread_mutex_unlock(L1);
        goto top;
    }
```
위와 같이 두 락을 모두 점유할 수 없는 경우 현재 점유 중인 락을 풀어주도록 하면 데드락은 발생하지 않는다.
- **livelock**이 발생할 수 있다.
    - 두 개의 스레드가 이 시퀀스를 반복해서 어느 스레드도 두 개의 락을 점유하지 못하고 위의 시퀀스를 반복하는 상황
- L1 외에 코드를 실행하며 얻은 다른 리소스들도 top으로 돌아갈 때 다시 해제해줘야 하는데, 이것을 구현하는 과정이 복잡해질 수 있다.
- 위의 방법은 선점을 추가하는 방법은 아니다.

### 상호 배제 예방
강력한 하드웨어 instruction을 사용해서 락이 필요하지 않은 데이터 구조를 만든다.

예: 하드웨어에서 atomic한 compare-and-swap instruction을 제공할 경우
```c
int CompareAndSwap(int *address, int expected, int new) {
    if (*address == expected) {
        *address = new;
        return 1; // success
    }
    return 0; // failure
}

void AtomicIncrement(int *value, int amount) {
    do {
        int old = *value;
    } while (CompareAndSwap(value, old, old + amount) == 0);
}
```
- 락을 점유하고 해제하는 대신, compare-and-swap을 사용해서 값을 업데이트 한다.

예: 리스트 삽입의 경우
```c
void insert(int value) {
    node_t *n = malloc(sizeof(node_t));
    assert(n != NULL);
    n->value = value;
    n->next = head;
    head = n;
}
```
- 위의 코드가 여러 스레드에 의해 동시에 호출되면 경쟁 상태가 발생한다.
    - `head = n`을 실행하기 전에 인터럽트가 발생할 경우 경쟁 상태가 발생

위의 문제를 락을 사용해 해결하면 다음과 같다.
```c
void insert(int value) {
    node_t *n = malloc(sizeof(node_t));
    assert(n != NULL);
    n->value = value;
    pthread_mutex_lock(listlock);
    n->next = head;
    head = n;
    pthread_mutex_unlock(listlock);
}
```

락을 사용하지 않고 compare-and-swap을 사용해 해결하려고 하면 다음과 같다.
```c
void insert(int value) {
    node_t *n = malloc(sizeof(node_t));
    assert(n != NULL);
    n->value = value;
    do {
       n->next = head; 
    } while (CompareAndSwap(&head, n->next, n) == 0);
}
```
- 만약 다른 스레드가 중간에 `head`를 바꾼 경우, 이 스레드는 바뀐 `head`로 다시 compare-and-swap을 시도하기 때문에 실패할 것이다.

### 스케줄링을 사용한 데드락 회피
각 스레드가 실행될 때 어떤 락을 점유할 지 미리 알고 있으면, 데드락이 발생하지 않도록 스레드들을 스케줄링할 수 있다.

### Detect and Recover (교착 상태 검출 후 회복)
교착 상태가 가끔 일어나게 두고, 교착 상태가 감지되면 해결하는 것
데드락 감지 체계가 주기적으로 실행되어 리소스 그래프를 그리고 순환 여부를 확인한다.
- 순환이 발생하면(데드락 발생), 시스템을 다시 시작한다.
