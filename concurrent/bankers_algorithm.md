### Banker's Algorithm
- 데드락을 회피하기 위한 알고리즘입니다.
- 은행이 기업에게 돈을 빌려주듯이 은행의 한정된 돈(자원)에서 기업이 필요한 미리 결정된 양의 돈(자원)을 대출하고 상환하는 알고리즘입니다.
- 그런데 은행이 돈을 다 사용했는데도 불구하고 기업들이 필요한 돈을 채우지 못해 상환을 하지 못하면 정체 상태(데드락)가 됩니다.
- 비유가 확 닿지 않을 수 있는데, 코드 예시를 보면서 설명하겠습니다.
```rust

use std::sync::{Arc, Mutex};

// Resource struct를 통해 다음을 정의하는 데요
// 현재 이용 가능한 리소스 (은행의 남은 돈)/ 스레드 i가 사용하고 있는 리소스 (기업이 빌려간 돈) / 스레드 i가 필요로 하는 리소스의 최댓값 (미리 결정된 양의 기업이 필요한 돈)
struct Resource<const NRES: usize, const NTH: usize> {
    available: [usize; NRES],         // 이용 가능한 리소스
    allocation: [[usize; NRES]; NTH], // 스레드 i가 확보 중인 리소스
    max: [[usize; NRES]; NTH],        // 스레드 i가 필요로 하는 리소스의 최댓값
}

impl<const NRES: usize, const NTH: usize> Resource<NRES, NTH> {
    fn new(available: [usize; NRES], max: [[usize; NRES]; NTH]) -> Self {
        Resource {
            available,
            allocation: [[0; NRES]; NTH],
            max,
        }
    }

    // 현재 상태가 데드록을 발생시키지 않는가 확인
    fn is_safe(&self) -> bool {
        let mut finish = [false; NTH]; // 스레드 i는 리소스 획득과 반환에 성공했는가?
        let mut work = self.available.clone(); // 이용 가능한 리소스의 시뮬레이션값

        loop {
            // 모든 스레드 i와 리소스 j에 대해,
            // finish[i] == false && work[j] >= (self.max[i][j] - self.allocation[i][j])
            // 을 만족하는 스레드를 찾는다.
            let mut found = false;
            let mut num_true = 0;
            for (i, alc) in self.allocation.iter().enumerate() {
                if finish[i] {
                    num_true += 1;
                    continue;
                }

                // need[j] = self.max[i][j] - self.allocation[i][j] 를 계산하고, 
                // 모든 리소스 j에 대해, work[j] >= need[j] 인가를 판정한다.
                let need = self.max[i].iter().zip(alc).map(|(m, a)| m - a);
                let is_avail = work.iter().zip(need).all(|(w, n)| *w >= n);
                if is_avail {
                    // 스레드 i가 리소스 확보 가능
                    found = true;
                    finish[i] = true;
                    for (w, a) in work.iter_mut().zip(alc) {
                        *w += *a // 스레드 i가 현재 확보하고 있는 리소스를 반환
                    }
                    break;
                }
            }

            if num_true == NTH {
                // 모든 스레드가 리소스 확보 가능하면 안전함
                return true;
            }

            if !found {
                // 스레드가 리소스를 확보할 수 없음
                break;
            }
        }

        false
    }

    // id번 째의 스레드가 resource를 하나 얻음
    fn take(&mut self, id: usize, resource: usize) -> bool {
        // 스레드 번호, 리소스 번호 검사
        if id >= NTH || resource >= NRES || self.available[resource] == 0 {
            return false;
        }

        // 리소스 확보를 시험해 본다
        self.allocation[id][resource] += 1;
        self.available[resource] -= 1;

        if self.is_safe() {
            true // 리소스 확보 성공
        } else {
            // 리소스 확보에 실패했으므로 상태 원복
            self.allocation[id][resource] -= 1;
            self.available[resource] += 1;
            false
        }
    }

    // id번 째의 스레드가 resource를 하나 반환
    fn release(&mut self, id: usize, resource: usize) {
        // 스레드 번호, 리소스 번호를 검사
        if id >= NTH || resource >= NRES || self.allocation[id][resource] == 0 {
            return;
        }

        self.allocation[id][resource] -= 1;
        self.available[resource] += 1;
    }
}

#[derive(Clone)]
pub struct Banker<const NRES: usize, const NTH: usize> {
    resource: Arc<Mutex<Resource<NRES, NTH>>>,
}

impl<const NRES: usize, const NTH: usize> Banker<NRES, NTH> {
    pub fn new(available: [usize; NRES], max: [[usize; NRES]; NTH]) -> Self {
        Banker {
            resource: Arc::new(Mutex::new(Resource::new(available, max))),
        }
    }

    pub fn take(&self, id: usize, resource: usize) -> bool {
        let mut r = self.resource.lock().unwrap();
        r.take(id, resource)
    }

    pub fn release(&self, id: usize, resource: usize) {
        let mut r = self.resource.lock().unwrap();
        r.release(id, resource)
    }
}

```
- Resource struct를 보면, `is_safe` / `take` / `release` 의 action이 가능하고.특히 `is_safe` 를 통해 데드락이 되는지 시뮬레이션 하게 됩니다.
- `is_safe`는 스레드가 더 필요한 리소스(`need`)를 계산하고 현재 사용할 수 있는 `work` 값보다 작은지를 체크하게 됩니다.
- 이를 통해 데드락이 걸리는 스레드가 존재하지 않게 됩니다.

그러면 이 Banker 알고리듬을 사용해서 식사하는 철학자 문제를 해결해보겠습니다.
```rust
mod banker;

use banker::Banker;
use std::thread;

const NUM_LOOP: usize = 100000;

fn main() {
    // 이용 가능한 포크의 수, 철학자가 이용하는 포크 최대 수 설정
    let banker = Banker::<2, 2>::new([1, 1], [[1, 1], [1, 1]]);
    let banker0 = banker.clone();

    let philosopher0 = thread::spawn(move || {
        for _ in 0..NUM_LOOP {
            // 포크 0과 1을 확보
            while !banker0.take(0, 0) {}
            while !banker0.take(0, 1) {}

            println!("0: eating");

            // 포크 0과 1을 반환
            banker0.release(0, 0);
            banker0.release(0, 1);
        }
    });

    let philosopher1 = thread::spawn(move || {
        for _ in 0..NUM_LOOP {
            // 포크 1과 0을 확보
            while !banker.take(1, 1) {}
            while !banker.take(1, 0) {}

            println!("1: eating");

            // 포크 1과 0을 반환
            banker.release(1, 1);
            banker.release(1, 0);
        }
    });

    philosopher0.join().unwrap();
    philosopher1.join().unwrap();
}
```
- 이렇게 코드를 작성하면 스핀락 상태에서 철학자가 포크를 얻는 과정이 수행되게 되고 포크를 두개 얻으면 식사를 마치고 스레드가 종료되게 되고 리소스를 반환하게 됩니다.
- 그래서 이러한 방법으로 데드락을 Banker's Algorithm을 사용해서 해결할 수 있습니다. 
- 그렇지만 각각의 스레드 혹은 프로세스가 사용할 리소스의 최대값과 그 수를 미리 파악하고 있어햐 하는 단점이 있습니다.