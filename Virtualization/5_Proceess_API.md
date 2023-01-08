# 5 Process API

유닉스 시스템에서 프로세스를 어떻게 생성하는지 알아볼것임

## 5.1 The `fork()` System Call

`fork()` 시스템 호출은 새로운 프로세스를 생성하는데 사용한다.

```c
// p1.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *arvg[]) {
    printf("hello world (pid%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child process
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (main)
        printf("hello, I am parent of %d (pid:%d)\n", rc, (int) getpid());
    }
    return 0;
}
```
```
hello world (pid40755)
hello, I am parent of 40765 (pid:40755)
hello, I am child (pid:40765)
```

프로세스가 `fork()` 시스템 호출을 호출해서 OS가 새로운 프로세스를 생성한다. 생성된 프로세스는 `fork()`를 호출한 프로세스와 거의 똑같은 복사본이다. OS에게는 `p1`이라는 프로그램의 복사본 두 개가 실행되고 있는 것처럼 보이고, 두 프로세스 모두 `fork()` 시스템 호출에서 반환하기 직전이다. 새로 생성된 자식 프로세스는 `main()`에서 실행을 시작하지 않고, 자기 자신(`child` 프로세스)가 `fork()`를 호출한 뒤에야 실행된다. `child`와 `parent`의 실행 순서는 보장되지 않는다 (위 예시에서는 `parent`가 먼저 출력되었지만, `child`가 먼저 출력될 수도 있음).
→ `child`가 `parent`와 동일한 복사본이 아니다. `child`는 자신만의 주소 공간, 레지스터, PC 등을 가지며, `fork()`를 호출한 대상에게 반환해주는 값이 다르다. 부모 프로세스가 새로 생성된 자식 프로세스의 PID를 받는 동안, 자식 프로세스는 반환 코드 0을 받는다.



## 5.2 The `wait()` System Call

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *arvg[]) {
    printf("hello world (pid%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child process
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (main)
        int rc_wait = wait(NULL);
        printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int) getpid());
    }
    return 0;
}
```
```
hello world (pid41185)
hello, I am child (pid:41198)
hello, I am parent of 41198 (rc_wait:-1) (pid:41185)
```
