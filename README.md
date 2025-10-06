# Package or


Пакет `or` предоставляет утилиту для объединения нескольких done-каналов в один. Это особенно полезно в конкурентных сценариях, когда необходимо ожидать завершения одной из нескольких операций.

## Установка

```bash
go get github.com/pozedorum/or
```

## Использование

### Базовый пример

```go
package main

import (
    "fmt"
    "time"
    
    "github.com/pozedorum/or"
)

func main() {
    sig := func(after time.Duration) <-chan interface{} {
        c := make(chan interface{})
        go func() {
            defer close(c)
            time.Sleep(after)
        }()
        return c
    }

    start := time.Now()
    <-or.Or(
        sig(2*time.Hour),
        sig(5*time.Minute),
        sig(1*time.Second),
        sig(1*time.Hour),
        sig(1*time.Minute),
    )

    fmt.Printf("done after %v\n", time.Since(start))
    // Output: done after ~1s
}
```

### Пример с таймаутом

```go
package main

import (
    "fmt"
    "time"
    
    "github.com/pozedorum/or"
)

func main() {
    longOperation := make(chan interface{})
    timeout := make(chan interface{})
    
    go func() {
        time.Sleep(50 * time.Millisecond)
        close(timeout)
    }()
    
    go func() {
        time.Sleep(100 * time.Millisecond)
        close(longOperation)
    }()
    
    <-or.Or(longOperation, timeout)
    fmt.Println("Operation finished or timed out")
}
```

## API

### `func Or(channels ...<-chan interface{}) <-chan interface{}`

Функция `Or` принимает переменное количество каналов и возвращает единый канал, который закрывается, когда закрывается **любой** из переданных входных каналов.

**Параметры:**
- `channels` - список каналов для отслеживания

**Возвращает:**
- `<-chan interface{}` - канал, который закрывается при закрытии любого из входных каналов

**Особенности:**
- Если не передано ни одного канала, возвращается немедленно закрытый канал
- Если передан только один канал, он возвращается как есть
- Все горутины корректно завершаются, утечек ресурсов не происходит

## Тестирование

```bash
cd task1/or
go test -v
```

## Примеры использования

### Ожидание нескольких горутин

```go
func waitForAnyGoroutine() {
    ch1 := make(chan interface{})
    ch2 := make(chan interface{})
    ch3 := make(chan interface{})
    
    // Запускаем различные операции
    go performTask1(ch1)
    go performTask2(ch2) 
    go performTask3(ch3)
    
    // Ждем завершения любой из них
    <-or.Or(ch1, ch2, ch3)
    fmt.Println("One of the tasks completed!")
}
```

### Таймаут операции

```go
func withTimeout(operation chan interface{}, timeout time.Duration) {
    timeoutCh := make(chan interface{})
    go func() {
        time.Sleep(timeout)
        close(timeoutCh)
    }()
    
    select {
    case <-or.Or(operation, timeoutCh):
        fmt.Println("Operation completed or timed out")
    }
}
```
