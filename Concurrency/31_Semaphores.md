# 31 Semaphores

세마포어를 락이나 조건 변수 대신 사용하는 방법
세마포어의 정의
이진 세마포어란
락과 조건 변수를 이용해서 세마포어를 만들거나, 그 역이 가능한가?

## 31.1 Semaphores: A Definition

세마포어: 두 가지 루틴을 활용해서 조작할 수 있는 정수 값을 가진 객체
- 두 가지 루틴: `wait()`, `post()`
- 세마포어의 초기값이 세마포어의 behavior를 결정하기 때문에, 루틴을 호출해서 세마포어를 조작하기 전에 반드시 세마포어를 어떤 값으로 초기화해야 한다.
```c
sem_t s;
sem_init (&s, 0, 1);
```
- 세마포어 `s`를 선언한 후, 세마포어의 값을 `1`로 초기화한다.
- `sem_init`의 두 번째 매개변수 `0`은 세마포어가 같은 프로세스의 스레드끼리는 공유된다는 것을 의미한다.
- 세마포어를 초기화 하고 난 후에는 세마포어를 조작하기 위해 `sem_wait()` 또는 `sem_post()` 함수를 호출할 수 있다.

```c
init sem_wait(sem_t *s) {
    // s -= 1
    // wait if s < 0
}

init sem_post(sem_t *s) {
    // s += 1
    // if there are one or more threads waiting, wake one
}
```
- `sem_wait()`
    - `sem_wait()`을 호출했을 때 세마포어의 값이 1 이상일 경우 바로 return
    - 아닐 경우 `post()` 될 때까지 호출자의 실행을 대기시킨다.
    - 여러 개의 스레드가 `sem_wait()`을 호출하면 그 스레드들은 큐에 들어간다.
- `sem_post()`
    - 세마포어의 값을 + 1한 후, 대기중인 스레드가 있다면 대기중인 스레드 중 하나를 깨운다.
- 세마포어의 값이 음수일 경우, 그 값은 대기 중인 스레드의 개수와 동일하다.

## 31.2 Binary Semaphores (Locks)

세마포어를 락으로 사용하기
```c
sem_t m;
sem_init(&m, 0, X);

sem_wait(&m);
// critical section
sem_post(&m);
```
- 임계 구역을 `sem_wait()`과 `sem_post()`의 쌍으로 감싼다.
- 세마포어의 초기값(`X`)이 `1`이어야 한다.

락으로 사용되는 세마포어를 이진 세마포어라고도 부른다 (락은 점유/비점유 두 가지 상태만 존재하기 때문).

## 31.3 Semaphores for Ordering

동시적인 프로그램에서 이벤트의 순서를 정할 때(순서대로 진행되게 할 때)에도 세마포어를 사용한다.
- 예: 어떤 스레드가 어떤 리스트가 빈 상태가 아니기를 기다리고 싶을 경우 (리스트에서 원소 하나를 삭제하기 위해서)
    - 1번 스레드는 어떤 일이 일어나기를 기다린다(**wait**).
    - 다른 스레드가 그 일을 일어나게 하고, 그 일이 일어났다는 것을 알려서(**signal**), 대기 중인 1번 스레드를 깨운다.
    - 이렇게 사용한 세마포어는 조건 변수와 비슷하게 동작한다는 것을 알 수 있다.

```c
sem_t s;

void *child(void *arg) {
    printf("child\n");
    sem_post(&s); // signal: child is done
    return NULL;
}

int main(int args, char *argv[]) {
    sem_init(&s, 0, X);
    printf("parent: begin\n");
    pthread_t c;
    Pthread_create(&c, NULL, child, NULL);
    sem_wait(&s); // wait for child
    printf("parent: end\n");
    return 0;
}
```
- 메인 스레드(부모)에서 새로운 스레드(자식)을 생성한다.
- 부모 스레드는 자식 스레드가 완료될 때까지 기다린다(`sem_wait`).
- 자식 스레드가 완료되면 `sem_post`로 알린다.
- 세마포어의 초기값 `X`는 몇으로 설정되어야 하는가? `0`으로 설정되어야 한다.
    - 케이스 1: 부모 스레드가 자식 스레드를 생성하지만, 자식 스레드가 아직 실행되지 않은 경우
        - 부모 스레드는 자식 스레드가 `sem_post()`를 호출하기 전에 `sem_wait()`을 호출한다.
        - 부모 스레드는 자식 스레드가 실행되는 것을 기다려야 하는데, 세마포어의 값이 0보다 작거나 같을 때에만 기다린다 (세마포어 -= 1 한 후 음수일 때에만 wait하기 때문).
        - 따라서 세마포어의 초기값은 0보다 클 수 없다.
    - 케이스 2: 부모 스레드가 `sem_wait()`을 호출하기 전에 자식 스레드가 실행&완료될 경우
        - 자식 스레드가 `sem_post()`를 호출해서 세마포어의 값을 += 1 (0에서 1이 된다).
        - 부모 스레드가 `sem_wait()`을 호출하면 세마포어의 값이 `1`이기 때문에, 세마포어 -= 1을 해도 0이기 때문에 대기하지 않고 바로 `wait` 함수에서 return 한다.

세마포어의 초기값을 어떻게 결정해야 할까?
세마포어를 초기화 한 후에 바로 주고 싶은 리소스의 개수를 고려해야 한다.
- 락의 경우, 초기화 한 후에 바로 락을 걸고 싶기 때문에 세마포어의 값을 1로 설정했다.
- 순서를 정하는 경우(조건 변수처럼 사용한 경우), 초기화 한 후 바로 수행할 것이 없기 때문에 (자식 스레드가 완료된 후에 자원이 생기기 때문에) 0으로 설정했다.

## 31.4 The Producer/Consumer (Bounded Buffer) Problem

### 해결 방안 1: 세마포어 2개 사용하기
`empty`와 `full` 두 개의 세마포어를 사용해서 각각 버퍼가 빈 상태인지 꽉 찬 상태인지를 확인한다.

```c
int buffer[MAX];
int fill = 0;
int use = 0;

void put(int value) {
    buffer[fill] = value;
    fill = (fill + 1) % MAX;
}

int get() {
    int tmp = buffer[use];
    use = (use + 1) % MAX;
    return tmp;
}

sem_t empty;
sem_t full;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty);
        put(i);
        sem_post(&full);
    }
}

void *consumer(void *arg) {
    int tmp = 0;
    while (tmp !- -1) {
        sem_wait(&full);
        tmp = get();
        sem_post(&empty);
        printf("%d\n", tmp);
    }
}

int main(int argc, char *argv[]) {
    ...
    sem_init(&empty, 0, MAX); // MAX are empty
    sem_init(&full, 0, 0); // 0 are full
    ...
}
```
- 소비자는 버퍼가 빌 때까지 기다린 후 데이터를 버퍼에 넣는다.
- 생산자는 버퍼를 사용하기 전에, 버퍼가 차기를 기다린다.
- `MAX = 1`인 경우 (버퍼의 크기가 1인 경우)
    - 두 개의 스레드(생산자1, 소비자1)이 존재하는 경우
        - 만약 소비자 스레드가 먼저 실행된다면, `sem_wait(&full)`을 호출한다.
        - `full`은 0으로 초기화 되었기 때문에(`sem_init(&full, 0, 0)`), `wait()` 호출이 `full`의 값을 -= 1 해서 `full`의 값은 -1이 되고, 소비자 스레드가 잠들어 다른 스레드가 `sem_post(&full)`을 호출하기를 기다리게 된다.
        - 그 다음 생산자 스레드가 실행되어 `sem_wait(&empty)`를 호출한다.
        - `empty`의 값은 `MAX`로 초기화되었기 때문에, 생산자는 `empty`의 값을 -= 1해서 `empty`의 값은 0이 되고 버퍼에 데이터 1개를 넣는다.
        - 그 다음 생산자 스레드는 `sem_post(&full)`을 호출해서 `full`의 값을 += 1해서 0으로 만들고, 대기 중인 소비자 스레드를 깨운다 (blocked 큐에서 ready 큐로 옮긴다).
        - 이 때 두 가지 일이 일어날 수 있다. 생산자 스레드가 계속 실행될 경우 `empty`의 값이 `0`이기 때문에 `empty` -= 1은 -1이 되어 생산자 스레드가 blocked 상태가 된다. 만약 인터럽트가 발생해서 소비자 스레드가 실행된 경우, `full`의 값이 0이기 때문에 `wait()`에서 return해서 버퍼의 데이터를 사용한다.
- `MAX`의 값이 1보다 큰 경우
    - 생산자와 소비자 스레드가 모두 여러 개 존재하는 경우
        - 경쟁 상태가 발생한다.
        - 생산자1과 생산자2가 거의 동시에 `put()`을 호출하면, 생산자1이 먼저 호출되어 버퍼에 데이터를 채우고 `fill`에 += 1 하기 전에 인터럽트 될 경우, 생산자2가 실행되어 생산자1이 버퍼의 첫 인덱스에 채운 요소를 overwrite하게 된다.

### 해결 방안: 상호 배제 추가
버퍼를 채우는 코드와 index 값을 증가시키는 코드는 임계 구역이기 때문에, 이진 세마포어를 사용하고 락을 추가해서 해결할 수 있다.

```c
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty);
        sem_wait(&mutex);
        put(i);
        sem_post(&mutex);
        sem_post(&full);
    }
}

void *consumer(void *arg) {
    int tmp = 0;
    while (tmp !- -1) {
        sem_wait(&full);
        sem_wait(&mutex);
        tmp = get();
        sem_post(&mutex);
        sem_post(&empty);
        printf("%d\n", tmp);
    }
}
```
- `mutex`를 잠그고 푸는 것을 `full`/`empty`의 값을 변경하는 코드 밖에 작성할 경우 데드락이 발생한다.
    - 사용자 스레드가 먼저 실행될 경우, 사용자가 `wait(&mutex)`를 통해 락을 점유한 후 `wait(&full)`을 호출해서 대기 상태에 들어갈 경우, 생산자 스레드가 락을 점유할 수 없어 데이터를 생산할 수 없어 데드락 상태가 된다.
- 위와 같이 `mutex` 락이 잠그는 범위를 임계 구역 바로 앞뒤로 설정해서 해결할 수 있다.


## 31.5 Reader-Writer Locks

리스트에 데이터를 추가하는 작업과 리스트의 데이터를 확인하는 작업이 있는 경우, 데이터를 추가하는 작업은 리스트의 상태를 변경하지만 데이터를 확인하는 작업은 리스트의 상태를 변경하지 않는다. 따라서 데이터를 확인하는 작업은 동시에 여러 개가 수행되어도 된다. 이런 작업을 수행하기 위한 락을 **Reader-Writer Lock**이라고 한다.

```c
typedef struct _rwlock_t {
    sem_t lock; // binary semaphore (basic lock)
    sem_t writelock; // allow ONE writer / MANY readers
    int readers; // readers in critical sectio
} _rwlock_t;

void rwlock_init(rwlock_t *rw) {
    rw->readers = 0;
    sem_init(&rw->lock, 0, 1);
    sem_init(&rw->writelock, 0, 1);
}

void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers++;
    if (rw->readers == 1) // first reader gets writelock
        sem_wait(&rw->writelock);
    sem_post(&rw->lock);
}

void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers--;
    if (rw->readers == 0) // last reader leaves
        sem_post(&rw->writelock);
    sem_post(&rw->lock);
}

void rwlock_acquire_writelock(rwlock_t *rw) {
    sem_wait(&rw->writelock);
}

void rwlock_release_writelock(rwlock_t *rw) {
    sem_post(&rw->writelock);
}
```
- 어떤 스레드가 데이터를 추가하고 싶을 경우
    - `rw_acquire_writelock()`을 호출해서 write lock을 점유하고, `rwlock_release_writelock()`을 사용해서 락을 해제한다. 이 두 루틴은 오직 하나의 생산자 스레드만이 락을 점유해서 임계 구역에 진입할 수 있도록 한다.
    - 소비자(read only) 스레드가 락을 점유할 때, 먼저 이진 세마포어인 `lock`을 먼저 점유하고, `readers`를 += 1한다.
    - 첫 번째 소비자가 `rw-acquire_readlock()`을 점유하면, `sem_wait(&rw->writelock)`을 통해 `writelock`도 같이 점유해서 생산자 스레드가 데이터를 쓰는 것을 막는다.
    - 그 후 `sem_post(&rw->lock)`으로 `lock`을 해제해서 다음 소비자(read only)가 락을 점유할 수 있게 한다.
    - 마지막 소비자가 락을 해제할 때에는 `sem_post(&rw->writelock)`을 통해 `writelock`을 해제해서 생산자 스레드가 데이터를 쓸 수 있게 한다.
- 위와 같은 접근 방법은 공정성 측면에서 단점이 있다. 소비자 스레드에 의해 생산자 스레드가 기아 상태가 될 확률이 높기 때문이다. 이를 해결하기 위해 생산자가 대기 중일 때에는 소비자가 락을 새롭게 점유하지 않도록 하는 방법도 있다.
- 오버헤드를 더 추가하는 경향이 있기 때문에 간단한 락(spin-lock)을 사용하는 경우만큼 성능 개선이 이루어지지 않을 수 있다.

## 31.6 The Dining Philosopher's Problem

원형 식탁에 앉은 각 철학자 사이에 1개의 포크만이 존재하며 (철학자의 수 = 포크의 수), 각 철학자는 식사를 하지 않을 때에는 포크를 사용하지 않고 식사를 하기 위해서는 1명당 포크가 2개씩 필요하다.
-> 동시성 프로그래밍에서 일어나는 동기화 문제

각 철학자의 loop는 다음과 같이 나타낼 수 있다
```c
while (1) {
    think();
    get_forks(p);
    eat();
    put_forks(p);
}
```
- 각 철학자는 자신만의 unique한 스레드 `p0`~`p4`를 가진다고 가정한다.
- 교착 상태가 발생하지 않도록 `get_forks()`와 `put_forks()`를 어떻게 구현할 것인가?

```c
int left(int p) { return p; }
int right(int p) { return (p + 1) % 5; }
sem_t forks[5];
```
- 철학자 `p`가 왼쪽 포크를 집을 때는 `left(p)`를, 오른쪽 포크를 집을 때는 `right(p)`를 호출한다.
- 세마포어가 각 포크별로 1개씩 존재한다고 가정하자 (총 세마포어의 개수 = 5개).

```c
void get_forks(int p) {
    sem_wait(&forks[left(p)]);
    sem_wait(&forks[right(p)]);
}

void put_forks(int p) {
    sem_post(&forks[left(p)]);
    sem_post(&forks[right(p)]);
}
```
- 위와 같이 작성할 경우, 어느 누구도 오른쪽 포크를 집기 전에 모두가 왼쪽 포크를 집는다면 데드락이 발생한다.

### 해결 방법: Breaking the Dependency
```c
void get_forks(int p) {
    if (p == 4) {
        sem_wait(&forks[right(p)]);
        sem_wait(&forks[left(p)]);
    } else {
        sem_wait(&forks[left(p)]);
        sem_wait(&forks[right(p)]);
    }
}
```
- 철학자 중 최소 1명이 포크를 집는 방법을 바꿔서 해결할 수 있다.
- 위의 경우 5번째 철학자만 오른쪽 포크를 먼저 잡도록 해서 만약 0번 포크가 점유된 상태일 경우 5번째 철학자는 왼쪽/오른쪽을 다 잡지 않기 때문에 데드락이 해결될 수 있다.

## 31.7 Thread Throttling

너무 많은 스레드가 한 번에 작업을 해서 시스템을 다운시키는 상황을 어떻게 예방할 것인가?
"너무 많은"의 값을 정하고, 세마포어를 사용해서 해당 코드를 동시에 수행하는 스레드의 수를 제한한다.
이러한 접근법을 **스로틀링**(**Throttling**)이라고 한다.
