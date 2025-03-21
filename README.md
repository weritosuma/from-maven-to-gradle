Вот полный пример мульти-модульного Gradle-проекта с **Spring Boot**, **MapStruct**, **Lombok**, **Nexus** и централизованным управлением версиями:

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
│   ├── build.gradle
│   └── src/main/java/com/example/...
└── db/
    ├── build.gradle
    └── src/main/java/com/example/...
```

---

### **`gradle.properties`
```properties
# Версии зависимостей
springBootVersion=3.1.2
javaVersion=17
hibernateVersion=6.2.5.Final
mapstructVersion=1.5.5.Final
lombokVersion=1.18.30

# Nexus репозиторий
nexusUrl=https://nexus.example.com/repository/maven-releases/
nexusUser=deployer
nexusPassword=secure_password_123

# Правила обновления зависимостей
dependencyChecksums=true
dependencyCacheDuration=7d
```

---

### **`settings.gradle`
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
    id 'net.ltgt.apt' version '0.21'  // Для аннотационных процессоров
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

### **`parent/build.gradle` (BOM)
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
        api "org.junit.jupiter:junit-jupiter:5.8.1"
        
        // MapStruct
        api "org.mapstruct:mapstruct:${mapstructVersion}"
        annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
        
        // Lombok
        api "org.projectlombok:lombok:${lombokVersion}"
        annotationProcessor "org.projectlombok:lombok:${lombokVersion}"
    }
}
```

---

### **`impl/build.gradle`
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

### **`db/build.gradle`
```groovy
dependencies {
    implementation platform(project(':parent'))
    
    // Hibernate
    implementation 'org.hibernate:hibernate-core'
    
    // Базовые зависимости
    runtimeOnly 'com.mysql:mysql-connector-j'
    
    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // Тесты
    testImplementation 'org.junit.jupiter:junit-jupiter'
}
```

---

### **`~/.gradle/init.gradle` (глобальные настройки)
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

### **Пример кода с MapStruct и Lombok**

#### **User.java (с Lombok)**
```java
package com.example.model;

import lombok.Data;

@Data
public class User {
    private Long id;
    private String name;
    private String email;
}
```

#### **UserDto.java**
```java
package com.example.dto;

public class UserDto {
    private String name;
    private String email;
}
```

#### **UserMapper.java (с MapStruct)**
```java
package com.example.mapper;

import com.example.dto.UserDto;
import com.example.model.User;
import org.mapstruct.Mapper;
import org.mapstruct.factory.Mappers;

@Mapper
public interface UserMapper {
    UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);
    UserDto toDto(User user);
}
```

---

### **Ключевые аналогии с Maven**
| **Maven**                          | **Gradle**                                      |
|------------------------------------|------------------------------------------------|
| `<properties>`                     | `gradle.properties` + `ext`                    |
| `<dependencyManagement>`           | Модуль BOM с `java-platform`                   |
| `maven-compiler-plugin`            | `java.toolchain.languageVersion`               |
| `lombok-maven-plugin`              | `compileOnly` + `annotationProcessor`          |
| `mapstruct-processor`              | `annotationProcessor` + `net.ltgt.apt` плагин  |
| `<repositories>` + `<servers>`     | `repositories { maven { credentials { ... } }` |
| `<distributionManagement>`         | `publishing { repositories { ... } }`          |

---

### **Особенности**
1. **Кэширование зависимостей**:
   ```groovy
   cacheDynamicVersionsFor 7, 'days'  // Кэширование SNAPSHOT-версий
   ```

2. **Генерация кода**:
   - MapStruct: `src/main/java` → `build/generated/sources/annotationProcessor/java`
   - Lombok: Генерация геттеров/сеттеров на лету

3. **Публикация в Nexus**:
   ```bash
   ./gradlew publish  # Аналог mvn deploy
   ```

4. **Компиляция**:
   ```bash
   ./gradlew build  # Аналог mvn clean install
   ```

Этот пример демонстрирует:
- Полную интеграцию Spring Boot, MapStruct и Lombok
- Работу с приватным Nexus репозиторием
- Централизованное управление версиями через BOM
- Настройки кэширования и проверки зависимостей
- Генерацию исходного кода через аннотационные процессоры
