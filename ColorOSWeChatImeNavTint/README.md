# ColorOS 微信输入法「系统输入法切换键」取色修复

下面的内容皆由 AI 生成。

---

给做输入法/ROM 适配的朋友：一个只需 20KB、不打包任何图标的 LSPosed 小模块，修好了
ColorOS 上微信输入法系统导航栏那两个按键（切换输入法 / 收起键盘）的图标取色。

## 一、现象

ColorOS 下，微信输入法键盘底部系统导航栏里的「切换输入法键」和「收起键」颜色经常和键盘底栏
对不上（该黑时是白、该白时是黑，或者看不清）。微信输入法自己调过颜色，但没用。

## 二、定位（3 个关键发现）

1. **那两个按钮不在微信输入法里。**
   反编译 `com.tencent.wetype`（测试 1.3.0 monet）后，`resources.arsc` 和三个 dex 里都搜不到
   `input_method_nav_ime_switcher` / `input_method_nav_back`。它们其实是 **framework（android 包）
   的 id**，只是渲染在输入法进程内的 IME 窗口里 —— 所以微信输入法根本碰不到这两个 View。

2. **微信输入法只控制「导航栏颜色」这一个杠杆。**
   这部分代码没混淆，在 `com.tencent.wetype.plugin.hld.WxHldService`（它的 InputMethodService）：
   - `onCreate()` / `reset()` / 主题切换时都会
     `window.setNavigationBarColor(键盘结束色 ime_keyboard_end_color)`
   - 但它**从没设过** `SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR`。

3. **ColorOS 会把「导航栏颜色」报成透明。**
   于是 framework 按透明色去算图标配色，结果就是错的。原版 `WeType-Tool` 模块的做法是：
   采样真实底色 → 算黑白 → **替换成它自带的图标** + 拦掉系统 `onDraw`。

> 逆向小技巧（可复用）：
> - 反编译用 jadx；字符串被 StringFog 加密时，直接按 `decrypt(byte[], "key", "xor")` 的
>   `byte[i] ^ key[i % len]` 解一遍就能还原。
> - jadx 报 "Method not decompiled" 的大方法，用
>   `jadx --show-bad-code --single-class 包.类` 就能拿到方法体。

## 三、根因

系统导航栏那个「输入法切换键」由 framework 绘制，颜色依据导航栏颜色/明暗标志决定；
微信输入法把导航栏设成了 ColorOS 会报透明的值，且没给明暗标志，framework 就算错了。

## 四、方案：保留系统原图标，只改 tint

不换图、不打包矢量的独立 LSPosed 模块（modern libxposed API 102）：

1. 用 framework id 在 IME 窗口里找到那两个按键；
2. **取色**：先看 `Window.getNavigationBarColor()`；若透明，就沿按钮父链最多 6 层找
   `ColorDrawable` / `GradientDrawable` 的真实底色；
3. **对比色**：sRGB 相对亮度 `0.2126R + 0.7152G + 0.0722B`，> 0.5 给黑，否则给白；
4. 只做 `setImageTintList` / `setForegroundTintList`，并在 `ImageView.onDraw` 前逐帧复贴，
   防止系统改回去；
5. 同步 `LIGHT_NAVIGATION_BAR` 标志，并把颜色写进 OPPO 私有 `mOplusLayoutParams`，避免被 ColorOS 覆盖；
6. 触发点：`InputMethodService` 的 `onCreate` / `onConfigureWindow` / `onWindowShown` /
   `onStartInputView`，外加 hook `PhoneWindow.setNavigationBarColor`（切主题时会调）。
7. 所有 hook 用 `ExceptionMode.PROTECTIVE`，出错也不会崩输入法。日志 tag：`ImeNavTint`。

## 五、复现步骤

1. 反编译目标微信输入法，确认两个 id 不在其 dex/资源里（说明是 framework）。
2. 写 libxposed 模块，作用域 `com.tencent.wetype`：
   - `META-INF/xposed/java_init.list` 写入口类
   - `META-INF/xposed/scope.list` 写 `com.tencent.wetype`
   - `META-INF/xposed/module.prop`：
     ```
     minApiVersion=102
     targetApiVersion=102
     staticScope=true
     exceptionMode=protective
     ```
   - 入口 `extends io.github.libxposed.api.XposedModule`，在 `onPackageReady` 里装 hook
     （新 API 用 `Hooker.intercept(Chain)` + `chain.proceed()`，没有 `XposedHelpers`，
     也不用 `assets/xposed_init`）
3. 打包，装到 LSPosed、勾作用域、强停微信输入法重开键盘，看 tag `ImeNavTint` 的日志。

## 六、构建

- 依赖：`compileOnly("io.github.libxposed:api:102.0.0")`（Maven Central 上是 `.aar`）。
- 没装 Gradle 也能构建：`aapt2 compile/link` → `javac -classpath android.jar` →
  `d8` → `zipalign` → `apksigner`。
  （注意：别把 android.jar 放 `-bootclasspath`，会顶掉 JDK 的 `LambdaMetafactory`，lambda 编译报错。）

## 七、注意

- 两个 id 是 framework 资源，只有 ColorOS/OPPO 系才有；别的 ROM 上模块空转、不报错。
- 本方案在 1.3.0 monet 上做的静态分析，真机验证请以 LSPosed 日志为准。
