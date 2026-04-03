---
created: "03/04/2026"
---
---

## Идиомы

**Идиомы языка** - список правил и приемов для написания правильного кода на Go. Идиомы в Go направлены на **читаемость**, **простоту** и **явность**.

Полезные материалы:
- Основной список идиом языка: [Go Proverbs](https://go-proverbs.github.io/)
- Гайд "Эффективный Go": [Effective Go](https://go.dev/doc/effective_go)
- Дополнение к Effective Go: [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)

## Наименование пакетов

Рекомендуется называть пакеты максимально кратко, чтобы не усложнять себе жизнь при импортах.

Если название пакета содержит 2 слова, то разделять их `_` не надо: `package userinfo`

Также стоит выбирать такие названия пакетов, чтобы их наименования не совпадали с именами переменных в импортируемых пакетах, в ином случае придется использовать alias пакета:

```go
import (
  usersrv github.com/..../user
)
func Do() {
  user, err := usersrv.TryFind() // переменная user и пакет user
}
```

Еще одно важное правило - стоит давать осмысленные имена пакетам, не стоит называть их обезличено (`utils`, `common`, `helper`, `model` ...). Хорошие примеры: `email`, `auth`, `config`

## Названия методов и функций

Важное правило - избегать дублирования имени пакета в именах его сущностей:

**Плохой пример:**

```go
package db

func CreateDbConnection() (*DbConn, error) {}
```

Дублирование в использовании: `db.CreateDbConnection`, `db.DbConn`.

**Хороший пример:**

```go
package db

type Conn struct {}

func CreateConn() (*Conn, error) {}
```

Где-то в другом пакете:

```go
c, err := db.CreateConn()

func Do(c *db.Conn) {}
```


Также в Go стоит избегать функции-геттеры с префиксом `get`:

```go
package user

type User struct {
  Id int
  Address string
}

// Плохой пример
// Использование: address := user.GetAddress()
func (u User) GetAddress() string {
  return u.Address
}

// Хороший пример
// Использование: address := user.Address()
func (u User) Address() string {
  return u.Address
}
```

Тут есть небольшое замечание: в пакете `user` есть структура `User`, она полностью дублирует название пакета и по сути является главной сущностью пакета. Это нормально.

## Интерфейсы

Если интерфейс имеет единственный метод, то его называют по его именованию:
- `Read()` -> `Reader`
- `Write()` -> `Writer`

Интерфейс стоит объявлять там, где он используется, а не в месте, где реализуются его методы:

```go
package format

type Formatter interface {
  Format() string
}

type Service struct {}

func NewService() *Service {
  return &Service{}
}

func (s *Service) Format() string {}
```

Где-то в другом пакете будет не важно, что мы используем: метод напрямую или интерфейс. Мы все равно притянем пакет format

```go
package example

import github.com/..../format

func example(f format.Formatter) {
  svc := format.NewService()
  svc.Format()
  f.Format()
}
```

Лучше всё же вынести интерфейс в место использования. Так не получим лишних импортов и полностью абстрагируем логику

```go
package example

type Formatter interface {
  Format() string
}

func example(f Formatter) {
  f.Format()
}
```

## nil-мапы и слайсы

Не стоит проверять мапы и слайсы на то, что они неинициализированы сравнением с `nil`. Достаточно использовать `len()`, т.к она вернет в таком случае 0.

## Горутины

Когда запускаем горутину, нужно сразу же понять, где и при каких обстоятельствах она закроется. 

Если без горутины можно обойтись, лучше так и сделать.

## Каналы

По умолчанию всегда стоит делать каналы небуферизированными, и только в случае дальнейшей необходимости - объявляем с буфером.

## Параметры функции

Вместо подобной сигнатуры с кучей параметров:

```go
func Example(one int, two string, three uint, four, five uuid.UUID, six any, seven string, eight bool) (map[int]string, uint, []int, any, error) {}
```

Лучше использовать отдельную структуру для параметров и для возврата в сигнатуре:

```go
type ExampleRequest struct {
  one int
  two string
  three uint
  four, five uuid.UUID
  six any
  seven string
  eight bool
}

type ExampleResp struct {...}

func Example(req ExampleRequest) (ExampleResp, error) {}
```

## Ошибки и паники

Всегда стоит обрабатывать ошибки и оборачивать их в контекст через `fmt.Errorf` при возврате.

Ну и конечно, не стоит злоупотреблять вызовом паник.

## Дженерики

Дженерики стоит использовать только там, где они действительно необходимо. Использовать их с мыслью "код универсальный, пригодится в будущем" неправильно.

## Глобальные переменные

Никаких глобальных переменных по возможности.