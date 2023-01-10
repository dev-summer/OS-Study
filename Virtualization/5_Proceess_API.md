# 5 Process API

유닉스 시스템에서 프로세스를 어떻게 생성하는지 알아볼것임

## 5.1 `fork()` System Call

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

프로세스가 `fork()` 시스템 호출을 호출해서 OS가 새로운 프로세스를 생성한다. 생성된 프로세스는 `fork()`를 호출한 프로세스와 거의 똑같은 복사본이다. OS에게는 `p1`이라는 프로그램의 복사본 두 개가 실행되고 있는 것처럼 보이고, 두 프로세스 모두 `fork()` 시스템 호출에서 반환하기 직전이다.
- 새로 생성된 자식 프로세스는 `main()`에서 실행을 시작하지 않고, 자기 자신(`child` 프로세스)가 `fork()`를 호출한 뒤에야 실행된다.
- `child`와 `parent`의 실행 순서는 보장되지 않는다 (위 예시에서는 `parent`가 먼저 출력되었지만, `child`가 먼저 출력될 수도 있음).
- `child`가 `parent`와 동일한 복사본이 아니다. `child`는 자신만의 주소 공간, 레지스터, PC 등을 가지며, `fork()`를 호출한 대상에게 반환해주는 값이 다르다. 부모 프로세스는 새로 생성된 자식 프로세스의 PID를 받고, 자식 프로세스는 반환 코드 0을 받는다.

## 5.2 `wait()` System Call

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *arvg[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
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
hello world (pid:3576)
hello, I am child (pid:3577)
hello, I am parent of 3577 (rc_wait:3577) (pid:3576)
```

부모 프로세스가 `wait()`을 호출해서 자식 프로세스의 실행이 끝날 때까지 부모 프로세스의 실행을 미룬다. 자식 프로세스가 끝나면 `wait()`은 부모 프로세스로 돌아간다.
- 반드시 `child`가 먼저 실행되는 것을 보장할 수 있다. 만약 `parent`가 먼저 실행될 경우, `wiat()`을 호출해서 `child`가 실행되고 exit할 때까지 기다린다.

### 5.3 `exec()` System Call

`exec()`을 호출하는 프로그램과 다른 프로그램을 실행하고 싶을 때 사용하는 시스템 호출.
- `fork()`를 호출하면 같은 프로그램이 하나 더 실행된다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child process
        printf("hello, I am child (pid:%d)\n", (int) getpid());
        char *myargs[3];
        myargs[0] = strdup("wc"); // program: "wc" (word count)
        myargs[1] = strdup("p3.c"); // argument: file to count
        myargs[2] = NULL; // marks end of array
        execvp(myargs[0], myargs); // runs word count
        printf("this shouldn't print out");
    } else {
        int rc_wait = wait(NULL);
        printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int) getpid());
    }
    return 0;
}
```
```
hello world (pid:3867)
hello, I am child (pid:3868)
wc: p3.c: open: No such file or directory
hello, I am parent of 3868 (rc_wait:3868) (pid:3867)
```

자식 프로세스가 `execvp()`를 호출해서 `wc`라는 프로그램을 실행한다.
- `wc`라는 프로그램은 소스파일 `p3.c`에 존재하고, 파일이 몇 줄, 몇 단어, 몇 바이트로 이루어졌는지를 알려준다.
- `exec()`에게 executable(실행 가능한 것? `wc`)의 이름과 인자(`p3.c`)를 주면, 그 executable에서 코드와 정적 데이터를 로드하고, 현재의 코드 세그먼트와 현재의 정적 데이터를 그걸로 덮어쓴다. 그리고 프로그램의 힙, 스택과 같은 메모리 공간이 재초기화(re-initialize)된다. 그 다음에 OS는 인자를 해당 프로세스의 `argv`로 전달해서 그 프로그램을 실행한다. 그러므로, **새로운 프로세스를 생성하지 않고**, 현재 실행중인 프로그램(`p3`)을 다른 실행중인 프로그램(`wc`)으로 변형시킨다(transform). 자식 프로세스의 `exec()` 후에는 `p3.c`가 실행된 적이 없는 것과 같다. `exec()`를 성공적으로 호춣하면 return 하지 않는다.

### 5.4 Motivating the API

`fork()`랑 `exec()`이 분리되어 있기 때문에, `fork()` 다음 & `exec()` 이전에 오는 코드를 shell이 실행할 수 있게 해주기 때문이다.
shell은 사용자 프로그램이다(shell에는 tcsh, bash, zsh 등이 있음). shell은 사용자에게 프롬프트를 보여주고, 사용자가 프롬프트에 무언가를 입력하기를 기다린다. 프롬프트에 커맨드를 입력하면 shell이 executable이 파일 시스템의 어디에 있는지 알아내고 `fork()`를 호출해서 새로운 자식 프로세스를 생성해서 커맨드를 실행하고, `exec()`을 호출해서 커맨드를 실행하고, 그리고 `wait()`을 호출해서 커맨드가 완료되기를 기다린다. 자식 프로세스가 완료되면, shell은 `wait()`에서 return해서 새로운 프롬프트를 프린트하고 사용자가 다음 커맨드를 입력하기를 기다린다.
위의 예시에서 프로그램 `wc`의 결과물은 산출된 파일 `newfile.txt`로 리다이렉트 된다. shell은 이 작업을 다음과 같이 수행한다. 자식 프로세스가 생성되면, `exec()`을 호출하기 전에 shell은 standard output을 닫고, `newfile.txt`를 연다. 그렇게 함으로써 곧 실행될 프로그램 `wc`의 산출물은 모두 화면이 아닌 파일로 보내진다.
