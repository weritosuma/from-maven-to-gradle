Вот полная конфигурация Gradle-проекта с **Liquibase**, **Spring Boot**, **MapStruct**, **Lombok** и **Nexus**:

---

### **Структура проекта**
```
parent-project/
├── settings.gradle
├── gradle.properties
├── build.gradle
├── parent/
│   └── build.gradle
├── impl/
│   └── build.gradle
└── db/
    ├── build.gradle
    ├── liquibase.properties
    └── src/main/resources/db/changelog/
        ├── changelog-master.xml
        └── changelog-1.0.xml
```

---

### **`gradle.properties` (версии и настройки)
```properties
# Версии зависимостей
springBootVersion=3.1.2
javaVersion=17
hibernateVersion=6.2.5.Final
mapstructVersion=1.5.5.Final
lombokVersion=1.18.30
postgresqlVersion=42.6.0
liquibaseVersion=4.23.1

# Nexus репозиторий
nexusUrl=https://nexus.example.com/repository/maven-releases/
nexusUser=deployer
nexusPassword=secure_password_123

# Параметры Liquibase
db.url=jdbc:postgresql://localhost:5432/mydb
db.username=postgres
db.password=mysecretpassword
db.driver=org.postgresql.Driver
db.changeLogFile=src/main/resources/db/changelog/changelog-master.xml

# Правила обновления зависимостей
dependencyChecksums=true
dependencyCacheDuration=7d
```

---

### **`settings.gradle` (модули)
```groovy
include 'parent', 'impl', 'db'
rootProject.name = 'parent-project'
```

---

### **Корневой `build.gradle`
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version "${springBootVersion}" apply false
    id 'maven-publish'
    id 'net.ltgt.apt' version '0.21'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(javaVersion)
    }
    withSourcesJar()
}

repositories {
    mavenCentral()
    maven {
        url = nexusUrl
        credentials {
            username = nexusUser
            password = nexusPassword
        }
        metadataSources {
            artifact()
            ignoreGradleMetadataRedirection()
        }
        content {
            releasesOnly()
        }
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'net.ltgt.apt'
    
    configurations.all {
        resolutionStrategy {
            cacheDynamicVersionsFor dependencyCacheDuration.toInteger(), 'days'
            cacheChangingModulesFor dependencyCacheDuration.toInteger(), 'days'
            enableDependencyVerification = dependencyChecksums.toBoolean()
        }
    }
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
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
    id 'java-platform'
}

javaPlatform {
    allowDependencies()
}

dependencies {
    constraints {
        // Spring
        api "org.springframework.boot:spring-boot-starter:${springBootVersion}"
        
        // Hibernate
        api "org.hibernate:hibernate-core:${hibernateVersion}"
        
        // Базовые зависимости
        api "com.mysql:mysql-connector-j:8.0.33"
        api "org.postgresql:postgresql:${postgresqlVersion}"
        api "org.junit.jupiter:junit-jupiter:5.8.1"
        
        // MapStruct
        api "org.mapstruct:mapstruct:${mapstructVersion}"
        annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
        
        // Lombok
        api "org.projectlombok:lombok:${lombokVersion}"
        annotationProcessor "org.projectlombok:lombok:${lombokVersion}"
        
        // Liquibase
        api "org.liquibase:liquibase-core:${liquibaseVersion}"
    }
}
```

---

### **`impl/build.gradle` (модуль реализации)
```groovy
plugins {
    id 'org.springframework.boot'
    id 'io.spring.dependency-management'
    id 'net.ltgt.apt'
}

dependencies {
    implementation platform(project(':parent'))
    implementation project(':db')
    
    // Spring
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // Базовые зависимости
    implementation 'com.mysql:mysql-connector-j'
    implementation 'com.google.guava:guava:31.1-jre'
    
    // MapStruct
    implementation 'org.mapstruct:mapstruct'
    annotationProcessor 'org.mapstruct:mapstruct-processor'
    
    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // Тесты
    testImplementation 'org.junit.jupiter:junit-jupiter'
}

// Настройки MapStruct
tasks.withType(JavaCompile) {
    options.annotationProcessorGeneratedSourcesDirectory = file("$buildDir/generated/sources/annotationProcessor/java")
}

// Настройки Lombok
tasks.withType(JavaCompile) {
    options.compilerArgs += [
        "-A lombok.addLombokGeneratedAnnotation=true",
        "-A lombok.addNullAnnotations=javax.annotation.Nonnull"
    ]
}

// Spring Boot конфигурация
bootJar {
    mainClass = 'com.example.Application'
    archiveClassifier = 'boot'
}
```

---

### **`db/build.gradle` (модуль БД с Liquibase)
```groovy
plugins {
    id 'java'
    id 'org.liquibase.gradle' version '2.2.0'  // Плагин Liquibase
}

dependencies {
    implementation platform(project(':parent'))
    
    // Hibernate
    implementation 'org.hibernate:hibernate-core'
    
    // PostgreSQL
    runtimeOnly 'org.postgresql:postgresql'
    
    // Liquibase
    liquibaseRuntime 'org.liquibase:liquibase-core'
    liquibaseRuntime 'org.postgresql:postgresql'
    
    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // Тесты
    testImplementation 'org.junit.jupiter:junit-jupiter'
}

// Конфигурация Liquibase
liquibase {
    activities {
        main {
            changeLogFile = project.property('db.changeLogFile')
            url = project.property('db.url')
            username = project.property('db.username')
            password = project.property('db.password')
            driver = project.property('db.driver')
            logLevel = 'info'
            defaultSchemaName = 'public'
        }
    }
    runList = 'main'
}

// Задача для выполнения миграций
task liquibaseUpdate(type: org.liquibase.gradle.LiquibaseTask) {
    args 'update'
}

// Задача для отката миграций
task liquibaseRollback(type: org.liquibase.gradle.LiquibaseTask) {
    args 'rollbackCount', '1'
}
```

---

### **`db/liquibase.properties`
```properties
url=jdbc:postgresql://localhost:5432/mydb
username=postgres
password=mysecretpassword
driver=org.postgresql.Driver
changeLogFile=src/main/resources/db/changelog/changelog-master.xml
```

---

### **`db/src/main/resources/db/changelog/changelog-master.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">

    <include file="changelog-1.0.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>
```

---

### **`db/src/main/resources/db/changelog/changelog-1.0.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">

    <changeSet id="1" author="dev">
        <createTable tableName="users">
            <column name="id" type="bigint" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="name" type="varchar(255)"/>
            <column name="email" type="varchar(255)"/>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

---

### **`~/.gradle/init.gradle` (глобальные настройки Nexus)
```groovy
allprojects {
    repositories {
        maven {
            url 'https://nexus.example.com/repository/maven-releases/'
            credentials {
                username = System.getenv('NEXUS_USER') ?: ''
                password = System.getenv('NEXUS_PASS') ?: ''
            }
        }
    }
}
```

---

### **Ключевые возможности**
1. **Liquibase**:
   ```bash
   # Выполнить миграции
   ./gradlew :db:liquibaseUpdate
   
   # Откатить последнюю миграцию
   ./gradlew :db:liquibaseRollback
   ```

2. **Параметризация**:
   ```bash
   # Переопределение параметров через командную строку
   ./gradlew build -Pdb.url=jdbc:postgresql://prod-db:5432/mydb
   ```

3. **Публикация в Nexus**:
   ```bash
   ./gradlew publish
   ```

---

### **Аналогии с Maven**
| **Maven**                          | **Gradle**                                      |
|------------------------------------|------------------------------------------------|
| `<properties>`                     | `gradle.properties` + `ext`                    |
| `<dependencyManagement>`           | Модуль BOM с `java-platform`                   |
| `maven-compiler-plugin`            | `java.toolchain.languageVersion`               |
| `liquibase-maven-plugin`           | `org.liquibase.gradle`                         |
| `<distributionManagement>`         | `publishing { repositories { ... } }`          |

---

Этот пример включает:
- Полноценную мульти-модульную структуру
- Централизованное управление версиями через BOM
- Интеграцию Liquibase с PostgreSQL
- Работу с приватным Nexus
- Настройки для Spring Boot, MapStruct и Lombok
- Готовые задачи для миграций и публикации
