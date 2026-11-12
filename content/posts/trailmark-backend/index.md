+++
title = "履痕 TrailMark：旅行足迹系统的后端设计"
date = "2026-11-12T00:00:00+08:00"
draft = false
description = "以履痕的 Java 源码为例，介绍地图足迹系统的分层、数据模型、坐标约定、接口兼容与照片存储。"
summary = "履痕用 Spring Boot 和 MySQL 管理旅行足迹。本文介绍地点与行政区怎么建模，地图坐标怎么进出数据库，以及 Java 后端如何接替原来的 FastAPI 服务。"
categories = ["项目实践"]
tags = ["Java", "Spring Boot", "MySQL", "项目笔记"]
+++

履痕（TrailMark）用来记录到过的地方。比如一次旅行去了成都，用户可以把整座城市标为到访，再把博物馆、茶馆和徒步起点分别记成地图上的点。照片挂在具体的点下，回看时可以按年份、地区或标签找到那次旅行。

这两种记录回答的问题不同。行政区表示「到过哪里」，地点表示「具体在哪、做了什么」。在地图上标记一个城市，并不意味着用户到过它下面的每个区县。这个区别决定了后端的数据模型，也影响了列表和统计接口的设计。

## 应用分层与代码结构

主线后端在 `backend-java/`，使用 Java 21、Spring Boot 3.5.16、MyBatis-Plus 3.5.17 和 MySQL 8。原来的 FastAPI 实现仍放在 `legacy/backend-python/`，可以用来对照已有接口和坐标算法。Java 版依旧是一套服务：开发时由 Vite 转发 API 请求，生产模式下由 Spring Boot 同时提供 API 和前端构建产物。

主代码位于 `src/main/java/com/trailmark/footprint/`。读代码可以从 `MarkerController` 开始。它把请求绑定到 `MarkerDtos`，从认证上下文取用户 ID，再调用 `MarkerService`。服务负责坐标转换、数据归属和照片组装，通过 `MarkerMapper`、`PhotoMapper` 访问数据库。HTTP 返回的是 DTO，和表对应的实体留在持久化层。

| 目录 | 职责 |
| --- | --- |
| `controller/`、`dto/` | 定义请求和响应，承接参数校验与 JSON 契约 |
| `service/` | 处理足迹、照片、行政区、统计、搜索和导入导出 |
| `entity/`、`mapper/` | 映射 MySQL 表，执行查询和写入 |
| `security/`、`config/` | JWT 认证、分页插件、应用配置和前端资源托管 |
| `common/`、`util/` | 统一错误响应，提供坐标和时间工具 |

## 数据库怎么组织足迹

用户、地点、行政区和照片构成核心关系。地点和行政区各自关联用户。照片跟随地点，一个点可以有多张照片。照片表保存磁盘文件名、原始文件名、大小、上传时间和封面状态，图片内容单独落盘。

| 数据 | 设计和用途 |
| --- | --- |
| `users` | 邮箱唯一。记录密码哈希、启用状态和管理员标记 |
| `markers` | 按用户存地点坐标、访问日期、描述和标签。常用列表查询按用户和更新时间排序 |
| `regions` | 用 `(user_id, adcode)` 唯一键防止同一个用户重复标记同一行政区 |
| `photos` | 用 `marker_id` 关联地点。查询时先排封面，再排上传时间 |

几个索引直接来自实际读法。`markers` 的 `(user_id, updated_at)` 对应个人地图列表的过滤和排序，`photos` 的 `(marker_id, is_cover, uploaded_at)` 则对应按点查照片。返回多个地点时，服务先收集标记点 ID，再用一次 `IN` 查询取照片并按 ID 分组。这让附带照片的列表保持两次主数据查询，分页接口可能另发一条计数查询。

标签存在 MySQL 的 JSON 列里。用户输入关键词时，服务对名称、描述和标签文本做模糊匹配。用户勾选标签时，查询逐个加上 `JSON_CONTAINS` 条件，结果需要匹配全部所选标签。访问日期目前是字符串，年份筛选直接用前缀匹配，因此依赖统一的日期格式。数据量增大后，可以优先考虑让地图按视口取点，并检查标签查询和统计扫描的成本。

## 地图坐标与外部数据

履痕把 WGS-84 当作持久化坐标，把 GCJ-02 当作高德地图的交互坐标。用户在高德底图上点选位置，API 收到的是 GCJ-02，写库前转成 WGS-84。读取记录时按相反方向转换。这样更换底图或导出数据时，数据库里的值仍然有明确含义。

~~~java
// MarkerService 的坐标边界，省略 DTO 和其他字段。
var stored = CoordTransform.gcj02ToWgs84(
        request.longitude(), request.latitude());
marker.setLongitude(stored.lng());
marker.setLatitude(stored.lat());

var onMap = CoordTransform.wgs84ToGcj02(
        marker.getLongitude(), marker.getLatitude());
~~~

`CoordTransform` 的反向转换先用正向公式估算该位置的偏移，再从输入坐标减去偏移量。它是一次近似计算。仓库用 Python 版的固定样例校验输出，也测了往返误差。另一个容易忽略的地方是部分更新：经纬度应成对提交。当前服务只收到其中一项时会用库值补另一项，可能混用两套坐标。现有前端表单始终成对提交。

高德地点搜索也走单独的适配器。`AmapSearchService` 把 POI 转成统一的地点、地址、坐标和类别，最多返回 12 条，配置默认对有结果缓存一小时，对空结果缓存一分钟。返回的坐标已经是 GCJ-02。缺少 Key 时接口返回 503，上游查询异常则降级为空列表，因此用户仍可以在地图上手动标点。

导入导出走数据库这一侧。GeoJSON 的 Point、GPX 的航点和 KML 坐标都按 WGS-84 读入。导出支持 GeoJSON、KML 和 CSV，直接使用库中坐标。GPX 导入读取的是航点，当前没有把轨迹线展开成连续路径。个人统计则从标记点读取省市、访问日期和标签，按年份、标签以及去重后的省份汇总。大区图里同一个省有多个点时，仍然只算一个到访省份。

## 接口兼容与用户隔离

迁移时，前端仍然按 `access_token`、`visit_date` 和 `photo_count` 取数据。Java 里的 record 用驼峰命名，JSON 交给 Jackson 统一转换。涉及旧接口的特定字段，DTO 也会用 `@JsonProperty` 显式指定。

~~~yaml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
~~~

前端的错误提示读取 `detail`。MVC 层用全局异常处理器响应业务错误。认证失败和无权访问可能在过滤链就被拒绝，Spring Security 的两个处理器也按同样的结构写出 401 和 403。参数校验是迁移时主动做的一处调整：旧版的 422 和错误数组，在 Java 版改为 400 和可直接显示的字符串。

~~~java
@ExceptionHandler(ApiException.class)
public ResponseEntity<Map<String, Object>> handleApiException(ApiException ex) {
    return ResponseEntity.status(ex.getStatus())
            .body(Map.of("detail", ex.getMessage()));
}
~~~

登录使用 BCrypt 检查密码，用 HS256 签发 JWT。过滤器验签后仍查一次用户，确保启用状态和管理员角色以数据库当前值为准。这个设计多一次查询，换来禁用账号后下一次受保护请求就能生效。到了业务层，标记点用 ID 和当前用户 ID 一起查询。照片写操作通过关联的标记点复用同一归属检查。图片的读取路由目前允许直接引用 URL，这是与编辑权限分开的访问策略。

## 照片为什么不存进 MySQL

履痕把照片当作文件，MySQL 只存元数据。这种方式让 API 可以直接返回图片资源，查询地点和照片列表时也不需要把图片内容读进结果。部署和备份需要同时照顾数据库与上传目录，两边用磁盘文件名关联。

上传流程先校验标记点归属、文件类型和大小，再用 UUID 生成文件名，写入上传目录并插入元数据。上传并设为封面时，服务还会清除同一地点原有的封面状态。下面是现有实现的顺序，代码略去了参数检查和实体字段赋值。

~~~java
file.transferTo(target);
if (isCover) {
    clearCover(markerId);
}
photoMapper.insert(photo);
~~~

这个顺序有一个明确的失败边界：MySQL 事务回滚时，文件系统不会撤回已经完成的写入。插表或提交失败，可能留下无记录引用的文件。后续可以在写入前注册事务完成回调，只在确定回滚后清理本次文件，再用定期对账处理进程崩溃或清理失败。若要保证并发修改时每个点最多一张封面，还需要让所有封面写入操作按同一标记点串行执行。这两项是改进方向，当前源码未包含对应的补偿和锁定逻辑。

## 运行和验证

Java 服务启动时检查数据库，并初始化缺少的基础记录。表由 `sql/01_schema.sql` 创建，旧数据迁移通过 SQL 导入，照片文件单独复制。生产模式下，Spring Boot 会从 `frontend/dist` 提供静态文件。前端深链接回落到 `index.html`，而 `/api/**` 由后端路由处理。

改动坐标算法时，先跑 `CoordTransformTest`。改动 DTO 时，跑 `DtoJsonContractTest` 核对关键字段。服务启动后，可以用 `scripts/smoke_test.py` 验证登录、地图数据、照片和导入导出。脚本依赖测试库里的样例数据，地点搜索还需要高德 Key。顺着这些测试回到 `MarkerService` 和 `PhotoService`，就能看到数据库记录、地图显示和文件状态分别在哪里交接。
