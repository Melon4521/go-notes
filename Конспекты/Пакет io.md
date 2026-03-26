---
created: "26/03/2026"
---
---

## Интерфейсы `io`

Пакет `io` содержит в себе полезные интерфейсы:

- `io.Reader` - типы, читающие байты из ресурса и записывающие их в `p` методом `Read(p []byte) (int, error)` (возвращает число прочитанных байтов и ошибку)
- `io.Writer` - типы, записывающие байты из `p` в ресурс методом `Write(p []byte) (int, error)` (возвращает число записанных байтов и ошибку)
- `io.Closer` - типы, закрывающие ресурс методом `Close() error`
- `io.ReadWriter` - объединяет интерфейсы `io.Reader` и `io.Writer`
- `io.ReadCloser`, `io.WriteCloser`, `io.ReadWriteCloser` - объединяют интерфейсы `io.Reader`+`io.Closer` / `io.Writer` + `io.Closer` / все вместе

