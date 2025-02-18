## 러스트의 동시성 프로그래밍
rust에서는 동시성 프로그래밍을 지원하기 위한 스레드, 채널, async/await 등 다양한 기능이 제공됩니다.

```rust
use std::fs::File;
use std::io::{BufReader, BufRead};
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        let file = File::open("file.txt").unwrap();
        let reader = BufReader::new(file);
        for line in reader.lines() {
            let txt = line.unwrap();
            println!("{}", txt);
        }
    });

    match handle.join() {
        Ok(_) => {},
        Err(e) => {
            println!("스레드 내부에서 오류가 발생했습니다. {:?}", e);
        }
    };
}
```
- join()을 사용해서 스레드 할당한 작업이 모두 완료될때까지 기다릴 수 있습니다.
- 심지어 생성한 스레드에서 에러가 나서 panic 처리가 되면 이를 받아서 에러 핸들링이 가능하게 됩니다.

### 채널로 스레드 간 데이터 전송

mpsc 모듈의 channel을 사용하면 송신자(transmitter)와 수신자(receiver)의 인스턴스가 튜플형태로 반환됩니다.
송신자는 수신자에게 값을 전달하는데 사용되면 수신자는 송신자가 보낸값을 읽는데 사용합니다.

송신자는 여러개가 가능하고(multiple producer), 수신자는 하나만 가능하게됩니다.(single consumer)

```rust
use std::thread;
use std::sync::mpsc;
// 송신자가 두개인 코드
fn main() {
    let (tx1, rx) = mpsc::channel();
    let tx2 = mpsc::Sender::clone(&tx1); // tx1복제

    // 1부터 50까지의 합
    thread::spawn(move || {
        let mut sum = 0;

        for i in 1..=50 {
            sum = sum + i;
        }

        tx1.send(sum).unwrap();
    });

    // 51부터 100까지의 합
    thread::spawn(move || {
        let mut sum = 0;

        for i in 51..=100 {
            sum = sum + i;
        }

        tx2.send(sum).unwrap();
    });

    let mut sum = 0;
    
    for val in rx {
        println!("수신: {}", val);
        sum = sum + val;
    } 

    println!("1부터 100까지의 합: {}", sum);
}

```

### async/await
async/await 구문은 메인 스레드를 멈추지 않으면서도 동기식 프로그래밍과 유사한 방식으로 비동기 프로그래밍을 할 수 있는 기법을 제공합니다.
`block_on()` 함수를 사용해 future의 수행이 가능합니다.
```rust
use futures::executor::block_on;

async fn hello_world() {
    println!("future안에서 실행");
}

fn main() {
    let future = hello_world();
    println!("main함수에서 실행");
    block_on(future);
    println!("future종료 이후 실행");
}
// -----------------------------------------------
use futures::executor::block_on;

async fn calc_sum(start: i32, end: i32) -> i32 {
    let mut sum = 0;

    for i in start..=end {
        sum += i;
    }

    sum
}

fn main() {
    let future = calc_sum(1, 100);
    let sum = block_on(future);
    println!("1부터 100까지의 합: {}", sum);
}
```
### 이벤트 루프 모델과 스레드 모델의 장단점

[concurrent vs event loop](../images/model_diff.png)


### Mutex
멀티스레드 환경에서 자원을 공유하는 상황에서 lock()을 사용해 자원의 소유권을 얻고 race condition을 방지할 수 있습니다.
```rust
use std::thread;
use std::sync::Mutex;

static counter: Mutex<i32> = Mutex::new(0); // counter를 전역변수로 정의

fn inc_counter() {
    let mut num = counter.lock().unwrap();
    *num = *num + 1;
} // inc_counter를 벗어나는 순간 counter는 unlock됩니다.

fn main() {
    let mut thread_vec = vec![];

    for _ in 0..100 {
        let th = thread::spawn(inc_counter);
        thread_vec.push(th);
    }

    for th in thread_vec {
        th.join().unwrap();
    }

    println!("결과: {}", *counter.lock().unwrap());
}
```

### Semaphore
복수의 자원을 나눠가질때 사용

```rust
use std::sync::{Arc, Mutex};
use tokio::sync::Semaphore;

static counter: Mutex<i32> = Mutex::new(0);

#[tokio::main]
async fn main() {
    // 동시에 2개의 thread가 접근 가능하도록 세마포어 설정
    let semaphore = Arc::new(Semaphore::new(2));
    let mut future_vec = vec![];

    for _ in 0..100 {
        // semaphore 획득
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        let future = tokio::spawn(async move {
            let mut num = counter.lock().unwrap();
            *num = *num + 1;

            drop(permit); // semaphore 해제
        });
        future_vec.push(future);
    }

    for future in future_vec {
        future.await.unwrap();
    }

    println!("결과: {}", *counter.lock().unwrap());
}
```