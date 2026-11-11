# 苍穹外卖 Java 后端学习路线

用一个能跑起来的外卖项目练习 Spring MVC、MyBatis、Redis 和订单业务。上游基线是 [shuhongfan/sky-take-out 的 267313f](https://github.com/shuhongfan/sky-take-out/tree/267313f948bba2001fbc497995a39ef61f2c621f)。学习版把外部服务凭据改成环境变量，并从 day12 课程资料补齐了后端报表模板。

前置基础是 Java 集合、异常和注解，SQL 增删改查，以及 HTTP 请求和 JSON。建议用两周，每周四次，每次 60–90 分钟。每次先操作一个功能，再沿代码下断点，最后用 HTTP 返回值、数据库记录或 Redis 键验证自己的解释。

## 在 Windows 本地启动

准备 JDK 17、Maven、MySQL 8 和 Redis。以下命令从仓库根目录执行。课程 SQL 包含删表语句，导入到专用 MySQL 实例或可重建的学习库。

```powershell
mysql --default-character-set=utf8mb4 -h 127.0.0.1 -u root -p
```

在 MySQL 客户端执行 `source <sky.sql 的绝对路径>`。文件位于 [资料/day01/数据库/sky.sql](资料/day01/数据库/sky.sql)，会创建 `sky_take_out` 库。Redis 启动后可用 `redis-cli PING` 检查。

```powershell
$env:SKY_DB_USER = 'root'
$env:SKY_DB_PASSWORD = Read-Host '本地 MySQL 密码' -MaskInput
mvn -f sky-take-out/pom.xml clean package
java -jar sky-take-out/sky-server/target/sky-server-1.0-SNAPSHOT.jar --server.address=127.0.0.1
```

默认连接 `127.0.0.1:3306` 和 `127.0.0.1:6379`。端口不同可在启动 Java 的终端设置 `SKY_DB_PORT` 和 `SKY_REDIS_PORT`，Redis 有密码时设置 `SKY_REDIS_PASSWORD`。Windows PowerShell 5.1 用户可用 IntelliJ 的本地运行配置传入数据库密码。

打开 [Knife4j 接口文档](http://127.0.0.1:8080/doc.html)。管理端使用仓库附带的 `nginx-1.20.2` 静态页面。复制该目录到纯英文路径，进入副本，创建 `temp` 目录，在 `conf/nginx.conf` 中把 `listen 80` 改为 `listen 127.0.0.1:80`，并将管理端代理的 `localhost:8080` 改为 `127.0.0.1:8080`。附带管理端的 WebSocket 指向 `localhost` 的 80 端口。然后执行：

```powershell
.\nginx.exe -t
if ($LASTEXITCODE -eq 0) { Start-Process .\nginx.exe -WindowStyle Hidden }
```

访问 [管理端](http://localhost/)，课程账号为 `admin / 123456`。从管理端切换营业状态，再打开 [用户端营业状态接口](http://localhost/user/shop/status)，应返回 `code=1` 和对应状态。停止 Nginx 时，在同一目录执行 `.\nginx.exe -s quit`。

小程序在微信开发者工具中导入 `mp-weixin`，使用自己的 AppID，并给后端设置 `SKY_WECHAT_APP_ID` 和 `SKY_WECHAT_SECRET`。本地模拟器的 API 地址见 `mp-weixin/common/vendor.js` 的 `baseUrl`。提交真实订单还需要百度地图的 `SKY_BAIDU_AK`，微信支付需要商户配置、证书和可达的回调地址。图片上传依赖自己的阿里云 OSS。其余变量见 [application-dev.yml](sky-take-out/sky-server/src/main/resources/application-dev.yml)。

## 第一周

下面的 Java 文件路径以 `sky-take-out/sky-server/src/main/java/com/sky/` 为起点。MyBatis XML 在同模块的 `src/main/resources/mapper/`，DTO、实体、VO 在 `sky-pojo` 模块。

### 01 · 启动与管理端登录（大纲）

**问题**　浏览器提交密码后，后端如何查询员工并签发令牌？

主链　Nginx `POST /api/employee/login` → `/admin/employee/login` → 校验员工 → 返回 JWT → 后续请求经拦截器。

1. `SkyApplication.java` 和 `application.yml`，看应用入口和数据源配置，找到 Knife4j。
2. `controller/admin/EmployeeController.java` 的 `login` 接收 `EmployeeLoginDTO` 并调用 Service。到 `service/impl/EmployeeServiceImpl.java` 的 `login` 看员工不存在、密码错误、账号停用三个分支。
3. `mapper/EmployeeMapper.java` 和 `EmployeeMapper.xml` 的 `getByUsername` 查表。返回 Controller，看 `JwtUtil.createJWT` 和 `EmployeeLoginVO`。
4. `config/WebMvcConfiguration.java` 注册拦截器。`interceptor/JwtTokenAdminInterceptor.java` 的 `preHandle` 从 `token` 请求头取令牌并保存员工 ID。

验证　用正确和错误的密码各发一次请求，再用合法令牌请求员工分页。小程序登录留到第 04 课。

### 02 · 菜品、口味与数据库事务（大纲）

**问题**　新增一道带口味的菜，为什么要写两张表？

主链　管理端新增菜品 → DTO → 保存菜品 → 取自增 ID → 批量写入口味 → 事务提交。

1. `controller/admin/DishController.java` 的 `save` 接收 `DishDTO`，交给 `DishServiceImpl.saveWithFlavor`。
2. `service/impl/DishServiceImpl.java` 的 `saveWithFlavor` 将 DTO 转成菜品实体，写入菜品后为口味填充 `dishId`。
3. `DishMapper.xml` 的 `insert` 回填主键，`DishFlavorMapper.xml` 的 `insertBatch` 使用该主键写入明细。结合 `aspect/AutoFillAspect.java` 看创建人和创建时间来自哪里。

验证　新增一道带两种口味的练习菜，在 `dish` 与 `dish_flavor` 查关联行。缓存失效留到第 03 课。

### 03 · 营业状态与菜单缓存（大纲）

**问题**　商家修改营业状态或菜品后，用户如何读到新值？

主链　管理端写营业状态 → Redis → 用户端读状态。用户查菜单 → 查缓存 → 未命中查 MySQL → 写缓存。

1. `controller/admin/ShopController.java` 的 `setStatus` 写 Redis，同名用户端 Controller 的 `getStatus` 读该键。
2. `controller/user/DishController.java` 的 `list` 按分类构造缓存键，未命中交给 `DishServiceImpl.listWithFlavor`。看管理端 `DishController` 如何在修改后删除菜品缓存。
3. 对照 `controller/user/SetmealController.java` 的 `list` 和 `controller/admin/SetmealController.java` 的写接口，比较 `@Cacheable`、`@CacheEvict` 与显式调用 Redis 的做法。

验证　切换营业状态，比较两端返回值。用同一分类请求菜单两次，查看 SQL 日志，再修改一项菜品观察缓存变化。用户鉴权进入第 04 课。

### 04 · 微信登录与购物车（大纲）

**问题**　后端如何识别当前顾客，同一菜品的两种口味如何计数？

主链　微信登录码 → 换 openid → 查询或创建用户 → JWT → 购物车按当前用户读写。

1. `controller/user/UserController.java` 的 `login` 接收登录码，`service/impl/UserServiceImpl.java` 的 `wxLogin`、`getOpenid` 换取 openid 并落库，Controller 签发用户 JWT。
2. `interceptor/JwtTokenUserInterceptor.java` 的 `preHandle` 从 `authentication` 头读取用户令牌并设置用户 ID。
3. `controller/user/ShoppingCartController.java` 的 `add` 调用 `service/impl/ShoppingCartServiceImpl.java` 的 `addShoppingCart`，沿 `ShoppingCartMapper.xml` 的 `list`、`updateNumberById`、`insert` 区分增量和新行。

验证　有微信联调配置时，比较连续添加同一口味和切换口味后的购物车行数。下单后的清理留到第 05 课。

## 第二周

### 05 · 下单、订单快照与事务边界（大纲）

**问题**　用户点提交后，订单与明细保存了哪些信息，购物车何时清空？

主链　`POST /user/order/submit` → 查地址和配送范围 → 读购物车 → 写订单和明细 → 清空购物车。

1. `controller/user/OrderController.java` 的 `submit` 接收 `OrdersSubmitDTO`，交给 `OrderServiceImpl.submitOrder`。
2. `service/impl/OrderServiceImpl.java` 的 `submitOrder` 检查地址，调用 `checkOutOfRange`，再按当前用户读购物车。检查地址缺失和购物车为空两个分支。
3. 沿 `OrderMapper.xml`、`OrderDetailMapper.xml` 和 `ShoppingCartMapper.xml` 找到三次写操作，核对订单状态和菜品快照。查看当前方法的事务边界，推演其中一步失败时数据库会留下什么。

验证　有地图和微信配置时，下单后对照 `orders`、`order_detail`、`shopping_cart`。支付进入第 06 课。

### 06 · 支付回调与订单状态（大纲）

**问题**　支付通知到达后，待付款订单如何进入商家处理流程？

主链　用户发起支付 → 微信支付 → 回调 → 查单并更新状态 → 商家接单、派送、完成。

1. `controller/user/OrderController.java` 的 `payment` → `OrderServiceImpl.payment` → `sky-common` 的 `WeChatPayUtil.pay`。
2. `controller/notify/PayNotifyController.java` 的 `paySuccessNotify` → `OrderServiceImpl.paySuccess`，检查订单号、付款状态和订单状态的更新。
3. `controller/admin/OrderController.java` 的接单、派送、完成接口调用 `OrderServiceImpl.confirm`、`delivery`、`complete`。对照 `sky-pojo` 中的 `Orders` 常量和 `OrderMapper.xml` 的更新语句画状态图。

验证　先用状态图核对各入口的前置条件。有商户联调环境时再验证真实支付和退款。超时和提醒进入第 07 课。

### 07 · 定时任务与 WebSocket（大纲）

**问题**　系统如何处理超时订单，商家如何收到来单和催单消息？

主链　定时器 → 查询符合条件的订单 → 更新状态。支付成功或用户催单 → 广播 → 已连接的管理端。

1. `SkyApplication.java` 启用调度。`task/OrderTask.java` 的 `processTimeoutOrder` 和 `processDeliveryOrder` 按各自条件查找并更新订单。
2. `config/WebSocketConfiguration.java` 注册端点，`websocket/WebSocketServer.java` 的 `onOpen`、`onClose` 维护连接。
3. `OrderServiceImpl.paySuccess` 和 `reminder` 调用 `WebSocketServer.sendToAllClient`，比较来单与催单的消息字段。

验证　在学习库构造超时待支付样例，等调度触发后检查状态和日志。在浏览器网络面板检查 WebSocket 握手。统计留到第 08 课。

### 08 · 工作台、统计与 Excel（大纲）

**问题**　营业额用什么口径计算，导出的 Excel 如何取得每天的汇总？

主链　工作台或报表请求 → 根据时间和状态聚合 → 返回 VO，或填充 Excel 模板后下载。

1. `controller/admin/WorkSpaceController.java` 的 `businessData` → `WorkspaceServiceImpl.getBusinessData`，追到订单和用户 Mapper，核对有效订单口径。
2. `controller/admin/ReportController.java` 的 `turnoverStatistics`、`top10` → `ReportServiceImpl.getTurnover`、`getSalesTop10`，检查查询范围和排序。
3. `ReportController.export` → `ReportServiceImpl.exportBusinessData` → 读取 `src/main/resources/template/运营数据报表模板.xlsx` → 填充近 30 天数据并写回 HTTP 响应。

验证　从管理端导出报表，对照任意一天的 SQL 汇总。第一轮覆盖鉴权、菜品、缓存、购物车、订单、调度与报表，员工和分类的其他 CRUD、地址簿全量接口、OSS 上传及前端源码适合后续选读。
