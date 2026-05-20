# Kotlin DSL 和 buildSrc 的 Gradle 统一管理

## 目标

采用 **Kotlin DSL** 作为 Gradle 构建语言，并通过 **buildSrc** 集中管理所有依赖版本、插件 ID 和依赖声明，实现：

- 类型安全的版本和依赖访问
- 消除模块间的重复配置代码
- 通过 Convention Plugin 标准化 Android 应用和库模块的构建配置
- 支持从 `gradle.properties` 动态读取应用版本号

---

## 核心要求

### 迁移范围（可选，适用于从 Groovy 迁移的场景）

- 将 `settings.gradle` 转换为 `settings.gradle.kts`
- 将所有模块的 `build.gradle` 转换为 `build.gradle.kts`
- 保留原有的自定义任务和构建逻辑，仅转换语法

> **新项目说明：** 如果是全新项目，直接使用 Kotlin DSL 编写构建文件即可，无需迁移步骤。

### buildSrc 结构
创建 `buildSrc` 目录，主要包含以下文件：

| 文件 | 职责 |
|------|------|
| `Versions.kt` | 定义所有版本常量（SDK 版本、插件版本、依赖库版本） |
| `Dependencies.kt` | 定义所有依赖坐标，引用 `Versions.kt` 中的版本 |
| `Plugins.kt` | 定义所有插件 ID 常量 |
| `DependencyExtensions.kt` | 定义 `DependencyHandler` 的扩展函数（`implementation`、`api` 等） |
| `ApplicationConventionPlugin.kt` | Android 应用模块的约定插件 |
| `LibraryConventionPlugin.kt` | Android 库模块的约定插件 |

### 版本管理规范

**SDK 版本定义**

`compileSdk`、`minSdk`、`targetSdk` 统一在 `Versions.kt` 中定义，不得在模块中硬编码。

**依赖库版本**

所有依赖库版本在 `Versions.kt` 中定义为常量，在 `Dependencies.kt` 中通过字符串模板引用。

### Convention Plugin 规范

**ApplicationConventionPlugin 职责**

- 应用 `com.android.application` 和 `org.jetbrains.kotlin.android` 插件
- 配置 `compileSdk`、`minSdk`、`targetSdk`
- 配置 `versionCode` 和 `versionName`（支持从 `gradle.properties` 动态读取）
- 添加通用依赖（如 `kotlin-stdlib`、`core-ktx`、`material`）

**LibraryConventionPlugin 职责**

- 应用 `com.android.library` 和 `org.jetbrains.kotlin.android` 插件
- 配置 `compileSdk`、`minSdk`、`targetSdk`
- 添加通用依赖（如 `kotlin-stdlib`）

### 模块配置规范
各模块 `build.gradle.kts` 配置方式：
- 通过 `plugins { id(...) }` 应用 Convention Plugin
- 仅配置模块特有属性（如 `namespace`、`applicationId`）

### 验证要求
完成后必须能成功运行以下命令：
```bash
./gradlew build
```

## 目录结构
```
project-root/
├── gradle.properties                     # 应用版本号定义
├── settings.gradle.kts                   # 原 settings.gradle 转换
├── build.gradle.kts                      # 根目录构建文件
├── buildSrc/
│   ├── build.gradle.kts                  # buildSrc 构建配置（必需）
│   └── src/main/kotlin/
│       ├── Versions.kt
│       ├── Dependencies.kt
│       ├── Plugins.kt
│       ├── DependencyExtensions.kt
│       ├── ApplicationConventionPlugin.kt
│       └── LibraryConventionPlugin.kt
├── app/
│   └── build.gradle.kts                  # 应用模块配置
└── lib/                                  # 其他模块
    └── build.gradle.kts                  # 库模块配置
```

## 示例代码

```properties
# gradle.properties

# 应用版本号
VERSION_CODE=1
VERSION_NAME=1.0.0
```

```kotlin
// buildSrc/build.gradle.kts

plugins {
    `kotlin-dsl`
}

repositories {
    google()
    mavenCentral()
}

dependencies {
    implementation("com.android.tools.build:gradle:8.13.0")
    implementation("org.jetbrains.kotlin:kotlin-gradle-plugin:1.9.25")
}

gradlePlugin {
    plugins {
        create("applicationConvention") {
            id = "com.example.application.convention"
            implementationClass = "ApplicationConventionPlugin"
        }
        create("libraryConvention") {
            id = "com.example.library.convention"
            implementationClass = "LibraryConventionPlugin"
        }
    }
}

```

```kotlin
// buildSrc/src/main/kotlin/Versions.kt

object Versions {
    const val compileSdk = 35
    const val minSdk = 23
    const val targetSdk = 35

    const val agp = "8.13.0"
    const val kotlin = "1.9.25"
    // 其他依赖版本...
}
```

```kotlin
// buildSrc/src/main/kotlin/Dependencies.kt

object Dependencies {
    const val kotlinStdlib = "org.jetbrains.kotlin:kotlin-stdlib:${Versions.kotlin}"
    // 其他依赖...
}

```

```kotlin
// buildSrc/src/main/kotlin/Plugins.kt
object Plugins {
    const val kotlinAndroid = "org.jetbrains.kotlin.android"
    const val androidApplication = "com.android.application"
    const val androidLibrary = "com.android.library"
        
    // Convention 插件 (需与build.gradle.kt中的插件ID定义保持一致)
    const val applicationConvention = "com.example.application.convention"
    const val libraryConvention = "com.example.library.convention"
    
    // 其他插件...
}

```

```kotlin
// buildSrc/src/main/kotlin/DependencyExtensions.kt

import org.gradle.api.artifacts.dsl.DependencyHandler

/**
 * DependencyHandler 扩展函数
 * 用于简化依赖声明，所有函数声明为 internal 避免污染其他模块
 */

internal fun DependencyHandler.implementation(dependencyNotation: Any) =
    add("implementation", dependencyNotation)

internal fun DependencyHandler.api(dependencyNotation: Any) =
    add("api", dependencyNotation)

internal fun DependencyHandler.compileOnly(dependencyNotation: Any) =
    add("compileOnly", dependencyNotation)

internal fun DependencyHandler.testImplementation(dependencyNotation: Any) =
    add("testImplementation", dependencyNotation)

internal fun DependencyHandler.androidTestImplementation(dependencyNotation: Any) =
    add("androidTestImplementation", dependencyNotation)

//...

```

```kotlin
// buildSrc/src/main/kotlin/ApplicationConventionPlugin.kt

import com.android.build.api.dsl.ApplicationExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies

class ApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        // 应用插件
        pluginManager.apply(Plugins.androidApplication)
        pluginManager.apply(Plugins.kotlinAndroid)
        
        // 配置 Android 应用
        extensions.configure<ApplicationExtension> {
            compileSdk = Versions.compileSdk
            
            defaultConfig {
                minSdk = Versions.minSdk
                targetSdk = Versions.targetSdk
                
                // 从 gradle.properties 读取
                versionCode = target.findProperty("VERSION_CODE").toString().toInt()
                versionName = target.findProperty("VERSION_NAME").toString()
            }
        }
        
        // 添加通用依赖
        dependencies {
            implementation(Dependencies.kotlinStdlib)
            //其他通用依赖
        }
    }
}

```

```kotlin
// buildSrc/src/main/kotlin/LibraryConventionPlugin.kt

import com.android.build.api.dsl.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies

class LibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        // 应用插件
        pluginManager.apply(Plugins.androidLibrary)
        pluginManager.apply(Plugins.kotlinAndroid)
        
        // 配置 Android 库
        extensions.configure<LibraryExtension> {
            compileSdk = Versions.compileSdk
            
            defaultConfig {
                minSdk = Versions.minSdk
                targetSdk = Versions.targetSdk
            }
        }
        
        // 添加通用依赖
        dependencies {
            implementation(Dependencies.kotlinStdlib)
            //其他通用依赖
        }
    }
}

```

```kotlin
// app/build.gradle.kts 

plugins {
    id(Plugins.applicationConvention)
}

android {
    namespace = "com.example.myapp"
    
    defaultConfig {
        applicationId = "com.example.myapp"
        // versionCode 和 versionName 已在 Convention Plugin 中配置
    }
}

dependencies {
    // 根据需要添加模块特有依赖
}

```

```kotlin
// lib/build.gradle.kts

plugins {
    id(Plugins.libraryConvention)
}

android {
    namespace = "com.example.mylib"
}

dependencies {
    // 根据需要添加模块特有依赖
}

```
