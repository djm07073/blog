
## Go routine

고루틴은 다른 함수와 동시에 실행할 수 있는 함수입니다. 고루틴을 생성하려면 `go` 키워드 뒤에 함수 호출을 사용합니다:

```go
package main

import "fmt"

func f(n int) {
    for i := 0; i < 10; i++ {
        fmt.Println(n, ":", i)
    }
}

func main() {
    go f(0)
    var input string
    fmt.Scanln(&input)
}
```

이 프로그램은 두 개의 고루틴으로 구성됩니다. 첫 번째 고루틴은 암시적이며 `main` 함수 자체입니다. 두 번째 고루틴은 `go f(0)`을 호출할 때 생성됩니다. 일반적으로 함수를 호출하면 프로그램은 함수의 모든 문을 실행한 다음 호출 후의 다음 줄로 돌아갑니다. 고루틴을 사용하면 즉시 다음 줄로 돌아가고 함수가 완료될 때까지 기다리지 않습니다. 이것이 `Scanln` 함수를 호출한 이유입니다. 그렇지 않으면 프로그램이 모든 숫자를 출력할 기회를 얻기 전에 종료됩니다.

고루틴은 가볍고 쉽게 수천 개를 생성할 수 있습니다. 프로그램을 수정하여 10개의 고루틴을 실행할 수 있습니다:

```go
func main() {
    for i := 0; i < 10; i++ {
        go f(i)
    }
    var input string
    fmt.Scanln(&input)
}
```

이 프로그램을 실행하면 고루틴이 동시에 실행되지 않고 순서대로 실행되는 것처럼 보일 수 있습니다. `time.Sleep`과 `rand.Intn`을 사용하여 함수에 지연을 추가해 보겠습니다:

```go
package main

import (
    "fmt"
    "time"
    "math/rand"
)

func f(n int) {
    for i := 0; i < 10; i++ {
        fmt.Println(n, ":", i)
        amt := time.Duration(rand.Intn(250))
        time.Sleep(time.Millisecond * amt)
    }
}

func main() {
    for i := 0; i < 10; i++ {
        go f(i)
    }
    var input string
    fmt.Scanln(&input)
}
```

`f`는 0에서 10까지의 숫자를 출력하며 각 숫자 후 0에서 250ms 사이의 시간을 대기합니다. 이제 고루틴이 동시에 실행됩니다.

## Channel

채널은 두 고루틴이 서로 통신하고 실행을 동기화할 수 있는 방법을 제공합니다. 다음은 채널을 사용하는 예제 프로그램입니다:

```go
package main

import (
    "fmt"
    "time"
)

func pinger(c chan string) {
    for i := 0; ; i++ {
        c <- "ping"
    }
}

func printer(c chan string) {
    for {
        msg := <- c
        fmt.Println(msg)
        time.Sleep(time.Second * 1)
    }
}

func main() {
    var c chan string = make(chan string)

    go pinger(c)
    go printer(c)

    var input string
    fmt.Scanln(&input)
}
```

이 프로그램은 "ping"을 계속 출력합니다(엔터 키를 눌러 중지). 채널 타입은 `chan` 키워드 뒤에 채널에서 전달되는 항목의 타입을 사용하여 나타냅니다(이 경우 문자열을 전달합니다). `<-`(왼쪽 화살표) 연산자는 채널에서 메시지를 보내고 받는 데 사용됩니다. `c <- "ping"`은 "ping"을 보낸다는 의미입니다. `msg := <- c`는 메시지를 받아 `msg`에 저장한다는 의미입니다. `fmt` 라인은 다음과 같이 작성할 수도 있습니다: `fmt.Println(<-c)` 이 경우 이전 줄을 제거할 수 있습니다.

이와 같이 채널을 사용하면 두 고루틴이 동기화됩니다. `pinger`가 채널에 메시지를 보내려고 할 때 `printer`가 메시지를 받을 준비가 될 때까지 기다립니다(이를 차단이라고 합니다). 프로그램에 다른 송신자를 추가하고 어떤 일이 발생하는지 확인해 보겠습니다. 다음 함수를 추가합니다:

```go
func ponger(c chan string) {
    for i := 0; ; i++ {
        c <- "pong"
    }
}
```

그리고 `main`을 수정합니다:

```go
func main() {
    var c chan string = make(chan string)

    go pinger(c)
    go ponger(c)
    go printer(c)

    var input string
    fmt.Scanln(&input)
}
```

이제 프로그램은 "ping"과 "pong"을 번갈아 출력합니다.

### 채널 방향

채널 타입에 방향을 지정하여 송신 또는 수신으로 제한할 수 있습니다. 예를 들어, `pinger`의 함수 시그니처를 다음과 같이 변경할 수 있습니다:

```go
func pinger(c chan<- string)
```

이제 `c`는 송신만 가능합니다. `c`에서 수신하려고 하면 컴파일러 오류가 발생합니다. 마찬가지로 `printer`를 다음과 같이 변경할 수 있습니다:

```go
func printer(c <-chan string)
```

이러한 제한이 없는 채널은 양방향 채널이라고 합니다. 양방향 채널은 송신 전용 또는 수신 전용 채널을 사용하는 함수에 전달할 수 있지만, 그 반대는 불가능합니다.

### Select

Go에는 채널을 위한 스위치처럼 작동하는 특별한 문인 `select`가 있습니다:

```go
func main() {
    c1 := make(chan string)
    c2 := make(chan string)

    go func() {
        for {
            c1 <- "from 1"
            time.Sleep(time.Second * 2)
        }
    }()

    go func() {
        for {
            c2 <- "from 2"
            time.Sleep(time.Second * 3)
        }
    }()

    go func() {
        for {
            select {
            case msg1 := <- c1:
                fmt.Println(msg1)
            case msg2 := <- c2:
                fmt.Println(msg2)
            }
        }
    }()

    var input string
    fmt.Scanln(&input)
}
```

이 프로그램은 2초마다 "from 1"을 출력하고 3초마다 "from 2"를 출력합니다. `select`는 준비된 첫 번째 채널을 선택하여 수신(또는 송신)합니다. 여러 채널이 준비된 경우 무작위로 선택합니다. 채널이 준비되지 않은 경우 문이 차단되어 하나가 준비될 때까지 기다립니다.

`select` 문은 종종 타임아웃을 구현하는 데 사용됩니다:

```go
select {
case msg1 := <- c1:
    fmt.Println("Message 1", msg1)
case msg2 := <- c2:
    fmt.Println("Message 2", msg2)
case <- time.After(time.Second):
    fmt.Println("timeout")
}
```

`time.After`는 채널을 생성하고 주어진 기간 후에 현재 시간을 채널에 보냅니다(시간에 관심이 없으므로 변수를 저장하지 않았습니다). 기본 케이스를 지정할 수도 있습니다:

```go
select {
case msg1 := <- c1:
    fmt.Println("Message 1", msg1)
case msg2 := <- c2:
    fmt.Println("Message 2", msg2)
case <- time.After(time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("nothing ready")
}
```

기본 케이스는 채널이 준비되지 않은 경우 즉시 실행됩니다.

### Buffered Channel

채널을 생성할 때 `make` 함수에 두 번째 매개변수를 전달할 수도 있습니다:

```go
c := make(chan int, 1)
```

이것은 용량이 1인 버퍼링된 채널을 생성합니다. 일반적으로 채널은 동기식입니다. 채널의 양쪽은 다른 쪽이 준비될 때까지 기다립니다. 버퍼링된 채널은 비동기식입니다. 채널이 이미 가득 차지 않은 한 메시지를 보내거나 받는 것이 기다리지 않습니다.