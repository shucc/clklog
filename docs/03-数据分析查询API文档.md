# ClkLog 数据分析查询 API 文档

> 面向读者：想把 ClkLog 的统计数据接进自己系统（BI 大屏、运营后台、移动端看板）的开发者。
> 本文覆盖 `clklog-api`（统计查询，61 个接口）与 `clklog-manage`（管理，23 个接口）的鉴权方式、参数约定与接口清单，全部由源码注解与模型类提取。

---

## 1. 两个服务，两套用途

| 服务 | 建议访问前缀 | 接口数 | 用途 | 鉴权 |
|---|---|---|---|---|
| `clklog-manage` | `/manage` | 23 | 登录、项目管理、用户管理、API 密钥、全局设置 | 登录拿 JWT |
| `clklog-api` | `/api` | 61 | 趋势、地域、渠道、设备、受访页面、访客、崩溃等报表查询 + Excel 导出 | JWT 或 API Key |

**两者共用同一个 `token_secret` 和同一个 Redis**：在 manage 登录拿到的 token，直接可以调 api。

---

## 2. 鉴权

### 2.1 方式一：JWT（适合有人登录的场景）

**第一步，登录（调 manage）：**

```http
POST /manage/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "clklog"
}
```

响应：

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiJ9...."
  }
}
```

**第二步，带 token 调接口：**

```http
POST /api/flow/getFlow
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9....
Content-Type: application/json
```

Token 有效期由 `token.expireTime` 控制（**单位：分钟**，默认 `300` 即 5 小时），存在 Redis 里；剩余不足 20 分钟时会自动续期。过期后需要重新登录。

### 2.2 方式二：API Key（适合服务端对接、无人工登录）

在管理后台「密钥管理」里创建 API Key（存在 MySQL `tbl_api_key` 表），然后：

```http
POST /api/flow/getFlow
X-API-Key: <你的 API Key>
Content-Type: application/json
```

> 请求头名字是 **`X-API-Key`**（源码常量 `API_KEY_HEADER`），不是 `Authorization`。
> 服务端用 `clklog:apikey:<key>` 这个 Redis Key 校验，如果 Redis 被清空需要重新在后台保存一次密钥。

### 2.3 无需鉴权的路径

`/profile/**`、静态资源、`/logout` 放行；`/actuator/**` 走独立的 Basic 认证（用户名密码 = `SPRING_SECURITY_USER_NAME` / `SPRING_SECURITY_USER_PASSWORD`）。

---

## 3. 统一响应格式

所有接口（含 manage 与 api）都返回同样的信封：

```json
{
  "code": 200,
  "msg": null,
  "data": { }
}
```

| code | 含义 |
|---|---|
| `200` | 成功 |
| `403` | 未登录 / token 失效（前端会跳登录页） |
| `423` | 无该项目的数据访问权限（前端跳「无权限」页） |
| `500` | 服务端异常 |
| `501` / `502` | manage 专用：登录失败 / 密码错误 |

> 注意：HTTP 状态码通常也是 200，**业务状态要看 body 里的 `code`**。

---

## 4. 通用请求参数

### 4.1 时间 + 维度（大部分接口都有）

| 参数 | 类型 | 示例 | 说明 |
|---|---|---|---|
| `startTime` | string | `"2026-10-01"` | 开始日期，`yyyy-MM-dd` |
| `endTime` | string | `"2026-10-20"` | 结束日期，含当天 |
| `timeType` | string | `"day"` / `"hour"` | 时间粒度 |
| `channel` | string[] | `["网站","安卓"]` | 渠道过滤，**中文**：`网站` / `安卓` / `苹果` / `微信小程序` / `鸿蒙` |
| `projectName` | string | `"clklogapp"` | 项目编码，对应 MySQL `tbl_project.project_name` |

> `channel` 的中文取值来自 `api` 的 `application.yml`：
> `all=全部`、`android=Android,安卓`、`ios=iOS,苹果`、`js=js,网站`、`miniprogram=MiniProgram,微信小程序`、`harmonyos=HarmonyOS,鸿蒙`。
> 传中文还是英文取决于该接口内部的映射，**建议先用 `["网站"]` 这种中文试，报错再换英文**，或直接开 Swagger 看 example。

### 4.2 分页 + 排序（分页类接口）

| 参数 | 类型 | 示例 | 说明 |
|---|---|---|---|
| `pageNum` | int | `1` | 页码，从 1 开始 |
| `pageSize` | int | `50` | 每页条数 |
| `sortName` | string | `"pv"` | 排序字段：`pv` / `visitCount` / `newUv` / `uv` / `ipCount` / `avgPv` / `avgVisitTime` / `bounceRate` 等 |
| `sortOrder` | string | `"desc"` | `desc` / `asc` |
| `visitorType` | string | `"新访客"` | 新老访客过滤 |

分页响应结构统一为：

```json
{
  "code": 200,
  "data": {
    "total": 128,
    "rows": [ { }, { } ]
  }
}
```

---

## 5. 接口清单

### 5.1 统计查询（clklog-api，61 个，全部 POST）

> 路径前缀 `/api`（按第 9 节的 nginx 配置），服务内实际路径是 `/flow/getFlow` 这种。

#### 趋势分析 `/flow`

| 路径 | 说明 |
|---|---|
| `/flow/getFlow` | 流量概览及同环比（当前/上周期/同期/当日预测） |
| `/flow/getFlowTrend` | 流量趋势（按日/小时） |
| `/flow/getFlowTotal` | 流量合计 |
| `/flow/getFlowDetail` | 流量明细（PV/UV/访问次数/IP 数/平均访问时长/跳出率） |
| `/flow/getFlowDetailByCompare` | 流量明细对比 |
| `/flow/getFlowTrendDetail` | 已弃用 |

#### 地域分析 `/area`

| 路径 | 说明 |
|---|---|
| `/area/getArea` | 地域访问统计（国家/省/市） |
| `/area/getAreaDetailTotal` | 地域合计 |
| `/area/getAreaDetailList` | 分页：按省 |
| `/area/getAreaDetailCountryList` | 分页：按国家 |
| `/area/getAreaDetailCityList` | 分页：按省过滤城市 |
| `/area/getAreaDetailCountryListByCompare` | 分页：国家同比 |
| `/area/getAreaDetailProvinceListByCompare` | 分页：省份同比 |
| `/area/getAreaDetailTop10` | 已弃用 |

#### 渠道分析 `/channel`

| 路径 | 说明 |
|---|---|
| `/channel/getChannelDetail` | 渠道访问统计 |
| `/channel/getChannelList` | 项目渠道列表 |

#### 来源网站 `/sourcewebsite`

| 路径 | 说明 |
|---|---|
| `/sourcewebsite/getSourceWebSiteTop10` | Top10 来源网站 |
| `/sourcewebsite/getSourceWebSiteTotal` | 来源网站合计 |
| `/sourcewebsite/getSourceWebSiteDetail` | 分页：来源网站明细 |
| `/sourcewebsite/getSourceWebsite` / `getSourceWebSiteDetailTop10` | 已弃用 |

#### 受访页面 `/visituri`

| 路径 | 说明 |
|---|---|
| `/visituri/getVisitUri` | Top10 受访页面 |
| `/visituri/getVisitUriTotal` | 受访页面合计 |
| `/visituri/getVisitUriDetailList` | 分页：受访页面明细 |
| `/visituri/getVisitUriPathTreeTotal` | 受访页面树形结构 |
| `/visituri/getVisitUriListOfUriPath` | 指定路径下的 Top10 页面 |

#### 访客分析 `/visitor`

| 路径 | 说明 |
|---|---|
| `/visitor/getVisitor` | 新老访客统计 |
| `/visitor/getVisitorDetail` | 新老访客明细 |
| `/visitor/getVisitorTotal` | 访客类型合计 |
| `/visitor/getVisitorChannel` | 按渠道的访客统计 |
| `/visitor/getVisitorList` | 访客列表 |
| `/visitor/getVisitorSessionList` | 访客访问明细（会话级） |
| `/visitor/getVisitorSessionUriList` | 单次访问内的页面明细 |
| `/visitor/getVisitorDetailinfo` | 访客基本信息 |
| `/visitor/getLogAnalysisList` | 原始访问日志 |

#### 忠诚度分析 `/uservisit`

| 路径 | 说明 |
|---|---|
| `/uservisit/getUserPv` | 按访问页数区间统计访客数 |
| `/uservisit/getUserVisit` | 按访问次数区间统计访客数 |
| `/uservisit/getUserVisitTime` | 按访问时长区间统计访客数 |
| `/uservisit/getUserLatestTime` | 按上次访问时间区间统计访客数 |

#### 设备 / 系统

| 路径 | 说明 |
|---|---|
| `/device/getDeviceDetailList` | 分页：设备统计 |
| `/device/getDeviceDetail` | 已弃用 |
| `/os/getOsDetail` | 操作系统统计 |
| `/appCrashed/totalSummary` | App 崩溃概览 |
| `/appCrashed/trendSummary` | 崩溃趋势 |
| `/appCrashed/groupedSummary` | 按渠道/版本分组 |
| `/appCrashed/getPagedSummary` | 分页：崩溃设备分组 |
| `/appCrashed/getPage` | 分页：崩溃记录 |

#### 搜索词 `/searchword`

| 路径 | 说明 |
|---|---|
| `/searchword/getSearchWordTop10` | Top10 搜索词 |
| `/searchword/getSearchWordDetail` | 搜索词明细 |

#### Excel 导出 `/download`

| 路径 | 说明 |
|---|---|
| `/download/exportFlowTrendDetail` | 导出流量趋势 |
| `/download/exportAreaDetail` | 导出地域 |
| `/download/exportVisitUriDetail` | 导出受访页面 |
| `/download/exportSourceWebsiteDetail` | 导出来源网站 |
| `/download/exportVisitorDetail` | 导出新老访客 |
| `/download/exportDeviceDetail` | 导出设备 |
| `/download/exportChannelDetail` | 导出渠道 |
| `/download/exportSearchWordDetail` | 导出搜索词 |
| `/download/exportVisitorList` | 导出用户行为 |
| `/download/exportVisitor` | 导出用户忠诚度 |

导出接口返回的是 **Excel 二进制流**（`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`），不是 JSON，调用时要按文件流处理。

#### 其他

| 路径 | 方法 | 说明 |
|---|---|---|
| `/profile` | GET | 获取服务 id |
| `/profile/subscribe` | POST | 提交激活状态 |

### 5.2 管理接口（clklog-manage，23 个，全部 POST）

| 模块 | 路径 | 说明 |
|---|---|---|
| 权限 | `/auth/login` | 用户登录 |
| | `/auth/getUser` | 获取当前用户信息 |
| | `/auth/modifyPassword` | 修改自己的密码 |
| 用户 | `/user/getlistpage` | 分页用户列表 |
| | `/user/get` / `/user/add` / `/user/edit` / `/user/delete` | 用户增删改查 |
| | `/user/modifyPassword` | 管理员改他人密码 |
| 项目 | `/project/getlist` | 项目列表 |
| | `/project/getMyList` | 我授权的项目 |
| | `/project/get` / `/project/add` / `/project/edit` / `/project/delete` | 项目增删改查 |
| API 密钥 | `/apikey/list` / `/apikey/get` / `/apikey/add` / `/apikey/edit` / `/apikey/delete` | 密钥管理 |
| 全局设置 | `/global/getSetting` / `/global/saveSetting` | 排除 IP、排除 UA、排除 URL 参数、站内搜索词配置 |
| | `/global/getStat` | 全局统计（记录数、占用空间） |

> 新增项目后，manage 会在 **30 秒内**把项目配置刷进 Redis（`ProjectSettingKey`），receiver 才能识别这个 `project`。

---

## 6. 调用示例

### 6.1 流量概览 `/api/flow/getFlow`

请求：

```bash
curl -X POST http://127.0.0.1:8082/flow/getFlow \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{
        "timeType": "day",
        "channel": ["网站"],
        "startTime": "2026-10-01",
        "endTime": "2026-10-20",
        "projectName": "clklogapp"
      }'
```

响应（`data` 下四个同结构对象）：

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "current": {
      "statTime": "2026-10-01 ~ 2026-10-20",
      "pv": 15230,
      "visitCount": 4210,
      "uv": 3120,
      "ipCount": 2980,
      "avgPv": 3.62,
      "avgVisitTime": 128.45,
      "bounceRate": 0.41,
      "channel": "网站"
    },
    "currentPrediction": { "...": "当日预测，字段同上" },
    "previous": { "...": "上一个周期，字段同上" },
    "samePeriod": { "...": "同期，字段同上" }
  }
}
```

字段含义：

| 字段 | 含义 |
|---|---|
| `pv` | 浏览量 |
| `visitCount` | 访问次数（会话数） |
| `uv` | 访客数 |
| `ipCount` | IP 数 |
| `avgPv` | 平均访问页数 |
| `avgVisitTime` | 平均访问时长（秒） |
| `bounceRate` | 跳出率（0~1） |

### 6.2 分页查地域 `/api/area/getAreaDetailList`

```bash
curl -X POST http://127.0.0.1:8082/area/getAreaDetailList \
  -H 'Content-Type: application/json' \
  -H 'X-API-Key: <你的API Key>' \
  -d '{
        "pageNum": 1,
        "pageSize": 20,
        "sortName": "pv",
        "sortOrder": "desc",
        "channel": ["网站"],
        "startTime": "2026-10-01",
        "endTime": "2026-10-20",
        "projectName": "clklogapp"
      }'
```

```json
{
  "code": 200,
  "data": {
    "total": 34,
    "rows": [
      {
        "country": "cn",
        "province": "guangdong",
        "city": "shenzhen",
        "pv": 5210,
        "pvRate": 0.34,
        "visitCount": 1420,
        "newUv": 380,
        "uv": 980,
        "ipCount": 910,
        "avgPv": 3.67,
        "avgVisitTime": 132,
        "bounceRate": 0.38,
        "newUvRate": 0.39
      }
    ]
  }
}
```

> `country` / `province` / `city` 返回的是**拼音/英文小写**（如 `cn`、`guangdong`、`shenzhen`），前端靠 Redis 里的中英文对照表翻译，你自己接的时候也需要维护一份映射。

### 6.3 登录 + 查询（完整脚本）

```bash
BASE_MANAGE=http://127.0.0.1:8081
BASE_API=http://127.0.0.1:8082

TOKEN=$(curl -s -X POST $BASE_MANAGE/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"clklog"}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['data']['token'])")

curl -s -X POST $BASE_API/flow/getFlowTrend \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"timeType":"day","channel":["网站"],"startTime":"2026-10-01","endTime":"2026-10-20","projectName":"clklogapp"}' \
  | python3 -m json.tool
```

---

## 7. 打开在线 Swagger（强烈推荐）

源码里 `clklog-manage` 的 Swagger **默认开启**，`clklog-api` 的**默认关闭**（`application.yml` 里 `springdoc.api-docs.enabled: false`）。

启动 api 时加两个参数即可打开：

```bash
java -jar clklog-api-1.3.1.jar --server.port=8082 \
  --springdoc.api-docs.enabled=true \
  --springdoc.swagger-ui.enabled=true
```

然后访问：

| 服务 | 地址 |
|---|---|
| clklog-manage | `http://<host>:8081/swagger-ui/index.html` |
| clklog-api | `http://<host>:8082/swagger-ui/index.html` |

> 两个服务的 pom 里都引了 `knife4j-springdoc-ui`，如果它正常生效，还可以访问更好用的中文界面 `http://<host>:<port>/doc.html`。
> 生产环境建议保持关闭：这两个页面会把全部接口和参数结构暴露出来。

---

## 8. 常见问题

| 现象 | 原因 |
|---|---|
| 401 / `code: 403` | token 过期、`token_secret` 与 manage 不一致、Redis 被清空 |
| `code: 423` | 当前用户没有被授权访问这个 `projectName` |
| 所有指标为 0 | 明细表没有数据；或数据不是当天的（聚合只算今天）；或 `lib`/`event` 不符合 PV 口径 |
| 分页 `total` 有值但 `rows` 为空 | `pageNum` 从 1 开始，传 0 会返回空 |
| 导出接口拿到乱码 | 导出返回的是二进制流，要用 `curl -o out.xlsx` 或按 blob 处理 |
| `channel` 传英文报错 | 该接口按中文匹配，改成 `"网站"` / `"安卓"` |

---

## 9. nginx 前缀约定

前端和外部系统都按下面这套前缀访问，所以 nginx 必须这样配（详见《部署与运行手册》第 9.1 节）：

| 外部路径 | 转发到 |
|---|---|
| `/api/*` | `http://clklog-api:8080/*`（剥掉 `/api`） |
| `/manage/*` | `http://clklog-manage:8080/*`（剥掉 `/manage`） |

也就是说：文档里的 `/flow/getFlow` 对外要写成 `/api/flow/getFlow`，`/auth/login` 对外要写成 `/manage/auth/login`。
