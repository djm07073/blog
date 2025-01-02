### Readers-Writer Lock
- race condition이 발생하는 이유는 쓰기 작업 때문이라고 할 수 있습니다. 그래서 Read와 Write 작업을 하는 프로세스를 분리하여 Read락을 획득한 프로세스끼리는 동시성을 가질 수 있도록 합니다.
- 다음과 같은 특징이 있습니다.
  - 1. 락을 획득 중인 Reader는 같은 시각에 다수 존재할 수 있다.
  - 2. 락을 획득 중인 Writer는 같은 시각에 1개만 존재할 수 있다.
  - 3. Reader와 Writer는 같은 시각에 락획득 상태가 될 수 없다.
- Reader용 락 획득과 Wrtier 용 락획득을 분리하여 적절하게 판단해서 사용하면 됩니다.
```c
// Reader용 록 획득 함수 ❶
void rwlock_read_acquire(int *rcnt, volatile int *wcnt) {
    for (;;) {
        while (*wcnt); // Writer가 있으면 대기 ❷
        __sync_fetch_and_add(rcnt, 1); // ❸
        if (*wcnt == 0) // Writer가 없으면 록 획득 ❹
            break;
        __sync_fetch_and_sub(rcnt, 1);
    }
}

// Reader용 록 반환 함수 ❺
void rwlock_read_release(int *rcnt) {
    __sync_fetch_and_sub(rcnt, 1);
}

// Writer용 록 획득 함수 ❻
void rwlock_write_acquire(bool *lock, volatile int *rcnt, int *wcnt) {
    __sync_fetch_and_add(wcnt, 1); // ❼
    while (*rcnt); // Reader가 있으면 대기
    spinlock_acquire(lock); // ❽
}

// Writer용 록 반환 함수 ❾
void rwlock_write_release(bool *lock, int *wcnt) {
    spinlock_release(lock);
    __sync_fetch_and_sub(wcnt, 1);
}

// 공유 변수
int  rcnt = 0;
int  wcnt = 0;
bool lock = false;

void reader() { // Reader용 함수
    for (;;) {
        rwlock_read_acquire(&rcnt, &wcnt);
        // 크리티컬 섹션(읽기만)
        rwlock_read_release(&rcnt);
    }
}

void writer () { // Writer용 함수
    for (;;) {
        rwlock_write_acquire(&lock, &rcnt, &wcnt);
        // 크리티컬 섹션(읽기 및 쓰기)
        rwlock_write_release(&lock, &wcnt);
    }
}
```
- c언어 구현을 보면 `wcnt`와 `rcnt`를 분리해서 관리하게 됩니다.
### [요약]
- rw락을 이용해야하는 상황은 대부분 읽기 처리이며 쓰기는 거의 일어나지않는 상황에 쓰는게 좋습니다.
- 쓰기 작업이 빈번하게 일어나면 읽기 작업이 전혀 실행하지 못하는 현상이 발생하기 때문에 쓰기작업이 많은 경우에는 mutex를 사용하는것이 좋습니다.