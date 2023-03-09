# 30 Condition Variables

어떤 스레드를 실행하기 전에 특정 조건이 참인지 확인하고 싶은 상황이 있다.
- 예: 부모 스레드를 계속 실행하기 전에 자식 스레드가 완료되었는지 확인해야 할 경우(`join()`)
    - 자식 스레드가 완료되었는지 확인하기 위해 parent가 spin-wait 하는 것은 비효율적이기 때문에, 기다리는 동안 parent 스레드를 sleep 시킬 방법이 필요하다.

## 30.1 Definition and Routines

특정 조건이 참일 때까지 기다리기 위해서 스레드는 **조건 변수**(**Condition Variable**)을 사용해야 한다.
- 조건 변수: 원하는 조건이 만족되지 않았을 때, 스레드들을 넣는 큐
    - 스레드를 이 큐에 넣는 것을 **wait**
    - 다른 스레드가 조건의 상태를 바꾸면, 대기 중인 스레드 중 하나를 깨워서 실행될 수 있도록 하는 것을 **signal**

```c
int done = 0;
pthrad_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c = PTHREAD_COND_INITIALIZER;

void thr_exit() {
    Pthread_mutex_lock(&m);
    done = 1;
    Pthread_cond_signal(&c);
    Pthread_mutex_unlock(&m);
}

void *child(void *arg) {
    printf("child\n");
    thr_exit();
    return NULL;
}

void thr_join() {
    Pthread_mutex_lock(&m);
    while (done == 0)
        Pthread_cond_wait(&c, &m);
    Pthread_mutex_unlock(&m);
}

int main(int argc, char *argv[]) {
    printf("parent: begin\n");
    pthread_t p;
    Pthread_create(&p, NULL, child, NULL);
    thr_join();
    printf("parent: end\n");
    return 0;
}
```
- `c`: 조건 변수
- 조건 변수는 `wait()`과 `signal()` 작업과 관계되어 있다.
    - `wait()`: 스레드가 자신을 sleep 상태로 전환하고 싶을 때 수행된다.
    - `signal()`: 어떤 스레드가 프로그램의 무언가를 변경해서, 이 조건이 변경되기를 기다리고 있던 sleep 상태의 스레드를 깨우고 싶을 때 수행된다.
- wait은 매개변수로 mutex를 하나 받는다.
    - `wait()`이 호출되었을 때에는 이 mutex가 잠긴 상태라고 가정한다.
    - `wait()`은 락을 해제하고, 호출한 스레드를 재운다.
    - 스레드가 다른 스레드의 signal로 인해 깨어나면, 락을 다시 점유하고(잠그고) 난 후에 signal을 호출한 곳에 return 된다.
    - 이렇게 하는 이유는 스레드를 sleep 상태로 전환할 때 발생할 수 있는 경쟁 상태를 예방하기 위해서이다.
- 스레드를 재울 때 발생할 수 있는 경쟁 상태
    - 예시1: 싱글 프로세서에서, 부모 스레드가 자식 스레드를 생성하지만 자기 자신(부모 스레드)을 계속 실행할 경우, 자식 스레드가 완료되는 것을 기다리기 위해 즉시 `thr_join()`을 호출한다.
        - 이 경우, 부모 스레드는 락을 점유하고, 자식 스레드가 완료되었는지 확인하고 (완료되지 않았을 경우), `wait()`을 호출해서 자신을 재운다 (그러면서 락을 해제한다).
        - 자식 스레드가 실행된 후, `thr_exit()`을 호출해서 부모 스레드를 깨운다. 그러면 락을 점유해서 상태 변수를 `done`으로 설정하고, 부모 스레드에 signal해서 깨운다. 
        - 부모 스레드는 락을 점유한 상태로 `wait()`으로부터 return해서 실행되고, 락을 해제한다.
    - 예시2: 자식 스레드가 생성 되자마자 실행되서 `done`을 1로 설정하고, 자고 있는 스레드를 깨우기 위해 `signal()`을 호출한다.
        - 하지만 자고 있는 스레드가 없기 때문에, 그냥 return 하고 완료된다.
        - 그 다음 부모 스레드가 실행되서 `thr_join()`을 호출한다. `done`이 1인 것을 확인하고 wait하지 않고 바로 부모 스레드가 실행된다.
- 조건 변수를 사용할 때에는 signal하는 동안 락을 점유하고 있는 것이 좋다. 

## 30.2 The Producer/Consumer (Bounded Buffer) Problem (생산자/소비자 문제, 유한 버퍼 문제)

생산자 스레드가 1개 이상 존재하고 소비자 스레드가 1개 이상 존재한다고 가정한다. 생산자 스레드는 데이터를 생산해서 버퍼에 넣는다. 소비자 스레드는 버퍼에서 데이터를 가져다 쓴다.
유한 버퍼는 한 프로그램(프로세스)의 결과물을 다른 프로그램(프로세스)으로 pipe out할 때 사용된다. UNIX shell은 결과물을 **pipe** 시스템 콜에 의해 생성되는 UNIX pipe에 재전송(redirect)하고, 그 pipe의 다른쪽 끝은 그 결과물 데이터를 사용하고자 하는 프로그램(프로세스)에 연결되어 있다.
유한 버퍼는 공유 자원이기 때문에 경쟁 상태를 방지하기 위해서 유한 버퍼에 대한 접근은 synchronized 되어야 한다.

```c
int buffer;
int count = 0; // initially empty

void put(int value) {
    assert(count == 0);
    count = 1;
    buffer = value;
}

int get() {
    assert(count == 1);
    count = 0;
    return buffer;
}

void *producer(void *arg) {
    int i;
    int loops = (int) arg;
    for (i = 0; i < loops; i++) {
        put(i);
    }
}

void *consumer(void *arg) {
    while (1) {
        int tmp = get();
        printf("%d\n", tmp);
    }
}
```
- `buffer`: 숫자 하나를 담는 공유 버퍼
- `put()`: 버퍼가 비었다고 가정하고, 버퍼에 값을 넣은 후 `count`를 1로 설정해서 버퍼가 꽉 찼다고 표시한다.
- `get()`: 버퍼를 비우고(`count`를 0으로 설정하고), 버퍼의 값을 반환한다.
- `count`가 0일 때에만 버퍼에 값을 쓰고, `count`가 1일 때에만 버퍼에서 값을 가져온다. -> assert에 정의되어 있음.
- `producer`: 생산자 스레드. 공유 버퍼에 `loop`번 만큼 정수 값을 넣는다.
- `consumer`: 소비자 스레드. 공유 버퍼에서 값을 꺼내서, 꺼낸 값을 출력한다 (계속&영원히).

생산자 1명, 소비자 1명인 경우를 가정해보자.
`put()`, `get()`은 임계 구역을 가지고 있다 (`buffer`에 접근). 하지만 `put()`과 `get()`을 호출하는 앞뒤로 락을 설정하는 방법은 유효하지 않다.
```c
int loops;
cond_t cond;
mutex_t mutx;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        if (count == 1)
            Pthread_cond_wait(&cond, &mutex);
        put(i);
        Pthread_cond_signal(&cond);
        Pthread_mutex_unlock(&mutex);
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        if (count == 0)
            Pthread_cond_wait(&cond, &mutex);
        int tmp = get();
        Pthread_cond_signal(&cond);
        Pthread_mutex_unlock(&mutex);
        printf("%d\n", tmp);
    }
}
```
- 생산자가 버퍼를 채우고 싶으면, 버퍼가 빌 때까지 기다린다. 소비자가 버퍼의 데이터를 가져다 쓰고 싶으면, 버퍼가 찰 때까지 기다린다.
- 생산자와 소비자가 각각 1명인 경우에는 위의 코드가 작동한다. 하지만 생산자/소비자 스레드가 하나 이상인 경우(예: 소비자가 2명인 경우)에는 문제가 생긴다.
    - 생산자 1, 소비자 2의 경우
        - 소비자A가 락을 점유하고, 버퍼에 데이터가 있는지 확인한다. 데이터가 없는 것을 확인하면 대기 상태가 되고 락을 해제한다.
        - 생산자가 실행된다. 생산자가 락을 점유하고 버퍼가 꽉 찼는지 확인한다. 버퍼가 꽉 차지 않았다면 버퍼를 채운다.
        - 버퍼를 채운 후 생산자 스레드가 버퍼가 채워졌다고 signal을 호출한다. 그러면 소비자A가 sleep에서 ready 큐로 옮겨져 실행할 준비가 된다.
        - 생산자 스레드는 버퍼가 꽉 찰 때까지 데이터를 넣고, 버퍼가 꽉 차면 sleep 한다.
        - 다른 소비자B가 실행해서 버퍼에 존재하는 값을 사용한다.
        - 그 후 소비자A가 실행되기 위해 `wait()`으로부터 return 하기 직전에 다시 락을 점유한 후 return 한다. 소비자A가 `get()`을 호출하지만, 버퍼에 값이 없어서 (B가 이미 다 씀) assertion이 fire한다.
        - 이 문제가 발생하는 원인은 생산자가 소비자A를 깨운 후, 솝자A가 실행되기 전에 소비자B에 의해 유한 버퍼의 상태가 바뀌기 때문이다. `signal`로 인해 깨어난 스레드가 언제 실행 될 지, 그 스레드가 깨어났을 때 조건 변수가 원하는 상태일지를 보장할 수 없다.
    - `if`를 `while`로 바꾸면 위의 문제를 해결할 수 있다.
        - 소비자A가 깨어나자 마자 공유 자원의 상태를 확인하기 때문에, 버퍼가 비어있다면 다시 잔다. 
- 조건 변수가 1개이기 때문에 발생하는 문제
    - 두 소비자A, B가 먼저 실행되고 둘 다 잠들어 버릴 경우
        - 생산자가 실행되서 버퍼에 값을 넣고, 소비자 중 하나(A)를 깨운다.
        - 생산자는 루프를 반복하면서(락을 풀었다/점유했다를 반복), 버퍼에 데이터를 채운다. 버퍼가 꽉 차면 생산자가 대기가 되어 sleep 상태가 된다.
        - 소비자A는 실행 준비가 된 상태(ready)이고, 자고 있는 스레드는 소비자B와 생산자이다.
        - 소비자A는 `wait()`에서 return 함으로써 깨어나고, 조건을 다시 확인한 후 버퍼가 꽉 찼기 때문에 버퍼의 값을 소비한다. 버퍼의 값을 소비한 후 소비자A는 `signal()`을 통해 스레드를 깨우는데, 단 하나의 스레드만을 깨운다. 이 때 어떤 스레드를 깨워야 하는가?
        - 소비자가 버퍼를 모두 비웠기 때문에 생산자를 깨워야 한다. 하지만 만약 소비자B를 깨운다면(같은 대기 큐에 적재되어 있을 경우 그럴 수 있음) 문제가 발생한다.
        - 소비자B가 깨어나서 버퍼가 빈 것을 확인하면 다시 잠든다.
        - 생산자 스레드는 계속 잠든 상태이다.
        - 소비자A도 잠든다.
        - 모든 스레드가 잠든 상태가 된다.
    - 조건 변수를 2개 활용해서 위의 문제를 해결할 수 있다.
        - 조건 변수를 `empty`, `fill` 2개 선언한다.
        - 생산자 스레드는 `empty`를 기다리고(wait), `fill`을 알린다(signal).
        - 소비자 스레드는 `fill`을 기다리고, `empty`를 알린다.
        - 이렇게 하면 소비자가 소비자를 깨우거나, 생산자가 생산자를 깨우는 일이 발생하지 않는다.
- 생산자/소비자 문제를 일반적으로 해결하기 위해서는 (more concurrent & efficient)
    - 버퍼의 구조와 `put()`, `get()`을 수정해야 한다.
    - 생산자는 모든 버퍼가 채워져 있을 때에만 sleep 하도록, 소비자는 모든 버퍼가 비워져 있을 때에만 sleep 하도록 로직을 수정한다.
