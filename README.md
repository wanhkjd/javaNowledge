# 万佳商城现代化前端项目 (geminiHmall)

本项目是针对 `hmall` Spring Cloud 微服务后端定制打造的现代化、高颜值、零构建门槛的电商前台与运营管理后台系统。

---

## 🚀 独立运行与启动方式 (自带 nginx.exe)

本项目已在 `D:\javaweb\workspace\hmall\fronted\geminiHmall` 目录下配置了**独立的 `nginx.exe` 服务器**与配置文件，完全独立运行：

- **一键启动**：双击运行目录下的 `start.bat`（默认监听 `18088` 端口，自动代理后端 `8080` 网关）
- **一键停止**：双击运行目录下的 `stop.bat`
- **配置热载**：双击运行目录下的 `reload.bat`

浏览器访问入口：
- **万佳商城首页**：[http://localhost:18088/index.html](http://localhost:18088/index.html)
- **商品搜索与多维筛选**：[http://localhost:18088/search.html](http://localhost:18088/search.html)
- **我的购物车**：[http://localhost:18088/cart.html](http://localhost:18088/cart.html)
- **确认订单结算**：[http://localhost:18088/order-confirm.html](http://localhost:18088/order-confirm.html)
- **统一收银台**：[http://localhost:18088/pay.html](http://localhost:18088/pay.html)
- **支付成功凭证**：[http://localhost:18088/paysuccess.html](http://localhost:18088/paysuccess.html)
- **限时秒杀抢购**：[http://localhost:18088/seckill.html](http://localhost:18088/seckill.html)
- **用户登录页**：[http://localhost:18088/login.html](http://localhost:18088/login.html)
- **商品运营后台**：[http://localhost:18088/admin.html](http://localhost:18088/admin.html)（含「商品管理」与「秒杀活动」两个标签页）

> 💡 **测试账号**：
> - 用户名：`jack`，密码：`123`
> - 账户余额支付密码：`123`（登录页内置一键快速填充按钮）

---

## 📁 目录结构
```text
d:\javaweb\workspace\hmall\fronted\geminiHmall\
├── nginx.exe             # 独立的 Nginx 服务器运行时
├── start.bat             # 一键启动服务脚本
├── stop.bat              # 一键停止服务脚本
├── reload.bat            # 一键重新加载配置脚本
├── conf/
│   ├── mime.types        # MIME 类型映射文件
│   └── nginx.conf        # 专属于 geminiHmall 的反向代理配置 (监听 18088 代理 8080)
├── html/                 # Nginx 托管的前端页面与静态资产目录
│   ├── index.html        # 万佳商城主站首页（轮播图、分类导购、真实热卖好物、快捷加购）
│   ├── search.html       # 商品搜索与多维筛选（关键词检索、品类/品牌/价格区间、排序、分页）
│   ├── cart.html         # 购物车管理（选品、调整数量、单选/全选联动、实时总价、一键结算）
│   ├── order-confirm.html# 结算确认订单（选择用户收货地址、核对选购商品与总价、生成微服务订单）
│   ├── pay.html          # 统一收银台（订单详情、30分钟倒计时、余额支付与密码校验）
│   ├── paysuccess.html   # 支付成功凭证（展示订单编号、支付方式、扣减金额、继续选购）
│   ├── seckill.html       # 限时秒杀抢购页（活动倒计时、库存/秒杀价、抢购下单、异步订单轮询、去支付）
│   ├── login.html        # 用户登录页（账号密码校验、内置快速测试账号按钮、Token持久化）
│   ├── admin.html        # 运营管理后台（商品管理 + 秒杀活动创建/预热/进入抢购页）
│   ├── css/
│   │   ├── common.css    # 全局设计规范（配色变量、导航栏、底部、商品卡片、按钮、徽章）
│   │   ├── element.css   # Element UI 样式库（已本地离线化）
│   │   └── fonts/        # Element UI 图标字体文件
│   ├── js/
│   │   ├── config.js     # 全局环境配置
│   │   ├── common.js     # 通用工具函数（分与元转换 formatPrice、时间格式化、会话存储、秒杀活动本地缓存 seckillCache）
│   │   ├── api.js        # 微服务 Axios 实例封装（JWT 拦截注入、统一响应解包与错误捕获）
│   │   ├── header.js     # 全局通用顶部导航与页脚组件
│   │   ├── vue.js        # Vue 核心库
│   │   ├── element.js    # Element UI 组件库
│   │   ├── axios.min.js  # Axios HTTP 请求库
│   │   └── qrcode.min.js # 二维码渲染库
│   └── img/              # 静态素材（Logo、支付渠道图标、Banner轮播图、商品占位图）
├── logs/                 # Nginx 运行日志
├── temp/                 # Nginx 临时缓存
└── README.md
```

---

## 🔗 微服务后端接口对应关系

| 业务模块 | 前端调用接口 | 后端微服务与控制器 | 功能说明 |
| :--- | :--- | :--- | :--- |
| **用户认证** | `POST /users/login` | `user-service: UserController` | 校验用户名密码，返回 JWT Token 与账户初始余额 |
| **收货地址** | `GET /addresses` | `user-service: AddressController` | 读取当前登录用户的可用收货地址列表 |
| **商品分页** | `GET /items/page` | `item-service: ItemController` | 分页加载商品数据，供前台首页与后台表格使用 |
| **商品检索** | `GET /search/list` | `item-service: SearchController` | 多条件检索商品（分类、品牌、价格区间、排序） |
| **商品增删改** | `POST/PUT/DELETE /items` | `item-service: ItemController` | 运营后台新增、编辑、删除与上下架切换 |
| **我的购物车** | `GET /carts` | `cart-service: CartController` | 查询当前用户购物车内所有条目 |
| **添加购物车** | `POST /carts` | `cart-service: CartController` | 将商品 SKU 加购并记录属性规格 |
| **修改加购量** | `PUT /carts` | `cart-service: CartController` | 购物车内即时增减商品数量 |
| **删除购物车** | `DELETE /carts/{id}` | `cart-service: CartController` | 删除指定购物车商品 |
| **提交订单** | `POST /orders` | `trade-service: OrderController` | 选定收货地址与选购条目，扣减库存生成订单 |
| **订单详情** | `GET /orders/{id}` | `trade-service: OrderController` | 收银台获取待支付订单状态与应付金额 |
| **生成支付单** | `POST /pay-orders` | `pay-service: PayController` | 生成唯一微服务支付流水单 (`payType: 5`) |
| **余额扣款支付** | `POST /pay-orders/{id}` | `pay-service: PayController` | 校验用户支付密码并执行扣款，更新支付状态 |
| **创建秒杀活动** | `POST /seckill/activity` | `item-service: SeckillController` | 后台录入商品、秒杀价、库存与起止时间，返回活动ID（雪花大整数，字符串） |
| **活动预热** | `POST /seckill/activity/{id}/preheat` | `item-service: SeckillController` | 将活动信息与库存载入 Redis，开抢前必须预热 |
| **抢购下单** | `POST /seckill/{activityId}` | `item-service: SeckillController` | Lua 原子扣减 Redis 库存并发 MQ 异步建单，返回预生成订单号 |
| **抢购结果查询** | `GET /seckill/result/{activityId}` | `item-service: SeckillController` | 查询当前用户在该活动是否已抢到及其订单号（防重复下单） |

---

## 🛠️ 参考项目缺陷修复说明

1. **修复首页无后端数据问题**：
   - 旧版 `hmall-portal/index.html` 全为 1200+ 行死数据；
   - `geminiHmall/index.html` 实时调用后端 `GET /items/page` 加载在售真实商品，并动态统计全商城好物总数。
2. **修复搜索页与“立即购买”断链**：
   - 旧版搜索页点击“立即购买”跳转时丢失商品结构，导致订单确认页读取 `selectedCarts` 报空；
   - `geminiHmall` 构造标准订单直购项，无论从首页、搜索页还是购物车，均能顺畅流转结算。
3. **修复收银台接口不匹配问题**：
   - 后端目前核心支持余额扣款 (`payType: 5`)，原版调用 WeChat/Alipay 接口直接 500 异常；
   - `geminiHmall` 重点突出钱包余额支付，直观对比“账户余额”与“本次扣减金额”，密码校验成功后自动更新前台余额缓存。
4. **统一前后台体验与鉴权管理**：
   - 顶部导航组件 `mall-header` 统一集成于所有页面；
   - 实时显示当前登录用户（如 `Jack`）与可用余额（`￥xx.xx`），退出登录后自动清空会话并拦截受限操作。
