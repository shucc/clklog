# ClkLog 数据上报 API 文档（客户端接入）

> 面向读者：需要把 App / Web / 小程序 的埋点数据接进 ClkLog 的开发者。
> 本文基于 `clklog-receiver` 源码逐行核对，包含**唯一一个上报接口**的完整参数、编码规则、字段映射与可运行示例。

---

## 1. 接口一览

ClkLog 的**数据接收服务（clklog-receiver）只暴露一个接口**，所有端（Web / Android / iOS / 小程序 / uni-app）都往这里发：

| 项 | 值 |
|---|---|
| 路径 | `/api/gp` |
| 方法 | `GET` 或 `POST`（两者等价，推荐 POST） |
| Content-Type | `application/x-www-form-urlencoded` |
| 认证 | 无 Header 认证，靠 `project` + `token` 参数识别项目 |
| 返回 | `text/plain`，纯字符串，**HTTP 状态码永远是 200** |
| 单请求上限 | 默认 **2MB**（`receiver.max-request-size`，压缩前/后的字节数都算） |

基础地址形如：

```
http://<receiver-host>:<receiver-port>/api/gp
# 例：http://127.0.0.1:8083/api/gp
```

---

## 2. 请求参数

参数可以放在 **query string** 或 **form body** 里，服务端两者都认（先由 Spring 绑定，绑定不到再手工解析 body）。

| 参数 | 必填 | 类型 | 说明 |
|---|---|---|---|
| `project` | ✅ | string | 项目编码，必须与 MySQL `tbl_project.project_name` 完全一致。默认项目是 `clklogapp` |
| `token` | ⚠️ | string | 项目凭据。**默认不校验**（`receiver.enable-check-project-token=false`）；一旦运维把开关打开，就必须传且要等于 `tbl_project.token` |
| `data_list` | ✅ 二选一 | string | 批量数据。**URL 编码后的 Base64 字符串**，解出来后是 JSON **数组** |
| `data` | ✅ 二选一 | string | 单条数据，规则同上，解出来是 JSON **对象**。代码里 `data_list` 优先 |
| `gzip` | ❌ | `0` / `1` | 默认 `0`。为 `1` 时，服务端会先做 GZIP 解压再 Base64 解码 |
| `crc` | ❌ | string | 校验码，仅落库不校验，可传空 |
| `ext` | ❌ | string | 扩展信息（URL 编码的 `k=v&k=v`），里面的 `crc` 会被提取出来 |

> **只用 `data_list` 就够了**。`data` 是给单条场景留的口子，批量上报统一用 `data_list`。

---

## 3. 数据编码规则（最容易出错的地方）

服务端的处理链是**反过来的三步**，你发送时要正着做：

```
你这边：  事件 JSON 数组
            │ ① 序列化成 UTF-8 字符串
            ▼
         JSON 字符串
            │ ② 可选：GZIP 压缩成 byte[]   （gzip=1 时）
            ▼
          byte[]
            │ ③ Base64 编码（标准 Base64，不要 URL-safe，不要换行）
            ▼
        Base64 字符串
            │ ④ URL 编码（把 + / = 变成 %2B %2F %3D）
            ▼
        data_list 的值
```

服务端：`URL 解码 → Base64 解码 → (GZIP 解压) → JSON 解析`

### 三条硬性规则

1. **必须是标准 Base64**（字母表含 `+` `/`，用 `=` 补位）。**不要**用 URL-safe 变体（`-` `_`）。
2. **不要带换行**。Java/Android 用 `Base64.NO_WRAP`，Python 用 `base64.b64encode()`（默认不换行）。
3. **Base64 结果一定要 URL 编码**后再拼进 `data_list=`。因为标准 Base64 里的 `+` 在表单里会被解析成空格。

> 服务端代码里有个兼容判断：如果字符串里**已经含有** `+` `/` `=`，就认为没被 URL 编码、直接 Base64 解码；否则先 URL 解码。所以两种写法都能跑通，但**URL 编码是官方 Web SDK 的做法，最稳**。

---

## 4. 事件（Event）JSON 结构

一次 `data_list` 可以带多条事件，是一个数组：

```json
[
  {
    "distinct_id": "a1b2c3d4-0001",
    "anonymous_id": "a1b2c3d4-0001",
    "type": "track",
    "event": "$AppViewScreen",
    "time": 1729392000000,
    "_track_id": 1729392000123,
    "_flush_time": 1729392000200,
    "identities": {
      "$identity_cookie_id": "a1b2c3d4-0001"
    },
    "lib": {
      "$lib": "Android",
      "$lib_method": "auto",
      "$lib_version": "1.0.0",
      "$app_version": "2.3.0"
    },
    "properties": {
      "$os": "Android",
      "$os_version": "14",
      "$brand": "Xiaomi",
      "$model": "23127PN0CC",
      "$network_type": "WIFI",
      "$carrier": "China Mobile",
      "$screen_width": 1080,
      "$screen_height": 2400,
      "$device_id": "8f3c1e2a9b7d",
      "$app_id": "com.example.app",
      "$app_name": "示例App",
      "$app_state": "foreground",
      "$url": "android://com.example.app/home",
      "$url_path": "/home",
      "$title": "首页",
      "$timezone_offset": 8,
      "$is_first_day": true,
      "$is_first_time": true
    }
  }
]
```

### 4.1 顶层字段

| JSON 字段 | 必填 | 说明 | 落库列 |
|---|---|---|---|
| `distinct_id` | ✅ | 用户唯一标识（登录后为业务用户 ID，未登录为设备 ID） | `distinct_id` |
| `anonymous_id` | ❌ | 匿名 ID。不传时服务端自动赋值为 `distinct_id` | （内部使用） |
| `type` | ✅ | 事件大类，一般固定 `track` | `typeContext` |
| `event` | ✅ | 事件名，见第 5 节 | `event` |
| `time` | ✅ | 事件发生时间，**毫秒时间戳** | `time` / `log_time` / `stat_date` / `stat_hour` |
| `_track_id` | ❌ | 埋点流水号 | `track_id` |
| `_flush_time` | ❌ | 上报时间戳（毫秒）。不传时等于 `time` | `flush_time` |
| `identities` | ❌ | 身份信息对象 | — |
| `lib` | ✅ | SDK 信息对象，**`$lib` 决定这条数据算哪个端** | `lib` / `lib_method` / `lib_version` |
| `properties` | ✅ | 事件属性对象 | 见 4.4 |

### 4.2 `lib` 对象（决定「渠道」，非常重要）

| 字段 | 说明 | 取值 |
|---|---|---|
| `$lib` | 端类型 | **`Android`** / `iOS` / `js` / `MiniProgram` / `HarmonyOS` |
| `$lib_method` | 埋点方式 | `auto`（全埋点）/ `code`（代码埋点） |
| `$lib_version` | SDK 版本 | 任意字符串 |
| `$app_version` | App 版本 | 任意字符串 |

> ⚠️ **Android 端必须传 `"$lib": "Android"`**（首字母大写）。
> 服务端的统计 SQL 里，PV 的口径写死为：
> - `lib = 'js'` 且 `event = '$pageview'`
> - `lib IN ('iOS','Android')` 且 `event = '$AppViewScreen'`
> - `lib = 'MiniProgram'` 且 `event = '$MPViewScreen'`
>
> `$lib` 传错（比如传 `android` 小写），数据会入库，但**所有报表都是 0**。

### 4.3 `identities` 对象

| 字段 | 说明 |
|---|---|
| `$identity_cookie_id` | Web 端 cookie ID（App 端可传设备 ID） |
| `user_key` | 业务用户 ID。**传了非空值，服务端会把 `is_logined` 置为 1** |

### 4.4 `properties` 对象（服务端会逐字段提取）

不是所有字段都落库，只有下面这些会被 `ExtractUtil.extractToLogBean()` 提取。其余字段会被丢弃（社区版明细表是宽表，不支持自定义属性列）。

**通用/环境类**

| 字段 | 说明 |
|---|---|
| `$os` / `$os_version` | 操作系统及版本 |
| `$brand` / `$model` / `$manufacturer` | 设备品牌 / 机型 / 厂商 |
| `$network_type` | 网络类型（WIFI / 4G / 5G） |
| `$carrier` | 运营商 |
| `$wifi` | 是否 WiFi |
| `$screen_width` / `$screen_height` | 屏幕分辨率 |
| `$viewport_width` / `$viewport_height` / `$viewport_position` | 视口尺寸/滚动位置（Web） |
| `$device_id` | 设备 ID |
| `$timezone_offset` | 时区偏移（如 `8`） |
| `$screen_name` / `$screen_orientation` | 屏幕名 / 横竖屏 |
| `$latitude` / `$longitude` | 经纬度 |
| `$country` / `$province` / `$city` | 国家/省/市。**会被服务端按 IP 解析出的地域覆盖**（IP 解析优先） |

**App 专属**

| 字段 | 说明 |
|---|---|
| `$app_id` / `$app_name` / `$app_state` | 应用 ID / 名称 / 前后台状态 |
| `app_crashed_reason` | 崩溃原因（注意：**没有 `$` 前缀**） |
| `event_duration` 或 `$event_duration` | 页面停留时长（秒，浮点） |
| `$event_session_id` | 会话 ID（用于把一次访问内的页面串起来） |

**页面 / 流量来源**

| 字段 | 说明 |
|---|---|
| `$url` / `$url_path` / `$title` | 页面地址 / 路径 / 标题（App 端可传自定义 scheme） |
| `$referrer` / `$referrer_host` | 上级来源 |
| `$latest_referrer` / `$latest_referrer_host` | 最近一次站外来源 |
| `$latest_search_keyword` | 最近一次搜索引擎关键词 |
| `$latest_traffic_source_type` | 最近一次流量来源类型 |
| `$latest_landing_page` | 最近一次落地页 |
| `$latest_scene` / `$latest_share_method` | 小程序场景值 / 分享方式 |
| `$utm_source` / `$utm_medium` / `$utm_campaign` / `$utm_content` / `$utm_term` | UTM 参数 |
| `$latest_utm_*` | 最近一次 UTM（同上，前缀 `$latest_`） |
| `$first_channel_name` / `$first_channel_ad_id` / `$first_channel_click_id` 等 | 首次渠道信息 |

**元素点击（全埋点）**

| 字段 | 说明 |
|---|---|
| `$element_id` / `$element_name` / `$element_type` | 元素标识 / 名称 / 类型 |
| `$element_class_name` / `$element_content` / `$element_selector` | 元素 class / 内容 / 选择器 |
| `$element_position` / `$element_target_url` | 元素位置 / 跳转地址 |

**新老访客（新客指标全靠它）**

| 字段 | 说明 |
|---|---|
| `$is_first_day` | 是否首日访问。统计 SQL 里按字符串 `'true'` 比较，**必须传布尔 `true` 或字符串 `"true"`** |
| `$is_first_time` | 是否首次触发该事件 |

---

## 5. 事件名约定

| 端 | 页面浏览事件 | 页面离开事件 |
|---|---|---|
| Web（`$lib=js`） | `$pageview` | `$WebPageLeave` |
| Android / iOS（`$lib=Android`/`iOS`） | **`$AppViewScreen`** | `$AppPageLeave` |
| 小程序（`$lib=MiniProgram`） | `$MPViewScreen` | `$MPPageLeave` |

自定义业务事件随便起名（如 `click_buy_button`），会入库，但**不会计入 PV/UV/访问次数**，只在「用户行为」明细里可见。

---

## 6. 返回结果

HTTP 状态码固定 200，返回体是纯文本。可能的值：

| 返回值 | 含义 | 排查方向 |
|---|---|---|
| `success` | 接收成功（已入内存队列，异步写库） | — |
| `parse_data_failed` | 解析失败 | Base64/GZIP 编码错、JSON 非法、`data`/`data_list` 为空 |
| `raw_message_empty` | 请求体为空 | 检查 body 是否真的发出去了 |
| `project_not_exist` | 项目不存在 | `project` 与 `tbl_project.project_name` 不一致；或 manage 未启动导致 Redis 里没有项目缓存 |
| `project_token_error` | 项目 token 校验失败 | 服务端开了 `enable-check-project-token`，但你传的 `token` 不匹配 |

> `parse_data_failed` 前面可能带一段原因前缀（代码里 `reason + type`），例如 `data或data_list为空parse_data_failed`，判断时用 `contains()` 而不是 `equals()`。

**注意**：返回 `success` 只代表**接收成功**，不代表已经写进 ClickHouse。receiver 是「入内存队列 → 后台线程每 1000 条或队列空闲时批量 INSERT」，有秒级延迟。

---

## 7. 可运行示例

### 7.1 curl（不压缩，最快验证）

```bash
# 1) 准备 JSON（一行，UTF-8 无 BOM）
cat > /tmp/evt.json <<'EOF'
[{"distinct_id":"test-user-001","type":"track","event":"$AppViewScreen","time":1729392000000,"lib":{"$lib":"Android","$lib_method":"code","$lib_version":"1.0.0"},"properties":{"$os":"Android","$os_version":"14","$brand":"Xiaomi","$model":"MI 13","$app_id":"com.example.app","$app_name":"示例App","$url":"android://com.example.app/home","$url_path":"/home","$title":"首页","$is_first_day":true,"$is_first_time":true}}]
EOF

# 2) Base64（macOS 用 base64，Linux 用 base64 -w0）
B64=$(base64 -i /tmp/evt.json | tr -d '\n')

# 3) URL 编码后上报
curl -X POST 'http://127.0.0.1:8083/api/gp' \
  --data-urlencode "project=clklogapp" \
  --data-urlencode "data_list=$B64" \
  -H 'Content-Type: application/x-www-form-urlencoded'
# 期望输出：success
```

> macOS 的 `base64` 默认会折行，必须 `tr -d '\n'`；Linux 加 `-w0`。

### 7.2 curl（GZIP 压缩）

```bash
gzip -c /tmp/evt.json | base64 | tr -d '\n' > /tmp/evt.b64
B64=$(cat /tmp/evt.b64)

curl -X POST 'http://127.0.0.1:8083/api/gp' \
  --data-urlencode "project=clklogapp" \
  --data-urlencode "gzip=1" \
  --data-urlencode "data_list=$B64"
```

### 7.3 Android（Kotlin + OkHttp）

```kotlin
import android.util.Base64
import okhttp3.*
import okhttp3.MediaType.Companion.toMediaType
import org.json.JSONArray
import org.json.JSONObject
import java.io.ByteArrayOutputStream
import java.net.URLEncoder
import java.util.zip.GZIPOutputStream

object ClkLogReporter {

    // receiver 地址，注意末尾是 /api/gp
    private const val ENDPOINT = "http://10.0.2.2:8083/api/gp"   // 模拟器访问宿主机
    private const val PROJECT  = "clklogapp"

    private val client = OkHttpClient.Builder().build()
    private val FORM = "application/x-www-form-urlencoded".toMediaType()

    /** 上报一批事件；gzip=true 时压缩后再 Base64 */
    fun report(events: JSONArray, gzip: Boolean = true, callback: (String) -> Unit = {}) {
        // ① JSON -> UTF-8 bytes
        var bytes = events.toString().toByteArray(Charsets.UTF_8)

        // ② 可选 GZIP
        if (gzip) {
            val bos = ByteArrayOutputStream()
            GZIPOutputStream(bos).use { it.write(bytes) }
            bytes = bos.toByteArray()
        }

        // ③ 标准 Base64，禁止换行
        val b64 = Base64.encodeToString(bytes, Base64.NO_WRAP)

        // ④ URL 编码
        val body = FormBody.Builder()
            .add("project", PROJECT)
            .add("gzip", if (gzip) "1" else "0")
            .add("data_list", URLEncoder.encode(b64, "UTF-8"))
            .build()

        val req = Request.Builder().url(ENDPOINT).post(body).build()
        client.newCall(req).enqueue(object : Callback {
            override fun onFailure(call: Call, e: java.io.IOException) = callback("io_error: ${e.message}")
            override fun onResponse(call: Call, response: Response) {
                callback(response.body?.string() ?: "")
            }
        })
    }

    /** 构造一条 Android 页面浏览事件 */
    fun buildPageViewEvent(distinctId: String, pageName: String, isFirstDay: Boolean): JSONObject {
        return JSONObject().apply {
            put("distinct_id", distinctId)
            put("anonymous_id", distinctId)
            put("type", "track")
            put("event", "$AppViewScreen")          // 必须是 $AppViewScreen
            put("time", System.currentTimeMillis())  // 毫秒
            put("_flush_time", System.currentTimeMillis())
            put("identities", JSONObject().put("user_key", ""))
            put("lib", JSONObject().apply {
                put("$lib", "Android")               // 首字母大写
                put("$lib_method", "code")
                put("$lib_version", "1.0.0")
                put("$app_version", "1.0.0")
            })
            put("properties", JSONObject().apply {
                put("$os", "Android")
                put("$os_version", android.os.Build.VERSION.RELEASE)
                put("$brand", android.os.Build.BRAND)
                put("$model", android.os.Build.MODEL)
                put("$app_id", "com.example.app")
                put("$app_name", "示例App")
                put("$url", "android://com.example.app/$pageName")
                put("$url_path", "/$pageName")
                put("$title", pageName)
                put("$is_first_day", isFirstDay)
                put("$is_first_time", true)
            })
        }
    }
}
```

调用：

```kotlin
val arr = JSONArray()
arr.put(ClkLogReporter.buildPageViewEvent("user-001", "home", true))
ClkLogReporter.report(arr, gzip = true) { result ->
    // result == "success" 即接收成功
    Log.d("ClkLog", "report -> $result")
}
```

> **Android 模拟器访问本机**用 `10.0.2.2`；真机用电脑局域网 IP，并确保 `AndroidManifest.xml` 里配了 `usesCleartextTraffic`（HTTP 明文）或直接用 HTTPS。

### 7.4 Python（压测/造数据）

```python
import base64, gzip, json, time, urllib.parse, urllib.request

def send(events, project="clklogapp", endpoint="http://127.0.0.1:8083/api/gp", use_gzip=True):
    raw = json.dumps(events, ensure_ascii=False).encode("utf-8")
    if use_gzip:
        raw = gzip.compress(raw)
    b64 = base64.b64encode(raw).decode("ascii")

    data = urllib.parse.urlencode({
        "project": project,
        "gzip": "1" if use_gzip else "0",
        "data_list": b64,
    }).encode("utf-8")

    req = urllib.request.Request(
        endpoint, data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    with urllib.request.urlopen(req, timeout=10) as r:
        return r.read().decode("utf-8")

if __name__ == "__main__":
    now = int(time.time() * 1000)
    events = [{
        "distinct_id": f"u{i}",
        "type": "track",
        "event": "$AppViewScreen",
        "time": now,
        "lib": {"$lib": "Android", "$lib_method": "code", "$lib_version": "1.0.0"},
        "properties": {
            "$os": "Android", "$os_version": "14",
            "$url": "android://app/home", "$url_path": "/home", "$title": "首页",
            "$is_first_day": True,
        },
    } for i in range(100)]

    print(send(events))   # success
```

---

## 8. 服务端还做了哪些处理（了解这些，才知道数据为什么「不对」）

上报成功后，receiver 会对每条事件做以下加工，**这些字段你不需要传，传了也会被覆盖**：

| 处理 | 结果 |
|---|---|
| 取客户端 IP | 依次读 `X-Forwarded-For` → `Proxy-Client-IP` → `WL-Proxy-Client-IP` → TCP 对端地址；非法 IP 置空 |
| IP 解析地域 | 用内置 IP2Location 库解析国家/省/市（结果缓存在 Redis，默认 7 天） |
| UA 解析 | 用 yauaa 从 `User-Agent` 解析浏览器、浏览器版本、机型、品牌、OS；**`properties.$os` 为空时才用 UA 结果兜底** |
| 时间派生 | 由 `time` 生成 `log_time`、`stat_date`（yyyy-MM-dd）、`stat_hour` |
| URL 清洗 | 按项目配置剔除 `excludedUrlParams` 里的参数（默认剔除 `time/t/timestamp/r`） |
| 路径规则 | 按项目配置的 `uriPathRules` 归一化 `$url_path`（例如把 `/order/12345` 归一成 `/order/*`） |
| 站内搜索词 | 从 URL 里按 `searchwordKey`（默认 `searchStr`）提取 |
| 数据过滤 | 命中项目 `excludedIp` / `excludedUa` 的事件会被丢弃 |

---

## 9. 联调检查清单

1. `project` 参数和 MySQL `tbl_project.project_name` 一致吗？
2. `lib.$lib` 是 `Android` 吗？`event` 是 `$AppViewScreen` 吗？（两个都错 = 报表全 0）
3. `time` 是**毫秒**吗？（传成秒会变成 1970 年）
4. Base64 是标准变体、无换行吗？
5. 返回的是 `success` 吗？不是就看第 6 节的对照表。
6. 返回 `success` 但 ClickHouse 没数据？
   ```bash
   docker exec -it clklog-clickhouse clickhouse-client -q \
     "SELECT count() FROM clklog.log_analysis WHERE project_name='clklogapp'"
   ```
   还是 0 的话看 receiver 日志 `log/store.log`，里面有每条原始报文。
7. 有数据但报表是 0？等 `clklog-init` 跑完一轮（约 5 分钟），并且只统计**当天**的数据。
