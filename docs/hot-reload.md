# Hot Reload Support for HRJBP Plugin

## 概述

本文档详细介绍如何将 HRJBP 项目配置为支持动态加载的 JetBrains 插件，并在开发过程中启用 Auto-Reload 功能。

## 什么是动态插件 (Dynamic Plugins)

动态插件是一种特殊类型的 IntelliJ Platform 插件，支持以下功能：
- **安装**: 无需重启 IDE 即可安装插件
- **更新**: 无需重启 IDE 即可更新插件到新版本
- **卸载**: 无需重启 IDE 即可卸载插件
- **热重载**: 开发过程中代码更改可以立即生效

### Auto-Reload 功能

Auto-Reload 是专门为开发阶段设计的功能，允许：
- 在沙盒 IDE 实例运行时检测代码更改
- 自动重新加载插件而无需重启整个 IDE
- 大大加快开发和测试周期

## IntelliJ Platform Gradle Plugin (2.x) 配置

### Auto-Reload 默认设置

对于使用 IntelliJ Platform Gradle Plugin (2.x) 的项目：

- **默认启用**: Auto-Reload 功能默认已启用
- **禁用方法**: 如需禁用，设置 `intellijPlatform.autoReload = false`
- **触发重载**: 
  1. 启动沙盒 IDE 实例
  2. 修改插件代码后运行 `buildPlugin` 任务
  3. 切换焦点回沙盒实例以触发重载

### 重要限制

> ⚠️ **调试模式限制**: Auto-Reload 在调试器模式下不工作
> 
> ⚠️ **任务冲突**: 必须显式禁用 `buildSearchableOptions` 任务以避免 "只能运行一个 IDEA 实例" 的问题

### 沙盒目录位置

插件开发实例的沙盒目录位置：
- **Windows**: `$PROJECT_DIRECTORY$\build\$TARGET_IDE$\idea-sandbox`
- **Linux/macOS**: `$PROJECT_DIRECTORY$/build/$TARGET_IDE$/idea-sandbox`

可通过 `intellijPlatform.sandboxContainer` 属性自定义沙盒位置。

## 动态插件要求和限制

要使插件支持动态加载，必须满足以下所有限制：

### 1. 禁用组件使用
- **要求**: 不得使用任何 Components
- **解决方案**: 必须将现有组件迁移到 services、extensions 或 listeners

### 2. Action Group 必须有 ID
- **要求**: 所有 `<group>` 元素必须声明唯一的 `id`
- **示例**: `<group id="MyActionGroup" text="My Actions">`

### 3. 仅使用动态扩展点
- **要求**: 所有使用的扩展点必须明确标记为动态
- **注意**: 某些已弃用的扩展点（如 `com.intellij.configurationProducer`）故意设为非动态

### 4. 扩展点动态标记
- **要求**: 自定义扩展点必须明确声明为动态
- **方法**: 在扩展点定义中添加动态标记

### 5. Configurable 依赖处理
- **要求**: 依赖动态扩展点的 `Configurable` 必须实现 `Configurable.WithEpDependencies`

### 6. 禁用服务覆盖
- **要求**: 不允许使用 `overrides="true"` 的应用程序、项目和模块服务

## HRJBP 项目现状分析

### 当前配置分析

经过对 HRJBP 项目的分析，发现以下现状：

**✅ 符合要求的部分:**
1. **使用 IntelliJ Platform Gradle Plugin (2.x)**: 项目已配置正确的 Gradle 插件
2. **无组件依赖**: 项目未使用任何已弃用的 Components
3. **服务架构**: 使用现代的 `@Service` 注解定义服务
4. **现代启动活动**: 使用 `ProjectActivity` 而非已弃用的组件

**⚠️ 需要检查的部分:**
1. **扩展点验证**: 需要验证所有使用的扩展点都支持动态加载
2. **Action Groups**: 当前项目没有定义 action groups，符合要求
3. **插件验证**: 需要运行插件验证工具确认动态兼容性

### 配置文件分析

**plugin.xml 配置:**
```xml
<extensions defaultExtensionNs="com.intellij">
    <toolWindow factoryClass="com.github.jiecmsft.hrjbp.toolWindow.MyToolWindowFactory" id="MyToolWindow"/>
    <postStartupActivity implementation="com.github.jiecmsft.hrjbp.startup.MyProjectActivity" />
</extensions>
```

- ✅ `toolWindow`: 支持动态加载
- ✅ `postStartupActivity`: 支持动态加载

## 实现 Auto-Reload 的行动计划

### 第一阶段: 验证动态兼容性

1. **运行插件验证检查**
   ```powershell
   ./gradlew verifyPlugin
   ```

2. **在 IDE 中运行动态插件验证检查**
   - 打开 Code → Analyze Code → Run Inspection by Name
   - 运行 "Plugin DevKit | Plugin descriptor | Plugin.xml dynamic plugin verification"

### 第二阶段: 配置 Auto-Reload

由于项目已使用 IntelliJ Platform Gradle Plugin (2.x)，Auto-Reload 默认已启用。

如需确保配置正确，可在 `build.gradle.kts` 中显式设置：

```kotlin
intellijPlatform {
    // Auto-Reload 默认启用，这里仅作示例
    autoReload = true
    
    // 可选：自定义沙盒目录
    sandboxContainer = layout.buildDirectory.dir("custom-sandbox")
}
```

### 第三阶段: 开发工作流程

1. **启动开发实例**
   ```powershell
   ./gradlew runIde
   ```

2. **开发循环**
   - 修改代码
   - 运行构建任务: `./gradlew buildPlugin`
   - 切换焦点回沙盒 IDE 实例
   - 观察插件自动重载

3. **可选：禁用 buildSearchableOptions**
   如遇到"只能运行一个实例"的问题，在 `build.gradle.kts` 中添加：
   ```kotlin
   tasks {
       buildSearchableOptions {
           enabled = false
       }
   }
   ```

### 第四阶段: 验证和测试

1. **本地分发测试**
   - 构建插件分发包: `./gradlew buildPlugin`
   - 在 IDE 中手动安装本地构建的插件
   - 验证动态安装是否成功

2. **更新测试**
   - 修改插件版本号
   - 重新构建并尝试更新安装
   - 验证无需重启即可更新

3. **卸载测试**
   - 测试插件卸载是否无需重启

## 故障排除

### 日志监控

所有动态插件事件都在 IDE 日志文件中的 `com.intellij.ide.plugins.DynamicPlugins` 类别下跟踪。

### 内存泄漏诊断

如果插件无法正确卸载：

1. **启用诊断模式**
   - 添加 JVM 参数: `-XX:+UnlockDiagnosticVMOptions`
   - 在 Gradle 中: `runIde.jvmArgs += "-XX:+UnlockDiagnosticVMOptions"`

2. **启用快照生成**
   - 设置注册表键 `ide.plugins.snapshot.on.unload.fail` 为 `true`

3. **分析内存快照**
   - 查看用户主目录中生成的 `.hprof` 文件
   - 寻找 `PluginClassLoader` 引用以定位内存泄漏

## 详细 Gap 分析

### 当前项目状态 vs 动态插件要求

| 要求项目 | 当前状态 | 是否符合 | 需要的行动 |
|---------|---------|----------|-----------|
| **无组件使用** | 未使用任何 Components | ✅ 符合 | 无需行动 |
| **Action Group ID** | 未定义 Action Groups | ✅ 符合 | 无需行动 |
| **动态扩展点** | 使用 `toolWindow`, `postStartupActivity` | ✅ 符合 | 验证兼容性 |
| **自定义扩展点标记** | 未定义自定义扩展点 | ✅ 符合 | 无需行动 |
| **Configurable 依赖** | 未使用 Configurable | ✅ 符合 | 无需行动 |
| **服务覆盖** | 未使用服务覆盖 | ✅ 符合 | 无需行动 |
| **插件描述符配置** | 未设置 require-restart | ✅ 符合 | 确保不设置此标记 |

### 技术架构分析

**现有代码结构分析:**

1. **服务层** (`MyProjectService`)
   - ✅ 使用 `@Service(Service.Level.PROJECT)`
   - ✅ 正确实现项目级服务
   - ✅ 无状态设计，支持动态加载/卸载

2. **工具窗口** (`MyToolWindowFactory`)
   - ✅ 实现 `ToolWindowFactory` 接口
   - ✅ 使用依赖注入获取服务
   - ✅ UI 组件使用 IntelliJ 标准组件

3. **启动活动** (`MyProjectActivity`)
   - ✅ 实现 `ProjectActivity` 接口
   - ✅ 使用现代异步启动机制
   - ✅ 无长期持有的资源

4. **插件配置** (`plugin.xml`)
   - ✅ 使用标准扩展点
   - ✅ 未设置 `require-restart="true"`
   - ✅ 扩展点配置简洁明确

### 构建配置分析

**Gradle 配置检查:**

```kotlin
// build.gradle.kts 当前配置
plugins {
    alias(libs.plugins.intelliJPlatform) // ✅ 使用 2.x 版本
}

intellijPlatform {
    // ✅ 标准配置，Auto-Reload 默认启用
    pluginConfiguration { ... }
}
```

**潜在问题:**
1. **版本兼容性**: 需要确保 Platform 版本兼容性
2. **构建问题**: 当前存在构建错误需要解决

## 实施路线图

### 阶段 1: 环境修复 (优先级: 高)

**问题**: 当前构建失败，需要先修复基础环境

1. **修复 JDK 路径问题**
   ```powershell
   # 检查当前 JDK 配置
   ./gradlew --version
   
   # 如需要，设置正确的 JAVA_HOME
   $env:JAVA_HOME = "C:\Program Files\Java\jdk-21"
   ```

2. **更新插件版本**
   ```kotlin
   // 在 build.gradle.kts 中更新插件版本
   plugins {
       alias(libs.plugins.intelliJPlatform) // 确保使用最新版本
   }
   ```

3. **平台版本验证**
   ```properties
   # gradle.properties 中验证版本
   platformVersion = 2024.3.6  # 确保版本存在且可用
   ```

### 阶段 2: 动态插件验证 (优先级: 高)

1. **运行插件验证**
   ```powershell
   ./gradlew verifyPlugin
   ```

2. **IDE 内检查**
   - 打开项目
   - Code → Analyze Code → Run Inspection by Name
   - 运行 "Plugin.xml dynamic plugin verification"

3. **手动验证清单**
   ```
   □ 无 Components 使用
   □ 无 require-restart 标记
   □ 扩展点都支持动态加载
   □ 无服务覆盖
   □ 无内存泄漏风险
   ```

### 阶段 3: Auto-Reload 配置和测试 (优先级: 中)

1. **确认 Auto-Reload 配置**
   ```kotlin
   // build.gradle.kts 显式配置（可选）
   intellijPlatform {
       autoReload = true  // 默认已启用
       
       // 可选：自定义沙盒目录
       sandboxContainer = layout.buildDirectory.dir("idea-sandbox")
   }
   ```

2. **开发工作流测试**
   ```powershell
   # 1. 启动开发实例
   ./gradlew runIde
   
   # 2. 在另一个终端中，修改代码后构建
   ./gradlew buildPlugin
   
   # 3. 观察沙盒实例是否自动重载
   ```

3. **故障排除配置**
   ```kotlin
   // 如遇到 buildSearchableOptions 冲突
   tasks {
       buildSearchableOptions {
           enabled = false
       }
   }
   ```

### 阶段 4: 高级功能和优化 (优先级: 低)

1. **内存泄漏预防**
   ```kotlin
   // 实现 DynamicPluginListener（如需要）
   class PluginLifecycleListener : DynamicPluginListener {
       override fun beforePluginUnload(pluginDescriptor: IdeaPluginDescriptor) {
           // 清理资源
       }
   }
   ```

2. **性能优化**
   - 确保服务实现 `Disposable`（如需要）
   - 避免在全局范围持有 PSI 引用
   - 使用 `SmartPsiElementPointer` 替代直接 PSI 引用

## 验证清单

### 构建验证
- [ ] `./gradlew build` 成功执行
- [ ] `./gradlew verifyPlugin` 通过检查
- [ ] `./gradlew buildPlugin` 生成有效插件

### 动态加载验证
- [ ] IDE 内插件验证检查通过
- [ ] 本地插件安装无需重启
- [ ] 插件更新无需重启
- [ ] 插件卸载无需重启

### Auto-Reload 验证
- [ ] `./gradlew runIde` 启动沙盒成功
- [ ] 代码修改后 `buildPlugin` 执行成功
- [ ] 沙盒实例自动检测并重载插件
- [ ] 重载后功能正常工作

### 稳定性验证
- [ ] 多次重载无内存泄漏
- [ ] 日志中无错误或警告
- [ ] 插件状态正确保持

## 总结

HRJBP 项目已经具备了成为动态插件的**优秀基础**：

**现有优势:**
- ✅ 使用现代 IntelliJ Platform Gradle Plugin (2.x)
- ✅ 完全避免已弃用的组件系统
- ✅ 采用推荐的服务和扩展架构
- ✅ 代码结构清晰，符合动态插件最佳实践

**主要待完成任务:**
1. **修复构建环境** - 解决当前的 JDK 和版本兼容性问题
2. **验证动态兼容性** - 运行官方验证工具确认
3. **测试 Auto-Reload 工作流** - 建立高效的开发循环

**期望结果:**
完成上述计划后，HRJBP 项目将支持：
- 🚀 **快速开发** - 代码更改立即生效，无需重启 IDE
- 🔄 **动态部署** - 安装、更新、卸载都无需重启
- 🛠️ **现代工具链** - 充分利用 IntelliJ Platform 最新功能

这将显著提升插件开发效率和用户体验。
