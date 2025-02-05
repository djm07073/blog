### Rust Pointer
rust에는 정말 다양한 포인터가 존재하고 각각의 포인터는 경우에 따라 적절히 사용해야 오버헤드가 적습니다.

### Rust Ownership Move/Borrow & Copy
포인터를 다루기 전에 소유권 이전과 빌림 그리고 복사에 대해 알아야 합니다.
#### Move
```rust
fn main() {
    let s = String::from("Hello");
    let s2 = s // s의 메모리 소유권이 s2에게 넘어감.
    println!("{}", s); // s를 사용하는 순간 컴파일 오류 발생
}

// 함수의 파라미터로 사용해도 소유권 이전이 발생하게 됩니다.
fn main() {
    let s = String::from("Hello");
    push_str(s); // push_str에 소유권이 이관
    println!("{}", s); // s를 사용하는 순간 컴파일 오류 발생
}

fn push_str(mut s: String) {
    s.push_str(" Rust!");
}
```
소유권 이전이 발생하게 되면 다시 해당 변수를 사용하는것이 불가능하게 됩니다.
해당 변수가 가진 메모리가 함수에서 사용되어서 이미 해제되었다는 것을 뜻합니다.

#### Clone/Copy
``` rust
fn main() {
    // 새로운 문자열 변수를 생성
    let s1 = String::from("Hello Rust!");

    // s1을 복사하여 s2에 저장
    let s2 = s1.clone();

    // s1은 여전히 오너쉽을 가지고 있기 때문에 문제없음
    println!("{}", s1);
}
// Copy trait는 스택에서 값을 복사해서 사용할 수 있습니다.
// .clone()을 사용하지 않고도 자동으로 값을 복사해서 사용하고 소유권이전이 발생하지 않습니다.
#[derive(Copy, Clone)]
struct Student {
    age: i32,
}

fn main() {
    let mut s1 = Student { age: 10 };
    let s2 = s1; // s1을 복사하여 s2에 저장

    println!("s1: {} s2: {}", s1.age, s2.age);

    s1.age = 20; // s1의 나이를 20으로 변경

    println!("s1: {} s2: {}", s1.age, s2.age);
}

// String은 Copy만 사용불가능하고 Clone만 사용가능
// String은 힙메모리를 사용해 값을 다루기 때문에 스택만 사용하는 Copy는 사용이 불가능합니다.
// 명시적으로 .clone()을 사용해야 합니다.
#[derive(Clone)]
struct Student {
    age: i32,
    name: String,
}

fn main() {
    let mut s1 = Student { age: 10, name: String::from("Luna") };
    let s2 = s1.clone(); // s1을 명시적으로 복제하여 s2에 저장

    println!("s1: {} s2: {}", s1.name, s2.name);

    s1.name = String::from("Rust"); // s1의 이름을 변경

    println!("s1: {} s2: {}", s1.name, s2.name);
}
```

다시 쓰고 싶을때는 해당 변수를 clone을 통해 복사하고 새롭게 s2는 복제된 값에 대한 소유권을 획득하게 됩니다.
사실상 사용하는 메모리는 두배인거죠. 이렇게 되면 불필요한 메모리를 사용해서 비효율적입니다. 특히 다른 함수에 값을 전달하는 경우 소유권을 상실하기 때문에 다음과 같은 문제가 추가로 발생하게 됩니다.

#### Borrow
그래서 기존 변수의 소유권은 그대로 두고 잠깐 빌려 쓰는 방식이 있습니다.
```rust
fn main() {
    let mut s = String::from("Hello");

    // s의 소유권을 push_str에 대여
    push_str(&mut s);

    // s는 소유권을 유지하고 있기에 정상 동작
    println!("{}", s);
}

fn push_str(s: &mut String) {
    s.push_str(" Rust!");
}
```
기존 프로그래밍 언어에서도 많이 쓰는 방식이지만 참조값을 전달하면 소유권을 빌려주게 됩니다.
Borrow 방식을 사용하면 값은 전달하되 소유권을 유지할 수 있게 됩니다.

### Ponter
동적 메모리 할당은 런타임에 프로그램이 필요한 메모리를 할당하는 것을 말합니다. 보통 컴파일 타임에 프로그램의 정확한 메모리 사용량을 예측하기 어려운 경우 사용합니다.
대표적으로 Box와 Rc가 있습니다.

#### Box
Box를 사용해 힙영역에 동적으로 메모리를 할당받을 수 있습니다. 일종의 스마트 포인터 역할을 수행합니다.
```rust
fn main() {
    let mut x = Box::new(10);
    println!("x: {}", x);

    *x = 20; // x를 변경
    println!("x: {}", x);
}
```
동적으로 생성한 x를 따로 free해주지 않아도 데이터가 더이상 사용되지 않는 시점에 알아서 해제됩니다.
이또한 소유권 이전이 존재합니다. 그렇다보니 여러 곳에서 공유해서 사용하지 못하고 빌려서 줘야합니다.
공유할 수 있는 포인터는 Rc 입니다.

#### Rc
.clone()을 통해 값을 사용할 수 있습니다. 실제 메모리를 더 사용하는 것은 아니고 참조 카운트를 통해 참조를 핸들링하게 됩니다.
이또한 참조횟수가 0이면 참조하고 있는 변수가 없다고 생각하고 메모리를 해제하게 됩니다.
```rust
use std::rc::Rc;

struct Person {
    age: i32,
}

fn main() {
    // person을 공유 객체로 생성
    let person = Rc::new(Person { age: 10 });

    // person복제
    let p1 = person.clone();
    println!("person: {} p1: {}", person.age, p1.age);
    println!("RefCount: {}", Rc::strong_count(&person));
    
    // person복제
    let p2 = person.clone();
    println!("RefCount: {}", Rc::strong_count(&person));

    {
        // person복제
        let p3 = person.clone();
        println!("RefCount: {}", Rc::strong_count(&person));

        // p3소멸
    }

    println!("RefCount: {}", Rc::strong_count(&person));
}
```

#### RefCell
```rust
use std::rc::Rc;

struct Person {
    name: String,
    age: i32,
    next: Option<Rc<Person>>, // next는 수정 불가
}

fn main() {
    let mut p1 = Rc::new(Person {
        name: String::from("Luna"),
        age: 30,
        next: None,
    });

    let mut p2 = Rc::new(Person {
        name: String::from("Rust"),
        age: 10,
        next: None,
    });

    p1.next = Some(p2.clone()); // 컴파일 오류 발생
}
```
Rc는 다 좋지만 처음이후에 값을 변경할 수 없다는 단점이 존재합니다. 그래서 RefCell을 사용해 변경 불가능한 변수를 임시로 변경가능하게 해줍니다.
이때는 mutable 참조 포인터를 받아오는것으로 borrow_mut라는 메서드를 사용해 포인터를 가져옵니다. 
``` rust
use std::rc::Rc;
use std::cell::RefCell;

struct Person {
    name: String,
    age: i32,
    next: RefCell<Option<Rc<Person>>>, // next는 수정 가능
}

fn main() {
    let mut p1 = Rc::new(Person {
        name: String::from("Luna"),
        age: 30,
        next: RefCell::new(None),
    });

    let mut p2 = Rc::new(Person {
        name: String::from("Rust"),
        age: 10,
        next: RefCell::new(None),
    });

    let mut next = p1.next.borrow_mut();
    *next = Some(p2.clone()); // p1뒤에 p2를 추가
}
```

#### Weak
그런데 만약 참조를 순환참조방법을 사용하게 되면 메모리릭이 발생하게 됩니다.
아래 코드가 그 상황입니다.
``` rust
use std::rc::Rc;
use std::cell::RefCell;

struct Person {
    id: i32,
    next: RefCell<Option<Rc<Person>>>,
}

fn main() {
    let mut p1 = Rc::new(Person {
        id: 1,
        next: RefCell::new(None),
    });

    let mut p2 = Rc::new(Person {
        id: 2,
        next: RefCell::new(None),
    });

    let mut p3 = Rc::new(Person {
        id: 3,
        next: RefCell::new(None),
    });

    // p1은 p2를/ p2는 p1을 참조하고 있는 상황
    let mut next = p1.next.borrow_mut();
    *next = Some(p2.clone()); // p1뒤에 p2를 추가
    
    let mut next = p2.next.borrow_mut();
    *next = Some(p1.clone()); // p2뒤에 p1를 추가

    println!("p1 RefCount: {} p2: RefCount: {}", 
        Rc::strong_count(&p1), Rc::strong_count(&p2));

    // p3는 Drop이 불리는데 p1과 p2는 Drop이 불리지 않음
}
```
Weak 포인터를 사용하게 되면 weak_count가 늘어나고 strong_count가 늘어나지 않습니다.
메모리 해제는 strong_count를 기준으로 판단하기 때문에 이를 사용하면 문제가 없습니다.
Rc는 downgrade를 사용해 Weak 포인터로, Weak 포인터는 upgrade를 통해 Rc로 변환이 가능합니다.
```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Person {
    id: i32,
    next: RefCell<Option<Weak<Person>>>,
}

impl Drop for Person {
    fn drop(&mut self) {
        println!("p{} Drop!", self.id);
    }
}

fn main() {
    let mut p1 = Rc::new(Person {
        id: 1,
        next: RefCell::new(None),
    });

    let mut p2 = Rc::new(Person {
        id: 2,
        next: RefCell::new(None),
    });

    let mut next = p1.next.borrow_mut();
    *next = Some(Rc::downgrade(&p2)); // weak 방식으로 p2 추가
    
    let mut next = p2.next.borrow_mut();
    *next = Some(Rc::downgrade(&p1)); // weak 방식으로 p1 추가

    println!("p1 RefCount: {} p2: RefCount: {}", 
        Rc::strong_count(&p1), Rc::strong_count(&p2));
}
```