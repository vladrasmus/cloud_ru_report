# cloud_ru_report Пархоменко В. Т. 2024
Отчет по тестовому заданию от Cloud.ru

## 1. Вопросы для разогрева

1. Расскажите, с какими задачами в направлении безопасной разработки вы сталкивались?
- с задачами безопасной разработки не сталкивался

2. Если вам приходилось проводить security code review или моделирование угроз, расскажите, как это было?
- проводить security code review не приходилось, однако был опыт установки, настройки и автоматического запуска системы статического анализа кода на основе Jenkins и SonarQube в рамках лабораторной работы.

3. Если у вас был опыт поиска уязвимостей, расскажите, как это было?
- в реальных проектах опыта поиска уязвимостей не было

4. Почему вы хотите участвовать в стажировке?
- есть желание расти в области ИБ, попробовать себя в сфере безопасной разработки

---

## 2. Security code review

### Часть 1. Security code review: GO

Код:
```
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "github.com/go-sql-driver/mysql"
)

var db *sql.DB
var err error

func initDB() {
    db, err = sql.Open("mysql", "user:password@/dbname")
    if err != nil {
        log.Fatal(err)
    }

err = db.Ping()
if err != nil {
    log.Fatal(err)
    }
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, "Method is not supported.", http.StatusNotFound)
        return
    }
-------------------------------------------------------------------------------------------
(2.1.1/2.2.2)
    # в строке ниже searchQuery берется напрямую из параметров URL без обработки/фильтрации:
searchQuery := r.URL.Query().Get("query")
if searchQuery == "" {
    http.Error(w, "Query parameter is missing", http.StatusBadRequest)
    return
}
-------------------------------------------------------------------------------------------
(2.1.1/2.2.2)
    # строка ниже создает риск SQL инъекции, так как searchQuery может содержать вредоносные SQL команды.
query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
rows, err := db.Query(query)
if err != nil {
    http.Error(w, "Query failed", http.StatusInternalServerError)
    log.Println(err)
    return
}
defer rows.Close()

var products []string
for rows.Next() {
    var name string
    err := rows.Scan(&name)
    if err != nil {
        log.Fatal(err)
    }
    products = append(products, name)
}

fmt.Fprintf(w, "Found products: %v\n", products)
}

func main() {
    initDB()
    defer db.Close()

http.HandleFunc("/search", searchHandler)
fmt.Println("Server is running")
-------------------------------------------------------------------------------------------

(2.1.1/2.2.2)
    # Отсутствие HTTPS
log.Fatal(http.ListenAndServe(":8080", nil))
}
```
2.1.1 Какие уязвимости присутствуют в этом фрагменте кода? и 2.1.2 Указать строки, в которых присутствуют уязвимости
- Неотфильтрованный пользовательский ввод, который может привести к SQL-инъекции
- Отсутствие HTTPS, слушается нешифрованный трафик, который может быть прослушан
(пометки прямо в коде по маркеру 2.1.1/2.2.2)

2.1.3 К каким последствиям может привести эксплуатация найденных уязвимостей злоумышленником?
- SQL инъекция: злоумышленник может выполнить произвольные SQL-запросы к базе данных, что может привести к чтению, модификации или удалению данных. Также возможно получение контроля над базой данных и выполнение административных операций в зависимости от уровня доступа.
- Отсутствие HTTPS: трафик между клиентом и сервером не защищен, что позволяет злоумышленнику перехватывать, читать и изменять передаваемые данные. Это также может привести к утечке информации.

2.1.4 Описать способы исправления уязвимостей.
- SQL инъекция: можно использовать параметризованные запросы для предотвращения SQL инъекций,обработка запроса должна выглядеть примерно так:
```
query := "SELECT * FROM products WHERE name LIKE ?"
rows, err := db.Query(query, "%"+searchQuery+"%")
```
"?" служит заполнителем для значения searchQuery, и подставляется безопасно, так что оно не может изменить структуру SQL запроса. Такой подход обрабатывает специальные символы в searchQuery будут таким образом, чтобы они интерпретировались не как часть SQL команды и не были связаны с внутренним кодом.
- Отсутствие HTTPS: нужно настроить сервер так, чтобы он использовал HTTPS для шифрования трафика, для этого понадобится сертификат. после настройки, нужно изменить код, заменив строчку в конце на:

```
log.Fatal(http.ListenAndServeTLS(":443", "server.crt", "server.key", nil))
```
Где server.crt и server.key — это пути к SSL сертификату и приватному ключу соответственно.

2.1.5 Если уязвимость можно исправить несколькими способами, необходимо перечислить их, выбрать лучший по вашему мнению и аргументировать свой выбор.
- SQL инъекция: можно вручную добавить кастомное правило обработки запроса, однако это не целесообразно и подход на данный момент является самым оптимальным


-------------------------------------------------------------------------------------------
### Часть 2: Security code review: Python

Код №1:
```
from flask import Flask, request
from jinja2 import Template

app = Flask(name)

@app.route("/page")
def page():
    name = request.values.get('name')
    age = request.values.get('age', 'unknown')
-------------------------------------------------------------------------------------------
    (2.2.1)
        # name и age, которые были получены из пользовательского ввода выше не фильтруются
    output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
return output

if name == "main":
    app.run(debug=True)
```

Тип уязвимости: Server-Side Template Injection (SSTI)
1 Указать строки, в которых присутствуют уязвимости.
- (пометка прямо в коде по маркеру (2.2.1))

2 К каким последствиям может привести эксплуатация данных уязвимостей злоумышленником?
- Это может позволить злоумышленнику манипулировать шаблоном, что может привести к выполнению произвольного кода, получению доступа к файлам сервера, а также к другим вредоносным действиям в зависимости от конфигурации сервера

3 Описать способы исправления уязвимостей.
- следует избегать непосредственного включения пользовательского ввода, также нужно ввести фильтрацию пользовательского ввода, с библиотекой jinja2 можно реализовать, например, такой подход:
```
output = Template('Hello {{ name }}! Your age is {{ age }}.').render(name=name, age=age)
```
то есть, будет происходить экранирование строк, что предотвратит внедрение вредоносных инструкций, при этом на сервере настройки Jinja2 не должны позволять выполнение произвольного кода

4 Если уязвимость можно исправить несколькими способами, необходимо перечислить их, выбрать лучший по вашему мнению и аргументировать свой выбор.
- описаный способо является оптимальным, так как испльзует встроенную функцию уже импортированной библиотеки

-------------------------------------------------------------------------------------------

Код №2:
```
from flask import Flask, request
import subprocess

app = Flask(name)

@app.route("/dns")
def dns_lookup():
    hostname = request.values.get('hostname')
-------------------------------------------------------------------------------------------
    (2.2.2)
        # hostname получается из пользовательского вывода и не фильтруется
    cmd = 'nslookup ' + hostname
    output = subprocess.check_output(cmd, shell=True, text=True)
return output
if name == "main":
    app.run(debug=True)
```
Тип уязвимости: Command Injection
1 Указать строки, в которых присутствуют уязвимости.
- (пометка прямо в коде по маркеру (2.2.2))

2 К каким последствиям может привести эксплуатация данных уязвимостей злоумышленником?
- Пользовательский ввод hostname добавляется непосредственно в строку команды nslookup, которая затем выполняется в оболочке, если пользовательский ввод hostname будет содержать вредоносные команды например, через ";", "&&", "||" (то есть добавление команд в придачу к nslookup), злоумышленник может выполнить произвольные команды на сервере.

3 Описать способы исправления уязвимостей.
- во-первых, можно избегать выполнения оболочечных команд с пользовательским вводом. Если возможно, можно использовать встроенные средства языка программирования или библиотеки для достижения желаемого функционала вместо обращения к внешним командам
- во-вторых, можно добавить проверку и очистку пользовательского ввода, например, с помощью регулярных выражений
- в-третьих, можно использовать встроенные безопасные методы, например, с помощью subprocess.run() и передчаей аргументов в виде списка
```
cmd = ['nslookup', hostname]
output = subprocess.run(cmd, capture_output=True, text=True).stdout
```

4 Если уязвимость можно исправить несколькими способами, необходимо перечислить их, выбрать лучший по вашему мнению и аргументировать свой выбор.
- способ с использованием безопасных методов - наиболее оптимальный в данном случае, при таком взимодействии, даже если пользователь ввел какую нибудь команду, она не будет выполняться в оболочке (по умолчанию у subprocess.run параметр shell=False)




