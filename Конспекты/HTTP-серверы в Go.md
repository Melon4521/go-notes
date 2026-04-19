---
created: "19/04/2026"
---
---

Для написания полноценного сервера на Go есть встроенная библиотека `net/http`

## Запуск сервера и обработчик запросов

Запустить сервер можно с помощью вызова `http.ListenAndServe(addr string, handler)`. Первый параметр является портом, на котором сервер запустится. Второй параметр - глобальный **http-обработчик** для всех поступающих на сервер запросов. Вместо него можно передать `nil`, если мы хотим разные обработчики для разных запросов, тогда будет использован глобальный маршрутизатор `http.DefaultServeMux`, который определяет какому эндоинту какой обработчик соответствует.

> [!important]
> В Go **обработчик** является интерфейсом `http.Handler` с методом `ServeHTTP(w http.ResponseWriter, r *http.Request)`, он вызывается при определенном HTTP-запросе. Параметры:
> - `http.ResponseWriter` - интерфейс для записи ответа, через него мы пишем данные и заголовки
> - `http.Request` - интерфейс для чтения информации о запросе (URL, метод, тело)
>  
> Если необходимо использовать функцию в качестве обработчика, то ее можно "преобразовать" к типу обработчика через `http.HandlerFunc(func)` 

Для привязки определенной функции к эндпоинту используется `http.HandleFunc(pattern, handler)`. `pattern` - строка с адресом эндпоинта, в ней можно указать метод и path-параметры. Функция `handler` должна соответствовать сигнатуре метода интерфейса обработчика `ServeHTTP`.

```go
func weatherHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Добро пожаловать в сервис погоды!")
}

func main() {
	http.HandleFunc("/weather", weatherHandler)
	fmt.Println("Сервер запущен на порту 8080...")
	
	if err := http.ListenAndServe(":8080", nil); err != nil {
		fmt.Println("Ошибка запуска сервера:", err)
	}
}
```

Теперь при запуске этого кода поднимется локальный сервер `localhost:8080`, а по запросу `/weather` мы получим соответствующий вывод.

Альтернативный способ создания сервера с единым глобальным обработчиком:

```go
func handler(w http.ResponseWriter, r *http.Request) {
	switch r.URL.Path {
	case "/":
		fmt.Fprintln(w, "Главная")
	case "/about":
		fmt.Fprintln(w, "О нас")
	default:
		http.NotFound(w, r)
	}
}

func main() {
	srv := &http.Server{
		Addr:           ":8080",
		// преобразование функции к типу обработчика
		Handler:        http.HandlerFunc(handler), 
		ReadTimeout:    5 * time.Second,
		WriteTimeout:   10 * time.Second,
	}
	
	log.Fatal(srv.ListenAndServe()) // запуск созданного сервера
}
```

Вариант через единую структуру приложения:

```go
type App struct{}

func (a *App) home(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Главная")
}

func (a *App) about(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "О нас")
}

func (a *App) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	switch r.URL.Path {
	case "/":
		a.home(w, r)
	case "/about":
		a.about(w, r)
	default:
		http.NotFound(w, r)
	}
}

func main() {
	app := &App{}
	srv := &http.Server{Addr: ":8080", Handler: app}
	log.Fatal(srv.ListenAndServe())
}
```

## Разбор запроса к серверу

Внутри интерфейса запроса `*http.Request` есть вся необходимая информация:
- **Метод запроса**: `r.Method` (строка вида `GET`, `POST`...)
- **Путь запроса**: `r.URL.Path` (например, `/tasks`)
- **Query-параметры**: `r.URL.Query()` возвращает объект с query-параметрами. Например, для запроса `/tasks?done=true` значение параметра done (значение всегда строка) можно получить через `r.URL.Query().Get("done")`
- **Заголовки**: `r.Header` - кастомный тип над `map[string][]string` с удобными методами взаимодействия с заголовками
- **Тело запроса**: доступно в `r.Body` (интерфейс `io.ReadCloser`)

> [!warning]
> Тело запроса `r.Body` можно читать только один раз (так как это поток, то после первого прочтения он просто будет пуст). Причем всегда принято закрывать тело через `defer r.Body.Close()`

- **Path-параметры**: `r.PathValue("id")`, при этом в эндпоинте должен быть указан этот параметр

```go
http.HandleFunc(
	"GET /users/{id}",
	func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id") // Строка из сегмента пути
		if id == "" {
			http.Error(w, "missing id", http.StatusBadRequest)
			return
		}
	}
)
```

## Отправка сообщения об ошибке

В случае неудачного исхода запроса можно отправить ошибку с необходимым статусом через `http.Error(w responseWriter, msg string, code int)`.

Например: `http.Error(w, "Не указан город", http.StatusBadRequest)`

## Пример полноценного обработчика

```go
type createUserReq struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

func createUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()
	
	var req createUserReq
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // опционально: лишние поля в JSON → ошибка
	
	if err := dec.Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}
	
	// работаем с ответом
	w.Header().Set("Content-Type", "application/json")
	
	_ = json.NewEncoder(w).Encode(map[string]string{
		"ok":    "true",
		"saved": req.Name,
	})
}

func main() {
	http.HandleFunc("POST /users", createUser)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Маршрутизация запросов

Когда мы биндим определенную функцию к эндпоинту через вызов `http.HandleFunc`, то по умолчанию эта функция закрепляется к дефолтному маршрутизатору `http.DefaultServeMux`. Это структура типа `http.ServeMux`, реализующая интерфейс `http.Handler`, которая управляет соотнесением обработчиков к нужным эндпоинтам запроса. Но иногда нам могут понадобится логически отделенные группы маршрутов, тогда для каждого можно определить свой маршрутизатор и биндить обработчики на него:

```go
mux := http.NewServeMux()

mux.HandleFunc("/tasks", tasksHandler)          // Список задач
mux.HandleFunc("/tasks/add", addTaskHandler)    // Создание новой задачи
mux.HandleFunc("/tasks/", taskNotFoundHandler)  // Обработчик по умолчанию для подмаршрутов

http.ListenAndServe(":8080", mux) // используем наш обработчик
```

Маршрут с завершающим слэшем `/tasks/` воспринимается как префик для более точных эндопоинтов. То есть, если клиент сделал запрос `/tasks/info` (условно), и для такого эндпоинта не установлен обработчик, то выполнится обработчик для `/tasks/`.

## Запросы к внешним API

Иногда нашему приложение необходимо сделать запрос в стороннему API. Для этого в пакете `net/http` есть соответствующие методы `http.Get`, `http.Post` и другие. Пример запроса к API погоды:

```go
type WeatherResponse struct {  
    CurrentWeather struct {  
       Temperature float64 `json:"temperature"`  
    } `json:"current_weather"`  
}  
  
type Coordinates struct {  
    Lat  float64  
    Long float64  
}  
  
var cityCoordinates = map[string]Coordinates{  
    "Москва": {Lat: 55.7558, Long: 37.6173},  
    "Лондон": {Lat: 51.5074, Long: -0.1278},  
    "Нью-Йорк": {Lat: 40.7128, Long: -74.0060},  
}  
  
func getWeather(city string) (float64, error) {  
    coords, exists := cityCoordinates[city]  
    if !exists {  
       return 0, fmt.Errorf("город не найден")  
    }
    
    baseURL, err := url.Parse("https://api.open-meteo.com/v1/forecast")  
    if err != nil {  
       return 0, err  
    }
    
    queryParams := url.Values{}  
    queryParams.Set("latitude", fmt.Sprintf("%f", coords.Lat))  
    queryParams.Set("longitude", fmt.Sprintf("%f", coords.Long))  
    queryParams.Set("current_weather", "true")  
    baseURL.RawQuery = queryParams.Encode()  
    
    resp, err := http.Get(baseURL.String()) // возвращает *http.Response
    // err != nil только в случае, если сам запрос не удалось выполнить
    // в случае 4xx, 5xx статус-кодов err все равно будет nil
    if err != nil {  
       return 0, err  
    }  
    defer resp.Body.Close()  
    
    var data WeatherResponse  
    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {  
       return 0, err  
    }  
    
    return data.CurrentWeather.Temperature, nil  
}
```

Пример POST-запроса:

```go
// http.Post(url, contentType, bodyReader)
resp, err := http.Post(
	"https://httpbin.org/post",
	"application/json",
	strings.NewReader(`{"x": 5}`)
)
```

Или можно поэтапно собрать запрос:

```go
req, _ := http.NewRequest("PUT", "http://example.com/resource/42", bodyReader)
req.Header.Set("Authorization", "Bearer TOKEN")
resp, err := http.DefaultClient.Do(req)
```

## Middlewares

Часто возникает потребность выполнять некоторый код до или после основного обработчика маршрута - например, логировать запросы, проверять авторизацию, обрабатывать ошибки единообразно или изменять выходные данные (сжатие, кеширование). Такой шаблон называется **middleware** (промежуточное ПО)

Это можно сделать через функцию-декоратор, которая принимает один http-обработчик и возвращает новый обработчик, который является оберткой изначального с добавленной логикой:

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	    // Логируем метод и путь
        fmt.Printf("=> %s %s\n", r.Method, r.URL.Path)
        
        // Вызываем следующий обработчик
        next.ServeHTTP(w, r)
        
        fmt.Println("<= отправлен ответ клиенту")
    })
}
```

Далее нам достаточно обернуть необходимые нам обработчики через этот middleware или применить ко всему маршрутизатору (т.к он тоже обработчик):

```go
mux := http.NewServeMux()
// ... регистрация маршрутов ...
loggedMux := loggingMiddleware(mux)
http.ListenAndServe(":8080", loggedMux)
```

