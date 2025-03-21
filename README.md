Вот полная конфигурация с **подробными комментариями**, объясняющими аналогии с Maven:

---

### **Структура проекта**
```
parent-project/
├── settings.gradle       // Аналог <modules> в Maven
├── gradle.properties     // Аналог <properties> в Maven
├── build.gradle          // Аналог родительского pom.xml
├── parent/               // Модуль BOM (аналог <dependencyManagement>)
│   └── build.gradle
├── impl/                 // Модуль реализации
│   └── build.gradle
└── db/                   // Модуль БД с Liquibase
    ├── build.gradle
    └── src/main/resources/db/changelog/  // Миграции Liquibase
```

---

### **`gradle.properties` (аналог `<properties>`)
```properties
# Версии зависимостей (аналог <properties> в Maven)
springBootVersion=3.1.2     // ${spring-boot.version}
javaVersion=17              // maven.compiler.source
postgresqlVersion=42.6.0    // postgresql.version
liquibaseVersion=4.23.1     // liquibase.version

# Параметры подключения к БД (аналог <properties> в profiles)
db.url=jdbc:postgresql://localhost:5432/mydb      // ${db.url}
db.username=postgres        // ${db.username}
db.password=mysecretpassword // ${db.password}

# Nexus репозиторий (аналог <distributionManagement>)
nexusUrl=https://nexus.example.com/repository/maven-releases/
nexusUser=deployer
nexusPassword=secure_password_123
```

---

### **`settings.gradle` (аналог `<modules>`)
```groovy
include 'parent', 'impl', 'db'  // Список модулей как в <modules>
rootProject.name = 'parent-project'  // <artifactId> корневого проекта
```

---

### **Корневой `build.gradle` (аналог родительского `pom.xml`)
```groovy
plugins {
    id 'java'  // Аналог maven-compiler-plugin
    id 'org.springframework.boot' version "${springBootVersion}" apply false  // Аналог spring-boot-maven-plugin
    id 'maven-publish'  // Аналог maven-deploy-plugin
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(javaVersion)  // Аналог <maven.compiler.source>
    }
}

repositories {
    mavenCentral()  // Аналог <repository> для Central
    maven {         // Аналог <repository> + <server> из settings.xml
        url = nexusUrl
        credentials {
            username = nexusUser  // ${nexusUser} из properties
            password = nexusPassword
        }
    }
}

subprojects {  // Аналог <build> секции для всех модулей
    apply plugin: 'java'
    
    // Настройки кэша (аналог <updatePolicy>)
    configurations.all {
        resolutionStrategy.cacheDynamicVersionsFor 7, 'days'
    }
}

publishing {  // Аналог <distributionManagement>
    repositories {
        maven {
            url = nexusUrl
            credentials(PasswordCredentials)
        }
    }
}
```

---

### **`parent/build.gradle` (аналог BOM)
```groovy
plugins {
    id 'java-platform'  // Плагин для создания BOM
}

javaPlatform {
    allowDependencies()  // Разрешаем объявлять зависимости
}

dependencies {
    constraints {
        // Spring Boot (аналог <dependencyManagement>)
        api "org.springframework.boot:spring-boot-starter:${springBootVersion}"
        
        // PostgreSQL (аналог <dependency> с версией в <dependencyManagement>)
        api "org.postgresql:postgresql:${postgresqlVersion}"
        
        // Liquibase (аналог управления версиями через BOM)
        api "org.liquibase:liquibase-core:${liquibaseVersion}"
    }
}
```

---

### **`impl/build.gradle` (аналог дочернего `pom.xml`)
```groovy
plugins {
    id 'org.springframework.boot'  // Аналог spring-boot-maven-plugin
}

dependencies {
    implementation platform(project(':parent'))  // Аналог <parent> с BOM
    implementation project(':db')  // Аналог <dependency> на другой модуль
    
    // Spring Web (аналог <dependency> без указания версии)
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // PostgreSQL (версия берется из BOM)
    runtimeOnly 'org.postgresql:postgresql'
}

// Настройки Spring Boot (аналог spring-boot-maven-plugin)
bootJar {
    mainClass = 'com.example.Application'  // Аналог <start-class>
}
```

---

### **`db/build.gradle` (аналог модуля с Liquibase)
```groovy
plugins {
    id 'org.liquibase.gradle' version '2.2.0'  // Аналог liquibase-maven-plugin
}

dependencies {
    liquibaseRuntime 'org.liquibase:liquibase-core'  // Аналог <dependency> для плагина
    liquibaseRuntime 'org.postgresql:postgresql'     // Драйвер БД для Liquibase
}

liquibase {
    activities {
        main {
            changeLogFile = project.property('db.changeLogFile')  // Аналог <changeLogFile>
            url = project.property('db.url')          // Аналог <url>
            username = project.property('db.username')  // Аналог <username>
            password = project.property('db.password')  // Аналог <password>
        }
    }
}

// Задача для миграций (аналог mvn liquibase:update)
task liquibaseUpdate(type: org.liquibase.gradle.LiquibaseTask) {
    args 'update'
}
```

---

### **Ключевые аналогии**
| **Maven**                          | **Gradle**                                      | **Где используется** |
|------------------------------------|------------------------------------------------|----------------------|
| `<properties>`                     | `gradle.properties`                            | Глобальные настройки |
| `<modules>`                        | `include` в `settings.gradle`                  | Структура проекта    |
| `<dependencyManagement>`           | Модуль `parent` с `java-platform`              | Управление версиями  |
| `mvn clean install`                | `./gradlew build`                              | Сборка проекта       |
| `mvn deploy`                       | `./gradlew publish`                            | Публикация в Nexus   |
| `liquibase-maven-plugin`           | `org.liquibase.gradle`                         | Миграции БД          |
| `<profiles>`                       | `-P` параметры или `gradle.properties`         | Параметризация       |

---

### **Сборка и запуск**
```bash
# Сборка всех модулей (аналог mvn clean install)
./gradlew build

# Выполнение миграций Liquibase (аналог mvn liquibase:update)
./gradlew :db:liquibaseUpdate

# Публикация в Nexus (аналог mvn deploy)
./gradlew publish
```

Этот пример показывает **прямые аналогии** между Maven и Gradle, сохраняя знакомую логику, но с использованием более гибкого Groovy-синтаксиса.
