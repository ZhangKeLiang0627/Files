# MVVM 调用链全解析

> 从 main() 到 renderSelectedIndex()，一条命令走完四层架构。
> 全链路追踪：`用户按键 → Model → ViewModel → View → LVGL 渲染`

本文以**按下 `Down` 键选择下一个文件**为例，逐行追踪代码的调用路径。

---

## 一、鸟瞰：四层架构 + 一个事件总线

```
┌─────────────────────────────────────────────────────────┐
│                     main.cpp (主循环)                     │
│   while (!quit) { keypad.poll(); lv_timer_handler();    │
│                    app.tick(); usleep(10000); }          │
└──────────────────────┬──────────────────────────────────┘
                       │ app.onLvglKeyState()
                       ▼
┌─────────────────────────────────────────────────────────┐
│        FilesApp（装配器/路由器）                            │
│   持有: Router, Model, VM×2, View×2                      │
│   职责: 转发按键 → 当前 VM，监听路由变化 → 切换页面        │
└──────────┬──────────────────────────────────────────────┘
           │ _current_vm->onKey(key)
           ▼
┌─────────────────────────────────────────────────────────┐
│   BrowserViewModel（业务逻辑调度器）                       │
│   持有: FilesModel& (引用)、自有 SingleObservables       │
│   职责: 决定 Down 键→调用哪个 Model 方法                 │
└──────────┬──────────────────────────────────────────────┘
           │ _model.browser().selectNext()
           ▼
┌─────────────────────────────────────────────────────────┐
│   FileBrowserModel（数据层）                              │
│   持有: SingleObservable<int> _selected_index            │
│   职责: 计算下一个选中项，_selected_index.set(next)       │
└──────────┬──────────────────────────────────────────────┘
           │ set() → notify() → 回调 BrowserView 的静态函数
           ▼
┌─────────────────────────────────────────────────────────┐
│   BrowserView（渲染层）                                   │
│   持有: LVGL 控件、注册在 VM/Observable 上的回调          │
│   职责: 收到通知 → 更新 LVGL 界面                         │
└─────────────────────────────────────────────────────────┘
```

### 事件总线：`SingleObservable<T>`

```
 Model 或 VM                     View
┌────────────┐                ┌──────────────┐
│ _selected  │─── observe() ──→│ onSelected   │
│ _index     │   (注册回调)    │ _IndexChanged │
│            │                │              │
│ .set(n) ───┼──→ notify() ──→│ callback(ctx,│
│            │   (自动触发)    │  newValue)   │
└────────────┘                └──────┬───────┘
                                     │ static_cast<BrowserView*>(ctx)
                                     ▼
                               renderSelectedIndex(n)
```

**关键设计**：`observe()` 注册后**立即回调一次**，所以 View 不需要额外的"初始化同步"步骤。

---

## 二、启动：app.start() 做了什么？

```cpp
// main.cpp:41
app.start();
```

`FilesApp::start()` 完成三件事：

```cpp
// files_app.cpp:101-109
void FilesApp::start()
{
    setupInputGroup();                    // ① 创建 LVGL 输入组
    _route_observer_id = _router
        .currentPage()
        .observe(this, onRouteChanged);   // ② 订阅路由变化
    setCurrentPage(_router.page());        // ③ 激活 Browser 页面
}
```

第③步 `setCurrentPage(PageId::Browser)` 会依次调用：

```cpp
// files_app.cpp:315-337
void FilesApp::setCurrentPage(PageId page)
{
    // 找到 BrowserViewModel 和 BrowserView
    _current_vm   = viewModelFor(page);   // → &_browser_vm
    _current_view = viewFor(page);        // → &_browser_view

    _current_vm->onEnter();               // → BrowserViewModel::onEnter()
    _current_view->onEnter(parent);       // → BrowserView::onEnter(lv_screen_active())
}
```

### `BrowserView::onEnter()` 全貌

```cpp
// browser_view.cpp:1230-1297
void BrowserView::onEnter(lv_obj_t* parent)
{
    // 1. 创建 LVGL 控件树（root → path_bar, file_list, action_menu, cursor...）
    _root       = make_unique<Container>(parent);
    _path_bar   = make_unique<Container>(_root->raw_ptr());
    _path_label = make_unique<Label>(_path_bar->raw_ptr());
    _file_list  = make_unique<FilesMenu>(_root->raw_ptr());
    // ... _action_menu, _cursor, _delete_confirm_dialog, _rename_confirm_dialog ...

    // 2. 注册所有 SingleObservable 的回调
    _vm.currentDirectory().observe(this, onDirectoryChanged);
    _vm.entries().observe(this, onEntriesChanged);
    _vm.selectedIndex().observe(this, onSelectedIndexChanged);
    _vm.status().observe(this, onStatusChanged);
    _vm.enterPressed().observe(this, onEnterPressedChanged);
    // ... 共 11 个 observe() 调用 ...

    // 3. 手动渲染当前状态（observe 注册时已回调过一遍，但这里又显式调了一次）
    renderDirectory(_vm.currentDirectory().get());
    renderEntries(_vm.entries().get());
    renderSelectedIndex(_vm.selectedIndex().get());
    // ...
}
```

> 第 2 步的 `observe()` 注册后立即回调，会执行 `renderXxx()`；第 3 步又手动调了一次 `renderXxx()`。
>
> **所以第 3 步是冗余的**——但无害，因为 `renderDirectory`、`renderEntries` 等方法都是幂等的。从语义上看，observe 回调是"值变化了通知我"，而 onEnter 末尾的显式调用是"进入页面时初始化一次"，重复但意图清晰。
>
> 按 ponytail 精神可以删掉第 3 步，但保留也不影响功能。这份代码选择保留，大概是为了强调"进入页面时状态必须同步"的意图，同时 observe 回调的职责更纯粹（只处理变更通知）。

---

## 三、主循环：每 10ms 跑一圈

```cpp
// main.cpp:44-53
while (!app.quitRequested()) {
    keypad.poll();             // 1. 轮询键盘 → 触发 onLvglKeyState()
    if (!app.hostRenderingSuspended())
        lv_timer_handler();    // 2. LVGL 内部定时器（动画、重绘）
    app.tick(lv_tick_get());   // 3. 应用层 tick
    usleep(10000);             // 4. 10ms ≈ 100Hz
}
```

`app.tick()` 的调用链：

```cpp
// files_app.cpp:256-268
void FilesApp::tick(uint32_t nowMs) {
    _current_vm->tick(nowMs);     // → BrowserViewModel::tick()
    _current_view->tick(nowMs);   // → BrowserView::tick()
}
```

- `BrowserViewModel::tick()`：处理长按连发逻辑（按住 Down 键不松手，每隔 90ms 自动触发一次选择）
- `BrowserView::tick()`：驱动动画（SmoothSelectorMenu 滚动、ActionMenu 滑入/滑出动画等）

---

## 四、按键：Down 键的完整调用链

### 第 1 站：`keypad.poll()` → `FilesApp::onLvglKeyState()`

```cpp
// files_app.cpp:124
bool FilesApp::onLvglKeyState(uint32_t lv_key, const char* utf8, bool pressed)
{
    // LV_KEY_UP / LV_KEY_DOWN / LV_KEY_LEFT / LV_KEY_RIGHT 分支
    case LV_KEY_DOWN:
        if (_current_vm)
            _current_vm->onKeyState(files_key::Down, pressed);  // 通知"按下状态"
        if (pressed)
            onKey(files_key::Down);                              // 触发"按键动作"
        return true;
}
```

这里有两路通知：
- `onKeyState(key, true)` → 给 ViewModel 用来检测**长按**
- `onKey(key)` → 给 ViewModel 处理**一次点击**

### 第 2 站：`BrowserViewModel::onKey()`

```cpp
// browser_view_model.cpp:98-103
void BrowserViewModel::onKey(uint32_t key)
{
    switch (key) {
        case files_key::Down:
            _model.browser().selectNext();  // ← 核心：调用 Model
            break;
    }
}
```

ViewModel 不做计算，只是根据按键选择调用哪个 Model 方法。

### 第 3 站：`FileBrowserModel::selectNext()`

```cpp
// file_browser_model.cpp:104-117
void FileBrowserModel::selectNext()
{
    const auto& list = _entries.get();    // 读取当前文件列表
    int next = _selected_index.get() + 1; // 当前选中 + 1
    if (next >= (int)list.size())
        next = 0;                         // 循环到开头

    _selected_index.set(next);            // ★ 关键：触发通知
}
```

### 第 4 站：`SingleObservable<int>::set()` → `notify()`

```cpp
// single_observable.hpp:33-37
void set(const T& newValue)
{
    _value = newValue;
    notify();                         // ← 通知注册的观察者
}

void notify()
{
    if (_callback)
        _callback(_context, _value);  // → 执行 BrowserView 注册的回调
}
```

### 第 5 站：静态回调 → 成员函数

```cpp
// browser_view.cpp (大约 1380 行左右)
void BrowserView::onSelectedIndexChanged(void* context, const int& index)
{
    auto* self = static_cast<BrowserView*>(context);
    self->renderSelectedIndex(index);
}
```

`void* context` 就是在 `observe(this, onSelectedIndexChanged)` 时传入的 `this`。

### 第 6 站：`renderSelectedIndex()` → LVGL 更新

```cpp
// browser_view.cpp:1376-1385
void BrowserView::renderSelectedIndex(int index)
{
    dismissTipsHud();                    // 如果有 TipsHUD 就关闭
    if (_file_list)
        _file_list->setSelectedIndex(index);  // LVGL 控件高亮选中项
    updateCursorTarget();                // 更新光标动画位置
}
```

### 完整链路图

```
main.cpp:44   while (!quit)
                  │
                  ├── keypad.poll()
                  │       ↓
                  │   FilesApp::onLvglKeyState(LV_KEY_DOWN, ...)
                  │       │
                  │       ├── _current_vm->onKeyState(files_key::Down, true)   ← 长按检测
                  │       │
                  │       └── onKey(files_key::Down)
                  │                ↓
                  │           BrowserViewModel::onKey(Down)
                  │                │
                  │                ↓
                  │           FileBrowserModel::selectNext()
                  │                │  _selected_index.set(next)
                  │                ↓
                  │           SingleObservable<int>::notify()
                  │                │  callback(context, next)
                  │                ↓
                  │           BrowserView::onSelectedIndexChanged(ctx, index)
                  │                │  static_cast<BrowserView*>(ctx)
                  │                ↓
                  │           BrowserView::renderSelectedIndex(index)
                  │                │
                  │                ├── _file_list->setSelectedIndex(index)
                  │                └── updateCursorTarget()
                  │
                  ├── app.tick(nowMs)
                  │       │
                  │       ├── _current_vm->tick(nowMs)      ← 长按重复逻辑
                  │       └── _current_view->tick(nowMs)    ← 驱动动画
                  │
                  └── usleep(10000)  ← 继续下一圈
```

---

## 五、逆流：页面切换

当用户选中一个文件夹（按 Enter 进入子目录）时，链路稍有不同：

```cpp
// BrowserViewModel::openSelected()
void BrowserViewModel::openSelected()
{
    FileEntry openedFile;
    FileOperationResult result = _model.browser().openSelected(&openedFile);
    if (!openedFile.path.empty()) {
        if (_model.preview().open(openedFile)) {
            _router.push(PageId::Preview);  // ★ 切换页面
        }
    }
}
```

`_router.push(PageId::Preview)` 的执行：

```cpp
// files_router.cpp:13-20
void FilesRouter::push(PageId page)
{
    _history.push_back(_current_page.get());  // 保存当前页到历史栈
    _current_page.set(page);                  // ← set() → Observable 通知
}
```

`_current_page` 是 `Observable<PageId>`（多观察者版本），谁在观察它？

```cpp
// files_app.cpp:108
_route_observer_id = _router.currentPage().observe(this, onRouteChanged);
```

→ `FilesApp::onRouteChanged()` → `FilesApp::setCurrentPage(PageId::Preview)`：

```cpp
// files_app.cpp:315-337
void FilesApp::setCurrentPage(PageId page)
{
    _current_view->onExit();   // BrowserView::onExit()
    _current_vm->onExit();     // BrowserViewModel::onExit()

    _current_vm   = &_preview_vm;
    _current_view = &_preview_view;

    _current_vm->onEnter();    // PreviewViewModel::onEnter()
    _current_view->onEnter();  // PreviewView::onEnter(lv_screen_active())
}
```

`BrowserView::onExit()` 销毁所有资源：

```cpp
// browser_view.cpp:1299-1301
void BrowserView::onExit() { destroy(); }

void BrowserView::destroy()
{
    // 1. 取消所有 observe 订阅
    _vm.magic().removeObserver();
    _vm.selectedIndex().removeObserver();
    // ... 共 11 个 removeObserver() ...

    // 2. 销毁所有 LVGL 控件（unique_ptr::reset → lv_obj_del）
    _magic_view.reset();
    _rename_confirm_dialog.reset();
    _cursor.reset();
    _action_menu.reset();
    _file_list.reset();
    _path_label.reset();
    _root.reset();
}
```

---

## 六、为什么这么设计？（ponytail 精神）

| 层 | 只负责 | 不负责 | 好处 |
|----|--------|--------|------|
| **Model** | 数据存储 + 文件操作 | 按键处理、UI | 纯逻辑，只处理 `selectNext()` 这种语义操作 |
| **ViewModel** | 按键 → 方法映射 | 数据存储、LVGL 渲染 | `switch(key)` 决定调哪个 Model 方法，中间层隔离开 |
| **View** | LVGL 控件创建 + 更新 | 文件操作、业务逻辑 | 收到通知就渲染，收到销毁就清理，纯粹的反应层 |
| **FilesApp** | 装配 + 转发 | 页面具体行为 | 只负责"当前是哪个页面"，不关心页面内部 |
| **Observable** | 事件通知 | 所有其他 | 解耦的核心，set() 自动通知，不需要手动管理"脏标记" |

### 为什么 ViewModel 要持有 Model 的 `SingleObservable&`？

```cpp
// browser_view_model.hpp
SingleObservable<std::string>& currentDirectory()
{
    return _model.browser().currentDirectory();
    //       ^^^^^ 直接透传 Model 的 Observable
}
```

因为 `SingleObservable` 是引用语义的——Model 和 ViewModel 共享同一个 `_current_directory` 对象。Model 调用 `set()` 时，ViewModel 和 View 都能收到通知，不需要 ViewModel 再转发一次。

### 为什么 View 用 C 函数指针而不是 `std::function`？

```cpp
using OnChangedCallback = void (*)(void* context, const T& newValue);
//                           ^^^^ C 函数指针
```

嵌入式环境（无堆分配、无 RTTI、无异常），`std::function` 的 type erasure 带来的开销不必要。裸函数指针 + `void* context` 是最轻量的回调方案。

---

## 七、记忆卡

```
observe(this, callback)    = "值变了叫我，我就是 callback"
set(newValue)              = "更新值 → 通知所有观察者"
removeObserver()           = "不用再叫我了"
onEnter()                  = "页面激活：造控件、注册回调"
onExit()                   = "页面离开：取消注册、销毁控件"
_current_page.set(page)    = "切换页面（自动触发路由切换）"
```

最简单的理解方式：

> **`main.cpp` 是发动机，`FilesApp` 是变速箱，`ViewModel` 是方向盘，`Model` 是引擎，`View` 是车轮，`SingleObservable` 是传动轴。**
> 按键是踩油门——传动轴把动力一路传到车轮，车轮转了（UI 更新），就可以踩下一个油门了（下一帧）。
