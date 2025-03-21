Вот **полный пример** с **подробными комментариями об аналогах Maven**, сохраняя логику для разработчика, знающего Maven:

---

### **Структура проекта**
```
parent-project/              // Корневая директория (аналог <project>)
├── settings.gradle          // Аналог <modules> в Maven
├── gradle.properties        // Аналог <properties>
├── build.gradle             // Корневая конфигурация (аналог pom.xml)
├── parent/                  // BOM-модуль (аналог <dependencyManagement>)
│   └── build.gradle
├── impl/                    // Модуль реализации (аналог <module>)
│   └── build.gradle
└── db/                      // Модуль БД с Liquibase
    ├── build.gradle
    └── liquibase.properties // Конфигурация Liquibase (аналог <configuration>)
```

---

### **`gradle.properties` (аналог `<properties>`)
```properties
# Версии зависимостей (аналог <properties> в Maven)
springBootVersion=3.1.2      // ${spring-boot.version}
javaVersion=17               // maven.compiler.source
mapstructVersion=1.5.5.Final // mapstruct.version
lombokVersion=1.18.30        // lombok.version
postgresqlVersion=42.6.0     // postgresql.version
liquibaseVersion=4.23.1      // liquibase.version

# Версии плагинов (аналог <pluginManagement>)
lombokPluginVersion=8.0.1    // io.freefair.lombok
mapstructPluginVersion=0.21  // net.ltgt.apt
liquibasePluginVersion=2.2.0 // org.liquibase.gradle

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
    id 'org.springframework.boot' version "${springBootVersion}" apply false  // spring-boot-maven-plugin
    id 'maven-publish'  // Аналог maven-deploy-plugin
    id 'net.ltgt.apt' version "${mapstructPluginVersion}" apply false  // Поддержка аннотационных процессоров
    id 'io.freefair.lombok' version "${lombokPluginVersion}" apply false  // Аналог lombok-maven-plugin
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(javaVersion)  // <maven.compiler.source>
    }
}

repositories {
    mavenCentral()  // Центральный репозиторий (аналог <repository>)
    maven {         // Приватный Nexus (аналог <repository> + <server>)
        url = nexusUrl
        credentials {
            username = nexusUser  // ${nexusUser} из properties
            password = nexusPassword
        }
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'net.ltgt.apt'  // Для MapStruct (аналог maven-processor-plugin)
    apply plugin: 'io.freefair.lombok'  // Для Lombok
    
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

### **`parent/build.gradle` (BOM-модуль, аналог `<dependencyManagement>`)
```groovy
plugins {
    id 'java-platform'  // Плагин для создания BOM
}

dependencies {
    constraints {
        // Spring Boot (аналог управления версиями в <dependencyManagement>)
        api "org.springframework.boot:spring-boot-starter:${springBootVersion}"
        
        // MapStruct (аналог <dependencyManagement>)
        api "org.mapstruct:mapstruct:${mapstructVersion}"
        annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
        
        // Lombok (аналог <dependencyManagement>)
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

### **`impl/build.gradle` (модуль реализации, аналог `<module>`)
```groovy
plugins {
    id 'org.springframework.boot'  // spring-boot-maven-plugin
    id 'net.ltgt.apt'  // Для MapStruct (аналог maven-processor-plugin)
    id 'io.freefair.lombok'  // Для Lombok
}

dependencies {
    implementation platform(project(':parent'))  // Аналог <parent>
    implementation project(':db')  // Зависимость на другой модуль (аналог <dependency>)
    
    // Spring Web (аналог <dependency> без версии)
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // MapStruct (аналог <dependency> + аннотационный процессор)
    implementation 'org.mapstruct:mapstruct'
    annotationProcessor 'org.mapstruct:mapstruct-processor'
    
    // Lombok (автоматически настраивается плагином)
    
    // PostgreSQL (версия из BOM)
    runtimeOnly 'org.postgresql:postgresql'
}

// Настройки MapStruct (аналог mapstruct-processor в Maven)
tasks.withType(JavaCompile) {
    options.annotationProcessorGeneratedSourcesDirectory = file("$buildDir/generated/sources/mapstruct")
}

// Настройки Lombok (аналог lombok-maven-plugin)
lombok {
    version = lombokVersion  // Из gradle.properties
    config.put("lombok.addLombokGeneratedAnnotation", "true")
    config.put("lombok.addNullAnnotations", "javax.annotation.Nonnull")
}

// Настройки Spring Boot (аналог spring-boot-maven-plugin)
bootJar {
    mainClass = 'com.example.Application'  // Аналог <start-class>
}
```

---

### **`db/build.gradle` (модуль БД, аналог `<module>`)
```groovy
plugins {
    id 'org.liquibase.gradle' version "${liquibasePluginVersion}"  // liquibase-maven-plugin
    id 'io.freefair.lombok'  // Для Lombok
}

// Загрузка настроек из файла (аналог <configuration><propertyFile>)
def liquibaseProps = new Properties()
file("liquibase.properties").withInputStream { liquibaseProps.load(it) }

dependencies {
    implementation platform(project(':parent'))  // Аналог <parent>
    
    // Lombok (автоматически настраивается плагином)
    
    // PostgreSQL (версия из BOM)
    runtimeOnly 'org.postgresql:postgresql'
    
    // Liquibase (аналог <dependency>)
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

### **`db/liquibase.properties` (аналог `<configuration>`)
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
| `lombok-maven-plugin`              | `id 'io.freefair.lombok'`                      | В модулях `impl`/`db` |
| `mapstruct-processor`              | `id 'net.ltgt.apt'` + `annotationProcessor`    | В модуле `impl`      |
| `liquibase-maven-plugin`           | `id 'org.liquibase.gradle'`                    | В модуле `db`        |

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

### **Особенности для разработчика из Maven**
1. **Зависимости**:
   ```groovy
   implementation 'org.springframework.boot:spring-boot-starter-web'  // Аналог <dependency> без версии
   runtimeOnly 'org.postgresql:postgresql'  // Аналог <scope>runtime</scope>
   ```

2. **Аннотационные процессоры**:
   ```groovy
   annotationProcessor 'org.mapstruct:mapstruct-processor'  // Аналог maven-processor-plugin
   ```

3. **Генерация кода**:
   - MapStruct: `build/generated/sources/mapstruct` (аналог `target/generated-sources`)
   - Lombok: Встроено в компилятор

Этот пример сохраняет **полную аналогию с Maven**:
- Все версии централизованы в `gradle.properties`
- BOM-модуль управляет версиями зависимостей
- Плагины и задачи соответствуют Maven-аналогам
- Комментарии объясняют каждую секцию с точки зрения Maven
