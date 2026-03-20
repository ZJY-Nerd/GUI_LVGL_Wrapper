# 在 CMSIS-Pack（RTE）中使用 GUI_LVGL_Wrapper 的 Scene 模板（`<name>` / `<NAME>`）

本文说明如何在 **CMSIS-Pack / RTE（例如 Keil MDK 的 Run-Time Environment）** 中使用 GUI_LVGL_Wrapper 提供的两套 Scene 模板来创建场景：

- **RTE 组件实例法（`%Instance%`）**：由 RTE 按实例数自动生成 `gui_scene_0/1/2/...`
- **User Code Template 法（`<name>` / `<NAME>`）**：按业务命名创建 `gui_scene_<name>`，并在入口处手动注册

> 关联文件（本仓库路径）：
> - User Scene Template：
>   - `template/gui_scene_template.h`
>   - `template/gui_scene_template.c`
> - RTE Scene 实例模板：
>   - `template/gui_scene.h`
>   - `template/gui_scene.c`
> - RTE 聚合头（按宏自动 include/init）：
>   - `template/gui_scene_include.h`

---

## 1. 两种“在 Pack 里添加 Scene”的方式（先搞清楚差别）

在 Pack/RTE 场景下，通常有两种添加 Scene 的方式：

### 1.1 RTE 组件实例法（`gui_scene_%Instance%`）

- 在 RTE 配置里启用并设置 `GUI_LVGL_Wrapper::Scene` 的实例数（最多 20）。
- RTE 会生成 `gui_scene_0.c/.h`、`gui_scene_1.c/.h` ...（生成目录通常在 `RTE/` 下）。
- `template/gui_scene_include.h` 会依据 `__RTE_Acceleration_GUI_LVGL_SceneN__` 宏自动 `#include "gui_scene_N.h"` 并提供 `__GUI_LV_ALL_SCENE_INIT()`。
- `gui_lvgl.c` 在 `#if defined(__RTE_Acceleration_GUI_LVGL_SCENE__)` 下会调用 `__GUI_LV_ALL_SCENE_INIT()` 自动完成所有实例的注册。

这套方式的特点是：**scene 的“注册入口函数”固定为数字命名**：`gui_lv_scene_0_init()`、`gui_lv_scene_1_init()` ...

### 1.2 User Code Template 法（自定义命名 `gui_scene_<name>`）

- 在 IDE 里通过 User Code Template 生成一对模板文件（或在 VS Code 里手工复制）。
- 你需要把模板中的 `<name>` / `<NAME>`（必要）以及 `%Instance%`（按需要）替换为你的命名。
- 这类“自定义命名”的 scene **不会被** `template/gui_scene_include.h` 自动包含/初始化：
  - 如果你不启用 `GUI_LVGL_Wrapper::Scene` 组件，则在 `gui_lvgl.c` 的 `__gui_all_scene_init()` 用户代码区手动调用你的 `gui_lv_scene_<name>_init()` 注册即可。
  - 如果你同时启用了 `GUI_LVGL_Wrapper::Scene`，你仍可额外手动注册自定义 scene，但需要你在工程入口里再补一次调用（属于进阶混用用法）。

如果你的目标是“我想要一个叫 `gui_scene_main_menu.c/h`，内部函数名也更可读”的场景，选择 **方法 1.2**。

---

## 2. 前置条件（RTE 里要勾哪些组件）

最小依赖：

- `GUI_LVGL_Wrapper::Core`
- `GUI_LVGL_Wrapper::USER`（提供可配置的 `gui_lvgl.c` / `gui_scene_id.h` 等）

可选：

- `GUI_LVGL_Wrapper::Helper`（如果你使用 helper 的控件/样式 API）

当你使用 **RTE 组件实例法** 时，还需要：

- `GUI_LVGL_Wrapper::Scene`（并设置实例数）

当你使用 **User Code Template 法** 时，确保能在 IDE 里看到模板项（通常来源于 `GUI_LVGL_Wrapper::Template` 组件）。如果列表里没有该模板，先确认已安装该 Pack 且已启用 `GUI_LVGL_Wrapper::USER`（必要时也启用 `GUI_LVGL_Wrapper::Template`）。

---

## 3. 生成/添加模板文件

### 3.1 在 Keil MDK（RTE 工程）里生成

**方式 A：RTE 组件实例法**

- 打开 RTE 配置窗口
- 选择 `GUI_LVGL_Wrapper::Scene`
- 把实例数调到你需要的数量

然后到生成的 `gui_scene_N.c/.h` 文件里填充 UI 逻辑，并把 `<NAME>` 改成你实际的 scene ID（见第 4 节）。

**方式 B：User Code Template 法**

- 在 Project 视图里选中一个 Group
- 右键 → **Add New Item to Group**
- 找到 **User Code Template**
- 选择 GUI_LVGL_Wrapper 提供的 `gui_scene_template.c/.h` 模板项

随后会把模板文件加入工程（位置取决于你当前 Group 的路径/设置）。建议立即按第 4 节完成重命名与占位符替换。

### 3.2 在 VS Code / 非 MDK 环境手工添加

- 直接复制：
  - `template/gui_scene_template.h`
  - `template/gui_scene_template.c`
- 放到你的工程源码目录（确保编译系统会编译 `.c`、并能找到 `.h`）

---

## 4. `<name>` / `<NAME>` / `%Instance%` 替换与重命名规则（核心）

模板里涉及三类“占位符/约定”：

### 4.1 `<name>`（小写/下划线，建议 `snake_case`）

用于：

- 静态回调/私有符号：`__on_scene_<name>_draw()`、`__on_scene_<name>_load()`、`__on_scene_<name>_depose()` 等

### 4.2 `<NAME>`（大写/下划线，建议全大写）

用于：

- Scene ID：`GUI_SCENE_<NAME>`（你必须在 `gui_scene_id.h` 里定义它）
- 头文件 include guard：`__GUI_SCENE_<NAME>_H__`

### 4.3 `%Instance%`（实例号：0/1/2/...，或你希望的后缀）

用于：

- 对外注册入口函数名：`gui_lv_scene_%Instance%_init()`

在 **RTE 组件实例法** 中：

- `%Instance%` 由 RTE 自动替换为 0/1/2/...（不要手改）

在 **User Code Template 法** 中：

- 你需要自己决定 `%Instance%` 最终是什么：
  - **推荐（自定义命名）**：把 `%Instance%` 也替换为 `<name>`，得到 `gui_lv_scene_<name>_init()`
  - **保持与实例法一致**：把 `%Instance%` 替换为数字（0/1/2/...），得到 `gui_lv_scene_0_init()` 这种形式

---

## 5. 典型流程（创建一个名为 `main_menu` 的 Scene）

下面以 `main_menu` 为例，演示 User Code Template 的推荐改法。

### 5.1 新建/重命名文件

- `gui_scene_template.c` → `gui_scene_main_menu.c`
- `gui_scene_template.h` → `gui_scene_main_menu.h`

### 5.2 全局替换占位符

- `<name>` → `main_menu`
- `<NAME>` → `MAIN_MENU`
- `%Instance%` → `main_menu`

替换完成后，你会得到：

- 头文件 include guard：`__GUI_SCENE_MAIN_MENU_H__`
- 注册入口：`void gui_lv_scene_main_menu_init(void);`
- scene ID：`.eId = GUI_SCENE_MAIN_MENU`

### 5.3 在 `gui_scene_id.h` 中添加 Scene ID

在 `gui_scene_id.h` 的 `gui_scene_id_t` enum 里加入：

```c
GUI_SCENE_MAIN_MENU,
```

并确保它位于 `GUI_SCENE_MAX` 之前。

### 5.4 在入口里注册并设置开机场景

如果你**不启用** `GUI_LVGL_Wrapper::Scene` 组件（即不走实例法），在 `gui_lvgl.c` 的用户代码区：

```c
static void __gui_sys_data_init(void)
{
    gui_lv_set_boot_scene_id(GUI_SCENE_MAIN_MENU);
}

static void __gui_all_scene_init(void)
{
    /* user code begin: scene register */
    gui_lv_scene_main_menu_init();
    /* user code end  : scene register */
}
```

随后调用 `gui_lv_init()`，框架会在注册完 scene 后自动执行：

- `gui_lv_scene_switch(gui_lv_get_boot_scene_id())`

也就是显示你设置的开机场景。

---

## 6. 常见问题与排查

### 6.1 编译报错：`GUI_SCENE_xxx` 未定义

- 检查是否已在 `gui_scene_id.h` 的 `gui_scene_id_t` 里添加对应枚举项（且在 `GUI_SCENE_MAX` 之前）
- 检查场景文件里 `.eId = GUI_SCENE_<NAME>` 是否已把 `<NAME>` 替换成实际名称

### 6.2 编译报错：源码里还残留 `<name>` / `<NAME>` / `%Instance%`

- 说明替换没做全；建议对工程全局搜索 `<name>`、`<NAME>`、`%Instance%` 并逐一替换

### 6.3 链接报错：找不到 `gui_lv_scene_xxx_init`

- 检查对应 `.c` 文件是否已加入工程并参与编译
- 检查头文件声明与 `.c` 定义的函数名是否一致（尤其是 `%Instance%` 的替换策略）

### 6.4 scene 已注册但启动显示为空

- 确认 `__gui_sys_data_init()` 中的 `gui_lv_set_boot_scene_id()` 传入的 scene ID 合法（小于 `GUI_SCENE_MAX`）
- 确认 `__gui_all_scene_init()` 在 `gui_lv_scene_switch()` 之前完成了注册（正常情况下 `gui_lv_init()` 已保证顺序）
- 确认你的 `__on_scene_<name>_draw()` 里确实创建了控件/布局

---

## 7. 何时用实例法，何时用 `<name>` 模板

- 你想让 **RTE 自动生成/聚合/初始化** N 个 scene → 用 `GUI_LVGL_Wrapper::Scene`（`gui_scene_%Instance%`）
- 你想要 **按业务命名文件与内部回调函数**，并更可控地手动注册 → 用 `gui_scene_template`（`<name>` / `<NAME>`）

两者都基于同一个场景管理框架（`gui_lv_scene_register()` / `gui_lv_scene_switch()`），可以按工程需要选择或混用。
