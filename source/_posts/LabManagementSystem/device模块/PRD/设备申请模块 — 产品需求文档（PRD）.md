---
title: "设备申请模块 — 产品需求文档（PRD）"
date: 2026-10-28 11:11:07
categories:
  - ProjectPRD
---

# 设备申请模块 — 产品需求文档（PRD）

|项|内容|
|---|---|
|文档版本|v1\.0|
|创建日期|2026\-07\-09|
|所属系统|LabManagementSystem（实验室预约系统）|
|关联模块|实验室预约（已实现）、用户管理、消息通知|
|文档状态|待评审|

## 修订记录

|版本|日期|作者|说明|
|---|---|---|---|
|v1\.0|2026\-07\-09|—<br>|初稿：基于现有 UI 与数据模型差距分析|



## 1\. 背景与目标

### 1\.1 背景

实验室预约系统已支持机房时段预约的全流程。设备申请页面（`/equipment`）已完成视觉与交互设计，师生可浏览设备并填写申请表单，但数据仅存于浏览器内存，管理员无法在后台审核，刷新页面后申请记录丢失。

后端已具备 `devices`（设备台账）与 `device_bookings`（借用申请）数据表及种子数据，具备快速打通的技术基础。

### 1\.2 产品目标

1. 师生可浏览**真实设备库存**并提交借用申请，数据持久化至数据库。

2. 管理员可在后台查看、搜索、**通过/拒绝**设备申请。

3. 师生可在「我的申请」查看审批状态、拒绝原因，并可**撤回待审申请**。

4. 与实验室预约保持一致的交互范式与状态定义，降低学习成本。

### 1\.3 成功指标（P0）

|指标|目标|
|---|---|
|申请提交成功率|合法表单提交后 100% 落库|
|审核闭环|管理员操作后用户端列表状态实时一致（刷新后）|
|数据一致性|同一设备待审\+已通过数量不超过 `available_quantity`（P0 可先校验，P1 自动扣减）|
|页面无 Mock 依赖|设备列表与我的申请均来自 API|



## 2\. 用户角色

|**角色**|**说明**|**核心诉求**|
|---|---|---|
|学生|主要申请人|快速找到可借设备、提交申请、跟踪进度|
|教师|可申请设备或作为指导教师|为学生项目借仪器/电脑|
|实验室管理员|审核与设备台账维护|控制出借、查看申请理由、拒绝不合规申请|
|超级管理员|系统配置|设备主数据维护（P1）|



## 3\. 用户场景（User Stories）

### 3\.1 学生 — 借用笔记本电脑

> 作为计算机专业学生，我需要在毕设期间借用一台高性能笔记本一周，以便在实验室外完成模型训练。

**路径：** 首页 → 设备申请 → 筛选「笔记本电脑」→ 选择可用设备 → 填写借用日期与事由 → 提交 → 在「我的申请」查看「待审批」。

### 3\.2 教师 — 拒绝后重新申请

> 作为指导教师，我帮学生提交仪器借用申请，因事由描述不清被管理员拒绝，我需要看到拒绝原因并修改后重新提交。

**路径：** 我的申请 → 查看详情（拒绝原因）→ 重新选择设备 → 再次提交。

### 3\.3 管理员 — 审核设备申请

> 作为实验室管理员，我需要在后台集中处理待审的设备申请，对合理申请点击通过，对资料不全的填写拒绝原因。

**路径：** 后台管理 → 设备借用 → 筛选待审 → 通过 / 拒绝 → 列表状态更新。

### 3\.4 学生 — 撤回误操作申请

> 作为学生，我误提交了错误的借用日期，在管理员审批前希望撤回申请。

**路径：** 我的申请 → 待审批记录 → 撤回申请 → 状态变为「已取消」。



## 4\. 业务流程

### 4\.1 主流程

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=OTUzNGIzMTc5ZWQ4ZmVmNTZlOTU4OGE0MjhjZmNkZDFfZmZkZWQzYjNmZDIwNTUyY2Y4NTJhOTcxNGI1YWI1YTBfSUQ6NzY2MDQyOTA5NjM1NDEwNjMwOF8xNzkzMTU3MDY2OjE3OTMyNDM0NjZfVjM)

### 4\.2 状态机

与实验室预约保持一致：

|**status**|**名称**<br>|**用户端文案**|**可执行操作（用户）**|**可执行操作（管理员）**|
|---|---|---|---|---|
|0|待审核|待审批|查看详情、撤回|通过、拒绝|
|1|已通过|已通过|查看详情|查看详情|
|2|已拒绝|已拒绝|查看详情（含拒绝原因）|查看详情<br>|
|3|已取消|已取消|查看详情|查看详情|

**状态流转规则：**

- `0 → 1`：管理员通过

- `0 → 2`：管理员拒绝（必填 `reject_reason`）

- `0 → 3`：用户撤回

- `1/2/3` 不可再变为其他状态（P0 不支持「重新审核」）



## 5\. 功能需求

### 5\.1 用户端 — 设备申请 Tab

**页面：** `frontend/src/views/EquipmentView.vue`（`/equipment`）

|**编号**|**功能**|**描述**|**优先级**|
|---|---|---|---|
|F\-U\-01|设备列表加载<br>|进入页面调用 `GET /api/devices`，替换硬编码 `deviceListByCategory`|P0|
|F\-U\-02|类别筛选<br>|侧栏类别与后端 `category` 映射：笔记本→0、手机→1、智能仪器→2、软件→3|P0|
|F\-U\-03|关键词搜索|按设备名称、序列号、型号、描述模糊搜索（前端过滤或后端 `q` 参数）|P0|
|F\-U\-04|品牌筛选|基于 `manufacturer` 字段筛选；无厂商时归「其他」|P1|
|F\-U\-05|可借状态展示|`available_quantity > 0` 且 `status==0` 显示「选择」，否则「已借出」并禁用|P0|
|F\-U\-06|分页|列表分页，默认每页 20 条，与现 UI 一致|P0|
|F\-U\-07|打开申请弹窗|点击「选择」打开 `EquipmentApplyModal`，带入设备信息|P0|
|F\-U\-08|申请表单校验|必填：电话、负责教师、借用起止日期、事由；结束日期 ≥ 开始日期|P0|
|F\-U\-09|提交申请|调用 `POST /api/device-bookings`，成功后展示成功弹窗|P0|
|F\-U\-10|提交失败提示|接口错误时 `alert` 或 Toast 展示后端 `detail`|P0|

### 5\.2 用户端 — 我的申请 Tab

|**编号**|**功能**|**描述**|**优先级**|
|---|---|---|---|
|F\-U\-11|申请列表|`GET /api/device-bookings`，替换 Mock 数据|P0|
|F\-U\-12|状态筛选|全部 / 待审批 / 已通过 / 已拒绝|P0|
|F\-U\-13|日期筛选<br>|按申请日期（`created_at`）区间过滤|P0|
|F\-U\-14|查看详情|打开 `EquipmentApplicationDetailModal`，展示完整字段含拒绝原因|P0|
|F\-U\-15|撤回申请|待审状态可撤回，调用 `PATCH /api/device-bookings/{id}/cancel`|P0|
|F\-U\-16|刷新|重新请求列表 API（不仅重置筛选）|P0|
|F\-U\-17|按用户过滤|登录体系接入后，仅展示当前用户申请|P1|

### 5\.3 管理端 — 设备借用

**页面：** 新建 `AdminDeviceBookingsView.vue`，替换 `AdminSectionView` 占位。

|**编号**|**功能**|**描述**|**优先级**|
|---|---|---|---|
|F\-A\-01|申请列表|`GET /api/device-bookings`，表格展示|P0|
|F\-A\-02|搜索|支持设备名、申请人、学号、事由关键字 `q`|P0|
|F\-A\-03|日期筛选|按借用开始日期或申请日期|P0|
|F\-A\-04|分页|与 `AdminAppointmentsView` 一致|P0|
|F\-A\-05|通过|`PATCH .../review`，`action=approve`，可填管理员备注|P0|
|F\-A\-06|拒绝|`action=reject`，必填拒绝原因|P0|
|F\-A\-07|查看详情|弹窗展示申请完整信息|P0|
|F\-A\-08|状态标签|待审/已通过/已拒绝/已取消配色与预约模块一致|P0|

### 5\.4 管理端 — 设备管理（P1）

**页面：** `/admin/devices`

|**编号**|**功能**|**描述**|**优先级**|
|---|---|---|---|
|F\-A\-09|设备 CRUD|增删改查 `devices` 表记录|P1|
|F\-A\-10|库存维护|修改 `total_quantity`、`available_quantity`|P1|
|F\-A\-11|状态维护|正常/维修中/报废|P1|

---

## 6\. 数据需求

### 6\.1 设备表 `devices`（已有，列表 API 只读）

|**字段**|**类型**|**说明**|
|---|---|---|
|id|int|主键|
|name|string|设备名称|
|serial\_number|string|序列号/编号（对应 UI `code`）|
|category|smallint|0电脑 1手机 2实验器材 3软件|
|manufacturer|string|制造商/品牌|
|model|string|型号|
|description|text|描述（对应 UI `spec` 摘要）|
|location|string|存放位置|
|department|string|所属部门|
|total\_quantity|int|总数量|
|available\_quantity|int|可借数量|
|status|smallint|0正常 1维修中 2报废|
|manager|string|设备管理员|
|icon|string|图标 URL|

### 6\.2 设备申请表 `device_bookings`（需扩展）

**已有字段：** `device_id`, `quantity`, `start_date`, `end_date`, `reason`, `special_requirement`, `user_id`, `user_name`, `openid`, `status`, `created_at`

**P0 建议新增字段：**

|字段|类型|说明|
|---|---|---|
|device\_name|string|冗余设备名|
|serial\_number|string|冗余设备编号|
|student\_id|string|学号/工号|
|class\_name|string|班级|
|major|string|专业|
|phone|string|联系电话|
|teacher\_id|string|指导教师 ID|
|teacher\_name|string|指导教师姓名|
|admin\_remark|text|管理员备注（用户不可见）|
|reject\_reason|text|拒绝原因（用户可见）|

### 6\.3 类别映射

|UI 类别 key|UI 文案|DB category|
|---|---|---|
|laptop|笔记本电脑|0|
|phone|手机|1|
|instrument|智能仪器|2|
|software|软件|3|

---

## 7\. 接口需求（REST API）

统一前缀：`/api`，响应格式与 `appointments` 模块保持一致。

### 7\.1 设备

|**方法**|**路径**|**说明**|
|---|---|---|
|GET|`/api/devices`<br>|设备列表。Query：`category`、`q`、`page`、`page_size`。仅返回 `status=0` 且可用于展示的记录|
|GET|`/api/devices/{id}`|单设备详情|

**列表项响应示例：**

```JSON
{
  "data": [
    {
      "id": 1,
      "name": "示波器 DS1054Z",
      "code": "DS1054Z-001",
      "spec": "四通道数字示波器",
      "category": 2,
      "manufacturer": "RIGOL",
      "model": "DS1054Z",
      "location": "12 号楼 301",
      "available_quantity": 4,
      "total_quantity": 5,
      "status": "available",
      "icon": ""
    }
  ],
  "pagination": { "page": 1, "page_size": 20, "total": 2, "total_pages": 1 }
}
```

> `status` 对外暴露为字符串：`available`（可借）/ `unavailable`（不可借）/ `maintenance` / `retired`，由 `devices.status` \+ `available_quantity` 计算。

### 7\.2 设备申请

|**方法**|**路径**|**说明**|
|---|---|---|
|GET|`/api/device-bookings`<br>|申请列表。Query：`q`、`status`、`date_from`、`date_to`、`page`、`page_size`|
|POST|`/api/device-bookings`|创建申请|
|PATCH|`/api/device-bookings/{id}/review`|管理员审核：\`action=approve|
|PATCH|`/api/device-bookings/{id}/cancel`|用户撤回（仅 `status=0`）|

**POST 请求体：**

```JSON
{
  "deviceId": 1,
  "quantity": 1,
  "studentId": "2024001001",
  "name": "张三",
  "className": "计科 2201",
  "major": "计算机科学与技术",
  "phone": "13800138000",
  "teacherId": "t1",
  "teacherName": "张明 教授",
  "borrowStart": "2026-07-10",
  "borrowEnd": "2026-07-17",
  "reason": "课程设计：嵌入式系统调试"
}
```

**POST 响应：**

```JSON
{ "ok": true, "id": 42 }
```

### 7\.3 业务校验（后端）

|规则|说明|
|---|---|
|V\-01|`deviceId` 必须存在且 `status==0`|
|V\-02|`available_quantity >= quantity`（P0 仅校验，不扣减）|
|V\-03|`borrowEnd >= borrowStart`|
|V\-04|`reason`、`phone` 非空|
|V\-05|拒绝时 `rejectReason` 必填|
|V\-06|仅 `status=0` 可撤回|
|V\-07|同一用户同一设备同一日期区间不可有重复待审申请（P1）|

## 8\. 交互与 UI 规范

### 8\.1 保持与现有 UI 一致

- 视觉：沿用 `EquipmentView` 已实现的 Rice 色系、字体层级（`font-headline` / `font-body`）

- 弹窗：复用 `EquipmentApplyModal`、`EquipmentApplicationDetailModal`、`BookAppointmentSuccessModal`

- 管理端：表格、筛选栏、审核按钮布局对齐 `AdminAppointmentsView`

### 8\.2 需改造的 UI 点

|位置|改造|
|---|---|
|`EquipmentView`|删除 `deviceListByCategory` 硬编码；`onEquipmentApplySubmit` 改调 API|
|`EquipmentView`|`refreshMyApplications` 改为重新 `GET` 列表|
|`EquipmentView`|「撤回申请」绑定 `cancel` API|
|`frontend/src/api/index.js`|新增设备相关 API 方法|
|`admin/router`|`device-bookings` 路由指向新页面|

### 8\.3 空态与错误态

|场景|展示|
|---|---|
|无设备|「暂无符合条件的设备，试试调整筛选或搜索关键词」|
|无申请记录|「暂无符合条件的申请记录」|
|API 失败|列表区展示错误提示 \+ 重试按钮|
|提交成功|成功弹窗 → 可跳转「我的申请」|

## 9\. 权限与安全

|项|P0|P1|
|---|---|---|
|API 认证|无（与预约模块一致，全开放）|JWT / Session|
|我的申请数据隔离|返回全量（与预约一致）|按 `user_id` / `openid` 过滤|
|审核接口|无角色校验|仅管理员可调用 `review`|
|输入消毒|字符串 trim、长度限制|防 XSS、SQL 注入（ORM 已规避）|

## 10\. 非功能需求

|类别|要求|
|---|---|
|性能|设备列表 \< 500ms（100 条以内）；申请提交 \< 1s|
|兼容性|与现有 Vue 3 前端、FastAPI 后端、MySQL 8\.0 一致|
|可维护性|API 路由文件 `device_bookings.py`、`devices.py`；复用 `appointments` 分页/审核模式|
|日志|审核、撤回操作记录到应用日志（P1 可写审计表）|
|部署|无新增中间件；迁移通过 `init_db` 补列或 Alembic（若后续引入）|

## 11\. 消息通知（P1 预留\-暂时不用做）

审核完成后写入 `messages` 表：

|字段|值|
|---|---|
|type|`device_booking`|
|receiver\_openid|申请人 openid|
|appointment\_id|可复用字段存 `device_booking.id` 或新增 `device_booking_id`|
|title|「设备申请已通过」/ 「设备申请未通过」|
|content|含设备名、借用区间、拒绝原因|

## 12\. 实施计划

### 12\.1 里程碑

|**阶段**|**交付物**|**预估**|
|---|---|---|
|**M1 后端**|模型补列、`devices` \+ `device-bookings` API、种子数据补充|2–3 人日|
|**M2 用户端**|API 对接、去掉 Mock、撤回功能|2 人日|
|**M3 管理端**|`AdminDeviceBookingsView` 审核页|1–2 人日|
|**M4 联调验收**|端到端测试、文档更新|1 人日|

### 12\.2 依赖项

|依赖|说明|
|---|---|
|MySQL 运行|Docker 或本地实例|
|实验室预约模块|参照实现模式，无强耦合|
|用户认证|非 P0 阻塞项|

## 13\. 验收标准（P0）

|编号|验收项|通过条件|
|---|---|---|
|AC\-01|设备列表|`/equipment` 展示数据库中的设备，非硬编码|
|AC\-02|提交申请|填写合法表单提交后，`device_bookings` 表新增记录，`status=0`|
|AC\-03|我的申请|提交后「我的申请」可见该记录；刷新页面仍在|
|AC\-04|管理端列表|`/admin/device-bookings` 可见待审记录|
|AC\-05|审核通过|管理员通过后，用户端状态变为「已通过」|
|AC\-06|审核拒绝|拒绝后用户端显示拒绝原因|
|AC\-07|撤回|待审状态可撤回，变为「已取消」|
|AC\-08|不可借设备|`available_quantity=0` 或维修/报废设备不可提交|
|AC\-09|错误处理|非法设备 ID、缺少必填项返回 4xx 及明确错误信息|

## 14\. 风险与对策

|风险|影响|对策|
|---|---|---|
|模型字段不足|前后端对不齐|PRD 已定义扩展字段，开发前先迁移|
|无认证导致数据串读|用户看到他人申请|P0 接受与预约一致；P1 同步上线认证|
|库存超借|多人同时申请最后一台|P0 提交时校验数量；P1 事务扣减|
|UI 类别与 DB 不一致|筛选错误|统一映射表，API 支持 `category` 参数|
|教师列表假数据|指导教师信息不可信|P1 从 `users` 表 `user_type=1` 拉取|

## 15\. 附录

### 15\.1 相关代码文件

|模块|路径|
|---|---|
|用户端页面|`frontend/src/views/EquipmentView.vue`|
|申请弹窗|`frontend/src/components/EquipmentApplyModal.vue`|
|详情弹窗|`frontend/src/components/EquipmentApplicationDetailModal.vue`|
|前端 API|`frontend/src/api/index.js`|
|设备模型|`backend/app/models/device.py`|
|申请模型|`backend/app/models/device_booking.py`|
|参照实现|`backend/app/api/routes/appointments.py`|
|管理端参照|`admin/src/views/AdminAppointmentsView.vue`|
|系统实现方案|`docs/implementation-plan.md`|



