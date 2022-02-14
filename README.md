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

Первый пользователь:
![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/postman-user1.png)

Второй пользователь:
![Alt-текст](https://github.com/aberdar/ktor-websockets-chat/blob/main/image/postman-user2.png)

This repository is the code corresponding to the hands-on lab [Creating a WebSocket Chat](https://ktor.io/docs/creating-web-socket-chat.html). 
