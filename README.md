# 外卖运营管理平台（catering-take-out）

基于 SpringBoot + MyBatis + MySQL + Redis + Vue2 的前后端分离餐饮外卖管理系统，面向中小餐饮商家，提供商家端（管理后台）与用户端（小程序 / H5）双端能力，并内置商家 AI 智能客服模块。


***

## 一、项目简介

面向中小餐饮商家的外卖运营平台，分为**管理端（admin）与用户端（user）**：



* **管理端**：员工管理、菜品管理、套餐管理、分类管理、订单管理、店铺营业状态、工作台数据看板、经营报表、AI 智能客服

* **用户端**：菜品 / 套餐浏览、购物车、下单支付、订单状态查询、收货地址管理、店铺状态查询

其中 **AI 智能客服模块为独立开发功能**，对接大模型 API（阿里云百炼 qwen3.6-flash），实现订单咨询问答，并增加超时捕获与限流异常降级处理。

## 二、技术栈



| 分类   | 技术                                            |
| ---- | --------------------------------------------- |
| 后端框架 | Spring Boot 2.7.3、Spring MVC                  |
| 持久层  | MyBatis、MyBatis-Plus 分页（PageHelper）、Druid 连接池 |
| 数据库  | MySQL 8.x            |
| 缓存   | Redis（店铺状态、全局 ID 生成）                          |
| 认证   | JWT（管理端 / 用户端双 Token）                         |
| 接口文档 | Knife4j（Swagger）                              |
| 中间件  | WebSocket（来单实时推送）、Spring Task（定时任务）           |
| 文件存储 | 阿里云 OSS（图片上传）                                 |
| 支付   | 微信支付 V3（下单支付、支付回调、退款）                         |
| 其他   | Lombok、Fastjson、Apache POI（报表导出）              |
| 前端   | Vue2 + ElementUI（Nginx 部署，反向代理后端）             |

## 三、核心功能



1. **Redis 缓存与防护**：店铺营业状态缓存，处理缓存穿透、击穿、雪崩问题；Redis 生成全局唯一 ID

2. **WebSocket 实时推送**：用户下单后向后端管理端实时推送来单提醒，支持语音 / 弹窗提示

3. **定时任务（Spring Task）**：

* `OrderTask`：订单超时未支付自动取消、释放库存

* `WebSocketTask`：定时向客户端推送店铺营业状态等提醒

* `MyTask`：自定义定时任务演示 / 扩展位

1. **AI 智能客服（自研）**：RestTemplate 封装第三方大模型 API（阿里云百炼），实现订单咨询问答，含超时捕获、限流异常降级；前端流式对话页面

2. **JWT 双端认证**：管理端、用户端分别签发 Token，通过拦截器校验身份

3. **菜品 / 套餐管理**：菜品及口味 CRUD、启售停售、分类管理、套餐组合（菜品 + 套餐关联）

4. **订单流转**：用户下单 → 支付回调 → 商家接单 / 拒单 → 派送 / 完成；订单状态机流转（待付款 / 待接单 / 已接单 / 派送中 / 已完成 / 已取消）

5. **经营数据看板**：工作台（今日数据 / 待处理订单 / 营业额）、经营报表（营业额 / 用户 / 订单 / 销量 Top10）、Apache POI 导出 Excel

6. **店铺营业状态**：管理端设置、用户端查询（Redis 缓存，`STORE_STATUS` 键）

## 四、项目结构



```
catering-take-out                       # Maven 父工程（com.catering:catering-take-out）
├── catering-common                     # 公共模块：工具类、全局异常、通用返回、常量
│   └── src/main/java/com/catering/
│       ├── constant/                   # 常量（JWT Claims、消息、状态、公共字段自动填充等）
│       ├── context/                    # 线程级上下文（当前登录用户 ID）
│       ├── enumeration/                # 枚举（数据库操作类型）
│       ├── exception/                  # 业务异常体系（Base + 各业务异常）
│       ├── json/                       # Jackson 序列化配置（Long→String）
│       ├── properties/                 # 配置属性类（JWT、微信、OSS）
│       ├── result/                     # 统一返回结果（Result、PageResult）
│       └── utils/                      # 工具类（JWT、HTTP、微信支付、OSS）
├── catering-pojo                       # 实体模块
│   └── src/main/java/com/catering/
│       ├── dto/                        # 入参对象（Staff/Product/Combo/Order/Cart/Customer 等）
│       ├── entity/                     # 数据库实体
│       └── vo/                         # 出参对象（含报表 VO、看板 VO）
└── catering-server                     # SpringBoot 主模块（端口 8080）
    └── src/main/
        ├── java/com/catering/
        │   ├── CateringApplication.java    # 启动类
        │   ├── annotation/ + aspect/       # @AutoSet 公共字段自动填充（AOP）
        │   ├── config/                     # Redis/WebSocket/OSS/WebMvc 配置
        │   ├── controller/
        │   │   ├── admin/                  # 管理端接口（含 AiChatController 客服）
        │   │   ├── customer/               # 用户端接口
        │   │   └── nofity/                 # 支付回调通知
        │   ├── handler/                    # 全局异常处理
        │   ├── interceptor/                # JWT 拦截器（管理端/用户端）
        │   ├── mapper/                     # MyBatis 数据访问层
        │   ├── service/                    # 业务逻辑层（接口 + impl）
        │   ├── task/                       # 定时任务（订单超时、WebSocket 提醒）
        │   └── websocket/                  # WebSocket 服务端
        └── resources/
            ├── application.yml             # 主配置
            ├── application-dev.yml         # 本地开发配置（数据库/微信/AI 密钥）
            └── mapper/                     # MyBatis 映射文件
```

## 五、数据模型（核心表）



| 表名              | 说明              | 对应实体            |
| --------------- | --------------- | --------------- |
| `employee`      | 员工（管理端账号）       | `Staff`         |
| `category`      | 分类（菜品分类 / 套餐分类） | `Category`      |
| `dish`          | 菜品              | `Product`       |
| `dish_flavor`   | 菜品口味            | `ProductFlavor` |
| `setmeal`       | 套餐              | `Combo`         |
| `setmeal_dish`  | 套餐 - 菜品关联       | `ComboProduct`  |
| `orders`        | 订单              | `Order`         |
| `order_detail`  | 订单明细            | `OrderDetail`   |
| `shopping_cart` | 购物车             | `Cart`          |
| `user`          | 用户（C 端）         | `Customer`      |
| `address_book`  | 收货地址            | `Address`       |

> 表结构如上表所列，库名统一为 `catering_take_out`。

## 六、接口概览

接口路径稳定，前端按此对接：



| 模块    | 管理端（/admin/\*\*）                             | 用户端（/user/\*\*）         |
| ----- | -------------------------------------------- | ----------------------- |
| 分类    | `/admin/category/**`                         | `/user/category/list`   |
| 菜品    | `/admin/dish/**`                             | `/user/dish/**`         |
| 套餐    | `/admin/setmeal/**`                          | `/user/setmeal/**`      |
| 订单    | `/admin/order/**`                            | `/user/order/**`        |
| 购物车   | —                                            | `/user/shoppingCart/**` |
| 地址    | —                                            | `/user/addressBook/**`  |
| 员工    | `/admin/employee/**`                         | —                       |
| 店铺状态  | `/admin/shop/**`                             | `/user/shop/status`     |
| 工作台   | `/admin/workspace/**`                        | —                       |
| 报表    | `/admin/report/**`                           | —                       |
| 通用上传  | `/admin/common/upload`                       | —                       |
| AI 客服 | `/admin/ai`（自研）                              | —                       |
| 支付回调  | `/notify/paySuccess`、`/notify/refundSuccess` | —                       |

接口文档（Knife4j）：启动后访问 `http://localhost:8080/doc.html`

## 七、命名约定

项目采用统一的餐饮主题命名体系：

* **包结构**：`com.catering`（common / pojo / server 三个模块）
* **核心实体**：员工 `Staff`、菜品 `Product`、口味 `ProductFlavor`、套餐 `Combo`、套餐-菜品关联 `ComboProduct`、订单 `Order`、订单明细 `OrderDetail`、购物车 `Cart`、用户 `Customer`、收货地址 `Address`、分类 `Category`
* **接口 / 实现分离**：`XxxService`（接口）+ `XxxServiceImpl`（实现）
* **数据访问层**：`XxxMapper`（Java 接口 + `mapper/*.xml` 映射）
* **自有 / 通用命名**：`AiChatController`（AI 客服）、`PayNotifyController`（支付回调）、`OrderTask`/`WebSocketTask`（定时任务）、`WorkspaceService`（工作台）

## 八、启动方式

### 环境要求



* JDK 8+（项目编译目标为 JDK 8 兼容，当前已验证 JDK 21 可编译）

* Maven 3.6+

* MySQL 8.x、Redis

### 步骤



1. **建库**：创建数据库 `catering_take_out`，导入项目 SQL 脚本（本机库名不一致时在配置中调整即可）

2. **改配置**：编辑 `catering-server/src/main/resources/application-dev.yml`，填入本机数据库账号密码、Redis、微信支付、OSS 与 AI 密钥（默认走 `dev` profile）

3. **启动**：运行 `catering-server` 模块的 `CateringApplication` 主类，端口 `8080`

4. **前端**：Vue2 前端构建产物由 Nginx 部署（`nginx/html/sky`），Nginx 反向代理 `/api` 至后端

## 九、配置说明（application.yml/application-dev.yml）



| 配置项                        | 说明                                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------------------- |
| `catering.jwt.*`           | 管理端 / 用户端 JWT 密钥、有效期、Token 名                                                                             |
| `catering.file.upload-dir` | 本地图片上传目录（`D:/waimaixiangmu/waimaixiangmu/catering-take-out/catering-server/src/main/resources/uploads/`） |
| `catering.alioss.*`        | 阿里云 OSS 配置（上传图片至 OSS）                                                                                    |
| `catering.wechat.*`        | 微信支付 V3 商户配置（mchid、APIv3 密钥、证书路径、回调地址）                                                                   |
| `catering.shop.address`    | 店铺地址展示配置                                                                                                 |
| `ai.*`                     | AI 智能客服：阿里云百炼 `base-url`、`api-key`、`model-name`（qwen3.6-flash）                                           |

> ⚠️ 
>
> `application-dev.yml`
>
>  中含微信支付密钥、AI API Key 等敏感信息，请勿提交到公开仓库。

## 十、注意事项



* **AI 客服**：`application.yml` 顶层 `ai.api-key` 为空，需在 `application-dev.yml` 中配置真实 Key 后客服功能方可使用

* **支付回调**：回调地址为 cpolar 内网穿透地址，本地调试时可自行替换

* **图片上传**：本地目录与 OSS 二选一，OSS 密钥为占位值（`your-*`），需填入真实凭证

* **数据库**：库名已统一为 `catering_take_out`，与代码配置保持一致
