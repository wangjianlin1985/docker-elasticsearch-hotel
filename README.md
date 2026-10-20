# 基于 ElasticSearch 的旅游酒店搜索网

> 微服务学习项目：使用 **Spring Boot + ElasticSearch + RabbitMQ + MySQL** 实现一个支持全文检索、拼音搜索、自动补全、附近酒店、条件过滤的旅游酒店搜索网站。

![Java](https://img.shields.io/badge/Java-1.8-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.3.10-brightgreen)
![ElasticSearch](https://img.shields.io/badge/ElasticSearch-7.12.1-blue)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3--management-ff6600)
![MySQL](https://img.shields.io/badge/MySQL-5.7%2B-4479A1)

## 目录

- [项目简介](#项目简介)
- [功能特性](#功能特性)
- [技术栈](#技术栈)
- [系统架构](#系统架构)
- [项目结构](#项目结构)
- [快速开始](#快速开始)
- [接口说明](#接口说明)
- [ElasticSearch 实现要点](#elasticsearch-实现要点)
- [常见问题](#常见问题)
- [声明](#声明)

## 项目简介

项目由两个独立的 Spring Boot 服务组成：

| 模块 | 端口 | 作用 |
| --- | --- | --- |
| `hotel-demo` | `8089` | **酒店搜索服务（前台）**：提供搜索、过滤、附近排序、自动补全等接口和前台页面，并监听 MQ 消息同步数据到 ElasticSearch |
| `hotel-admin` | `8099` | **酒店管理服务（后台）**：对 MySQL 中的酒店数据做增删改查，数据变更时发送 MQ 消息 |

数据以 MySQL 为准，通过 RabbitMQ 异步同步到 ElasticSearch，实现 **“数据库管理 + 搜索引擎检索”** 的读写分离。

## 功能特性

- **全文检索**：酒店名称、品牌、商圈合并到 `all` 字段统一检索，使用 IK 中文分词
- **拼音搜索**：集成拼音分词器，支持拼音、拼音首字母搜索（如输入 `rujia` 搜索 “如家”）
- **输入自动补全**：基于 ES `completion suggester`，边输入边提示
- **条件过滤**：品牌、城市、星级、价格区间组合筛选
- **动态筛选项**：通过聚合（Aggregation）实时生成当前搜索结果对应的品牌 / 城市 / 星级列表
- **附近的酒店**：基于 `geo_point`，传入用户位置后按距离升序排序并返回距离
- **广告置顶**：通过 `function_score` 给广告酒店加权，提高排名
- **分页查询**
- **数据同步**：后台新增 / 修改 / 删除酒店后，经 RabbitMQ 自动同步到 ElasticSearch
- **前端页面**：Vue + Axios，集成高德地图展示定位

## 技术栈

| 类别 | 技术 |
| --- | --- |
| 后端框架 | Spring Boot 2.3.10.RELEASE、Spring MVC |
| 搜索引擎 | ElasticSearch 7.12.1（RestHighLevelClient）、Kibana 7.12.1 |
| 中文 / 拼音分词 | IK Analyzer、elasticsearch-analysis-pinyin |
| 消息队列 | RabbitMQ（spring-boot-starter-amqp） |
| 数据库 | MySQL、MyBatis-Plus 3.4.2 |
| 其他 | Lombok、FastJson、Commons-Lang3 |
| 前端 | Vue 2、Axios、高德地图 JS API |
| 部署环境 | Docker |

## 系统架构

```
                         ┌────────────────────┐
   浏览器 (Vue) ───────▶ │  hotel-demo :8089   │ ──── 搜索 / 过滤 / 补全 ───▶ ┌───────────────┐
                         │  搜索服务（前台）    │                              │ ElasticSearch │
                         └─────────▲──────────┘ ◀─── 同步新增/删除 ──────────  │   (hotel 索引) │
                                   │ 监听消息                                  └───────────────┘
                                   │
                         ┌─────────┴──────────┐
                         │      RabbitMQ       │  exchange: hotel.topic
                         └─────────▲──────────┘
                                   │ 发送消息
                         ┌─────────┴──────────┐
   管理端 ─────────────▶ │  hotel-admin :8099  │ ──── 增删改查 ───▶ ┌───────┐
                         │  管理服务（后台）    │                    │ MySQL │
                         └────────────────────┘                    └───────┘
```

**MQ 约定**

| 项 | 值 |
| --- | --- |
| 交换机（Topic） | `hotel.topic` |
| 新增 / 修改队列 | `hotel.insert.queue`，RoutingKey：`hotel.insert` |
| 删除队列 | `hotel.delete.queue`，RoutingKey：`hotel.delete` |

## 项目结构

```
.
├── 完整版项目
│   ├── hotel-demo                  # 搜索服务（前台，端口 8089）
│   │   └── src/main
│   │       ├── java/cn/itcast/hotel
│   │       │   ├── HotelDemoApplication.java   # 启动类（含 ES 客户端配置）
│   │       │   ├── constants/                  # 索引、MQ 常量
│   │       │   ├── mapper/                     # MyBatis-Plus Mapper
│   │       │   ├── mq/HotelListener.java       # MQ 监听，同步数据到 ES
│   │       │   ├── pojo/                       # Hotel / HotelDoc / RequestParams / PageResult
│   │       │   ├── service/                    # 搜索、聚合、补全等核心业务
│   │       │   └── web/HotelController.java    # 搜索接口
│   │       ├── resources/static/               # 前台页面（index.html、css、js、img）
│   │       └── test/                           # ES 索引 / 文档 / 搜索 / 聚合 测试用例
│   └── hotel-admin                 # 管理服务（后台，端口 8099）
│       └── src/main
│           ├── java/cn/itcast/hotel/web/HotelController.java   # 酒店增删改查 + 发送 MQ
│           └── resources/static/index.html                     # 管理页面
├── tb_hotel.sql                    # MySQL 建表及初始数据脚本
├── 酒店的mapping.txt                # ES 索引库 DSL（含拼音分词、自动补全）
├── 自定义分词器.jsonc               # 拼音分词器、自动补全的学习示例
├── 安装elasticsearch.md             # ES / Kibana / IK 分词器 Docker 安装文档
├── 创建RabbitMQ容器命令.txt          # RabbitMQ Docker 启动命令
└── 【运行步骤看这个】.txt            # 运行步骤速查
```

## 快速开始

### 1. 环境准备

- JDK 1.8
- Maven 3.6+
- MySQL 5.7+
- Docker（用于部署 ElasticSearch、Kibana、RabbitMQ）

### 2. 用 Docker 部署 ElasticSearch、Kibana

```bash
# 创建网络，让 ES 和 Kibana 互联
docker network create es-net

# 部署单点 ES 7.12.1
docker run -d \
  --name es \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "discovery.type=single-node" \
  -v es-data:/usr/share/elasticsearch/data \
  -v es-plugins:/usr/share/elasticsearch/plugins \
  --privileged \
  --network es-net \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:7.12.1

# 部署 Kibana（用于执行 DSL）
docker run -d \
  --name kibana \
  -e ELASTICSEARCH_HOSTS=http://es:9200 \
  --network=es-net \
  -p 5601:5601 \
  kibana:7.12.1
```

> 更详细的步骤（含 IK 分词器离线安装、扩展词典）见 [安装elasticsearch.md](./安装elasticsearch.md)。

### 3. 安装分词器插件

需要安装 **两个** 插件，版本必须与 ES 保持一致（7.12.1）：

1. [elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik/releases)
2. [elasticsearch-analysis-pinyin](https://github.com/medcl/elasticsearch-analysis-pinyin/releases)

将解压后的目录分别放入 ES 插件数据卷（`docker volume inspect es-plugins` 可查看路径），然后重启：

```bash
docker restart es
```

### 4. 部署 RabbitMQ

```bash
docker run \
  -e RABBITMQ_DEFAULT_USER=itcast \
  -e RABBITMQ_DEFAULT_PASS=123321 \
  --name rabbitmq \
  --hostname mq1 \
  -p 15672:15672 \
  -p 5672:5672 \
  -d \
  rabbitmq:3-management
```

管理界面：<http://localhost:15672>

### 5. 创建 ES 索引库

打开 Kibana 的 DevTools（<http://localhost:5601>），把 [酒店的mapping.txt](./酒店的mapping.txt) 中的 `PUT /hotel { ... }` 整段执行。

> ⚠️ 请**不要**用测试类里的 `testCreateIndex` 创建索引，那样创建出来的索引不支持自动补全。

### 6. 导入 MySQL 数据

新建数据库 `heima`，并执行 [tb_hotel.sql](./tb_hotel.sql)：

```sql
CREATE DATABASE IF NOT EXISTS heima DEFAULT CHARSET utf8mb4;
USE heima;
SOURCE tb_hotel.sql;
```

### 7. 修改配置

根据你的实际环境修改以下配置（仓库中的默认值只是作者本机/虚拟机环境）：

| 文件 | 配置项 |
| --- | --- |
| `hotel-demo/.../HotelDemoApplication.java` | ES 地址：`http://192.168.79.130:9200` |
| `hotel-demo/src/main/resources/application.yaml` | MySQL 地址 / 账号 / 密码、RabbitMQ 地址 / 账号 / 密码 |
| `hotel-admin/src/main/resources/application.yaml` | MySQL 地址 / 账号 / 密码、RabbitMQ 地址 / 账号 / 密码 |
| `hotel-demo/src/main/resources/static/index.html` | 前端请求后端的地址：`axios.defaults.baseURL = "http://localhost:8089"` |

> 🔐 配置文件中的账号密码是教学用的示例值，请在你自己的环境中修改，**不要把真实密码提交到 GitHub**。

### 8. 将 MySQL 数据导入 ElasticSearch

先启动 `hotel-demo`，再运行测试类 `HotelDocumentTest` 中的 `testBulkRequest` 方法，把 MySQL 中的酒店数据批量写入 ES 的 `hotel` 索引。

### 9. 启动项目

```bash
# 搜索服务（前台）
cd 完整版项目/hotel-demo
mvn spring-boot:run

# 管理服务（后台），另开一个终端
cd 完整版项目/hotel-admin
mvn spring-boot:run
```

### 10. 访问

| 页面 | 地址 |
| --- | --- |
| 前台搜索页面 | <http://localhost:8089/index.html> |
| 后台管理页面 | <http://localhost:8099/index.html> |
| Kibana | <http://localhost:5601> |
| RabbitMQ 管理台 | <http://localhost:15672> |

## 接口说明

### 搜索服务 `hotel-demo`（8089）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/hotel/list` | 搜索酒店列表（关键词、过滤、排序、分页、附近） |
| `POST` | `/hotel/filters` | 获取当前搜索条件下的品牌 / 城市 / 星级筛选项 |
| `GET` | `/hotel/suggestion?key=xx` | 输入自动补全 |

`/hotel/list` 请求体示例：

```json
{
  "key": "如家",
  "page": 1,
  "size": 5,
  "sortBy": "default",
  "city": "上海",
  "brand": "如家",
  "starName": "二钻",
  "minPrice": 100,
  "maxPrice": 500,
  "location": "31.21,121.5"
}
```

| 字段 | 说明 |
| --- | --- |
| `key` | 搜索关键词 |
| `page` / `size` | 页码 / 每页条数 |
| `sortBy` | 排序方式（前端页面有 默认 / 评分 / 价格 选项，当前后端尚未处理该字段，可自行扩展） |
| `city` / `brand` / `starName` | 城市 / 品牌 / 星级过滤 |
| `minPrice` / `maxPrice` | 价格区间 |
| `location` | 用户位置（`纬度,经度`），传入后按距离升序排序 |

### 管理服务 `hotel-admin`（8099）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/hotel/{id}` | 根据 id 查询酒店 |
| `GET` | `/hotel/list?page=1&size=10` | 分页查询酒店 |
| `POST` | `/hotel` | 新增酒店（并发送 MQ 消息） |
| `PUT` | `/hotel` | 修改酒店（并发送 MQ 消息，`id` 不能为空） |
| `DELETE` | `/hotel/{id}` | 删除酒店（并发送 MQ 消息） |

## ElasticSearch 实现要点

**索引库设计（`hotel`）**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | `text` | 索引时 `ik_max_word` + 拼音；搜索时 `ik_smart`；`copy_to` → `all` |
| `brand` / `business` | `keyword` | `copy_to` → `all` |
| `city` / `starName` | `keyword` | 用于过滤和聚合 |
| `price` / `score` | `integer` | 范围过滤、排序 |
| `location` | `geo_point` | 距离排序、附近搜索 |
| `address` / `pic` | `keyword`（不索引） | 仅用于展示 |
| `all` | `text` | 聚合字段，关键词搜索入口 |
| `suggestion` | `completion` | 自动补全，使用 `keyword` + 拼音分词 |

**核心查询技术**

- `bool` 查询：`must` 放关键词全文检索，`filter` 放品牌 / 城市 / 星级 / 价格范围
- `function_score`：对 `isAD = true` 的酒店加权（权重 10）实现广告置顶
- `geoDistanceSort`：按距离排序
- `terms aggregation`：生成品牌、城市、星级筛选项
- `completion suggester`：输入自动补全
- `bulk`：批量导入数据
- `highlight`：测试类 `HotelSearchTest#testHighlight` 中有高亮查询示例

## 常见问题

**Q：启动报错连不上 ES / RabbitMQ / MySQL？**
检查上面“修改配置”中的地址是否改成了你自己的环境，并确认容器已启动、端口已开放。

**Q：搜索不到拼音 / 自动补全没有结果？**
确认已安装 IK 和 pinyin 两个插件并重启 ES，且索引库是通过 `酒店的mapping.txt` 在 Kibana 中创建的。

**Q：后台修改酒店后前台搜索结果没变化？**
确认 RabbitMQ 已启动，并且 `hotel-demo` 与 `hotel-admin` 连接的是同一个 RabbitMQ。

**Q：MySQL 驱动报错？**
项目使用 `com.mysql.jdbc.Driver`（MySQL 5.x 驱动写法）。若使用 MySQL 8，请将驱动类改为 `com.mysql.cj.jdbc.Driver`，并在 URL 中补充 `serverTimezone` 参数。

## 声明

本项目为微服务 / ElasticSearch 的**学习与交流**用途，请勿用于商业用途。


