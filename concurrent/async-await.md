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

### I/O 다중화와 aync/await
