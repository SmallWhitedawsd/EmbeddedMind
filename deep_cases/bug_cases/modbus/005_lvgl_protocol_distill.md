# 005: LVGL协议知识蒸馏

## 项目
EmbeddedMind (知识库项目)

## 环境
LVGL v8.x, 嵌入式GUI

## 现象
需要系统化理解LVGL协议/接口以避免常见错误

## 影响范围
所有LVGL相关开发

## 排查过程

### 假设1: LVGL初始化顺序
- 验证方法：分析lv_init→lv_port_indev_init→lv_port_disp_init
- 结果：顺序错误会导致显示异常

### 假设2: 显示缓冲区配置
- 验证方法：检查disp_drv配置
- 结果：buffer大小影响刷新率和内存占用

### 假设3: 输入设备回调
- 验证方法：检查indev_drv.read_cb
- 结果：回调函数需正确返回数据

## 根因
LVGL有严格的初始化顺序和接口规范

## 修改方案
```c
// 正确初始化顺序
lv_init();
lv_port_disp_init();  // 先初始化显示
lv_port_indev_init(); // 再初始化输入

// 显示驱动配置
static lv_disp_draw_buf_t draw_buf;
static lv_color_t buf1[128*10]; // 部分刷新
lv_disp_draw_buf_init(&draw_buf, buf1, NULL, 128*10);

// 输入设备回调
static void touchpad_read(lv_indev_drv_t *drv, lv_indev_data_t *data) {
    data->point.x = current_x;
    data->point.y = current_y;
    data->state = pressed ? LV_INDEV_STATE_PR : LV_INDEV_STATE_REL;
}
```

## 验证结果
LVGL正确初始化，显示和触摸正常

## 预防措施
- 严格遵循初始化顺序
- 显示缓冲区至少1/10屏幕大小
- 输入回调需实时返回

## 经验规则
- lv_task_handler()需定期调用(通常5ms)
- LVGL非线程安全，任务中需加锁
- flush_cb中等待DMA完成再返回

## 来源
ses_05eaf1df5ffebGEnPBvDZovqJ4 - 2026-07-25
