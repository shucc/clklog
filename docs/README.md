# ClkLog 文档索引

本目录是对 ClkLog 社区版源码（v1.3.1）通读后整理的落地文档，面向「没有 Java 后端部署经验」的开发者。

| 文档 | 解决什么问题 | 建议阅读顺序 |
|---|---|---|
| [01-部署与运行手册](./01-部署与运行手册.md) | 项目由哪些服务组成、数据库怎么建、本地怎么跑、怎么打包、生产怎么部署、出问题怎么查 | ① 先读 |
| [02-数据上报API文档](./02-数据上报API文档.md) | 客户端怎么把埋点数据打进来：接口、编码规则、字段映射、Android/curl/Python 示例 | ② 再读（接入 SDK 时用） |
| [03-数据分析查询API文档](./03-数据分析查询API文档.md) | 怎么用 token / API Key 调 84 个查询与管理接口，把统计结果接进自己的系统 | ③ 需要二次开发时读 |

---

## 30 秒了解这个项目

```
SDK ──/api/gp──► receiver ──► ClickHouse(log_analysis)
                                  │
                          init（每5分钟聚合）
                                  ▼
                          ClickHouse(汇总表) ──► api ──► ui(nginx)
                                                 ▲
                                          manage ──► MySQL + Redis
```

六个仓库：`receiver`（收数）、`init`（建表+聚合）、`api`（查数）、`manage`（管理）、`ui`（前端）、`processing`（Flink，社区版可不用）。

---

## 最短跑通路径（约 30 分钟）

```bash
# 1. 起中间件
docker run -d --name clklog-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=clklog123 -e TZ=Asia/Shanghai mysql:8.0
docker run -d --name clklog-clickhouse -p 8123:8123 -p 9000:9000 -e TZ=Asia/Shanghai clickhouse/clickhouse-server:22.8
docker run -d --name clklog-redis -p 6379:6379 redis:7

# 2. 建 MySQL 表
mysql -h127.0.0.1 -uroot -pclklog123 < init/scripts_init/mysql/mysql_clklog.sql

# 3. 起服务（四个终端，各自在模块目录下）
cd init     && java -jar target/clklog-init-1.3.1.jar --server.port=8084
cd manage   && java -jar target/clklog-manage-1.3.1.jar --server.port=8081
cd api      && java -jar target/clklog-api-1.3.1.jar --server.port=8082
cd receiver && java -jar target/clklog-receiver-1.3.1.jar --server.port=8083

# 4. 起前端
cd ui && npm run dev      # http://localhost:9527   admin / clklog
```

详细的环境变量、工作目录要求、Docker 编排、nginx 配置、排错手册，见 [01-部署与运行手册](./01-部署与运行手册.md)。

---

## 三个最容易踩的坑（先记住）

1. **六个子模块是 Git Submodule** —— 目录已补齐（v1.3.1），但里面没有 `.git`，不是可提交的仓库；要重新拉取见手册第 6 节。
2. **必须在各自模块目录下启动** —— init 读 `./scripts`、manage 读 `./setting`、receiver 读 `./iplib`，换目录就找不到文件。
3. **Android 端 `$lib` 必须传 `"Android"`，页面事件必须是 `$AppViewScreen`** —— 否则数据能入库，但所有报表都是 0。
