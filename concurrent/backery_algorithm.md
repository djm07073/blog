### Backery Algorithm
- LL/SC등 CPU가 제공하는 아토믹 명령을 이용한 동기처리 방법이 아닌, 이를 지원하지 않는 CPU에서도 할수 있는 동기처리 방법입니다.
- 프로세스 혹은 스레드가 티켓(순서)을 부여 받고,다른 티켓을 부여받은 어느 사람보다 순서가 빠르면 실행하게 되는 동기처리 방법입니다.
- 쉽게 말하자면, 번호표 알고리즘입니다.
```Rust
// 최적화 억제 읽기/쓰기용
use std::ptr::{read_volatile, write_volatile}; // ❶
// 메모리 배리어용
use std::sync::atomic::{fence, Ordering}; // ❷
use std::thread;

const NUM_THREADS: usize = 4;   // 스레드 수
const NUM_LOOP: usize = 100000; // 각 스레드에서의 루프 수

// volatile용 매크로 ❸ -> 메모리 읽기/쓰기 순서가 아웃오브 오더를 수행하는 CPU 때문에 꼬일 수 있어. 해당 메크로만 메모리를 접근 
macro_rules! read_mem {
    ($addr: expr) => { unsafe { read_volatile($addr) } };
}

macro_rules! write_mem {
    ($addr: expr, $val: expr) => {
        unsafe { write_volatile($addr, $val) }
    };
}

// 베이커리 알고리즘용 타입 ❹
struct BakeryLock {
    entering: [bool; NUM_THREADS],
    tickets: [Option<u64>; NUM_THREADS],
}

impl BakeryLock {
    // 록 함수. idx는 스레드 번호
    fn lock(&mut self, idx: usize) -> LockGuard {
        // 여기부터 티켓 취득 처리 ❺
        fence(Ordering::SeqCst); // 메모리 베리어를 수행해 아웃 오브 오더에서 메모리를 읽고 쓰는 것을 방지
        write_mem!(&mut self.entering[idx], true);
        fence(Ordering::SeqCst);

        // 현재 배포되어 있는 티켓의 최댓값 취득 ❻
        let mut max = 0;
        for i in 0..NUM_THREADS {
            if let Some(t) = read_mem!(&self.tickets[i]) {
                max = max.max(t);
            }
        }
        // 최댓값 + 1을 자신의 티켓 번호로 한다 ❼
        let ticket = max + 1;
        write_mem!(&mut self.tickets[idx], Some(ticket));

        fence(Ordering::SeqCst);
        write_mem!(&mut self.entering[idx], false); // ❽
        fence(Ordering::SeqCst);

        // 여기부터 대기 처리 ❾
        for i in 0..NUM_THREADS {
            if i == idx {
                continue;
            }

            // 스레드 i가 티켓 취득 중이면 대기
            while read_mem!(&self.entering[i]) {} // ❿

            loop {
                // 스레드 i와 자신의 우선 순위를 비교해
                // 자신의 우선 순위가 높거나,
                // 스레드 i가 처리 중이 아니면 대기 종료 ⓫
                match read_mem!(&self.tickets[i]) {
                    Some(t) => {
                        // 스레드 i의 티켓 번호보다
                        // 자신의 번호가 낮거나,
                        // 티멧 번호가 같고
                        // 자신의 스레드 번호가 작으면
                        // 대기 종료
                        if ticket < t ||
                           (ticket == t && idx < i) {
                            break;
                        }
                    }
                    None => {
                        // 스레드 i가 처리 중이 아니면
                        // 대기 종료
                        break;
                    }
                }
            }
        }

        fence(Ordering::SeqCst);
        LockGuard { idx }
    }
}

// 록 관리용 타입 ⓬
struct LockGuard {
    idx: usize,
}

impl Drop for LockGuard {
    // 록 해제 처리 ⓭
    fn drop(&mut self) {
        fence(Ordering::SeqCst);
        write_mem!(&mut LOCK.tickets[self.idx], None);
    }
}

// 글로벌 변수 ⓮
static mut LOCK: BakeryLock = BakeryLock {
    entering: [false; NUM_THREADS],
    tickets: [None; NUM_THREADS],
};

static mut COUNT: u64 = 0;

fn main() {
    // NUM_THREADS만큼 스레드를 생성
    let mut v = Vec::new();
    for i in 0..NUM_THREADS {
        let th = thread::spawn(move || {
            // NUM_LOOP 만큼 루프 반복하면서 COUNT를 인크리먼트
            for _ in 0..NUM_LOOP {
                // 록 획득
                let _lock = unsafe { LOCK.lock(i) };
                unsafe {
                    let c = read_volatile(&COUNT);
                    write_volatile(&mut COUNT, c + 1);
                }
            }
        });
        v.push(th);
    }

    for th in v {
        th.join().unwrap();
    }

    println!(
        "COUNT = {} (expected = {})",
        unsafe { COUNT },
        NUM_LOOP * NUM_THREADS
    );
}
```
- 아토믹 명령어를 사용하지 않는 CPU에서 사용하다보니 더 많은 스핀락이 필요해서 더 많은 리소스를 잡아먹게 됩니다.
- 그리고 아웃 오브 오더 실행을 막아 성능 저하가 우려될 수 있습니다.