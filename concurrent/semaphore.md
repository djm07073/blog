### 3.4 Semaphore
- 세마포어는 mutex의 일반화된 버전으로 최대 N개의 프로세스가 동시에 락을 획득을 할 수 있습니다.
```c
#define NUM 4

void semaphore_acquire(volatile int *cnt) { // 1
    for (;;) {
        while (*cnt >= NUM); // 2 : cnt가 NUM보다 크면 스핀락 상태
        __sync_fetch_and_add(cnt, 1); // 3
        if (*cnt <= NUM) // 4 : cnt가 NUM보다 작으면 락을 획득
            break;
        __sync_fetch_and_sub(cnt, 1); // 5
    }
}

void semaphore_release(int *cnt) {
    __sync_fetch_and_sub(cnt, 1); // 6: 락을 해제
}

#include "semtest.c"
```
- 위 코드처럼 TTAS 방법을 사용해 현재 cnt를 계속해서 확인하는 스핀 락 상태가 되고, cnt가 줄어드면 critical section을 실행할 수 있게 됩니다.
- `__sync_fetch_and_sub` 와 `__sync_fetch_and_add`는 atomic operation으로 증가된 cnt 값이 NUM을 초과한 경우, cnt 값을 원자적으로 감소시키고 다시 시도하게 되어 cnt의 race condition을 방지할 수 있습니다.
- 더 자세한 설명을 위해 본 코드를 컴파일 된 코드를 확인해보면,
```assembly
.LBB0_1:
    ldr     w8, [x0]     // while (*x0 > 3); ❶
    cmp     w8, #3
    b.hi    .LBB0_1
.Ltmp1:
    ldaxr   w2, [x0]     // w2 = [x0] ❷  // ll 명령어로 x0에 해당하는 데이터를 load
    cmp     w2, #4
    b.lo    .Ltmp2       // if (w2 < 4) then goto .Ltmp2 ❸
    clrex                // clear exclusive
    b       .LBB0_1      // goto .LBB0_1
.Ltmp2:
    add     w2, w2, #1   // w2 = w2 + 1 ❹
    stlxr   w3, w2, [x0] // [x0] = w2 // sc 명령어로 x0에 해당하는 데이터를 write
    cbnz    w3, .Ltmp1   // if (w3 != 0) then goto .Ltmp1
    ret
```
- 어셈블리어로 본 코드를 보면 크게 두개의 루프를 돌면서 세마포어가 구현된것을 알 수 있습니다.
  - 1. TTAS 작업을 위해 `w2`(cnt) 에 대한 스핀락을 수행합니다.
  - 2. ldaxr / stlxr와 같은 하드웨어 레벨에서 atomic operation을 보장하는 방법을 수행하고 성공할때까지 루프를 돌게 됩니다.

### POSIX Semaphore
- pthread api를 이용해 구현된 POSIX Semaphore는 아래와 같이 구현할 수 있습니다.
```c
#include <pthread.h> // ❶
#include <fcntl.h>
#include <sys/stat.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define NUM_THREADS 10 // 스레드 수
#define NUM_LOOP 10    // 스레드 안의 루프 수

int count = 0; // ❷

void *th(void *arg) { // 스레드용 함수
    // 이름이 있는 세마포를 연다 ❸
    sem_t *s = sem_open("/mysemaphore", 0);
    if (s == SEM_FAILED) {
        perror("sem_open");
        exit(1);
    }

    for (int i = 0; i < NUM_LOOP; i++) {
        // 대기 ❹
        if (sem_wait(s) == -1) {
            perror("sem_wait");
            exit(1);
        }

        // 카운터를 아토믹하게 인크리먼트
        __sync_fetch_and_add(&count, 1);
        printf("count = %d\n", count);

        // 10ms 슬립
        usleep(10000);
        // critical section

        // 카운터를 아토믹하기 디크리먼트
        __sync_fetch_and_sub(&count, 1);

        // 세마포 값을 증가시키고 ❺
        // 크리티컬 섹션에서 벗어난다
        if (sem_post(s) == -1) {
            perror("sem_post");
            exit(1);
        }
    }

    // 세마포를 닫는다 ❻
    if (sem_close(s) == -1)
        perror("sem_close");

    return NULL;
}

int main(int argc, char *argv[]) {
    // 이름이 붙은 세마보를 연다. 세마포가 없을 때는 생성한다.
    // 자신과 그룹을 이용할 수 있는 세미포로, 
    // 크리티컬 섹션에 들어갈 수 있는 프로세스는 최대 3개이다. ❼
    sem_t *s = sem_open("/mysemaphore", O_CREAT, 0660, 3);
    if (s == SEM_FAILED) {
        perror("sem_open");
        return 1;
    }

    // 스레드 생성
    pthread_t v[NUM_THREADS];
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&v[i], NULL, th, NULL);
    }

    // join
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(v[i], NULL);
    }

    // 세마포를 닫는다
    if (sem_close(s) == -1)
        perror("sem_close");

    // 세마포 파기 ❽
    if (sem_unlink("/mysemaphore") == -1)
        perror("sem_unlink");

    return 0;
}
```