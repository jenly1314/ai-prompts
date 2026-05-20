# 基于 Kotlin DSL 和 Version Catalog 的 Gradle 统一管理

## 目标

采用 **Kotlin DSL** 作为 Gradle 构建语言，并通过 **Version Catalog**（`libs.versions.toml`）集中管理所有依赖版本、插件 ID 和依赖声明，实现：

- 集中式的依赖版本管理
- 消除模块间的重复配置代码
- 标准化的 Android 应用和库模块构建配置
- 支持从 `gradle.properties` 动态读取应用版本号

---

## 核心要求

### 迁移范围（可选，适用于从 Groovy 迁移的场景）

- 将 `settings.gradle` 转换为 `settings.gradle.kts`
- 将所有模块的 `build.gradle` 转换为 `build.gradle.kts`
- 保留原有的自定义任务和构建逻辑，仅转换语法

> **新项目说明：** 如果是全新项目，直接使用 Kotlin DSL 编写构建文件即可，无需迁移步骤。

### Version Catalog 结构

创建 `gradle/libs.versions.toml` 文件，主要包含以下章节：

| 章节 | 职责 |
|------|------|
| `[versions]` | 定义所有版本常量（SDK 版本、插件版本、依赖库版本） |
| `[libraries]` | 定义所有依赖坐标，引用 `[versions]` 中的版本 |
| `[plugins]` | 定义所有插件 ID 和版本，引用 `[versions]` 中的版本 |
| `[bundles]` | 可选，定义依赖组合，便于批量引用 |

### 版本管理规范

**SDK 版本定义**

`compileSdk`、`minSdk`、`targetSdk` 统一在 `[versions]` 中定义，不得在模块中硬编码。

**依赖库版本**

所有依赖库版本在 `[versions]` 中定义为常量，在 `[libraries]` 中通过 `version.ref` 引用。

### 模块配置规范

各模块 `build.gradle.kts` 配置方式：
- 通过 `alias(libs.plugins.xxx)` 应用插件
- 通过 `libs.versions.xxx.get().toInt()` 读取 SDK 版本
- 通过 `libs.xxx` 引用依赖

### 验证要求

完成后必须能成功运行以下命令：

```bash
./gradlew build
```

## 目录结构

```
project-root/
├── gradle/
│   └── libs.versions.toml                # 版本目录文件
├── gradle.properties                     # 应用版本号定义
├── settings.gradle.kts                   # 原 settings.gradle 转换
├── build.gradle.kts                      # 根目录构建文件
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

```toml
[versions]
compileSdk = "35"
minSdk = "23"
targetSdk = "35"

agp = "8.13.0"
kotlin = "1.9.25"
# 其他依赖版本...

[libraries]
kotlin-stdlib = { group = "org.jetbrains.kotlin", name = "kotlin-stdlib", version.ref = "kotlin" }
# 其他依赖...

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
# 其他插件...

```

```kotlin
// app/build.gradle.kts

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
}

android {
    namespace = "com.example.myapp"
    compileSdk = libs.versions.compileSdk.get().toInt()

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = libs.versions.minSdk.get().toInt()
        targetSdk = libs.versions.targetSdk.get().toInt()

        // 从 gradle.properties 读取
        versionCode = properties["VERSION_CODE"].toString().toInt()
        versionName = properties["VERSION_NAME"].toString()
    }
    // ...
}

dependencies {
    implementation(libs.kotlin.stdlib)
    // 其他依赖...
}
```

```kotlin
// lib/build.gradle.kts

plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.kotlin.android)
}

android {
    namespace = "com.example.library"
    compileSdk = libs.versions.compileSdk.get().toInt()

    defaultConfig {
        minSdk = libs.versions.minSdk.get().toInt()
        targetSdk = libs.versions.targetSdk.get().toInt()
    }
    // ...
}

dependencies {
    implementation(libs.kotlin.stdlib)
    // 其他依赖...
}
```
