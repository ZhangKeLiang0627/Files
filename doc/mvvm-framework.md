# MVVM 框架实现详解

> 本项目在嵌入式 LVGL GUI 上实现了 **MVVM (Model-View-ViewModel)** 架构模式。核心是一条自制的响应式数据绑定通道：`Observable<T>` / `SingleObservable<T>`，由 `smooth_ui_toolkit` 库提供。

---

## 目录

- [整体架构](#整体架构)
- [核心：响应式数据绑定](#核心响应式数据绑定)
  - [Observable（多观察者）](#observable多观察者)
  - [SingleObservable（单观察者）](#singleobservable单观察者)
- [ViewModel 基类](#viewmodel-基类)
- [View 基类](#view-基类)
- [路由系统](#路由系统)
- [FilesApp 装配](#filesapp-装配)
- [完整数据流：Browser 页面](#完整数据流browser-页面)
  - [1. ViewModel 暴露状态](#1-viewmodel-暴露状态)
  - [2. View 订阅状态](#2-view-订阅状态)
  - [3. 用户输入驱动状态变更](#3-用户输入驱动状态变更)
  - [4. 通知回调触发渲染](#4-通知回调触发渲染)
- [预览 MVVM 子分层：AudioPreview](#预览-mvvm-子分层audiopreview)
- [设计要点总结](#设计要点总结)

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        FilesApp                             │
│  持有 Router / Model / ViewModel(×2) / View(×2)            │
│  负责：路由切换 → 激活对应的 VM + View                      │
└──────────────┬──────────────────────────────────────────────┘
               │ 路由变化
               ▼
┌──────────────────────┐     ┌──────────────────────┐
│   BrowserViewModel   │     │   PreviewViewModel    │
│                      │     │                       │
│  SingleObservable<T> │◄───►│  SingleObservable<T>  │
│  作为公开属性暴露     │     │  作为公开属性暴露      │
└─────────┬────────────┘     └───────────┬───────────┘
          │ observe()                     │ observe()
          ▼                               ▼
┌──────────────────────┐     ┌──────────────────────┐
│     BrowserView      │     │     PreviewView       │
│                      │     │                       │
│  订阅 VM 的 Observable │     │  订阅 VM 的 Observable │
│  onEnter 注册回调      │     │  onEnter 注册回调      │
│  onExit 移除回调       │     │  onExit 移除回调       │
└──────────────────────┘     └──────────────────────┘
          │                               │
          └────────── LVGL 渲染 ──────────┘
```

**数据流方向**：

```
用户按键  →  FilesApp  →  ViewModel.onKey()
                                   │
                                   ▼
                            Model 方法调用
                                   │
                                   ▼
                        Observable.set(新值)
                                   │
                                   ▼
                        View 回调函数被触发
                                   │
                                   ▼
                            LVGL UI 更新
```

---

## 核心：响应式数据绑定

整个 MVVM 的基石是 `Observable<T>` 和 `SingleObservable<T>`。它们是发布-订阅模式的模板类：当值变化时自动通知所有观察者。

### Observable（多观察者）

支持 **多个** 观察者同时监听同一个值（用于 Router 的 `PageId` 等全局状态）。

```cpp
// dependencies/smooth_ui_toolkit/src/tools/observable/observable.hpp
template <typename T>
class Observable {
public:
    using OnChangedCallback = void (*)(void* context, const T& newValue);

    // ── 构造 ──
    explicit Observable(T defaultValue) : _value(std::move(defaultValue)) {}

    // ── 写值 + 自动通知 ──
    void set(const T& newValue) { _value = newValue; notify(); }
    void set(T&& newValue)      { _value = std::move(newValue); notify(); }

    // ── 读值 ──
    const T& get() const { return _value; }

    // ── 订阅：注册回调，注册后立即回调一次（初始化同步） ──
    size_t observe(void* context, OnChangedCallback callback) {
        size_t id = ++_last_id;
        _observers.push_back({id, context, callback});
        if (callback) {
            callback(context, _value);      // ponytail: 注册即回调，省去手动初始化渲染
        }
        return id;                          // 返回 ID 供取消订阅用
    }

    // ── 取消订阅 ──
    void removeObserver(size_t id) { /* 从 vector 中移除 */ }
    void clearObservers()          { _observers.clear(); }

private:
    T _value;
    struct Observer_t {
        size_t id;
        void* context;
        OnChangedCallback callback;
    };
    std::vector<Observer_t> _observers;     // ponytail: vector + 裸函数指针，无 mutex（单线程 LVGL）
    size_t _last_id = 0;

    void notify() {
        for (auto& observer : _observers) {
            if (observer.callback) {
                observer.callback(observer.context, _value);
            }
        }
    }
};
```

**关键设计决策**：

| 决策 | 理由（ponytail 精神） |
|------|----------------------|
| `vector<Observer_t>` 裸存储 | 嵌入式环境不需要 `shared_ptr` 的开销 |
| C 函数指针而非 `std::function` | 避免类型擦除和堆分配 |
| 无 mutex | LVGL 是单线程模型，加锁是无意义的复杂度 |
| `observe()` 注册后立即回调 | 省去 View 在 `onEnter` 中手动初始化所有 UI 状态 |
| 返回 `size_t` observer ID | 比 `shared_ptr` 连接更轻量，取消订阅时直接传 ID |

### SingleObservable（单观察者）

只有一个观察者的优化版本，用于 ViewModel → View 的一对一绑定。接口相同，内部去掉了 `vector` 和 ID 分配。

```cpp
// dependencies/smooth_ui_toolkit/src/tools/observable/single_observable.hpp
template <typename T>
class SingleObservable {
public:
    using OnChangedCallback = void (*)(void* context, const T& newValue);

    explicit SingleObservable(T defaultValue) : _value(std::move(defaultValue)) {}

    void set(const T& newValue) { _value = newValue; notify(); }

    const T& get() const { return _value; }

    // 直接存储唯一的回调
    void observe(void* context, OnChangedCallback callback) {
        _context   = context;
        _callback  = callback;
        notify();                       // 注册即回调
    }

    void removeObserver() {
        _context   = nullptr;
        _callback  = nullptr;
    }

private:
    T _value;
    void* _context         = nullptr;
    OnChangedCallback _callback = nullptr;

    void notify() {
        if (_callback) {
            _callback(_context, _value);
        }
    }
};
```

> **Observable vs SingleObservable**：`Observable` 用于 Router 的 `PageId`（可以被 FilesApp 和多个地方同时观察）；`SingleObservable` 用于 ViewModel 暴露给 View 的状态（一个状态只被一个 View 订阅）。

---

## ViewModel 基类

每个页面（Browser / Preview）对应一个 ViewModel，继承自统一基类：

```cpp
// src/view_models/view_model.hpp
class ViewModel {
public:
    explicit ViewModel(FilesRouter& router) : _router(router) {}

    virtual ~ViewModel() = default;

    // 返回这个 VM 对应的页面 ID
    virtual PageId pageId() const = 0;

    // 页面激活/离开时调用
    virtual void onEnter() {}
    virtual void onExit()  {}

    // 输入事件 —— 这是 VM 的核心入口
    virtual void onKey(uint32_t key) {}              // 按键按下
    virtual void onKeyState(uint32_t key, bool pressed) {}  // 按键状态变化

    // 每帧 tick
    virtual void tick(uint32_t nowMs) {}

    // 是否暂停 LVGL 的帧渲染（视频播放等场景）
    virtual bool suspendsHostRendering() const { return false; }

protected:
    FilesRouter& _router;     // 持有路由引用，可切换页面
};
```

**ViewModel 的职责**：
1. 通过 `onKey()` / `onKeyState()` 接收来自 FilesApp 转发的用户输入
2. 调用 Model 的方法执行业务逻辑
3. 更新 `SingleObservable` 的值 → 自动通知 View 重渲染
4. 需要切换页面时调用 `_router.push()` / `_router.back()`

---

## View 基类

```cpp
// src/views/view.hpp
class View {
public:
    virtual ~View() = default;

    // 页面激活时创建 UI
    virtual void onEnter(lv_obj_t* parent) = 0;

    // 页面离开时销毁 UI
    virtual void onExit() = 0;

    // 每帧更新（驱动 SmoothSelectorMenu 动画等）
    virtual void tick(uint32_t nowMs) {}

protected:
    View() = default;
};
```

**View 的职责**：
1. `onEnter`：在 LVGL 容器上创建 UI 控件 + **订阅 ViewModel 的 Observable**
2. `onExit`：**取消订阅** + 销毁 UI 控件
3. `tick`：驱动动画引擎更新（SmoothSelectorMenu、AnimateValue 等）

View **不处理任何业务逻辑**，所有操作都通过 ViewModel 暴露的方法进行。

---

## 路由系统

`FilesRouter` 是一个简单的页面栈，通过 `Observable<PageId>` 驱动页面切换：

```cpp
// src/core/files_router.hpp
class FilesRouter {
public:
    Observable<PageId>& currentPage() { return _current_page; }
    PageId page() const               { return _current_page.get(); }

    void replace(PageId page);   // 替换当前页（无历史）
    void push(PageId page);      // 压栈新页面（可返回）
    void back();                 // 返回上一页

private:
    Observable<PageId> _current_page{PageId::Browser};
    std::vector<PageId> _history;
};
```

当 ViewModel 调用 `_router.push(PageId::Preview)` 时：
1. `_current_page.set(PageId::Preview)` → 触发 `onRouteChanged`
2. FilesApp 收到回调 → 调用 `setCurrentPage()`
3. 旧 View 调用 `onExit()`（取消订阅 + 销毁 UI）
4. 新 View 调用 `onEnter()`（创建 UI + 订阅）

---

## FilesApp 装配

`FilesApp` 是 MVVM 各层的组装者：

```cpp
// src/core/files_app.cpp
FilesApp::FilesApp(FilesConfig config)
    : _config(std::move(config)),
      _model(_config.start_directory),
      _browser_vm(_router, _model),      // VM ← Model + Router
      _preview_vm(_router, _model),       // VM ← Model + Router
      _browser_view(_browser_vm),         // View ← VM
      _preview_view(_preview_vm),         // View ← VM
      _view_models{&_browser_vm, &_preview_vm},
      _views{&_browser_view, &_preview_view}
{}

void FilesApp::start() {
    initFontAssets();
    setupInputGroup();
    // 订阅路由变化
    _route_observer_id = _router.currentPage().observe(this, onRouteChanged);
    setCurrentPage(_router.page());  // 初始为 Browser
}

void FilesApp::setCurrentPage(PageId page) {
    ViewModel* next_vm   = viewModelFor(page);
    View* next_view      = viewFor(page);

    if (_current_view) _current_view->onExit();
    if (_current_vm)   _current_vm->onExit();

    _current_vm   = next_vm;
    _current_view = next_view;

    _current_vm->onEnter();
    _current_view->onEnter(lv_screen_active());
}
```

**输入事件传递链路**：

```
Keyboard (LVGL key)
      │
      ▼
FilesApp::onLvglKeyState()        ← LVGL 事件回调
      │
      ├── 映射 ASCII / F/Z/X/C 到 files_key
      │
      ▼
ViewModel::onKey(key)             ← 业务逻辑入口
      │
      ├── _model.browser().selectNext()
      ├── _model.browser().openSelected(&openedFile)
      ├── _router.push(PageId::Preview)
      └── _enter_pressed.set(true)   ← Observable 通知 View
```

---

## 完整数据流：Browser 页面

以"用户按 Down 键选择下一个文件"为例，展示完整的 MVVM 数据流：

### 1. ViewModel 暴露状态

`BrowserViewModel` 将内部状态通过 `SingleObservable` 公开暴露：

```cpp
// src/view_models/browser_view_model.hpp
class BrowserViewModel : public ViewModel {
public:
    // 直接返回 Model 内部的 Observable（数据透传）
    SingleObservable<std::string>&    currentDirectory() { return _model.browser().currentDirectory(); }
    SingleObservable<std::vector<FileEntry>>& entries()  { return _model.browser().entries(); }
    SingleObservable<int>&            selectedIndex()    { return _model.browser().selectedIndex(); }

    // ViewModel 自己的 Observable（UI 专属状态）
    SingleObservable<bool>&           enterPressed()     { return _enter_pressed; }
    SingleObservable<bool>&           actionMenuOpen()   { return _action_menu_open; }
    SingleObservable<uint32_t>&       magic()            { return _magic; }

private:
    SingleObservable<bool>           _enter_pressed{false};
    SingleObservable<bool>           _action_menu_open{false};
    SingleObservable<uint32_t>       _magic{0};
};
```

### 2. View 订阅状态

`BrowserView::onEnter()` 中注册所有回调：

```cpp
// src/views/browser_view.cpp — BrowserView::onEnter()
void BrowserView::onEnter(lv_obj_t* parent) {
    // ... 创建 LVGL 控件树 ...

    // ── 订阅 ViewModel 的所有状态 ──
    _vm.currentDirectory().observe(this, onDirectoryChanged);
    _vm.entries().observe(this, onEntriesChanged);
    _vm.selectedIndex().observe(this, onSelectedIndexChanged);
    _vm.status().observe(this, onStatusChanged);
    _vm.enterPressed().observe(this, onEnterPressedChanged);
    _vm.actionMenuOpen().observe(this, onActionMenuOpenChanged);
    _vm.actionMenuSelectedIndex().observe(this, onActionMenuSelectedIndexChanged);
    _vm.pendingDelete().observe(this, onPendingDeleteChanged);
    _vm.pendingRename().observe(this, onPendingRenameChanged);
    _vm.pendingRenameName().observe(this, onPendingRenameNameChanged);
    _vm.magic().observe(this, onMagicChanged);

    // observe() 注册后会立刻回调一次，所以无需额外初始化
}
```

每个状态变化的回调都是一个静态函数，将 `void* context` 转回 `BrowserView*` 后调用实际渲染方法：

```cpp
// src/views/browser_view.cpp — 静态回调 → 成员函数
void BrowserView::onSelectedIndexChanged(void* context, const int& index) {
    static_cast<BrowserView*>(context)->renderSelectedIndex(index);
}

void BrowserView::onEntriesChanged(void* context, const std::vector<FileEntry>& entries) {
    static_cast<BrowserView*>(context)->renderEntries(entries);
}
```

`onExit()` 中取消所有订阅并销毁 UI：

```cpp
void BrowserView::onExit() {
    // 取消所有 Observable 订阅
    _vm.magic().removeObserver();
    _vm.pendingRenameName().removeObserver();
    // ... 其余同理 ...

    // 销毁 LVGL 对象（unique_ptr reset 自动触发 lv_obj_del）
    _magic_view.reset();
    _tips_hud.reset();
    // ... 其余同理 ...
}
```

### 3. 用户输入驱动状态变更

用户按下 Down 键时：

```cpp
// FilesApp 将按键转发给当前 ViewModel
void FilesApp::onLvglKeyState(uint32_t lv_key, const char* utf8, bool pressed) {
    // ... 按键映射 ...
    case LV_KEY_DOWN:
        _current_vm->onKeyState(files_key::Down, pressed);  // 先通知按下状态
        if (pressed) onKey(files_key::Down);                 // 再触发动作
        return;
}

// → BrowserViewModel::onKey()
void BrowserViewModel::onKey(uint32_t key) {
    switch (key) {
        case files_key::Down:
            _model.browser().selectNext();   // ← 调用 Model 方法
            break;
    }
}
```

### 4. Model 更新 → Observable 通知 → View 渲染

```cpp
// FileBrowserModel::selectNext()
void FileBrowserModel::selectNext() {
    int next = _selected_index.get() + 1;
    if (next >= static_cast<int>(_entries.get().size())) {
        next = 0;    // 循环选择
    }
    _selected_index.set(next);  // ← 触发 SingleObservable 的 notify()
}
```

`_selected_index.set(next)` 内部调用 `notify()`，遍历所有观察者执行回调：

```cpp
// SingleObservable::notify()
void notify() {
    if (_callback) {
        _callback(_context, _value);   // → onSelectedIndexChanged(context, next)
    }
}
```

`BrowserView` 收到回调，更新选中状态：

```cpp
void BrowserView::renderSelectedIndex(int index) {
    dismissTipsHud();
    if (_file_list) {
        _file_list->setSelectedIndex(index);  // 更新 SmoothSelectorMenu 选中项 + 带动画滚动
    }
    updateCursorTarget();                     // 更新光标位置
}
```

### 完整链路图

```
用户按 Down 键
    │
    ▼
LVGL 事件 → FilesApp::onLvglKeyState()
    │ 映射 files_key::Down
    ▼
BrowserViewModel::onKey(Down)
    │
    ▼
FileBrowserModel::selectNext()
    │  _selected_index.set(next)
    ▼
SingleObservable<int>::notify()
    │  执行已注册的 callback
    ▼
BrowserView::onSelectedIndexChanged(context, index)
    │  static_cast<BrowserView*>(context)
    ▼
BrowserView::renderSelectedIndex(index)
    │  _file_list->setSelectedIndex(index)
    ▼
SmoothSelectorMenu 动画滚动 + 光标位置更新
```

---

## 预览 MVVM 子分层：AudioPreview

音频预览内部也采用了完整的 MVVM 子分层：

```
AudioPreview (主控)
    │
    ├── AudioPreviewModel       ← 数据层：播放状态、进度、倍速
    ├── AudioPreviewViewModel   ← VM 层：onKey 逻辑、状态管理
    └── AudioPreviewView        ← View 层：LVGL 控件渲染
```

```cpp
// src/preview/audio/audio_preview_view.hpp
class AudioPreviewView {
    // 订阅 AudioPreviewViewModel 的 Observable
    _vm.playbackState().observe(this, onPlaybackStateChanged);
    _vm.currentTime().observe(this, onCurrentTimeChanged);
    _vm.speed().observe(this, onSpeedChanged);
};
```

这种"大 MVVM 套小 MVVM"的嵌套结构，体现了模式的一致性和可复用性。

---

## 设计要点总结

| 方面 | 实现 | ponytail 理由 |
|------|------|-------------|
| **绑定机制** | `SingleObservable<T>` 裸函数指针 + `void* context` | `std::function` 有类型擦除开销，裸指针在单线程嵌入式环境足够 |
| **线程安全** | 无锁 | LVGL 是单线程模型 |
| **订阅时间** | `onEnter()` 订阅，`onExit()` 取消 | 与页面生命周期严格绑定，避免野指针 |
| **初始化同步** | `observe()` 注册后立即回调一次 | 省去手写"初始化时同步一次状态"的模板代码 |
| **VM 数据源** | 一部分直接透传 Model 的 Observable，一部分自己持有 | 区分"纯数据状态"（目录列表）和"UI 状态"（菜单开关） |
| **View 的职责** | 仅渲染 + 动画，不处理业务逻辑 | 业务逻辑在 VM 中，方便测试和替换 View |
| **Router 解耦** | VM 持有 `FilesRouter&` 而非直接操作 View | VM 通过 `push()`/`back()` 表达"意图"，View 切换由 FilesApp 完成 |
| **通知即渲染** | 每个 Observable 都有对应的 `renderXxx()` 方法 | 一对一的映射关系，调试时一目了然 |
