# 校园跑腿服务小程序 (Campus Errand Mini-Program)

## 📖 项目简介
本项目是一个基于 C2C 模式的校园互助跑腿平台。致力于解决校园内最后一公里的配送与代办需求（如代拿快递、代买零食等）。项目实现了从用户身份核验、任务发布、接单大厅流转到上传凭证结案的完整业务闭环。

---

## 🛠️ 技术栈与核心组件 (Technology Stack)

### 前端生态 (Front-End)
* **核心框架**：uni-app / Vue.js (实现一次开发，多端发布，当前主打微信小程序端)
* **UI 组件**：原生小程序组件结合自定义 Flex 布局，实现响应式自适应。
* **状态管理**：利用 `uni.setStorageSync` 实现用户鉴权状态与本地数据的轻量级持久化。
* **核心 API**：
    * `wx.request` / `uni.request`：封装 HTTP 异步请求，与 Spring Boot 后端交互。
    * `uni.uploadFile`：实现本地图片转二进制流的表单上传。
    * `uni.showModal` / `uni.showToast`：全局状态拦截与用户交互提示。

### 后端架构 (Back-End)
* **核心框架**：Spring Boot 
* **持久层框架**：MyBatis
* **数据库**：MySQL 8.0 (InnoDB 引擎)
* **云原生服务**：阿里云 OSS (Object Storage Service) 用于海量图片的高可用存储。
* **连接池**：HikariCP (Spring Boot 默认，提供高性能数据库连接)

---

## 🔌 核心 API 接口设计 (Core APIs)

系统遵循 RESTful 风格与扁平化设计原则，主要接口包括：

1.  **`/api/sms/send` (POST)**：仿真短信网关接口。接收手机号，在后端生成并打印高强度动态随机码，模拟真实短信下发链路。
2.  **`/api/user/login` (POST)**：鉴权与注册一体化接口。校验手机号与验证码，自动完成新用户落库（`users` 表）与旧用户数据查询，返回 `code: 200` 及用户态实体。
3.  **`/getOrders` (GET)**：接单大厅数据流。返回 `status = 0` (待接单) 的全局任务列表。
4.  **`/getMyPublished` (GET)**：个人任务查询。基于 `studentId` 进行严格的数据权限隔离，返回当前用户的专属订单履历。
5.  **`/completeOrder` (POST)**：订单生命周期终结接口。接收订单 ID 与阿里云 OSS 图片 URL，更新 `status` 并记录精确的 `finish_time`，实现闭环。

---

## ✨ 项目亮点 (Project Highlights)

1.  **基于阿里云 OSS 的分布式存储架构**
    弃用传统的本地硬盘存储图片方式，引入阿里云 OSS。实现了前端直传（或后端代理上传），极大减轻了应用服务器的带宽压力，并保证了“送达凭证”的高可用性与防数据丢失。
2.  **完善的数据隔离与防越权设计**
    在数据库设计上，舍弃自增 ID 作为主要业务关联，采用 `student_id` (学号) 字符串作为强关联键。在查询逻辑中严格校验用户身份，确保多租户场景下的数据绝对安全（A同学绝对无法查阅B同学的私人订单）。
3.  **仿真验证码网关链路**
    在零成本的前提下，利用后端日志打印与前端弹窗联动，完美模拟了完整的企业级短信验证码鉴权链路，具备极高的可演示性与扩展性。
4.  **全生命周期双时间轴追踪**
    订单实体整合了 `@JsonFormat` 配合 `LocalDateTime`，精确记录 `create_time` (发单时刻) 与 `finish_time` (结案时刻)，为后续平台的“履约时效分析”提供了底层数据支撑。

---

## 🎯 解决的核心技术难题 (Problem Solved)

1.  **数据库架构与实体类类型阻抗失配 (Type Mismatch)**
    * **问题**：早期业务中，订单关联 ID 被定义为 `Integer`，但在接入学号（如 `STU853266`）时引发了 `SQLSyntaxErrorException` 与插入崩溃。
    * **解决**：通过 `ALTER TABLE` 热修改数据库 Schema，将关键外键改造为 `VARCHAR`，并同步重构了 Java 实体类与 MyBatis XML 映射文件，打通了字符型主键的流转。
2.  **前后端生命周期异步渲染的空指针防御**
    * **问题**：前端在请求订单详情时，由于网络延迟，页面渲染先于数据到达，导致 `Cannot read property 'createTime' of null` 造成白屏死机。
    * **解决**：在 Vue 模板中引入防御性编程，使用 `v-if="item"` 进行节点挂载拦截，并采用短路运算符 `{{ item.createTime || '未知' }}` 提供降级显示，保证了极端的弱网环境下程序的鲁棒性。
3.  **微服务端口冲突与环境脏数据清理**
    * **问题**：多开项目导致 8080 端口占用（`Web server failed to start. Port 8080 was already in use`）及旧项目数据库连接池配置失效。
    * **解决**：实施纯净项目迁移策略。废弃旧工程，在全新 `demo` 工程中重写 `application.properties`，精确定位 MySQL 时区参数及数据库名（`school_run`），一举解决驱动器加载失败的顽疾。