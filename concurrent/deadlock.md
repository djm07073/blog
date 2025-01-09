### Deadlock
- 자원을 공유하고 있는 두개 이상의 프로세스 혹은 스레드가 서로 자원이 비는것을 기다리며 더 이상 처리가 진행되지 않는 상태
![Deadlock](./images/process.png)
- Rust RwLock의 데드락 예시 코드
```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let val = Arc::new(RwLock::new(true));

    let t = thread::spawn(move || {
        let _flag = val.read().unwrap(); // ❶
        *val.write().unwrap() = false; // ❷
        println!("deadlock");
    
    });

    t.join().unwrap();
}
```

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let val = Arc::new(RwLock::new(true));

    let t = thread::spawn(move || {
        let _ = val.read().unwrap(); // ❶ 그 즉시 스택에서 해당 변수를 해제하고 락을 해제하도록 함.
        *val.write().unwrap() = false; // ❷
        println!("not deadlock");
    
    });

    t.join().unwrap();
}
```