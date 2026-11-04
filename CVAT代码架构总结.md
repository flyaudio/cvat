# CVAT 代码架构总结（develop 分支）

> 阅读对象：`D:\files\github\cvat`（CVAT develop 分支源码）
> 目的：为基于 CVAT 的二次开发建立代码地图
> 版本依据：`cvat-ui/package.json` v2.81.1、`cvat-core/package.json` v15.3.1、Python 3.12 / Django ~5.2
> 说明：本文所有结论均来自对仓库源码的阅读；标注了“推断/建议”的内容为二次开发视角的延伸，非源码事实。

---

## 1. 总览：这是一个什么形态的仓库

CVAT 是 **monorepo**：根目录下同时包含

- 一个 **Python 后端工程**（`cvat/`，Django + Django REST Framework）
- 多个 **JavaScript/TypeScript 前端包**（`cvat-ui` / `cvat-core` / `cvat-canvas` / `cvat-canvas3d` / `cvat-data`，通过 `yarn workspace` + `link:` 相互引用）
- 一个 **Python SDK**（`cvat-sdk/`）与命令行工具（`cvat-cli/`）
- 部署与运维配套（`docker-compose*.yml`、`supervisord/`、`helm-chart/`、`components/`）
- 自动标注函数（`serverless/`，Nuclio 部署的 AI 推理函数）

### 顶层目录地图

| 目录 | 作用 |
|---|---|
| `cvat/` | **后端主工程**：Django 项目（settings、urls）+ 全部业务 app |
| `cvat-ui/` | **前端主应用**：React SPA（标注界面、项目管理界面） |
| `cvat-core/` | 前端核心 JS 库：封装后端 REST API、定义客户端对象模型与标注业务逻辑 |
| `cvat-canvas/` | 2D 标注画布库（SVG 渲染 + 交互） |
| `cvat-canvas3d/` | 3D 标注画布库（Three.js） |
| `cvat-data/` | 媒体数据解码库（浏览器内视频解码 mp4.js/avc.wasm、zip 解包 worker） |
| `cvat-sdk/` | Python SDK（OpenAPI 生成客户端 + 数据集/自动标注工具） |
| `cvat-cli/` | Python CLI（命令行调用 CVAT API） |
| `serverless/` | Nuclio serverless 推理函数（onnx/openvino/pytorch/tensorflow） |
| `utils/` | 辅助工具（dataset_manifest 生成器、dicom 转换等） |
| `supervisord/` | 容器内进程管理（nginx、RQ worker 池） |
| `components/` | 配套组件（analytics: clickhouse/vector/grafana 配置等） |
| `tests/` | 端到端测试（playwright 等） |
| `docker-compose*.yml` | 一键部署编排 |

### 技术栈速查

**后端**：Python 3.12 · Django ~5.2 · DRF ~3.17 · django-rq（Redis 队列）· Datumaro（CVAT 维护的 fork，负责格式转换/数据模型）· django-allauth + dj-rest-auth（认证）· Open Policy Agent（RBAC）· ClickHouse + Vector + Grafana（事件分析）· PostgreSQL 15 · Redis（内存缓存/队列）· Kvrocks（磁盘缓存）

**前端**：TypeScript · React 18.2 · Redux + redux-thunk · Ant Design 5 · react-router 5 · Webpack 5 · axios + tus-js-client（断点续传上传）· onnxruntime-web（浏览器端模型推理）· OpenCV.js（浏览器端图像处理）· Three.js（3D）

---

## 2. 后端架构（`cvat/`）

### 2.1 Django 工程组织

```
cvat/
├── manage.py / rqscheduler.py / backend_entrypoint.sh   # 入口
├── urls.py          # 根路由：admin/、engine.urls、redis_handler、django-rq，以及按 INSTALLED_APPS 条件挂载的 app 路由
├── settings/        # base.py / development.py / production.py / testing.py / email_settings.py
├── asgi.py / nginx.conf / schema.yml
├── requirements/    # 依赖分层（base/all/development/production/testing）
└── apps/            # 全部业务应用
```

根 `urls.py` 的路由挂载方式很有代表性——**app 是否生效取决于 INSTALLED_APPS**（如 `log_viewer`、`events`、`lambda_manager`、`webhooks`、`quality_control`、`consensus`、`access_tokens`、`growth` 都是条件挂载），这是 CVAT 用"可裁剪 app"组织功能的典型手法，二次开发新增功能时可以沿用。

### 2.2 app 划分（cvat/apps/）

| app | 职责 |
|---|---|
| `engine` | **核心 app**：数据模型、ViewSet、任务创建管线、媒体服务、缓存、权限基类 |
| `iam` | 身份认证与授权：allauth 集成、自定义认证类、**Rego 规则（RBAC）**、Organization 过滤 |
| `organizations` | 组织/团队：成员、邀请、角色 |
| `dataset_manager` | 数据集导入导出：Datumaro 集成、格式注册表、Task/Project 管理、标注中间表示 |
| `dataset_repo` | （遗留/兼容）数据集 git 仓库支持 |
| `lambda_manager` | 自动标注函数管理：Nuclio serverless 函数列表/调用/结果收集 |
| `webhooks` | Webhook 订阅与投递 |
| `events` | 事件日志：写 ClickHouse、异常处理 |
| `quality_control` | 质量检测：冲突、质量报告、质量需求/设置（Datumaro 后端） |
| `redis_handler` | RQ 任务基础设施：RequestId 体系、任务状态/进度、cron 清理 |
| `health` | 健康检查命令 |
| `log_viewer` | 日志分析（可选，CVAT_ANALYTICS 时启用） |
| `consensus` | 共识管理（多人标注一致性） |
| `access_tokens` | 个人访问令牌 + **PolicyEnforcer（OPA 授权执行点）** |
| `growth` | A/B 实验/增长数据 |

### 2.3 settings 要点（base.py）

- `INSTALLED_APPS` 用列表拼装：基础 + 可选（`ANALYTICS_ENABLED` 时追加 `log_viewer`）
- `AUTH_USER_MODEL = "iam.User"` —— 用户模型在 iam app 内自定义
- `REST_FRAMEWORK` 全局约定（二次开发写新接口时继承这些行为）：
  - 认证：Token → AccessToken → Session → Basic（Basic 是自定义扩展）
  - 权限默认：`IsAuthenticated` + `PolicyEnforcer`（所有 API 默认经过 OPA 策略执行）
  - 分页：`engine.pagination.CustomPagination`（默认 PAGE_SIZE=10，上限 500）
  - 过滤后端：SimpleFilter / SearchFilter / OrderingFilter / **JsonLogicFilter**（前端复杂查询） / OrganizationFilterBackend
  - 版本化：AcceptHeaderVersioning，版本 2.0
  - Schema：`iam.schema.CustomAutoSchema`（drf-spectacular 定制）
  - 异常处理：`events.handlers.handle_viewset_exception`（统一转事件日志）
- 队列：`CVAT_QUEUES` 枚举定义 9 个 RQ 队列名：**import / export / annotation（自动标注）/ webhooks / notifications / quality_reports / cleaning / chunks / consensus**
- 存储：`DATA_ROOT`（`/data`，任务媒体与 chunk 落盘）、`MEDIA_DATA_ROOT`
- Redis 双库：`cvat_redis_inmem`（Redis，队列+缓存）与 `cvat_redis_ondisk`（Kvrocks，磁盘缓存）

### 2.4 API 路由

`cvat/apps/engine/urls.py` 用 DRF DefaultRouter 注册：

```
/api/projects        ProjectViewSet
/api/tasks           TaskViewSet
/api/jobs            JobViewSet
/api/users           UserViewSet
/api/server          ServerViewSet   （about/share/annotation_formats/plugins 等）
/api/issues          IssueViewSet
/api/comments        CommentViewSet
/api/labels          LabelViewSet
/api/cloudstorages   CloudStorageViewSet
/api/assets          AssetsViewSet
/api/guides          AnnotationGuidesViewSet
```

另挂载：`/api/auth/*`（登录注册）、`/api/organizations/*`、`/api/events/*`、`/api/webhooks/*`、`/api/quality/*`、`/api/functions|requests`（lambda）、`/api/consensus/*`、`/api/tokens|policies`（access_tokens）等。OpenAPI 文档：`/api/schema/`、`/api/swagger/`、`/api/docs/`。

### 2.5 核心数据模型（engine/models.py）

模型体系是理解 CVAT 的钥匙，层次关系：

```
Organization（organizations app）
└── Project（项目：任务容器，统一定义标注规范 labels）
    ├── Task（任务：一批数据 + 子集标注规范）
    │   ├── Data（媒体描述：帧数/尺寸/格式/存储位置/chunk 配置）
    │   │   ├── Image / Video / Audio（媒体实体）
    │   │   ├── ClientFile / ServerFile / RemoteFile / RelatedFile（数据来源）
    │   │   └── CloudStorage（云存储数据源）
    │   └── Segment（片段，按任务切分策略生成）
    │       └── Job（作业：标注单元，可分配给标注员）
    ├── Label（标签，树形：parent 支持子标签/skeleton 结构）
    │   └── Skeleton（骨架标签：sublabels 定义关键点）
    ├── AttributeSpec（标签属性定义：select/radio/checkbox/text/number）
    └── ...
```

**标注规范**：Project 级定义 Label → AttributeSpec；Task 可覆盖；Skeleton 用子标签定义关键点集。

**标注数据**（多表模式，而非 JSON 字段）——CVAT 标注的物理存储设计，值得二次开发时重点理解：

- `LabeledImage`（图片标签/分类）、`LabeledShape`（形状）、`LabeledTrack`（轨迹）、`LabeledInterval`（音频区间）
- `TrackedShape`（轨迹的每个关键帧形状）+ `ShapeAnnotationMixin`（点集/形状几何）
- 各类 `AttributeVal`（属性值）
- `SourceType`（manual / auto / acceptance 等来源标记）、`ScoredAnnotationMixin`（自动标注置信度）
- 派生：`Issue` / `Comment`（人工审核批注）、`AnnotationGuide` / `Asset`（标注指南）

**其他关键模型**：`CloudStorage`（S3/Azure/GCS 等）、`Storage`（源/目标存储抽象）、`ValidationParams/ValidationLayout/ValidationFrame`（验证任务/金标准）、`Profile`（用户偏好）。

### 2.6 视图层（DRF ViewSet 模式）

每个资源一个 ViewSet（`engine/views.py`），典型结构：

- `ProjectViewSet` / `TaskViewSet` / `JobViewSet` 继承 `(mixins, viewsets.ViewSet)` 组合
- 自定义 action 承载复杂操作：
  - `TaskViewSet.data`（上传/获取任务数据）、`annotations`（获取/更新标注，支持 bulk）、`export_dataset` / `import_dataset`（异步导入导出）、`backup`（项目/任务备份）
  - `JobViewSet.data`（**按 chunk 取帧数据，是前端渲染的食粮**）、`annotations`、`issues`、`commit`（提交）
  - `ServerViewSet.about` / `share` / `annotation_formats` / `plugins`
- 关键机制：`_DataGetter` 抽象（`_TaskDataGetter` / `_JobDataGetter`）统一从 **文件系统 / 云存储 / 内存缓存** 取帧；`Range` 头支持分帧请求（`_RangeHeaderSyntaxError` 等用于视频范围请求）

### 2.7 权限体系（IAM：认证 + Rego 规则授权）

这是 CVAT 一个特色且重要的架构点：

- **认证**：allauth 体系 + 自定义 `LoginViewEx`/`RegisterViewEx`；`BasicAuthenticationEx` 等自定义认证类
- **授权 = 两层**：
  1. DRF 默认权限 `IsAuthenticated` + `access_tokens.permissions.PolicyEnforcer`
  2. PolicyEnforcer 向 **OPA（Open Policy Agent）** 发起查询，OPA 的策略包（Rego 规则）由 `/api/auth/rules` 提供（`iam/rules/utils.rego`）
- **Rego 规则体系**：`iam/rules/utils.rego` 定义了完整的 scope 枚举（如 `view:annotations`、`update:annotations`、`create@job`、`update:assignee`、`upload:data` 等），各 app 的 `rules/` 目录（`engine/rules`、`webhooks/rules`、`quality_control/rules`……）按资源定义"谁在什么上下文下可以做什么"。**新增接口/新增权限维度 = 写 Rego 规则**
- **Organization 维度**：`OrganizationFilterBackend` 让所有查询自动按用户所属组织过滤
- 角色模型：`admin / user / worker`（每组织内）、对象可见性 `public / private / internal`

### 2.8 异步任务体系（RQ + Redis）

CVAT 把耗时操作全部异步化，架构分为三层：

1. **队列**（settings.RQ_QUEUES）：9 个队列（见 2.3），各队列超时/进度上报策略不同（如 import/export 4h、自动标注 24h、webhooks 25s）
2. **任务对象**：`redis_handler/rq.py` 的 `RequestId` 体系——每个请求有唯一 ID、关联 user/org/project/task/job，状态与进度写入 Redis；`redis_handler/background.py` 管理请求生命周期；前端通过 `/api/requests` 轮询进度
3. **worker 进程**：`supervisord/worker*.conf` + docker-compose 中的独立 worker 容器（见第 4 节）

**任务创建管线**（`engine/task.py`，核心流程，二次开发常改这里）：

```
TaskViewSet.data 请求
  → 校验数据（文件计数、manifest、帧过滤、视频/图像/音频类型探测）
  → 下载数据（本地/共享目录/云存储/URL，支持 manifest）
  → 切分 Segment、创建 Job（含验证任务 validation jobs、honeypot 金标准帧）
  → 生成媒体描述（Image/Video/Audio 记录）
  → 生成 chunk（媒体分块缓存，chunks 队列异步执行）
  → 生成任务预览图
  → 通知完成（webhook/事件）
```

### 2.9 媒体服务与缓存（engine/media_io + cache）

- `media_io/media_provider.py`：媒体访问抽象
- `media_io/frame_provider.py`：**帧提供器**（按 chunk 取帧，支持图像/视频）
- `media_io/media_chunks.py`：chunk 读写（chunk 是 CVAT 媒体缓存的核心单位：视频抽帧、图像打包成压缩 chunk）
- `media_io/audio_provider.py`：音频波形/片段
- `engine/cache.py`：Redis 内存缓存（热数据）；磁盘缓存用 Kvrocks（冷数据）
- `frame_provider` 同时被后端视图（`_DataGetter`）和 lambda_manager（自动标注取帧）复用

### 2.10 数据集导入导出（dataset_manager）

基于 **Datumaro**（CVAT fork）构建，架构：

- `formats/registry.py`：**格式注册表**——`COCO / PASCAL VOC / YOLO / MOT / LabelMe / Cityscapes / KITTI / CamVid / ICDAR / ImageNet / OpenImages / TFRecord / CVAT 自身 / Datumaro / 音频 TSV / 点云 / 自定义` 等 20+ 格式，每个格式一个 adapter 模块
- `annotation.py`：`AnnotationIR`（**标注中间表示**：tags/shapes/tracks/intervals）+ `AnnotationManager`/`TrackManager`（track 拆分、合并、压缩）
- `bindings.py`：`TaskManager` / `ProjectManager` 将 DB 模型 ↔ Datumaro 数据集 ↔ 外部格式互转；导出时任务按 chunk 流式拉取标注（`LazyList`、按帧流式 getter），避免大任务内存爆炸
- `task.py` / `project.py`：导入导出编排（异步 RQ：import/export 队列），含 `dumps` 的格式参数（图像质量、保存图片、压缩等）
- 关键设计：**导出走"数据库 → AnnotationIR → Datumaro Dataset → 目标格式"**；导入反向。要加新标注格式 = 在 registry 注册新 adapter

### 2.11 质量检测（quality_control）

- 后端基于 Datumaro 的 `QualityCalculator`（`backends/datumaro`），计算 GT（ground truth）与标注的匹配质量
- 模型：`QualityReport`、`QualityConflict`（人工审核冲突）、`QualityRequirement`（质量门槛）、`QualitySettings`、`QualityReportTarget`（project/task/job 级）
- 前端 `cvat-ui/src/components/quality-control` + `cvat-core/src/quality/` 对应实现
- 金标准来源：validation jobs（验证任务）+ honeypot（隐藏金标准帧）

### 2.12 事件与埋点（events + ClickHouse）

- `events/`：`handle_viewset_exception` 统一异常处理、`handle_function_call`（自动标注调用事件）等
- 日志流：Django → Vector → ClickHouse；Grafana 可视化（`/analytics`）
- 开关：`CVAT_ANALYTICS=1` 时启用 `log_viewer` app

### 2.13 自动标注（lambda_manager + serverless + cvat-sdk）

两条路线并存：

1. **外部函数**（serverless/，Nuclio 部署）：`lambda_manager` 通过 `LambdaGateway` 调用 Nuclio（`/api/functions` 列表、dashboard/direct 两种 invoke 方式），`FunctionViewSet`/`RequestViewSet` 暴露函数管理与调用请求；`DetectionResultConverter`/`DetectionResultCollector` 把函数返回的检测结果转成 CVAT 标注（shape/track）
2. **SDK 自动标注**（`cvat-sdk/cvat_sdk/auto_annotation/`）：`driver.py` + torchvision 函数族（classification/detection/instance_segmentation/keypoint_detection），走 CVAT 的函数调用接口（`/api/functions`）

---

## 3. 前端架构

### 3.1 前端包依赖关系

```
cvat-ui（React 应用）
  ├── cvat-core（API 封装 + 业务模型）←── cvat-canvas3d 也依赖它
  ├── cvat-canvas（2D 画布）
  ├── cvat-canvas3d（3D 画布，Three.js）
  └── cvat-data（媒体解码，被 cvat-core 依赖）
```

前端与后端之间的"唯一通道"是 **REST API（/api/）**，cvat-ui 不直接发请求，一律通过 cvat-core 封装。

### 3.2 cvat-core：前端核心库（业务大脑）

`cvat-core/src/` 关键模块：

| 模块 | 职责 |
|---|---|
| `api.ts` | **公共 API 面**：构建 `cvat` 单例对象，所有方法经 `PluginRegistry.apiWrapper` 包裹（插件可拦截/装饰每个 API 调用） |
| `api-implementation.ts` | API 实现（转发到 server-proxy） |
| `server-proxy.ts` | **REST 客户端**：axios + FormData + tus-js-client（断点续传上传）；序列化类型定义（server-response-types/server-request-types） |
| `session.ts` | `Session` 基类 → `Job` / `Task`（标注会话核心，含 frameData、annotations、save 等） |
| `project.ts` / `project-implementation.ts` | 项目模型 |
| `annotations-objects/` | **标注对象类体系**：`shape.ts`/`track.ts`/`tag.ts` 基类 + 具体类型（rectangle/polygon/polyline/points/ellipse/cuboid/mask/skeleton × shape/track）+ mixin（`pinnable`、`scored`、`drawn`） |
| `annotations.ts` | 标注集合与操作（增删改、合并、过滤） |
| `annotations-saver.ts` | 标注保存策略（全量/增量 PATCH） |
| `annotations-filter.ts` | JSON Logic 过滤（与后端 JsonLogicFilter 对应） |
| `annotations-history.ts` | 撤销/重做历史 |
| `annotations-actions/` | 批量动作：传播形状、简化多边形、删除过滤结果等 |
| `object-state.ts` | 画布 ↔ 标注对象的状态桥（clientID 体系） |
| `opencv/` | 浏览器端 OpenCV.js：智能剪刀（intelligent-scissors）、MIL 跟踪器（tracker-mil）、直方图均衡 |
| `quality/` | 质量模块前端模型 |
| `plugins.ts` | **插件注册表**：`PluginRegistry.register(plugin)` + `apiWrapper`（enter/leave 钩子、可阻止原方法），是官方提供的"不改源码扩展 API"机制 |
| `enums.ts` / `config.ts` / `exceptions.ts` / `logger.ts` | 基础设施 |

核心调用链（以"取一帧数据"为例）：

```
cvat-ui action → cvat.server.proxy... → cvat-core api.ts → api-implementation → server-proxy → axios → REST /api/jobs/:id/data
```

### 3.3 cvat-canvas：2D 标注画布（MVC）

`cvat-canvas/src/typescript/` 是经典 MVC：

- `canvas.ts`：对外 `Canvas` 接口 + `CanvasImpl`（组合 Controller/Model/View）
- `canvasController.ts`：交互控制（模式切换：idle/draw/edit/merge/group/split/slice/interact/select/zoom）
- `canvasModel.ts`：数据模型（frameData、objectStates、SVG 元素管理）
- `canvasView.ts`：SVG 渲染
- 各 handler：`drawHandler`（绘制各形状）、`editHandler`（编辑点）、`selectHandler`、`mergeHandler`、`groupHandler`、`splitHandler`、`sliceHandler`、`interactionHandler`（交互式标注：多边形追踪/智能剪刀）、`zoomHandler`、`autoborderHandler`（自动吸附边缘）、`masksHandler`（mask 编辑）、`cuboid.ts`（3D 长方体在 2D 的投影）、`paste.ts`（跨帧复制粘贴）

Canvas 公共 API（`setup`/`draw`/`edit`/`select`/`merge`/`group`/`split`/`slice`/`zoom`/`fit`/`activate` 等）就是"前端如何驱动画布"的契约。

### 3.4 cvat-ui：React 应用

**状态管理**：Redux + redux-thunk。目录分层严格对应 Redux 惯例：

```
src/
├── index.tsx            # 入口：createCVATStore + 应用初始化动作（认证/关于/格式/插件/模型/组织...）
├── cvat-store.ts        # store 创建（dev 模式挂 redux-logger）
├── cvat-core-wrapper.ts # 初始化 cvat-core 单例、注册插件
├── cvat-canvas-wrapper.ts / cvat-canvas3d-wrapper.ts  # 画布实例工厂
├── actions/             # 30+ 组 thunk actions（annotation/tasks/projects/jobs/models/quality...）
├── reducers/            # 与 actions 一一对应的 reducers + root-reducer
├── containers/          # 页面级容器（connect 层）
├── components/          # 展示组件（按页面组织）
└── utils/               # 工具（dayjs 等）
```

**主要页面/组件**（`components/` 下）：

- 认证：`login-page` / `register-page` / `reset-password-*`
- 项目管理：`projects-page` / `project-page` / `create-project-page` / `tasks-page` / `task-page` / `create-task-page` / `jobs-page` / `job-item`
- 标注：`annotation-page`（核心，见下）
- 数据：`cloud-storages-page` / `create-cloud-storage-page` / `import-dataset` / `export-dataset` / `export-backup` / `import-backup` / `file-manager`
- 系统：`models-page`（ML 模型）、`webhooks-page`、`organization-page`、`invitations-page`、`requests-page`（任务进度）、`quality-control`、`consensus-management-page`、`settings-modal`（快捷键/画布/工作区设置）、`shortcuts-dialog`

**标注页剖析**（`components/annotation-page` + `containers/annotation-page`）：

- 四种工作区（workspace）：`standard-workspace`（标准）、`attribute-annotation-workspace`（属性标注）、`review-workspace`（审核）、`single-shape-workspace`（单形状）
- 画布视图：`canvas/views/canvas2d`、`canvas3d`、`context-image`（上下文图）；`canvas/grid-layout`（画布网格布局）
- 侧栏：`controls-side-bar`（工具按钮：绘制/编辑/合并/分组/切片/分割/交互式标注等，每形状一个 control 组件）、`objects-side-bar`（对象列表/属性编辑）
- 顶部栏 `top-bar`（帧导航、播放、保存、撤销重做、任务切换）
- 审核：`review/`（issue 创建/解析、冲突标签）
- 标注操作通过 `containers/annotation-page/...` 的 connect 层把 Redux 状态注入

**Redux 标注状态流**（annotation-actions → annotation-reducer）：

```
切换帧/激活对象 → annotation-actions（thunk）→ cvat-core session（Job/Task）取数据
→ dispatch 更新 annotation-reducer → 容器重渲染 → canvas.setup(frameData, objectStates)
用户绘制 → canvas 事件回调 → action 更新对象状态（object-state）→ 同步 cvat-core 对象
保存 → annotations-saver 策略 → server-proxy PATCH /api/jobs/:id/annotations
```

### 3.5 cvat-data：媒体解码

- `mp4.js + avc.wasm`：**浏览器内 H.264 解码**，CVAT 视频任务不依赖后端逐帧出图，而是按 chunk 提供 mp4 片段由前端解码渲染（配合 Range 请求）
- `unzip_imgs.worker.ts`：图像 chunk 解包
- `cvat-data.ts`：对外接口（ChunkQuality 等）

### 3.6 3D 与音频

- `cvat-canvas3d`：Three.js 场景，支持点云/3D cuboid 标注；通过 `cvat-core` 与后端交互
- `cvat-ui/src/audio`：音频波形（wavesurfer.js）与区间标注（`cvat-core/annotations-objects/audio-interval*`）

---

## 4. 部署架构（docker-compose）

```
浏览器
  │
  ▼
traefik（8080 反向代理）
  ├── /  → cvat_ui（nginx 静态站点，端口 8000）
  └── /api/ /static/ /admin /django-rq → cvat_server（Django + mod_wsgi + nginx，8080）
                                          │
      ┌─────────────┬──────────────┬──────┴──────────┬───────────────┐
      ▼             ▼              ▼                 ▼               ▼
 cvat_db        cvat_redis     cvat_redis       cvat_clickhouse  cvat_opa
 (Postgres15)   _inmem(Redis)  _ondisk(Kvrocks)  +vector+grafana  (OPA 策略)
      │
      └── RQ 队列（Redis） → 各 worker 容器：
          cvat_worker_utils（notifications/cleaning）
          cvat_worker_import / export（--with-scheduler 定时任务）
          cvat_worker_annotation（自动标注）
          cvat_worker_webhooks（--with-scheduler）
          cvat_worker_quality_reports
          cvat_worker_chunks
          cvat_worker_consensus
```

要点：
- **前后端分离部署**：`cvat_server` 只服务 API；`cvat_ui` 是独立 nginx 容器
- **Worker 按队列拆分容器**，可独立水平扩展；`supervisord/worker.conf` 在容器内启动多个 rqworker 进程（`NUMPROCS`）
- OPA 独立容器，从 `/api/auth/rules` 拉取策略 bundle
- 开发模式：`docker-compose.dev.yml` + `cvat-ui` 本地 `webpack serve`（代理到后端 7000 端口）

---

## 5. 关键端到端流程

### 5.1 创建任务

```
前端 create-task-page → actions → cvat-core task.create()
→ POST /api/tasks（异步）→ 后端 TaskViewSet.data → task.py initialize_task
→（后台 RQ chunks/import 队列）下载数据、切分 Segment/Job、生成 chunk 与预览
→ 前端 requests-page 轮询 /api/requests 进度 → 完成
```

### 5.2 打开任务标注

```
annotation-page → cvat-core Job/Task 会话 → 拉 job 元数据 + 首帧 data
→ 后端 _JobDataGetter 按 chunk 返回帧（图像 chunk 解包 / 视频 chunk 走 mp4 流式解码）
→ canvas.setup(frameData, objectStates) → 渲染
```

### 5.3 保存标注

```
前端对象状态变化 → annotation-saver 计算增量 → PATCH /api/jobs/:id/annotations
→ 后端 LabeledDataSerializer 校验 → 写入 LabeledShape/LabeledTrack/AttributeVal 等表
→ 触发信号（events/webhooks/quality）
```

### 5.4 导入/导出数据集

```
导出：POST /api/tasks/:id/export_dataset → export 队列 → bindings.TaskManager.dumps
      → AnnotationIR → Datumaro → 目标格式（zip）→ 下载
导入：POST /api/tasks/:id/import_dataset → import 队列 → 格式探测 → Datumaro → DB
```

---

## 6. 二次开发切入点（建议）

基于上述架构，二次开发最值得利用的扩展点：

### 后端
1. **新增标注格式**：`dataset_manager/formats/` 注册新 adapter（继承 `registry.py` 的格式定义 + 实现 dump/load），自动出现在 `/api/server/annotation_formats`
2. **新增 API 资源**：engine 下建 ViewSet + 注册进 router；权限通过 `iam/rules` 写 Rego 规则（或复用现有 scope）
3. **新增自定义逻辑钩子**：`engine/signals.py`、`events/handlers.py`（事件）、`webhooks`（外部通知）
4. **新增 RQ 队列/worker**：settings.RQ_QUEUES 增加队列 + docker-compose 加 worker 容器 + `redis_handler` 的 RequestId 子类定义进度模型
5. **接入自有 AI 标注**：优先走 `serverless/`（Nuclio 函数）或 `cvat-sdk/auto_annotation`（driver + function），避免侵入核心
6. **存储/媒体适配**：`engine/media_io` 的 provider 抽象 + `CloudStorage`/`Storage` 模型支持自定义存储

### 前端
1. **不改源码扩展 API**：cvat-core `PluginRegistry.register()`（enter/leave 钩子包装任意 API 方法）——官方设计的最干净扩展点
2. **新增 UI 模块**：components/ 下建目录 + actions/reducers 配套（Redux 分层非常规整，照抄现有模块即可）
3. **自定义画布交互**：cvat-canvas 新增 handler（参考 drawHandler/editHandler），通过 Canvas 接口暴露，UI 侧加 control
4. **自定义标注对象类型**：`cvat-core/annotations-objects/` 增加 shape/track 子类 + 后端 models 对应扩展 + 画布渲染

### 推荐阅读顺序（从 0 到 1）
1. `cvat/apps/engine/models.py`（数据模型是根）
2. `cvat/apps/engine/views.py` 的 `TaskViewSet`/`JobViewSet`（API 行为）
3. `cvat/apps/engine/task.py` 的 `initialize_task`（任务创建管线）
4. `cvat-core/src/api.ts` + `session.ts`（前端如何抽象后端）
5. `cvat-ui/src/containers/annotation-page/annotation-page.tsx` + `components/annotation-page`（UI 如何组织）
6. `cvat-canvas/src/typescript/canvasController.ts`（画布交互）
7. `cvat/apps/iam/rules/`（权限规则，改权限前必读）

---

## 7. 附：版本与依赖速查

| 组件 | 版本/依赖 |
|---|---|
| Python | 3.12（requirements/.python-version） |
| Django | ~5.2.0 |
| DRF | ~3.17 |
| RQ | 1.16.0 + django-rq ≥2.10 |
| Datumaro | cvat-ai fork（GitHub zip 引用） |
| OpenCV | opencv-python-headless ~4.8 |
| 前端 React | 18.2.0 |
| Redux | 4.1.1 + redux-thunk |
| UI 库 | Ant Design 5.17 |
| 画布 | 自研 SVG（2D）/ Three.js 0.184（3D） |
| 媒体解码 | mp4.js + avc.wasm（浏览器内） |
| 数据库 | PostgreSQL 15 / Redis 7 / Kvrocks |
| 策略引擎 | Open Policy Agent 1.12 |
