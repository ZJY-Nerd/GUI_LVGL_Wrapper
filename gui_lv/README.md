# GUI_LVGL_Wrapper

GUI_LVGL_Wrapper 是一套基于 LVGL 8.x 的嵌入式 GUI 封装层，核心目标是把
UI 初始化、场景切换、页面栈、输入焦点管理和常用控件创建做成稳定接口，
让业务代码聚焦在“画什么”和“怎么交互”，而不是反复处理底层样板代码。

## 1. 功能概览

- 单入口初始化：`gui_lv_init()`
- 场景生命周期管理：注册 / 切换 / 返回
- 页面子栈管理：`page_push` / `page_back`
- 焦点索引保存与恢复（支持多 group）
- 常用控件快速创建 API（container/img/label/btn/bar/spinbox）
- 通用样式配置器（结构化 style + 标记位）
- 输入与参数编辑辅助模块（导航、加减、回环）
- 同时支持：
	- 直接源码集成（非 RTE）
	- CMSIS-Pack / RTE 组件化集成

## 2. 目录结构

`gui_lv/` 下核心目录说明：

```text
gui_lv/
├─ core/                     # 场景管理与基础控件封装
│  ├─ gui_lv_common.h/.c
│  └─ gui_lv_scene_manage.h/.c
├─ helper/                   # 辅助模块：style / data / input
│  ├─ include/
│  └─ source/
├─ port/                     # LVGL 显示与输入设备移植模板
│  ├─ lv_port_disp_template.h/.c
│  └─ lv_port_indev_template.h/.c
├─ template/                 # 场景模板（RTE 实例模板 + 用户模板）
├─ gui_lvgl.h/.c             # 对外主入口
├─ gui_lv_sys_data.h/.c      # 语言/开机场景/蜂鸣档位系统数据
├─ gui_scene_id.h            # 场景与页面 ID 枚举定义（用户维护）
├─ gui_lv_conf.h             # GUI 封装相关配置项
├─ gui_lv_utils.h            # 常用宏（颜色、定时器、group、对齐等）
└─ ZJY.GUI_LVGL_Wrapper.pdsc # CMSIS-Pack 描述文件
```

## 3. 快速开始

### 3.1 方式 A：直接源码集成（非 RTE）

1. 引入 LVGL 8.x，并完成你自己的 `lv_port_disp` / `lv_port_indev`。
2. 将 `gui_lv/` 下 `core`、`helper`、`gui_lvgl.*`、`gui_lv_sys_data.*`、
	 `gui_scene_id.h` 等加入工程编译。
3. 在 `gui_scene_id.h` 中定义业务场景/页面 ID。
4. 基于 `template/gui_scene_template.c/.h` 新建你的场景文件。
5. 在 `gui_lvgl.c` 的用户代码区：
	 - `__gui_sys_data_init()` 设置语言和开机场景
	 - `__gui_common_style_init()` 初始化通用样式
	 - `__gui_all_scene_init()` 注册所有场景
6. 系统启动时调用 `gui_lv_init()`；主循环周期调用 `lv_timer_handler()`。

### 3.2 方式 B：CMSIS-Pack / RTE 集成

在 RTE 中按顺序启用组件：

1. `GUI_LVGL_Wrapper::Core`
2. `GUI_LVGL_Wrapper::Helper`
3. `GUI_LVGL_Wrapper::USER`
4. `GUI_LVGL_Wrapper::Scene`（可多实例，最多 20）

说明：

- `Scene` 组件会生成 `gui_scene_%Instance%.c/.h`。
- `gui_lvgl.c` 在 `__RTE_GUI_LVGL_WRAPPER__` 下通过
	`template/gui_scene_include.h` 自动初始化已启用实例。
- 如果需要自定义命名模板（`<name>` / `<NAME>`），参考：
	`template/how_to_use_user_scene_template_in_pack_zh.md`

## 4. 初始化流程

`gui_lv_init()` 的执行顺序：

1. （非 `_WIN64`）`lv_init()`
2. （非 `_WIN64`）`lv_port_disp_init()`
3. （非 `_WIN64`）`lv_port_indev_init()`
4. `__gui_sys_data_init()`：系统参数初始化（语言/开机场景）
5. `__gui_common_style_init()`：通用样式初始化
6. `__gui_all_scene_init()`：场景注册
7. `gui_lv_scene_switch(gui_lv_get_boot_scene_id())`
8. `lv_timer_handler()`（首次触发）

建议：`gui_lv_init()` 只做一次；后续渲染时序交给系统主循环。

## 5. 场景与页面机制

### 5.1 Scene API

- `bool gui_lv_scene_register(const gui_scene_cfg_t *ptCfg)`
	- 注册场景（必填：`eId`、`pfInit`、`pfDeinit`）
- `void gui_lv_scene_switch(gui_scene_id_t eTargetId)`
	- 切换场景：销毁当前场景后重建目标场景
- `void gui_lv_scene_back(void)`
	- 返回历史场景（栈深默认 8）
- `gui_scene_id_t gui_lv_scene_get_current(void)`
- `lv_obj_t *gui_lv_scene_get_current_root(void)`

### 5.2 Page API（场景内子页面）

- `bool gui_lv_scene_page_register(const gui_page_cfg_t *ptCfg)`
- `void gui_lv_scene_page_push(gui_scene_page_id_t ePageId)`
- `void gui_lv_scene_page_back(void)`

页面机制特点：

- `push` 时会清空当前 root 下 UI，再创建新页面容器
- `back` 时通过再次调用上一层 `pfInit` 重建页面
- 自动保存并恢复 focus index（多 group）

### 5.3 容量与边界（当前实现）

- 场景回退历史深度：`GUI_SCENE_HISTORY_DEPTH = 8`
- 页面节点池大小：`GUI_PAGE_NODE_POOL_SIZE = 8`
- 单场景最大 group 记录数：`GUI_SCENE_GROUP_MAX = 4`

如果业务需要更大容量，请调整 `gui_lv_scene_manage.c` 相关宏。

## 6. 最小使用示例（非 RTE）

### 6.1 定义场景 ID

在 `gui_scene_id.h`：

```c
typedef enum {
		GUI_SCENE_HOME = 0,
		GUI_SCENE_SETTING,
		GUI_SCENE_MAX
} gui_scene_id_t;
```

### 6.2 注册场景

在你的场景文件中：

```c
static void __on_scene_home_draw(lv_obj_t *ptRoot)
{
		gui_lv_label_init(ptRoot, 10, 10, "Home", &lv_font_montserrat_20,
											rgb(255, 255, 255));
}

static void __on_scene_home_depose(void)
{
}

void gui_lv_scene_home_init(void)
{
		const gui_scene_cfg_t c_tCFG = {
				.eId      = GUI_SCENE_HOME,
				.ptEx     = NULL,
				.pfInit   = __on_scene_home_draw,
				.pfDeinit = __on_scene_home_depose,
		};
		gui_lv_scene_register(&c_tCFG);
}
```

### 6.3 在入口里挂接

在 `gui_lvgl.c` 用户代码区：

```c
static void __gui_sys_data_init(void)
{
		gui_lv_set_lang(GUI_LV_LANGUAGE_EN);
		gui_lv_set_boot_scene_id(GUI_SCENE_HOME);
}

static void __gui_all_scene_init(void)
{
		gui_lv_scene_home_init();
}
```

## 7. Helper 模块

### 7.1 `core/gui_lv_common.*`

常用控件快速创建：

- `gui_lv_container_init`
- `gui_lv_img_init`
- `gui_lv_label_init`
- `gui_lv_btn_init`
- `gui_lv_bar_init`
- `gui_lv_spinbox_init`
- `gui_lv_timer_init`

Group 焦点工具：

- `gui_lv_group_focus_nav`
- `gui_lv_group_get_focus_index`

### 7.2 `helper/style`

- 用 `ui_style_t` + `style_config_t` 描述样式
- 调用 `style_init()` 完成 LVGL 样式对象构建
- 使用 `style_apply()` / `style_remove()` / `style_reset()`
- 支持宏化字段，如：`bg_color(...)`、`radius(...)`、`text_font(...)`

### 7.3 `helper/input` 与 `helper/data`

- `ui_data_nav()`：处理网格导航（支持换行与回环）
- `ui_data_set()`：整型/浮点参数增减与边界控制
- `ui_data_save_to_flash()` / `ui_data_reset_to_default()`：参数持久化钩子

## 8. 资源与字体

本仓库资源生成工具位于 `gui/gui_resource`：

- 资源说明：`../gui/gui_resource/README.md`
- 自动化细节：`../gui/gui_resource/copilot.md`

典型流程：

1. 在 `assets/font_config.json`、`assets/img_config.json` 配置资源
2. 运行 `tools/resource_gen.py` 或 `tools/generate_all.py`
3. 使用 `generated/ui_resource.h` 中声明的字体/图片符号

Pack 模式通常直接消费 `generated/` 下预生成文件，不依赖 Python。

## 9. 配置项说明

`gui_lv_conf.h` 关键宏：

- 分辨率：`MY_DISP_HOR_RES`、`MY_DISP_VER_RES`
- Buffer：`__BUF_PX_SIZE__`、`__LV_DISP_USE_BUFFER__`
- 输入开关：
	- `__LV_USE_KEYPAD_INDEV__`
	- `__LV_USE_TOUCHPAD_INDEV__`
	- `__LV_USE_ENCODER_INDEV__`
	- `__LV_USE_MOUSE_INDEV__`
	- `__LV_USE_BUTTON_INDEV__`

建议先按硬件真实能力配置输入设备，再设计 group 绑定策略。

## 10. 移植检查清单

- 已实现并验证 `lv_port_disp_init()`
- 已实现并验证 `lv_port_indev_init()`
- LVGL Tick 与 `lv_timer_handler()` 调度正常
- `gui_scene_id.h` 已定义有效场景枚举
- 开机场景 ID 与已注册场景一致
- 若使用 keypad/group：确认 `indev_keypad` 非空并绑定 group
- 若使用页面栈：页面 ID 已注册且池大小满足需求

## 11. 常见问题

### Q1：`gui_lv_init()` 后没有显示内容

- 检查是否已注册开机场景
- 检查 `gui_lv_set_boot_scene_id()` 的 ID 是否有效
- 检查显示端口刷新与 `lv_timer_handler()` 调度

### Q2：`gui_lv_scene_page_push()` 没反应

- 检查 `gui_lv_scene_page_register()` 是否已调用
- 检查 page ID 是否越界
- 检查页面池是否耗尽（默认最多 8 个节点）

### Q3：模板文件编译报错（`<name>` / `%Instance%` 未替换）

- `template` 目录文件是模板，不应直接原样编译
- 非 RTE 模式请先复制并替换占位符

### Q4：RTE 模式下 scene 未自动初始化

- 检查是否启用了对应 `Scene` 实例组件
- 检查是否生成了 `__RTE_GUI_LVGL_SceneN__` 宏

## 12. License

本项目采用 Apache-2.0 License，详见 `LICENSE`。

