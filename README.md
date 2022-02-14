[![GitHub license](https://img.shields.io/badge/license-Apache%20License%202.0-blue.svg?style=flat)](https://www.apache.org/licenses/LICENSE-2.0)


# Chat App

## Содержание

  * [Конфигурация проекта](#конфигурация-проекта)
    * [Зависимости для серверного приложения](#зависимости-для-серверного-приложения)
    * [Конфигурация для серверного приложения](#конфигурация-для-серверного-приложения)
    * [Зависимости для клиентского проекта](#зависимости-для-клиентского-проекта)
  * [Эхо-сервер](#эхо-сервер)
    * [Реализация](#реализация-эхо-сервера)
    * [Запуск](#запуск-эхо-сервера)
  * [Обмен сообщениями](#обмен-сообщениями)
    * [Моделирование соединений](#моделирование-соединений)
    * [Реализация обработки соединений и распространения сообщений](#реализация-обработки-соединений-и-распространения-сообщений)
  * [Создание чат-клиента](#создание-чат-клиента)
    * [Улучшение решения](#улучшение-решения)
    * [Соединение](#соединение)
  * [Результат](#результат)      
___
## Конфигурация проекта

### Зависимости для серверного приложения

Файл зависимостей серверного приложения: ```server/build.gradle.kts```

```kotlin
dependencies {
    implementation("io.ktor:ktor-server-netty:$ktor_version")
    implementation("io.ktor:ktor-websockets:$ktor_version")
    implementation("ch.qos.logback:logback-classic:$logback_version")
}
```

### Конфигурация для серверного приложения

Ktor использует файл конфигурации HOCON для настройки своего базового поведения, такого как точка входа и порт развертывания.

Файл конфигурации серверного приложения: ```server/src/main/resources/application.conf```

```koltin
ktor {
    deployment {
        port = 8080
    }
    application {
        modules = [ com.jetbrains.handson.chat.ApplicationKt.module ]
    }
}
```

### Зависимости для клиентского проекта

Клиентское приложение указывает две зависимости в файле ```client/build.gradle.kts```

```kotlin
dependencies {
    implementation("io.ktor:ktor-client-websockets:$ktor_version")
    implementation("io.ktor:ktor-client-cio:$ktor_version")
}
```

## Эхо-сервер

### Реализация эхо-сервера
Служба “echo” принимает подключения к WebSocket, получает текстовое содержимое и отправляет его обратно клиенту.

Файл: ```server/src/main/kotlin/com/jetbrains/handson/chat/server/Application.kt```

```kotlin
fun Application.module() {
    install(WebSockets)
    routing {
        webSocket("/chat") {
            send("You are connected!")
            for(frame in incoming) {
                frame as? Frame.Text ?: continue
                val receivedText = frame.readText()
                send("You said: $receivedText")
            }
        }
    }
}
```
Сначала мы включаем функции, связанные с ```WebSocket```, предоставляемые платформой ```Ktor```, установив ```WebSockets``` плагин Ktor. Это позволяет нам определять конечные точки в нашей маршрутизации, которые отвечают протоколу ```WebSocket``` (в нашем случае это маршрут ```/chat```). В рамках функции ```webSocket``` маршрута мы можем использовать различные методы взаимодействия с нашими клиентами (через ```DefaultWebSocketServerSession``` - тип получателя). Это включает в себя удобные методы отправки сообщений и перебора полученных сообщений.

Поскольку нас интересует только текстовое содержимое, мы пропускаем любые нетекстовые ```Frame``` сообщения, которые мы получаем при повторении по входящему каналу.

### Запуск эхо-сервера

После завершения компиляции нашего проекта мы должны увидеть подтверждение того, что сервер запущен в окне инструмента IntelliJ IDEAs Run:

```Application - Responding at http://0.0.0.0:8080```

Чтобы опробовать услугу, мы можем использовать Postman для подключения ```ws://localhost:8080/chat``` и отправки запроса на веб-сайт.

![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/postman-connect.png)

## Обмен сообщениями

### Моделирование соединений

Ktor управляет соединением WebSocket с объектом типа ```DefaultWebSocketSession```, который содержит все необходимое для общения через WebSockets, включая каналы ```incoming``` и ```outgoing```, удобные методы для связи и многое другое. На данный момент мы можем упростить задачу назначения имен пользователей и просто дать каждому участнику автоматически сгенерированное имя пользователя на основе счетчика.

Файл ```server/src/main/kotlin/com/jetbrains/handson/chat/server/Connection.kt```

```kotlin
import io.ktor.http.cio.websocket.*
import java.util.concurrent.atomic.*

class Connection(val session: DefaultWebSocketSession) {
    companion object {
        var lastId = AtomicInteger(0)
    }
    val name = "user${lastId.getAndIncrement()}"
}
```

```AtomicInteger``` - потокобезопасная структура данных для счетчика. Это гарантирует, что два пользователя никогда не получат один и тот же идентификатор для своего имени пользователя, даже если два их объекта Connection создаются одновременно в разных потоках.

### Реализация обработки соединений и распространения сообщений

Настройте реализацию ```routing``` блока ```server/src/main/kotlin/com/jetbrains/handson/chat/server/Application.kt```

```kotlin
routing {
    val connections = Collections.synchronizedSet<Connection?>(LinkedHashSet())
    webSocket("/chat") {
        println("Adding user!")
        val thisConnection = Connection(this)
        connections += thisConnection
        try {
            send("You are connected! There are ${connections.count()} users here.")
            for (frame in incoming) {
                frame as? Frame.Text ?: continue
                val receivedText = frame.readText()
                val textWithUsername = "[${thisConnection.name}]: $receivedText"
                connections.forEach {
                    it.session.send(textWithUsername)
                }
            }
        } catch (e: Exception) {
            println(e.localizedMessage)
        } finally {
            println("Removing $thisConnection!")
            connections -= thisConnection
        }
    }
}
```

Наш сервер теперь хранит (поточно-безопасную) коллекцию ```Connections```. Когда пользователь подключается, мы создаем его ```Connection объект``` (который также присваивает себе уникальное имя пользователя) и добавляем его в коллекцию. Затем мы приветствуем нашего пользователя и сообщаем ему, сколько пользователей в настоящее время подключаются. Когда мы получаем сообщение от пользователя, мы добавляем к нему префикс уникального имени, связанного с его ```Connection объектом```, и отправляем его всем активным в данный момент соединениям. Наконец, мы удаляем объект клиента ```Connection``` из нашей коллекции, когда соединение завершается — либо изящно, когда закрывается входящий канал, либо с помощью , ```Exception``` когда сетевое соединение между клиентом и сервером неожиданно прерывается.

| user0 | user1 |
|:-----:|:-----:|
|![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/postman-user1.png)|![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/postman-user2.png)|

## Создание чат-клиента

Файл ```client/src/main/kotlin/com/jetbrains/handson/chat/client/ChatClient.kt```

```kotlin
import io.ktor.client.*
import io.ktor.client.features.websocket.*
import io.ktor.http.*
import io.ktor.http.cio.websocket.*
import io.ktor.util.*
import kotlinx.coroutines.*

fun main() {
    val client = HttpClient {
        install(WebSockets)
    }
    runBlocking {
        client.webSocket(method = HttpMethod.Get, host = "127.0.0.1", port = 8080, path = "/chat") {
            while(true) {
                val othersMessage = incoming.receive() as? Frame.Text ?: continue
                println(othersMessage.readText())
                val myMessage = readLine()
                if(myMessage != null) {
                    send(myMessage)
                }
            }
        }
    }
    client.close()
    println("Connection closed. Goodbye!")
}
```

Здесь мы сначала создаем ```HttpClient``` и настраиваем ```WebSocket``` плагин Ktor. Функции в Ktor, отвечающие за выполнение сетевых вызовов, используют механизм приостановки из сопрограмм Kotlin, поэтому мы заключаем наш сетевой код в ```runBlocking``` блок. Внутри обработчика WebSocket мы снова обрабатываем входящие сообщения и отправляем исходящие: игнорируем кадры, не содержащие текста, читаем входящий текст и отправляем пользовательский ввод на сервер.

### Улучшение решения

Лучшей структурой для нашего чат-клиента было бы разделение механизмов вывода и ввода сообщений, что позволило бы им работать одновременно: когда приходят новые сообщения, они печатаются немедленно, но наши пользователи по-прежнему могут начать составлять новое сообщение чата в любой момент.

Добавим вызываемую функцию ```outputMessages()``` в ```ChatClient.kt```

```kotlin
suspend fun DefaultClientWebSocketSession.outputMessages() {
    try {
        for (message in incoming) {
            message as? Frame.Text ?: continue
            println(message.readText())
        }
    } catch (e: Exception) {
        println("Error while receiving: " + e.localizedMessage)
    }
}
```

Поскольку функция работает в контексте a ```DefaultClientWebSocketSession```, мы определяем ```outputMessages()``` ее как функцию расширения типа. Также не забываем добавить ```suspend``` модификатор — потому что итерация по ```incoming``` каналу приостанавливает сопрограмму, пока нет доступных новых сообщений.

Определим вторую функцию, которая позволяет пользователю вводить текст.

```kotlin
suspend fun DefaultClientWebSocketSession.inputMessages() {
    while (true) {
        val message = readLine() ?: ""
        if (message.equals("exit", true)) return
        try {
            send(message)
        } catch (e: Exception) {
            println("Error while sending: " + e.localizedMessage)
            return
        }
    }
}
```

Еще раз определенная как приостанавливающая функция расширения на ```DefaultClientWebSocketSession```, единственной задачей этой функции является чтение текста из командной строки и отправка его на сервер или возврат, когда пользователь вводит ```exit```.

### Соединение

```kotlin
fun main() {
    val client = HttpClient {
        install(WebSockets)
    }
    runBlocking {
        client.webSocket(method = HttpMethod.Get, host = "127.0.0.1", port = 8080, path = "/chat") {
            val messageOutputRoutine = launch { outputMessages() }
            val userInputRoutine = launch { inputMessages() }

            userInputRoutine.join() // Wait for completion; either "exit" or error
            messageOutputRoutine.cancelAndJoin()
        }
    }
    client.close()
    println("Connection closed. Goodbye!")
}
```

Эта новая реализация улучшает поведение нашего приложения: как только соединение с нашим чат-сервером установлено, мы используем ```launch``` функцию из библиотеки Kotlin Coroutines для запуска двух долго работающих функций ```outputMessages()``` и ```inputMessages()``` новой сопрограммы (без блокировки текущего потока). Функция запуска также возвращает ```Job``` объект для обоих из них, который мы используем, чтобы программа работала до тех пор, пока пользователь не введет ```exit``` или не столкнется с сетевой ошибкой при попытке отправить сообщение. После ```inputMessages()``` возврата мы отменяем выполнение ```outputMessages()``` функции и ```close``` клиента.

## Результат

| Add users | Chat session user 1 | Chat session user 2 |
|:---------:|:-------------------:|:-------------------:|
|![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/cmd-add-user.png)|![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/chat-user1.png)|![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/chat-user2.png)|
___
This repository is the code corresponding to the hands-on lab [Creating a WebSocket Chat](https://ktor.io/docs/creating-web-socket-chat.html). 
