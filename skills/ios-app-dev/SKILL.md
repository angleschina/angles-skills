---
name: ios-app-dev
description: >
  开发 iOS/iPadOS 原生应用。可帮助规划 SwiftUI 应用结构、生成 Swift 源码、设计界面布局、处理数据持久化、碰触 Xcode 工程配置；尤其适合在 iPad 上编写 SwiftUI 并在 Mac 上编译，或指导生成完整可构建的 SwiftUI 工程骨架。
  当用户要求"做个 iOS 应用""写 SwiftUI 界面""生成 Swift 源码""规划 App 结构"时触发。
license: MIT
compatibility: 建议在 iPad(iSH) 或本机写源码；构建需 macOS + Xcode(本地或用 CI)，SwiftUI 需 iOS 14+
---

# iOS / SwiftUI 应用开发

面向 iPad 上以 angles-cli 辅助**编写并交付 SwiftUI 源码 / 工程骨架**的场景。macOS + Xcode 负责编译构建（可在 CI 完成）。

## When to use

- 用户要"开发一个 iOS / iPadOS 应用"
- 需要生成 SwiftUI 界面源码、数据模型、持久化方案
- 需要规划 App 目录结构 / project.pbxproj / Info.plist
- 从零搭一个可编译的最小工程

## 能力边界（先说清）

angles-cli 跑在 Linux 容器里，**没有 Xcode / 无法真机构建**。能可靠交付：
- ✅ SwiftUI 全部源码（视图/模型/持久化/扩展）
- ✅ Info.plist、Assets/AppIcon 规格说明、完整目录树
- ✅ project.pbxproj（手写最小可编译工程）或建议用 XcodeGen/Tuist 生成
- ❌ 直接 `xcodebuild` 产出 .app / .ipa（需 Mac CI）

所以工作流是：**angles 产出完整工程文本 → 用户在 Mac/iPad 上编译**。

## 工程骨架（最小 SwiftUI App）

标准 SwiftUI 生命周期工程（App 入口 + ContentView）：

```swift
// MyAppApp.swift
import SwiftUI

@main
struct MyAppApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

```swift
// ContentView.swift
import SwiftUI

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("你点击了 \(count) 次")
                .font(.largeTitle)
            Button("点击") { count += 1 }
                .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

## 常用视图模式（直接给用户选）

```swift
// 列表导航 NavigationStack + List + NavigationLink
NavigationStack {
    List(items) { item in
        NavigationLink(item.title) { DetailView(item: item) }
    }
    .navigationTitle("项目")
}

// 网格 LazyVGrid
LazyVGrid(columns: [GridItem(.adaptive(minimum: 120))], spacing: 16) {
    ForEach(photos) { photo in
        Image(uiImage: photo).resizable().scaledToFill()
    }
}
```

## 数据持久化方案选择表

| 需求 | 推荐 | Swift 写法关键点 |
|---|---|---|
| 轻量键值 | `@AppStorage` / `UserDefaults` | `@AppStorage("key") var x = 默认` |
| 结构化少量数据 | 自建 Codable + JSON 存文件 | `JSONEncoder/Decoder` |
| 关系型/大量 | SwiftData（iOS17+） | `@Model`, `@Query` |
| 文件加密同步 | 也可自编码后存 Keychain/文件 | `Security.framework` |

示例：`@AppStorage` 轻量持久化

```swift
struct Settings: View {
    @AppStorage("isDark") private var isDark = false
    var body: some View {
        Toggle("深色模式", isOn: $isDark)
    }
}
```

## 界面常见需求示例

```swift
// 表单 Form
Form {
    TextField("姓名", text: $name)
    DatePicker("日期", selection: $date, displayedComponents: .date)
    Picker("分类", selection: $cat) {
        Text("A").tag(1); Text("B").tag(2)
    }
    Section { Button("保存") { save() }.disabled(name.isEmpty) }
}

// 弹窗 alert / sheet
.alert("错误", isPresented: $showError) {
    Button("好", role: .cancel) {}
} message: { Text("出错了") }
.sheet(isPresented: $showSheet) { DetailView() }
```

## 关键工程文件（xcodebuild 必需）

**Info.plist 最小必需键**：
```xml
<key>CFBundleExecutable</key><string>$(EXECUTABLE_NAME)</string>
<key>CFBundleIdentifier</key><string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
<key>CFBundleName</key><string>$(PRODUCT_NAME)</string>
<key>CFBundlePackageType</key><string>APPL</string>
<key>CFBundleShortVersionString</key><string>1.0</string>
<key>CFBundleVersion</key><string>1</string>
<key>UIApplicationSceneManifest</key>...
```

**项目生成建议三层目录**：
```
MyApp/
├── MyApp.xcodeproj/project.pbxproj
├── MyApp/            # 源码
│   ├── MyAppApp.swift
│   ├── ContentView.swift
│   ├── Models/       # Model.swift
│   ├── Views/        # 各界面
│   └── Assets.xcassets
└── Info.plist
```

## 在 Mac 上构建这一步

把生成的目录放到 Mac 后：
```bash
# 首选 XcodeGen（从 project.yml 生成 xcodeproj，避免手写 pbxproj）
brew install xcodegen
cd MyApp && xcodegen generate
xcodebuild -scheme MyApp -destination 'generic/platform=iOS' CODE_SIGNING_ALLOWED=NO build

# 或用 xcrun simctl 装模拟器跑
```

`project.yml`（XcodeGen 简例）：
```yaml
name: MyApp
options: { bundleIdPrefix: com.example }
targets:
  MyApp:
    type: application
    platform: iOS
    deploymentTarget: "16.0"
    sources: [MyApp]
    settings:
      base: { PRODUCT_BUNDLE_IDENTIFIER: com.example.myapp }
```

## 常见坑

- **预览 vs 真机**：`#Preview` 是 Xcode16+；旧版用 `PreviewProvider`。给代码前确认用户 Xcode 版本
- **iOS 系统版本 API**：SwiftData 要 iOS17+；`NavigationStack` iOS16+。写代码注明最低需版本
- **中文字体**：SwiftUI 系统字体自带中文渲染，无需额外字体
- **iPad 多窗口/键盘**：常见是额外做 `navigationSplitView` + 键盘快捷键，别按 iPhone 单一窗口写死
- **首启白屏**：多为缺 `Info.plist` 的 `UIApplicationSceneManifest` 或入口被删，检查 `@main`
- **图标/Assets**：AppIcon 需 `1024x1024` 资源放 `AppIcon.appiconset` + `Contents.json`
- **模拟器 vs 真机签名**：本地模拟器免签名能跑，真机需开发者证书；自动帮助用户装真机描述文件属另一流程

## 交给模型（你）做的

- 按用户**一句话需求**先拆成 界面栈/数据模型/持久化/导航 四块
- 每块用最小 SwiftUI 代码给出，能编译、职责单一
- 可运行代码 → 整理成上面的目录骨架 → 说明在 Mac 上怎么 build
- 别一次性堆几百行全功能；迭代式交付，每步可跑

## 原则

**你负责写出正确、最小、SwiftUI 惯用的源码与工程骨架，构建在 Mac 上由 xcodebuild 完成。** 给出的代码必须语法正确（Swift 6 风格加分）、版本注释清楚，不交付无法编译的"臆想 API"。
