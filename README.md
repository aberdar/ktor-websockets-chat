[![GitHub license](https://img.shields.io/badge/license-Apache%20License%202.0-blue.svg?style=flat)](https://www.apache.org/licenses/LICENSE-2.0)


# Chat App

## Содержание

  * [Конфигурация проекта](#конфигурация-проекта)
    * [Зависимости для серверного приложения](#зависимости-для-серверного-приложения)
    * [Конфигурация для серверного приложения](#конфигурация-для-серверного-приложения)
    * [Зависимости для клиентского проекта](#зависимости-для-клиентского-проекта)  

## Конфигурация проекта

### Зависимости для серверного приложения

Файл зависимостей серверного приложения ```server/build.gradle.kts```

```kotlin
dependencies {
    implementation("io.ktor:ktor-server-netty:$ktor_version")
    implementation("io.ktor:ktor-websockets:$ktor_version")
    implementation("ch.qos.logback:logback-classic:$logback_version")
}
```

### Конфигурация для серверного приложения

Ktor использует файл конфигурации HOCON для настройки своего базового поведения, такого как точка входа и порт развертывания.
Файл конфигурации серверного приложения ```server/src/main/resources/application.conf```

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

Клиентское приложение указывает две зависимости в файлу ```client/build.gradle.kts```

```kotlin
dependencies {
    implementation("io.ktor:ktor-client-websockets:$ktor_version")
    implementation("io.ktor:ktor-client-cio:$ktor_version")
}
```


This repository is the code corresponding to the hands-on lab [Creating a WebSocket Chat](https://ktor.io/docs/creating-web-socket-chat.html). 
