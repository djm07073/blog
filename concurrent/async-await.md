### async/await
#### Future
Rust에서 Future는 미래의 언젠가의 시점에서 값이 결정되는 타입을 뜻합니다. 다른 언어에서는 Promise로 불리기도 합니다.
``` rust
fn main() {
    let executor = Executor::new();
    executor.get_spawner().spawn(Hello::new());
    executor.run();
}
```
위 코드에서 async/await를 사용해 명시적으로 코루틴을 실행하도록 작성할 수 있습니다
``` rust
fn main() {
    let executor = Executor::new();
    executor.get_spawner().spawn( async {
        let h = Hello::new();
        h.await;
    });
    executor.run();
}
```
async로 둘러싸인 처리 부분이 컴파일러에 의해 Future 트레이트를 구현한 타입의 값으로 변환됩니다. 그리고 실제 await를 호출하면 poll을 실행해서 
```rust
match h.poll(cx) {
    Poll::Pending => return Poll::Pending,
    Poll:Result(x) => x,
}
```
와 같이 실행하고 결과값을 다시 task queue에 저장하게 됩니다.

### 비동기 라이브러리 Tokio

rust에서 비동기 프로그래밍을 할때 사용하는 라이브러리인 Tokio가 제일 유명합니다. 코드를 보면서 왜 쓰는지 한번 살펴봅시다!

**tokio를 사용해서 구현하면 스레드를 생성하는 것이 아니라 미리 생성해둔 스레드(워커 스레드)를 이용해 각 태스크를 실행하게 됩니다.**
```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt}; // ❶
use tokio::io;
use tokio::net::TcpListener; // ❷

#[tokio::main] // ❸
async fn main() -> io::Result<()> {
    // 10000번 포트에서 TCP 리슨 ❹
    let listener = TcpListener::bind("127.0.0.1:10000").await.unwrap();

    loop {
        // TCP 커넥트 억셉트 ❺
        let (mut socket, addr) = listener.accept().await?;
        println!("accept: {}", addr);

        // 비동기 태스크 생성 ❻
        tokio::spawn(async move {
            // 버퍼 읽기 쓰기용 객체 생성 ❼
            let (r, w) = socket.split(); // ❽
            let mut reader = io::BufReader::new(r);
            let mut writer = io::BufWriter::new(w);

            let mut line = String::new();
            loop {
                line.clear(); // ❾
                match reader.read_line(&mut line).await { // ❿
                    Ok(0) => { // 커넥션 로스
                        println!("closed: {}", addr);
                        return;
                    }
                    Ok(_) => {
                        print!("read: {}, {}", addr, line);
                        writer.write_all(line.as_bytes()).await.unwrap();
                        writer.flush().await.unwrap();
                    }
                    Err(e) => { // 에러ー
                        println!("error: {}, {}", addr, e);
                        return;
                    }
                }
            }
        });
    }
}
```

이 경우에 spawn을 사용해서 구현해도 되지만 tokio를 사용해서 spawn하면 실행시 비용을 줄여줄 수 있습니다.
스레드 생성은 비용이 많이 드는 작업이므로 단위 시간당 컨넥션 도착 수가 증가하면 계산 자원이 부족해집니다.
**tokio를 사용해서 구현하면 스레드를 생성하는 것이 아니라 미리 생성해둔 스레드(워커 스레드)를 이용해 각 태스크를 실행하게 됩니다.**
이 스레드의 수는 CPU 코어의 개수에 비례해서 결정되게 됩니다.

**비동기용 워커스레드 환경에서 적합한 sleep!**
```rust
// 블록되는 슬립 sleep
// use std::{thread, time}; // ❶
//
// #[tokio::main]
// async fn main() {
//     // join으로 종료 대기
//     tokio::join!(async move { // ❷
//         // 10초 슬립 ❸
//         let ten_secs = time::Duration::from_secs(10);
//         thread::sleep(ten_secs);
//     });
// }

// Tokio 함수를 가진 슬립
use std::time;

#[tokio::main]
async fn main() {
    // join으로 종료 대기
    tokio::join!(async move {
        // 10초 슬립
        let ten_secs = time::Duration::from_secs(10);
        tokio::time::sleep(ten_secs).await; // ❶
    });
}
```
tokio의 sleep을 사용하면 다른 스레드의 코드들을 실행이 가능하고 std의 sleep을 사용하면 전체 워커 스레드가 정지하게 됩니다.


**워커스레드간에 공유와 소유권 이동이 가능한 Mutex**

```rust
use std::{sync::Arc, time};
use tokio::sync::Mutex;

const NUM_TASKS: usize = 8;

// 록만 하는 태스크 ❶
async fn lock_only(v: Arc<Mutex<u64>>) {
    let mut n = v.lock().await;
    *n += 1;
}

// 록 상태에서 await를 수행하는 태스크 ❷
async fn lock_sleep(v: Arc<Mutex<u64>>) {
    let mut n = v.lock().await;
    let ten_secs = time::Duration::from_secs(10);
    tokio::time::sleep(ten_secs).await; // ❸
    *n += 1;
}

#[tokio::main]
async fn main() -> Result<(), tokio::task::JoinError> {
    let val = Arc::new(Mutex::new(0));
    let mut v = Vec::new();

    // lock_sleep 태스크 생성
    let t = tokio::spawn(lock_sleep(val.clone()));
    v.push(t);

    for _ in 0..NUM_TASKS {
        let n = val.clone();
        let t = tokio::spawn(lock_only(n)); // lock_only 태스크 생성
        v.push(t);
    }

    for i in v {
        i.await?;
    }
    Ok(())
}
```
tokio::Mutex 를 사용하지 않으면, `lock_sleep`에서 락을 획득한 상태에서 `lock_only`에서 락을 획득하지 못하고 빠져나가지 못해 데드락 상태에 빠지게 됩니다. 그래서 이를 회피하기 위해 비동기 워커 스레드 환경에서 mutex의 락획득이 관리되는 tokio::Mutex를 사용해야 합니다.

### 요약
1. async는 비동기 프로그래밍 타입의 결과인 Future를 명시적으로 지정할때 사용하고, await는 실제로 코루틴의 poll을 실행하면서 완료될때까지 대기하게 됩니다.
2. 비동기 라이브러리인 tokio는 워커 스레드 환경에서 적합한 기능들을 제공하게 됩니다. 실행비용 단축, sleep, mutex등 비동기 환경에서 워커스레드가 잘 동작하도록 설계된 라이브러리입니다.