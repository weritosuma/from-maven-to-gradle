Вот **полный пример** с **Lombok**, **MapStruct**, **Liquibase** и комментариями об аналогах Maven:

---

### **Структура проекта**
```
parent-project/
├── settings.gradle       // Аналог <modules> в Maven
├── gradle.properties     // Аналог <properties>
├── build.gradle          // Корневая конфигурация
├── parent/               // BOM-модуль (аналог <dependencyManagement>)
│   └── build.gradle
├── impl/                 // Модуль реализации
│   └── build.gradle
└── db/                   // Модуль БД с Liquibase
    ├── build.gradle
    └── liquibase.properties  // Конфигурация Liquibase
```

---

### **`gradle.properties` (аналог `<properties>`)
```properties
# Версии зависимостей
springBootVersion=3.1.2     // ${spring-boot.version}
javaVersion=17              // maven.compiler.source
mapstructVersion=1.5.5.Final // mapstruct.version
lombokVersion=1.18.30       // lombok.version
postgresqlVersion=42.6.0    // postgresql.version
liquibaseVersion=4.23.1     // liquibase.version

# Nexus репозиторий
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
    id 'org.springframework.boot' version "${springBootVersion}" apply false  // spring-boot-maven-plugin
    id 'maven-publish'  // maven-deploy-plugin
    id 'net.ltgt.apt' version '0.21'  // Поддержка аннотационных процессоров (аналог maven-processor-plugin)
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(javaVersion)  // <maven.compiler.source>
    }
}

repositories {
    mavenCentral()  // Центральный репозиторий
    maven {         // Приватный Nexus (аналог <repository> + <server>)
        url = nexusUrl
        credentials {
            username = nexusUser
            password = nexusPassword
        }
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'net.ltgt.apt'  // Включаем поддержку аннотационных процессоров
    
    repositories {
        mavenCentral()
        maven { url = nexusUrl }
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

### **`parent/build.gradle` (BOM-модуль)
```groovy
plugins {
    id 'java-platform'  // Плагин для создания BOM
}

dependencies {
    constraints {
        // Spring Boot (аналог <dependencyManagement>)
        api "org.springframework.boot:spring-boot-starter:${springBootVersion}"
        
        // MapStruct (аналог управления версиями в <dependencyManagement>)
        api "org.mapstruct:mapstruct:${mapstructVersion}"
        annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
        
        // Lombok (аналог управления версиями)
        api "org.projectlombok:lombok:${lombokVersion}"
        annotationProcessor "org.projectlombok:lombok:${lombokVersion}"
        
        // PostgreSQL
        api "org.postgresql:postgresql:${postgresqlVersion}"
        
        // Liquibase
        api "org.liquibase:liquibase-core:${liquibaseVersion}"
    }
}
```

---

### **`impl/build.gradle` (модуль реализации)
```groovy
plugins {
    id 'org.springframework.boot'  // spring-boot-maven-plugin
}

dependencies {
    implementation platform(project(':parent'))  // Аналог <parent>
    implementation project(':db')  // Зависимость на модуль БД
    
    // Spring Web (аналог <dependency> без версии)
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // MapStruct (аналог <dependency> + аннотационный процессор)
    implementation 'org.mapstruct:mapstruct'
    annotationProcessor 'org.mapstruct:mapstruct-processor'
    
    // Lombok (аналог <dependency> с scope=provided)
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // PostgreSQL (версия из BOM)
    runtimeOnly 'org.postgresql:postgresql'
}

// Настройки Spring Boot (аналог spring-boot-maven-plugin)
bootJar {
    mainClass = 'com.example.Application'  // Аналог <start-class>
}

// Настройки MapStruct (аналог mapstruct-processor в Maven)
tasks.withType(JavaCompile) {
    options.annotationProcessorGeneratedSourcesDirectory = file("$buildDir/generated/sources/mapstruct")
}

// Настройки Lombok (аналог lombok-maven-plugin)
tasks.withType(JavaCompile) {
    options.compilerArgs += [
        "-A lombok.addLombokGeneratedAnnotation=true",
        "-A lombok.addNullAnnotations=javax.annotation.Nonnull"
    ]
}
```

---

### **`db/build.gradle` (модуль БД с Liquibase)
```groovy
plugins {
    id 'org.liquibase.gradle' version '2.2.0'  // liquibase-maven-plugin
}

// Загрузка настроек из файла (аналог <configuration><propertyFile>)
def liquibaseProps = new Properties()
file("liquibase.properties").withInputStream { liquibaseProps.load(it) }

dependencies {
    implementation platform(project(':parent'))
    
    // Lombok (аналог <dependency> с scope=provided)
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // PostgreSQL
    runtimeOnly 'org.postgresql:postgresql'
    
    // Liquibase
    liquibaseRuntime 'org.liquibase:liquibase-core'
}

// Конфигурация Liquibase (аналог <configuration> в Maven)
liquibase {
    activities {
        main {
            changeLogFile = liquibaseProps.getProperty('changeLogFile')  // Аналог <changeLogFile>
            url = liquibaseProps.getProperty('url')          // Аналог <url>
            username = liquibaseProps.getProperty('username')  // Аналог <username>
            password = liquibaseProps.getProperty('password')  // Аналог <password>
        }
    }
}

// Задачи для Liquibase (аналог mvn liquibase:update)
task updateDB(type: org.liquibase.gradle.LiquibaseTask) {
    args 'update'
}

task rollbackDB(type: org.liquibase.gradle.LiquibaseTask) {
    args 'rollbackCount', '1'
}
```

---

### **`db/liquibase.properties` (конфигурация Liquibase)
```properties
# Подключение к PostgreSQL (аналог <configuration> в Maven)
url=jdbc:postgresql://localhost:5432/mydb
username=postgres
password=mysecretpassword
changeLogFile=src/main/resources/db/changelog-master.xml
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
| `lombok-maven-plugin`              | `compileOnly` + `annotationProcessor`          | В модулях `impl`/`db` |
| `mapstruct-processor`              | `annotationProcessor` + `net.ltgt.apt`         | В модуле `impl`      |
| `liquibase-maven-plugin`           | `org.liquibase.gradle`                         | В модуле `db`        |

---

### **Примеры команд**
```bash
# Сборка проекта (аналог mvn clean install)
./gradlew build

# Запуск приложения (Spring Boot)
./gradlew :impl:bootRun

# Выполнение миграций (аналог mvn liquibase:update)
./gradlew :db:updateDB

# Откат миграций (аналог mvn liquibase:rollback)
./gradlew :db:rollbackDB

# Публикация в Nexus (аналог mvn deploy)
./gradlew publish
```

---

### **Особенности**
1. **Lombok**:
   ```groovy
   compileOnly 'org.projectlombok:lombok'  // Аналог <scope>provided</scope>
   annotationProcessor 'org.projectlombok:lombok'  // Аннотационный процессор
   ```

2. **MapStruct**:
   ```groovy
   implementation 'org.mapstruct:mapstruct'  // Основная зависимость
   annotationProcessor 'org.mapstruct:mapstruct-processor'  // Генерация кода
   ```

3. **Генерация кода**:
   ```groovy
   // Для MapStruct
   options.annotationProcessorGeneratedSourcesDirectory = file("$buildDir/generated/sources/mapstruct")
   
   // Для Lombok
   options.compilerArgs += ["-A lombok.addLombokGeneratedAnnotation=true"]
   ```

Этот пример сохраняет **все ключевые элементы**:
- Полноценная мульти-модульная структура
- Интеграция Lombok/MapStruct с комментариями об аналогах
- Вынос конфигурации Liquibase в отдельный файл
- Готовые задачи для работы с миграциями
- Подробные аналогии с Maven в таблице и комментариях
