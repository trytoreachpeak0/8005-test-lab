# 三个被测进程的可观测面盘点

> **这是 2026-09-04 的快照，不回改。**各仓 HEAD 见下一段，都已经往前走了。至少三处被后来的实测
> 改掉，全部见 [#3](https://github.com/trytoreachpeak0/8005-test-lab/issues/3) 与
> [#9](https://github.com/trytoreachpeak0/8005-test-lab/issues/9) 的修正评论：车载端
> `127.0.0.1:58007` 的自动化面**今天在 agv01 上可用**（本文记的是 Production 守卫拒绝启动，
> 那个判断在当时是对的，`8005-agv-onboard-hmi` PR #15 修好了它并已于 2026-09-08 合并）；
> 服务端到车载端的消息链在 `ProtocolInbox.RequestJson` 里带完整的 `messageId`/`correlationId`；
> 而 agv01 上的 `automation.enabled` 今天是 `true`。以票的决议为准，本文只作为当时盘点结果的记录。


研究票据：[#3](https://github.com/trytoreachpeak0/8005-test-lab/issues/3)（`wayfinder:research`，属于 #1）。
盘点日期 2026-09-04。核对时各仓 HEAD：`8005-agv-control-server` `90e8fc3`、
`8005-agv-onboard-hmi` `550dbe9`（分支 `w2g/recovery-entry-mid-session-readiness`）、
`slots-simulator` `fb5f7c5`、`8005-agv-protocol` `e54e988`。所有引用写成
`仓库/路径:行号`，路径相对该仓根；`remote-ops/` 相对工作区根。

## 1. 问题

票据要的是 WIRE_TO_GATE 三个进程**今天**的可观测面，作为日志关联与控制面设计的事实底座：
三个进程（`ControlServer.Host`、车载端 `SQCD.Agv.Wpf`、`slots-simulator`）各自把日志写到哪、
什么格式、是否结构化、协议信封里的 `correlationId` / `messageId` 有没有落进日志行；每个进程的
配置有没有能塞一个外部 `runId` 的口（环境变量、appsettings、命令行各自支持哪些）；客户端
`127.0.0.1:58007/api/v1`、模拟器 `127.0.0.1:58006/api/v1`、`ControlServer.FakeRiot` 与
`ControlServer.FakeMesIngest` 的 `/control/v1` 的全部端点及 openapi 文档位置；以及服务端 SQLite
里 lab 会读的表，以今天 L2 断言用到的那几张为准。

## 2. 一页结论

| | ControlServer.Host | SQCD.Agv.Wpf（车载端） | slots-simulator |
| --- | --- | --- | --- |
| 日志去哪 | 代码里只有 Console sink；产品安装时由安装脚本生成 `appsettings.Production.json` 加 File sink，写 `<DataRoot>\logs\controlserver-.ndjson`；L2 下 stdout 重定向到 `logs/control-server.out.log` | 自写的 `FileAppLogger`，`C:\8005\OnboardHmi\logs\agv-{yyyyMMdd}.log` | **没有日志文件**。只有窗口内的 `ListBox`，500 行封顶，退出即丢 |
| 格式 | Console：Serilog 默认文本模板；文件：`CompactJsonFormatter` NDJSON | TSV：`{Timestamp:O}\t{Severity}\t{Source}\t{Message}`，非结构化 | `HH:mm:ss.fff  {message}`，无日期，内存字符串 |
| 结构化 | 是（Serilog，`Enrich.FromLogContext`），但业务上**没有任何** `PushProperty`/`BeginScope` | 否 | 否 |
| `correlationId` / `messageId` 在日志里 | 否。仅 EventId 1101 拒收事件带 `RejectedMessageId` | 否。所有 `messageId=` 文本都在被 `DisabledRuleGateway` 替换掉的旧规则网关路径上 | 否（模拟器不在 W2G 协议上，本来就没有信封） |
| `runId` 不改代码能否注入 | **能**：`ReadFrom.Configuration` 已接，`Serilog__Properties__RunId=<v>`（env）或 `--Serilog:Properties:RunId=<v>` 即给每条事件打属性；但只在 CompactJson 文件 sink 里可见，Console 默认模板不打印 `{Properties}` | **不能**：无 env、无 CLI，只读 exe 旁的 `appsettings.json`；能改的自由字符串只有 `agvId`（但它是协议身份，服务端会校验）。自动化平面自己的 `RunId` 是每次启动的随机 GUID | **不能**（同样无 env/CLI）。唯一自由字符串是 `simulator.settings.json` 的 `instanceId`，会回显在每个 HTTP 响应里但不进日志；`runId` 是每次 reset 的随机 GUID |
| 控制面端点数 | 5 个 GET（`/health/live`、`/health/ready`、`/version`、`/api/runtime/sessions`、`/api/onboard/v1/vehicle-safety`），无管理 API | `/api/v1` 下 3 个 + `/openapi/v1.json` | `/api/v1` 下 8 个 + `/openapi/v1.json` |
| 替身 `/control/v1` | FakeRiot 8 个、FakeMesIngest 7 个、FakeOnboard 6 个、ClockSkewProxy 5 个；每个响应都带 `instanceId` / `runId` / `revision`，`instanceId` 可从 CLI 注入，`runId` 不可 | | |

两个横切事实：

1. 协议信封（`8005-agv-protocol/schemas/envelope.schema.json`）里 `messageId` 与 `correlationId`
   都是必填字段，但三个进程**没有一个**把它们写进日志行。关联链今天只存在于 ControlServer 的
   SQLite（`ProtocolInbox` / `ProtocolOutbox` 以 `MessageId` 为主键）与车载端 SQLite journal
   之间，日志不参与。
2. L2 编排器的 `$runId` 今天只到达 ControlServer 一处，且不是日志：
   `JourneyRuntime__admissionPolicyDeploymentId = "L2-$runId"`，落在 `AdmissionPolicyState.DeploymentId`
   （`8005-agv-control-server/scripts/l2/Invoke-L2Scenario.ps1:286`）。两个替身拿到的是常量
   `l2-riot` / `l2-mes`（`:192,205`），车载端与模拟器什么都拿不到。

还有一个**矛盾**必须先说：`remote-ops/onboard-hmi/scripts/11-drive-journey.ps1:21-27` 要求把车上的
`automation.enabled` 改成 `true` 再重启，而车上部署的文件 `"environment": "Production"`，
`OnboardAutomationSettings.Validate` 在 `Enabled && Production` 时直接抛异常
（`8005-agv-onboard-hmi/src/SQCD.Agv.Infrastructure/Configuration.cs:403-407`），客户端拒绝启动。
按脚本说的做，真车上的 58007 平面**起不来**。详见第 5.5 节。

## 3. ControlServer.Host

仓库 `8005-agv-control-server`。

### 3.1 日志

接线在 `src/ControlServer.Host/Program.cs:15-19`（已逐行核对）：

```csharp
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .WriteTo.Console(formatProvider: CultureInfo.InvariantCulture));
```

加 `app.UseSerilogRequestLogging();`（`Program.cs:83`）。

- **代码里的 sink 只有 Console**，默认模板 `[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}`。
  L2 实跑样本 `evidence/l2/20260903-normal-load-016/logs/control-server.out.log`：
  `[18:25:48 INF] Onboard NDJSON listener started on 127.0.0.1:58405; transport=plaintext`、
  `[18:25:48 INF] HTTP GET /health/live responded 200 in 6.1587 ms`。
- Enricher 只有 `FromLogContext()`（`Program.cs:18`）。
- `ReadFrom.Configuration` 已接（`Program.cs:16`），所以配置里任何 `Serilog:*` 节都生效。
  `Serilog.Settings.Configuration` 8.0.4 随 `Serilog.AspNetCore` 8.0.3 传递引入，同时带进
  `Serilog.Formatting.Compact` 2.0.0、`Serilog.Sinks.File` 5.0.0、`Serilog.Sinks.Console` 5.0.0
  （`src/ControlServer.Host/packages.lock.json:29-42`；版本锁在 `Directory.Packages.props:14`，
  引用在 `ControlServer.Host.csproj:13`）。**File sink 与 CompactJson formatter 已经在产物里**，
  不需要加包。
- 随包的 `src/ControlServer.Host/appsettings.json:64-71` 只有 `Serilog:MinimumLevel`
  （Default Information，`Microsoft.AspNetCore` Warning）。`src/` 下只有这一个 appsettings。
- **产品安装时的文件 sink**由安装脚本生成：`scripts/Install-ControlServerLocal.ps1:252-265`
  写出 `<InstallRoot>\appsettings.Production.json`（:267-271），内容 `Serilog:WriteTo[0]` =
  File，`path = <DataRoot>\logs\controlserver-.ndjson`，
  `formatter = Serilog.Formatting.Compact.CompactJsonFormatter`，`rollingInterval = Day`，
  `retainedFileCountLimit = 14`，`shared = true`。`$DataRoot` 默认
  `C:\ProgramData\8005\ControlServer`（:11）。服务以 `--environment Production` 注册（:278）并
  设 `DOTNET_ENVIRONMENT=Production`（:284-292）。文档 `docs/RELEASE-CANDIDATE.md:286-295`；
  升级脚本会把这个文件带过去（`scripts/Update-ControlServerLocal.ps1:164,210`）。
- 应用层日志事件**只有 9 个**，全部 `LoggerMessage` 定义，没有零散 `Log*` 调用，Host 以外没有
  `ILogger`：

| EventId | 级别 | 模板 | 位置 |
| --- | --- | --- | --- |
| 1001 | Warning | `Onboard transport is disabled; vehicle readiness cannot become READY.` | `src/ControlServer.Host/Transport/OnboardTcpServer.cs:113-115` |
| 1002 | Information | `Onboard NDJSON listener started on {Address}:{Port}; transport=plaintext` | `OnboardTcpServer.cs:117-119` |
| 1003 | Warning | `Onboard connection ended with a protocol or transport error.` | `OnboardTcpServer.cs:121-123`，触发 :45 |
| 1101 | Warning | `Onboard rejected {RejectedMessageType} {RejectedMessageId}: {ReasonCode} at {FieldPath} -- {DisplayMessage}` | `src/ControlServer.Host/Transport/OnboardMessageProcessor.cs:635-643`，触发 :408-414 |
| 2001 | Warning | `Journey runtime is disabled; MesIngest polling and movement dispatch are fail-closed.` | `src/ControlServer.Host/Runtime/JourneyRuntimeWorker.cs:11-14` |
| 2002 | Error | `Journey runtime iteration failed closed; no stage is inferred from memory.` | `JourneyRuntimeWorker.cs:15-18`，触发 :45 |
| 2101 | Warning | `MesIngest catalog polling failed closed; no journey was accepted.` | `src/ControlServer.Host/Runtime/JourneyRuntimeEngine.cs:30-33`，触发 :124 |
| 2102 | Warning | `SUBLOT_BOX_COUNT failed closed for demand {DemandId}.` | `JourneyRuntimeEngine.cs:34-37`，触发 :250 |
| 2103 | Warning | `RIoT Map station catalog failed closed; no new journey action was taken.` | `JourneyRuntimeEngine.cs:38-41`，触发 :65 |

其余全是框架噪音：EF Core `Executed DbCommand`、HttpClient、Kestrel、请求日志。另有一处
`Console.WriteLine`（`src/ControlServer.Host/Runtime/PackageCapacityImportCommand.cs:23-24`）。

### 3.2 日志里有哪些标识

基本没有。整个仓库没有 `LogContext.PushProperty` / `BeginScope` / `WithProperty`（只有
`Program.cs:18` 那个 enricher）。没有逐信封的日志事件；`ProcessAsync`
（`OnboardMessageProcessor.cs:20+`）对**被接受**的流量不写任何日志。

- 有的结构化标识：`RejectedMessageId`（1101，`:636,411`）及同事件的
  `RejectedMessageType` / `ReasonCode` / `FieldPath` / `DisplayMessage`；`DemandId`（2102，
  `:35,250`）；`Address` / `Port`（1002）。
- 没有的：`correlationId`（解析于 `OnboardMessageProcessor.cs:386`，只用于校验 `DurableAck`）、
  `sessionId` / `sessionGeneration`、`journeyId`、`agvId`、`vehicleKey`、`serverInstanceId`
  （`OnboardMessageProcessor.cs:18`，每进程随机 GUID，随 `SessionAccepted` 上线 :65）。这些
  全部只落 SQLite。

### 3.3 `runId` 入口

配置提供链是 `WebApplication.CreateBuilder(args)`（`Program.cs:13`）的默认链：
`appsettings.json` → `appsettings.{Env}.json` → user secrets（仅 Development）→ 环境变量
（无前缀，`__` 分层）→ 命令行 `--Section:Key=value`。没有自定义 `Add*`。环境名来自
`DOTNET_ENVIRONMENT` / `ASPNETCORE_ENVIRONMENT` 或 `--environment`；L2 两者都不设，实跑日志里是
`Hosting environment: Production`（.NET 未设时的默认值）。

Host 自己的命令行开关只有 `--import-package-capacity`、`--input <csv>`、`--version <int>`
（`Runtime/PackageCapacityImportCommand.cs:8-9,16-17,56-62`，分派于 `Program.cs:87-92`）。
安装脚本传 `--contentRoot "<InstallRoot>" --environment Production`
（`scripts/Install-ControlServerLocal.ps1:278`）。

`src/ControlServer.Host/appsettings.json` 键树：

| 节 | 消费处 | 备注 |
| --- | --- | --- |
| `ProtocolCandidate:*`（:2-12） | 无——用的是常量 `ProtocolCandidateIdentity`（`src/ControlServer.Domain/ProtocolCandidateIdentity.cs:3-14`） | 死配置 |
| `Health:url`（:13-15） | `Program.cs:20`，默认 `http://127.0.0.1:58007` | Kestrel 唯一绑定 |
| `OnboardTransport:{enabled,listenAddress,port,maxLineBytes,credentialEnvironmentVariable}`（:16-22） | `Transport/OnboardTransportOptions.cs:5-14`；`Program.cs:35-37`；`OnboardMessageProcessor.cs:527-529` | 拒绝 TLS 时代的键（`OnboardTransportOptions.cs:24-30`） |
| `ConnectionStrings:ControlServer`（:23-25） | `Program.cs:22-24` | |
| `MesIngest:{baseUrl,sharedSecretEnvironmentVariable}`（:26-29） | `Program.cs:56-57,73-74`；`Runtime/JourneyRuntimeOptions.cs:78-81` | |
| `RIoT:{baseUrl,callApiKeyEnvironmentVariable}`（:30-33）+ 未列出的 `RIoT:timeoutSeconds` | `Runtime/RiotSdkRegistration.cs:20,21,40` | |
| `RiotCreateDispatch:enabled`（:34-36） | `Runtime/RiotCreateDispatchOptions.cs:13-15` | |
| `RiotAbsentAtObservationCreateExperiment:*`（:37-39） | `Runtime/RiotAbsentAtObservationCreateExperimentOptions.cs:8-19` | |
| `OnboardSafetyProjection:{enabled,credentialEnvironmentVariable}`（:40-43） | `Runtime/OnboardSafetyProjectionOptions.cs:7-10`；`Program.cs:132` | |
| `JourneyRuntime:{enabled,pollInterval,maximumEvidenceAge,agvId,vehicleKey,agvLifecycleGeneration,mapId,mapIdentity,gateStationId,gateStationRiotId,dispatchZone,dispatchGeneration,minimumBatteryPercent,sublotBoxCountPath,allowedWorkTypes[],allowedDispatchZones[],admissionPolicyVersion,admissionPolicyDeploymentId}`（:44-63）+ `departureSafetyResultWait` | `Runtime/JourneyRuntimeOptions.cs:7-34` | `agvId` / `vehicleKey` / `admissionPolicyDeploymentId` 会持久化——**不适合当 run 标签** |
| `Serilog:MinimumLevel:*`（:64-71） | `Program.cs:16` | 任何 `Serilog:*` 都生效 |

不在随包 appsettings 里但代码直接读的键：`ControlServerBuild:commit`
（`OnboardMessageProcessor.cs:66`，默认 `WORKTREE_BUILD`；随 `SessionAccepted` 上线并持久化到
outbox；G2/G3 脚本设 `ControlServerBuild__commit`，如 `scripts/run-staged-g3.ps1:2191`）；
`Recovery:AuthenticationProofEnvironmentVariable`（`Transport/OnboardRecoveryCoordinator.cs:1008-1009`，
默认 `CONTROL_SERVER_RECOVERY_AUTHENTICATION_PROOF`）。

按名字读的环境变量：`CONTROL_SERVER_ONBOARD_CREDENTIAL`（`OnboardMessageProcessor.cs:527-529`、
`Runtime/OnboardVehicleSafetyEndpoints.cs:37`、`OnboardSafetyProjectionOptions.cs:22`）、
`CONTROL_SERVER_MES_INGEST_SHARED_SECRET`（`Program.cs:57-58,74-75`）、
`CONTROL_SERVER_RIOT_CALL_API_KEY`（`RiotSdkRegistration.cs:40-43`）、
`CONTROL_SERVER_RECOVERY_AUTHENTICATION_PROOF`。校验在 `JourneyRuntimeOptions.cs:78-83,94-102`。

**结论**：没有为日志准备的 `instanceId` / `deploymentId` / `siteId` / tags 字段；
`serverInstanceId` 每进程随机。**但**因为 `ReadFrom.Configuration` 已接，
`Serilog__Properties__RunId=<v>`（env）或 `--Serilog:Properties:RunId=<v>`（CLI）今天就能给每条
事件打上属性，零代码改动。属性在 CompactJson 文件 sink 里可见，在 Console 默认模板里**不可见**
（模板没有 `{Properties}`），所以 L2 下还需要同一机制再给一个
`Serilog__WriteTo__0__Args__outputTemplate` 或干脆加一个文件 sink。`ControlServerBuild:commit`
是今天唯一已存在的逐次注入字符串，但它是构建身份且上线，不能挪用。

测试侧的覆盖点供参考：`tests/ControlServer.Tests/OnboardMessageProcessorTests.cs:38,194`、
`JourneyRuntimeOptionsTests.cs:38`、`OnboardVehicleSafetyEndpointsTests.cs:124,139`。

### 3.4 HTTP 端点

单个 Kestrel 绑定（`Program.cs:20`，默认 `http://127.0.0.1:58007`；注意与车载端自动化平面
默认端口相同，两者在不同机器上）。无控制器，无 POST/PUT/DELETE，无 Swagger：

| 路由 | 位置 | 行为 |
| --- | --- | --- |
| `GET /health/live` | `Program.cs:94` | 200 `{"status":"live"}` |
| `GET /health/ready` | `Program.cs:95-107` | DB 可达且任一 `SessionRecoveries.Readiness == Ready` → 200；否则 503 `{"status":"not-ready","reason":"RECOVERY_HANDSHAKE_REQUIRED"\|"DATABASE_UNAVAILABLE"}` |
| `GET /version` | `Program.cs:108-119` | 协议身份常量 |
| `GET /api/runtime/sessions` | `Program.cs:120-131` | 逐 AGV `{AgvId, SessionGeneration, readiness, ReasonCode, UpdatedAt}` |
| `GET /api/onboard/v1/vehicle-safety` | `Runtime/OnboardVehicleSafetyEndpoints.cs:14-20`，仅当 `OnboardSafetyProjection:enabled`（`Program.cs:132-135`） | Bearer 对比环境变量（:37-50）；未设则 503；`{VehicleKey, MotionState, ObservedAt, Source, ReasonCodes}` |

没有恢复管理 HTTP；恢复走 NDJSON（`OnboardRecoveryCoordinator.cs:21,50`）。文档
`docs/RELEASE-CANDIDATE.md:253-261,455-456`、`ControlServer.http:1-6`。

### 3.5 SQLite

连接串：`Program.cs:22-25`，`GetConnectionString("ControlServer")` 缺省
`Data Source=%ProgramData%\8005\ControlServer\data\controlserver.db`，`ExpandDataSource`
（:139-154）。随包值 `appsettings.json:24`；安装脚本 `Install-ControlServerLocal.ps1:237,240`；
L2 覆盖为 `ConnectionStrings__ControlServer = "Data Source=<temp>\l2-<runId>\controlserver.db"`
（`scripts/l2/Invoke-L2Scenario.ps1:128-130,260`）。启动时跑迁移（`Program.cs:85,156-161`），
EF Core Sqlite 8.0.30。

Schema：13 个迁移在 `src/ControlServer.Infrastructure/Persistence/Migrations/`
（`20260825101420_InitialWireToGate` … `20260903110052_ResumeReplacementOperationResult`）；
快照 `ControlServerDbContextModelSnapshot.cs`；DbSet 在 `ControlServerDbContext.cs:9-37`。
**28 张表**（行号指快照文件）：

`AcceptedDemands`(:84)、`AdmissionDecisionSnapshots`(:111)、`AdmissionPolicyAudit`(:139)、
`AdmissionPolicyState`(:163)、`ConnectionRecoveries`(:190)、`ExceptionRecoverySessions`(:257)、
`ExperimentalRiotCreateAuthorizations`(:308)、`HardwareRecoveryRecords`(:360)、
`JourneyBacklog`(:396)、`JourneyRuntimes`(:556；`DemandId` 主键，`AgvId`、`Stage`、
`BlockReasonCode`、多列 `*MessageId`、`VehicleKey`…)、`MissingPackages`(:576)、
`OperationResults`(:629)、`OrderIntents`(:732)、`PackageCapacityRules`(:770，种子 :772-1052)、
`ProtocolInbox`(:1081；`MessageId` 主键，`ContentHash`、`FirstResponseJson`、`MessageType`、
`ReceivedAt`、`RequestJson`)、`ProtocolOutbox`(:1108；`MessageId` 主键，`AcknowledgedAt`、
`CreatedAt`、`FencedAt`、`MessageType`、`PayloadJson`)、`RecoveryDecisions`(:1136)、
`RecoveryResultEvidence`(:1174)、`RecoveryWorkflows`(:1250)、`RiotDispatchAuditEvents`(:1335)、
`SessionRecoveries`(:1422；`AgvId` 主键，`Readiness`、`ReasonCode`、`SessionGeneration`、
`DepartureSafe`…)、`StationOperations`(:1468)、`StationTaskTypeAdmissions`(:1484)、
`StopClosures`(:1497)、`TransportDemandCompletions`(:1524)、`UnloadBatches`(:1545)、
`VehicleDispatchLeases`(:1569)、`VehicleRecoveryGenerations`(:1585)。

没有 run / instance / deployment 表。

**L2 今天读的九张**（`scripts/l2/Invoke-L2Scenario.ps1:526-541`，已核对；每张 `SELECT *` 导出为
`snapshots\db-<Table>.json`）：`JourneyRuntimes`、`AcceptedDemands`、`JourneyBacklog`、
`OrderIntents`、`StationOperations`、`SessionRecoveries`、`OperationResults`、
`ExceptionRecoverySessions`、`RecoveryWorkflows`。断言写在 `assertions.json`，`SUMMARY.md` 的身份块
（`controlServerCommit`、`agvId`、`vehicleKey`、`stageRoot`、`rig`、可选对端 commit /
`clockSkewMs`）由 `Write-L2Evidence` 产出（:545-557；`scripts/l2/L2.psm1:653-733`）。
`timeline.jsonl` 是编排器自己的时间线（:132）。布局说明 `scripts/l2/README.md:138-144`。

### 3.6 L2 编排器如何起它

`scripts/l2/Invoke-L2Scenario.ps1`：`EvidenceRoot` 必须不存在（:29-30,112-114）；
`$runId = <UTC yyyyMMddTHHmmssfffZ>`（:116-117，已核对）；`$logRoot = <EvidenceRoot>\logs`、
`$snapshotRoot = <EvidenceRoot>\snapshots`（:123-126）；stage 根 `%TEMP%\l2-<runId>`（:128-130）。
Release 构建（:150）；跑 `src/ControlServer.Host/bin/Release/net8.0/win-x64/ControlServer.Host.exe`
（:169-171,303,314）。Host 配置**全部走 `__` 环境变量**（`$serverEnvironment` :258-287，已核对）：
`CONTROL_SERVER_ONBOARD_CREDENTIAL`、`ConnectionStrings__ControlServer`、`Health__url`（58407）、
`OnboardTransport__listenAddress/port`（58405）、`MesIngest__baseUrl`、`RIoT__baseUrl`、
`RIoT__callApiKeyEnvironmentVariable` + 假密钥、`RiotCreateDispatch__enabled`、
`JourneyRuntime__enabled/pollInterval/agvId/vehicleKey/mapId/mapIdentity/gateStationId/gateStationRiotId/admissionPolicyDeploymentId`
（**`"L2-$runId"`，:286——`$runId` 到达 Host 的唯一一处，落 `AdmissionPolicyState.DeploymentId`
而不是日志**）；条件性的 `Recovery__AuthenticationProofEnvironmentVariable` + proof（:288-291）、
`OnboardSafetyProjection__enabled`（:292-297）。**没有任何 `Serilog__*` 覆盖**，所以 L2 下只有
Console sink。

`Start-L2Process`（`scripts/l2/L2.psm1:250-284`）把 stdout/stderr 重定向到
`<logRoot>\<Name>.out.log` / `.err.log`（:265-270，已核对），`Start-Process -Environment`（:276）。
实跑目录 `evidence/l2/20260903-normal-load-016/logs/`：`build.log`、`control-server.{out,err}.log`、
`fake-*.{out,err}.log`、`package-capacity-import.{out,err}.log`。CI：`.github/workflows/l2.yml:59-66`
→ `EvidenceRoot = $RUNNER_TEMP\l2-<GITHUB_RUN_ID>-<GITHUB_RUN_ATTEMPT>\<scenario>`，产物名
`l2-evidence`（:81-86）。

## 4. 替身：FakeRiot / FakeMesIngest / FakeOnboard / ClockSkewProxy

仓库 `8005-agv-control-server`，全在 `tools/` 下。四个替身共用一个框架
`tools/ControlServer.TestDoubles/`：命令信封、响应信封、监听规则、openapi 文件服务。都是
ASP.NET Core minimal API，无控制器。

### 4.1 控制面端点

**FakeRiot**（接线 `tools/ControlServer.FakeRiot/FakeRiotHost.cs:41-42`）。产品面（冒充 RIoT RCS，
`tools/ControlServer.FakeRiot/RiotDataPlane.cs`）：`GET /api/task/vehicles/getVehicleInfoByDeviceKey?key=`(:21)、
`GET /api/task/v1/task/getVehicleInfo/{deviceKey}`(:50)、`GET /api/imap/v1/mapInfo/stations/{mapId:int}`(:84)、
`GET /api/order/v1/orderRecord/detailByUpperId/{upperId}`(:100)、
`GET /api/order/v1/orderRecord?filterByState=&pageSize=`(:111)、`POST /api/order/v1/add/byDefaultMissions`(:141)。
控制面组 `/control/v1` 在 `tools/ControlServer.FakeRiot/ControlPlane.cs:68`（已核对）：

| 方法 | 路由 | 位置 | 说明 |
| --- | --- | --- | --- |
| GET | `/control/v1/openapi.json` | `ControlPlane.cs:70` | |
| GET | `/control/v1/health` | :72 | `{status:"live", faultMode}`（:72-76） |
| GET | `/control/v1/snapshot` | :78 | `{faultMode, delayMs, mapStationReads, vehicles, orders, maps:[{mapId, stations}]}`（:78-91；`mapStationReads` 计数在引擎外，`RiotDataPlane.cs:267-274`，递增 :89） |
| POST | `/control/v1/reset` | :93 | :93-101 |
| PUT | `/control/v1/vehicle` | :103 | `VehicleCommand`（:10-31，:103-141） |
| PUT | `/control/v1/orders/{upperId}` | :143 | `OrderCommand {orderState 1..9, executeVehicleKey, endStationNo}`（:37-42，:143-169） |
| PUT | `/control/v1/maps/{mapId:int}/stations` | :171 | `StationsCommand {stations:{id→name}}`（:44-47，:171-206） |
| PUT | `/control/v1/faults/http` | :208 | `FaultCommand {mode: Normal\|NoResponse\|ServerError\|Delay, delayMs 1..60000}`（:49-54；枚举 `FakeRiotState.cs:62-68`；生效 `RiotDataPlane.cs:233-252`） |

**FakeMesIngest**（接线 `tools/ControlServer.FakeMesIngest/FakeMesIngestHost.cs:35-36`）。产品面
（冒充 MesIngest V2，`tools/ControlServer.FakeMesIngest/MesIngestDataPlane.cs`）：`GET /api/v2/contract`(:41，
路径常量 `src/ControlServer.Infrastructure/Adapters/HttpMesIngestCatalog.cs:12`)、
`GET /api/v2/externally-readable-demand-catalog`(:56，常量 `HttpMesIngestCatalog.cs:13`)、
`GET /api/v2/sublot-box-count?sublot=`(:98，字面量)。控制面组 `/control/v1` 在
`tools/ControlServer.FakeMesIngest/ControlPlane.cs:41`（已核对）：

| 方法 | 路由 | 位置 | 说明 |
| --- | --- | --- | --- |
| GET | `/control/v1/openapi.json` | `ControlPlane.cs:43` | |
| GET | `/control/v1/health` | :45 | `{status:"live", demandCount}`（:45-49） |
| GET | `/control/v1/snapshot` | :51 | `{historyEpoch, catalogRevision, breakContract, demands}`（:51-61） |
| POST | `/control/v1/reset` | :63 | 种子 `FakeMesIngestSeed.cs:16-21` |
| PUT | `/control/v1/demands/{demandId}` | :73 | `DemandCommand`（:9-27，:73-117） |
| DELETE | `/control/v1/demands/{demandId}` | :121-122 | **要求 JSON body**（`[FromBody]`） |
| PUT | `/control/v1/contract` | :133 | `{breakContract}`（:29-32；效果 `MesIngestDataPlane.cs:46-48`） |

**FakeOnboard**（`tools/ControlServer.FakeOnboard/ControlPlane.cs:49` 组）：`GET /control/v1/openapi.json`(:51)、
`GET /control/v1/health`(:53)、`GET /control/v1/snapshot`(:64)、`PUT /control/v1/policy`(:88)、
`PUT /control/v1/safety`(:101)、`PUT /control/v1/answer/{key}`(:141)。**没有 `/reset`**，没有产品面
HTTP——它是 ControlServer 的 TCP 对端（`FakeOnboardHost.cs:41`，端口 58009 于 :11）。它有一份线路
日志（`ControlPlane.cs:84`，最近 200 条）。

**ClockSkewProxy**（`tools/ControlServer.ClockSkewProxy/ControlPlane.cs:30` 组）：`GET /control/v1/openapi.json`(:32)、
`GET /control/v1/health`(:34)、`GET /control/v1/snapshot`(:44)、`PUT /control/v1/skew`(:51)、
`POST /control/v1/reset`(:61)。产品面 `GET /api/onboard/v1/vehicle-safety`
（`SafetyProxyDataPlane.cs:14,26`），端口 58090（`ClockSkewProxyHost.cs:89`），目标必须由
`--ClockSkewProxy:target` 给出（`ClockSkewProxyHost.cs:99-107`）。

共享形状（`tools/ControlServer.TestDoubles/ControlPlaneConventions.cs`）：请求信封
`CommandEnvelope { runId, commandId, expectedRevision? }`（:10-15）；读响应信封
`{schemaVersion, instanceId, runId, revision, observedAt, body}`（:33-41，已核对）；写响应加
`{commandId, changed, replayed, appliedRevision}`（:48-54）；拒绝 `{commandId, reasonCode}`，
`INVALID_ARGUMENT` 为 400，其余 409（:58-63；码表 `CommandEngine.cs:8-16`）；幂等
`CommandEngine.Apply`（:132-183）；重置 `CommandEngine.Reset`（:97-125）。

### 4.2 openapi 文档与漂移

每个项目旁各一份**手写** `openapi.json`，复制到输出目录（`ControlServer.FakeRiot.csproj:16`、
`ControlServer.FakeMesIngest.csproj:13`），由 `ControlPlaneConventions.OpenApiDocument` 从
`AppContext.BaseDirectory` 读出（`ControlPlaneConventions.cs:122-128`）。没有生成器，没有任何东西
比对文档与路由。

- FakeRiot：路由齐全。漂移：文档写 `VEHICLE_NOT_FOUND`（`openapi.json:138`）/ `ORDER_NOT_FOUND`（:155），
  代码抛的是 `NOT_FOUND`（`CommandEngine.cs:14`；`ControlPlane.cs:107,110,148`）。`servers.url`
  写死 `http://127.0.0.1:58008`（:8），L2 用 58408。
- FakeMesIngest：路由齐全。漂移：`DemandCommand` schema（:140-166）漏了 `demandId`
  （`ControlPlane.cs:11`）；DELETE（:99-112）没列 400，但 `runId` / `commandId` 缺失时
  `ControlPlaneConventions.Handle` 返回 400 `INVALID_ARGUMENT`（`ControlPlaneConventions.cs:78-80`）。
  `servers.url` 58088（:8），L2 用 58409。

### 4.3 监听地址与配置

`ControlPlaneConventions.ResolveLoopbackListener`（`ControlPlaneConventions.cs:98-119`）：
`{Section}:listenAddress` 默认 `127.0.0.1`（:104）；`{Section}:port` 默认 FakeRiot 58008
（`FakeRiotHost.cs:12`）、FakeMesIngest 58088（`FakeMesIngestHost.cs:11`）；
`{Section}:allowNonLoopbackListen`——否则非回环拒绝，退出码 2（:110-117）。通过
`ConfigureKestrel(options => options.Listen(listener))` 绑定（`FakeRiotHost.cs:38`、
`FakeMesIngestHost.cs:32`），所以 **`--urls` / `ASPNETCORE_URLS` 不生效**。节名 `FakeRiot`
（`FakeRiotHost.cs:33`）、`FakeMesIngest`（`FakeMesIngestHost.cs:27`）；配置走
`WebApplication.CreateBuilder(args)`，即命令行 `--FakeRiot:port=…` 与环境变量 `FakeRiot__port=…`
都行。**故意没有 appsettings.json**（`ControlServer.FakeRiot.csproj:11-15`、
`tools/ControlServer.FakeRiot/README.md:17-26`）。

### 4.4 日志

任何替身都**没有配置日志器**：没有 Serilog，没有 `AddConsole` / `AddJsonConsole`，没有请求日志，
`tools/` 下没有 `ILogger` 用法。生效的是 `WebApplication.CreateBuilder` 的默认（Console simple
formatter，Information），stdout 只有 hosting 生命周期行与未处理异常，**没有逐请求行**。显式写出的
只有：回环拒绝的 `Console.Error.WriteLine`（`ControlPlaneConventions.cs:113-115`）与 ClockSkewProxy
缺目标（`ClockSkewProxyHost.cs:103-105`）。`correlationId` / `messageId` 不出现在任何日志行。

L2 下 stdout/stderr 落 `logs/fake-riot.{out,err}.log`、`logs/fake-mes-ingest.{out,err}.log`
（`scripts/l2/L2.psm1:265-270`；名字 `Invoke-L2Scenario.ps1:188,201`；证据清单 `L2.psm1:715`）。

### 4.5 `runId` / `instanceId`

| 键 | 效果 | 位置 |
| --- | --- | --- |
| `FakeRiot:instanceId` / `FakeMesIngest:instanceId` | 自由文本，默认 `fake-riot-1` / `fake-mes-ingest-1`；回显为每个控制面响应的 `instanceId` | `FakeRiotHost.cs:27-29`、`FakeMesIngestHost.cs:22-24`、`CommandEngine.cs:73,82`、`ControlPlaneConventions.cs:36` |
| `FakeRiot:Seed:*` | 业务种子 | `FakeRiotSeed.cs:10-27`、`FakeRiotHost.cs:25` |
| `FakeMesIngest:Seed:historyEpoch` | 目录身份 GUID | `FakeMesIngestSeed.cs:14`、`FakeMesIngestHost.cs:20` |
| `{Section}:listenAddress` / `port` / `allowNonLoopbackListen` | 监听 | `ControlPlaneConventions.cs:104-111` |

**不可注入**的：替身自己的 `runId`——`"run-" + 8 位 hex`（`CommandEngine.cs:205`，已核对），在构造
（:77）与重置（:119）时生成；请求里的 `runId`（`ControlPlaneConventions.cs:12`）只用于校验
（`CommandEngine.cs:109-111,144-146`）。`commandId` 由调用方自选自由文本（:13），不进内容指纹
（`ControlPlaneConventions.cs:87`）。两个平面都没有 scenario / profile / tag 概念。

L2 起替身时给的是常量 `--FakeRiot:instanceId=l2-riot`、`--FakeMesIngest:instanceId=l2-mes`
（`Invoke-L2Scenario.ps1:192,205`，已核对）。`L2Double` 类（`L2.psm1:168-228`）每次 `Command()`
先读 `/snapshot` 取 `runId`（:215-216），缺 `commandId` 则生成 GUID（:203-205），仅在
`RequireExpectedRevision` 时对 409 重试（:213-223）；从不调 `/reset`。`Wait-L2Iterations` 轮询
`mapStationReads`（:146-158）。拆除时抓 `snapshots/fake-riot.json`、`snapshots/fake-mes-ingest.json`
（`Invoke-L2Scenario.ps1:499-501,515-525`）。`/faults/http`、`/maps/{mapId}/stations`、`/contract`、
`DELETE /demands`、`/reset` 今天没有任何 L2 场景用到。侧记：`tests/ControlServer.Tests/` 有
`FakeRiotTests.cs` 与 `ClockSkewProxyTests.cs`，没有任何东西引用 `FakeMesIngest`。

## 5. 车载端 SQCD.Agv.Wpf

仓库 `8005-agv-onboard-hmi`，分支 `w2g/recovery-entry-mid-session-readiness`。项目图
`SQCD.Agv.Wpf` → `SQCD.Agv.AutomationHost`、`Application`、`Core`、`Infrastructure`
（`src/SQCD.Agv.Wpf/SQCD.Agv.Wpf.csproj:9-12`）；HTTP 平面在 `SQCD.Agv.AutomationHost`
（FrameworkReference AspNetCore，csproj:9；`openapi.json` 内嵌，:13）。

### 5.1 58007 端点

Kestrel `CreateSlimBuilder`（`src/SQCD.Agv.AutomationHost/AutomationHttpServer.cs:68-73`，
`Args = []` :72，已核对），`builder.Logging.ClearProviders()`（:74，已核对），`UseUrls(Endpoint)`（:75），
`Endpoint => $"http://{ListenAddress}:{Port}"`（:15）。**`RunId = Guid.NewGuid().ToString("D")`
每个服务器实例一次**（:48，已核对）——不可配置。配置 `automation.listenAddress` / `port` / `enabled`
（`src/SQCD.Agv.Infrastructure/Configuration.cs:384-390`，默认 `127.0.0.1` / 58007 / **false**）。
校验：`OnboardAutomationHostOptions.Validate()` 回环 + 端口（`AutomationHttpServer.cs:17-25`）；
`OnboardAutomationSettings.Validate()` 禁止与 `wireToGate.port` / `ioModule.port` 撞车、要求
`wireToGate.enabled`（`Configuration.cs:409-418`）。启动于 `src/SQCD.Agv.Wpf/App.xaml.cs:177-194`
（已核对），仅当 `settings.Automation.Enabled`；日志 `车载端自动化接口已启动：{Endpoint}`（:190-193）。

路由（`src/SQCD.Agv.AutomationHost/AutomationApi.cs`，组 `/api/v1` :47，已核对）：

| 路由 | 位置 | 请求 | 响应 |
| --- | --- | --- | --- |
| `GET /api/v1/health` | :48-63 | — | `AutomationHealthResponse(SchemaVersion "1.0.0", AgvId, RunId, Revision, ObservedAt, Status)`；READY 当且仅当 `IoConnected && WireToGateSession.Connected && State != Faulted`，否则 DEGRADED（:51-55）；DTO `ApiContracts.cs:11-17` |
| `GET /api/v1/snapshot` | :65-75 | — | `AutomationSnapshotResponse(..., State: OnboardAutomationSnapshot)`（`ApiContracts.cs:19-25`）；快照记录 `src/SQCD.Agv.Core/OnboardAutomation.cs:7-17`：`AgvId, Onboard, WireToGateSession?, WireToGateJourney?, RecoveryState, CanSubmitSublot, ExpectedSublot?, CurrentOperationAttemptId?, CurrentOperationPhase?, ObservedAt` |
| `POST /api/v1/sublots/submit` | :77-163 | `AutomationSublotSubmitRequest(RunId?, CommandId?, ExpectedRevision?, Sublot?)`（`ApiContracts.cs:5-9`）；全部必填，`RunId` / `CommandId` 须 GUID "D"，`ExpectedRevision >= 1`（:81-99 → 400 `INVALID_HTTP_REQUEST`）；`RunId` 须匹配（:101-106 → 409 `RUN_ID_MISMATCH`）；按 `CommandId` + 指纹幂等（:119-129 → 409 `COMMAND_ID_CONFLICT` 或 `Replayed=true`）；`ExpectedRevision`（:131-136 → 409 `REVISION_CONFLICT`） | `AutomationSubmitResponse(..., CommandId, Accepted, Replayed, MessageId?, ReasonCode?, State)`（`ApiContracts.cs:27-38`）；200/409（:155-157）。**`MessageId` 就是 W2G `SublotSubmitted` 的 `messageId`**，来自 `WireToGateBusinessService.SubmitSublotAsync`（`src/SQCD.Agv.Wpf/WpfOnboardAutomationFacade.cs:72-79`） |
| `GET /openapi/v1.json`（不在 `/api/v1` 下） | :165-166 | — | 内嵌 `openapi.json`（:183-193） |

错误信封：`UseStatusCodePages`（:22-45）→ `AutomationErrorResponse(SchemaVersion, RunId, Revision,
ObservedAt, CommandId?, ReasonCode, Message)`（`ApiContracts.cs:40-47`），`PATH_NOT_FOUND` /
`METHOD_NOT_ALLOWED` / `HTTP_ERROR`。`Revision` 由 `AutomationRevisionState.Read()` 推导：序列化快照、
剥掉 `observedAt` / `updatedAt` / `startedAt`、指纹变则递增（:213-270）。无鉴权。单元测试确认
`POST /api/v1/slots/1/unlock` → 404 且不在 OpenAPI 里
（`tests/SQCD.Agv.UnitTests/OnboardAutomationHostTests.cs:53-61`）。

文档：仓内**没有**文档描述 58007 的路由，README 从不提 58007；
`docs/WANG_KUN_FIRST_INTEGRATION_WORK_PACKAGE.md:171-194` 描述的是模拟器的平面。
`src/SQCD.Agv.AutomationHost/openapi.json` 列了三条路由（:10-30），`servers.url` 58007（:8），
`/health` 无 schema（:10）；错误信封与 POST body 描述不足。

### 5.2 日志形态

自写 `FileAppLogger : IAppLogger`（`src/SQCD.Agv.Infrastructure/FileAppLogger.cs:7`），
`File.AppendAllText` 加锁（:30-35）。`src/` 下没有 Serilog / NLog / MEL。端口
`IAppLogger.Write(LogSeverity, string source, string message, Exception?)` + `EntryWritten`
（`src/SQCD.Agv.Core/Ports.cs:65-69`）；`LogEntry(Timestamp, Severity, Source, Message)`
（`src/SQCD.Agv.Core/DomainModels.cs:208`）。

- 配置 `logging.directory` 默认 `"logs"`、`logging.writeToConsole` 默认 `false`
  （`Configuration.cs:585-590`、`appsettings.json:123-126`）。目录 =
  `Path.GetFullPath(Path.Combine(AppContext.BaseDirectory, dir))`（`FileAppLogger.cs:16`）→
  车上是 `C:\8005\OnboardHmi\logs\`。
- 文件 `agv-{yyyyMMdd}.log`（:33），无按大小滚动，无保留期。
- 行模板（:26，已核对）：`$"{entry.Timestamp:O}\t{entry.Severity}\t{entry.Source}\t{entry.Message}{Environment.NewLine}"`
  ——TSV，ISO-8601 **本地时间**（`DateTimeOffset.Now` :25），非 JSON，无最低级别。异常折叠为
  `" | {Type}: {Message}"`（:24），无堆栈。写失败 → `Debug.WriteLine`（:37-44）。
- UI 日志面板内存 300 条：`MainViewModel.Logs`，来源 `OperatorEventPublished` / `OperatorRecord`
  （`src/SQCD.Agv.Wpf/ViewModels/MainViewModel.cs:76,176-181,381,432-437`）；
  `OperatorRecordFormatter` 明说技术状态走磁盘日志（`src/SQCD.Agv.Application/OperatorRecordFormatter.cs:20-23`）。
- 持久状态在 SQLite journal：表 `WireToGateJournalMetadata`、`WireToGateRecoveryState`、
  `WireToGateDurableOutbox`、`WireToGateAppliedJourneySnapshots`
  （`src/SQCD.Agv.Infrastructure/SqliteWireToGateJournal.cs:51-72`），路径 `wireToGate.journalPath`，
  支持 `%LOCALAPPDATA%` 展开（:26）。
- **没有传输层追踪**：`WireToGateSessionClient.cs` 零处 logger 引用；`WireToGateSlotOperationExecutor.cs` 零处。

### 5.3 信封标识在日志里

信封 `WireToGateEnvelope(ProtocolVersion, ProfileId, ProtocolReleaseVersion, ProtocolReleaseManifestSha256,
MessageType, MessageId, CorrelationId?, AgvId, SessionGeneration?, SentAt, Payload)`
（`src/SQCD.Agv.Contracts/WireToGateProtocol.cs:46-57`）。

- `correlationId`：从未写进日志。
- W2G 的 `messageId`：从未写进日志。所有 `messageId=` 文本都在旧规则网关路径
  （`src/SQCD.Agv.Application/OnboardController.cs:417,1000,1068,1165,1203`；`TcpJsonRuleGateway.cs:209`），
  而 W2G 下这条路径被 `DisabledRuleGateway` 替换（`App.xaml.cs:44-45`）。W2G 业务服务里
  `messageId` 只出现在 `DeduplicationKey` 字符串里（`WireToGateBusinessService.cs:421,763,804,847,917`）。
- 无结构化日志（`LogContext` / `PushProperty` / `BeginScope`：零）。
- 能以文本进文件的：`generation=…，readiness=…`（`WireToGateSessionService.cs:242`）；
  `demandId=…，revision=…`（`App.xaml.cs:148`）；`attempt=…`
  （`WireToGateBusinessService.cs:844,915,1013,1060,1075,1160`）；`session={recoveryId}`（:372）；
  启动时 `agvId=…，environment=…，version=…`（`App.xaml.cs:95`）；`code=/guidance=`
  （`MainViewModel.cs:468`）。没有 `sessionId` / `journeyId` / `deviceKey` / `vehicleKey`。
  `ARCHITECTURE.md:100` 要求的 `visitId` / `operationId` 只有旧路径满足。

### 5.4 配置键树与 `runId` 候选字段

单个 `appsettings.json`，位于 `Path.Combine(AppContext.BaseDirectory, ...)`（`App.xaml.cs:31`）→
`OnboardSettings.Load`（`Configuration.cs:28-41`）；大小写不敏感、允许注释与尾逗号（:160-165）。
**无环境分层，无 env / CLI 覆盖**（部署脚本也确认，`remote-ops/onboard-hmi/scripts/06-deploy-onboard-hmi.ps1:235-239`）。
`wireToGate.useTls` / `serverCertificateSha256` 按名字拒绝（:123-158）。**命令行：无**——`OnStartup`
从不碰 `e.Args`（`App.xaml.cs:25-27`），无 `GetCommandLineArgs`，`launchSettings.json:1-7` 无参数，
Kestrel `Args = []`。未知 JSON 键静默忽略（无 `UnmappedMemberHandling`，:160-165）。

键树（行号指 `Configuration.cs`）：`environment`(:8)、`agvId`(:10)、`onboardInstanceId`(:12)；
`ruleGateway.{host, port, connectTimeoutMs, requestTimeoutMs, heartbeatIntervalMs, resultRetryCount, reconnectDelaysMs[]}`(:422-436)；
`wireToGate.{enabled, host, port, onboardInstanceId, onboardBuildCommit, credentialEnvironmentVariable, connectTimeoutMs, messageTimeoutMs, capabilityVersion, safetyStateVersion, slotModelVersion, activeSlotConfigurationVersion, supportsBatchUnlock, journeySnapshotMaxAgeMs, operatorIdEnvironmentVariable, recoveryResumeEnabled, recoveryAuthenticationProofEnvironmentVariable, recoveryAdministratorRole, recoveryVerificationMethod, journalPath}`(:246-289；`recoveryResumeEnabled` 默认 `false` :280，用于 `App.xaml.cs:139`)；
`automation.{enabled, listenAddress, port}`(:384-390)；
`vehicleSafety.{enabled, endpoint, credentialEnvironmentVariable, expectedVehicleKey, maximumEvidenceAgeMs, clockSkewToleranceMs, pollIntervalMs, requestTimeoutMs}`(:173-191)；
`ioModule.{host, port, unitId, doStartAddress, diStartAddress, channelCount, pollIntervalMs, requestTimeoutMs, reconnectDelayMs, slots[8]{slotIndex, doChannel, lockFeedbackDiChannel, lightCurtainDiChannel}}`(:462-552)；
`workflow.{unlockFeedbackTimeoutMs, unlockOutputResetTimeoutMs, operationTimeoutMs, feedbackStableMs, ioSnapshotMaxAgeMs, maxSublotLength, maxReopenAttempts}`(:554-568)；
`logging.{directory, writeToConsole}`(:585-590)。

按名字读的环境变量：`CONTROL_SERVER_ONBOARD_CREDENTIAL`（`WireToGateSessionClient.cs:317`；
`ControlServerVehicleSafetySignalProvider.cs:46`）、`CONTROL_SERVER_OPERATOR_ID`
（`WireToGateBusinessService.cs:136,217,402`）、`CONTROL_SERVER_RECOVERY_PROOF`（:137,219）；
Production 下检查存在性（`Configuration.cs:76-93`）。

`runId` 候选载体：

| 字段 | 去向 | 能否借用 |
| --- | --- | --- |
| `agvId`(:10) | 信封 `AgvId`（服务端校验，`WireToGateProtocol.cs:161`）、每个自动化响应、启动日志；Production 下有占位符检查（:96-112） | **不能**——现场文件把它设为 RIoT `deviceName`（`remote-ops/onboard-hmi/config/site-agv01.json:7`） |
| 顶层 `onboardInstanceId`(:12) | 只喂旧 `TcpJsonRuleGateway`（`App.xaml.cs:49`）——W2G 下惰性 | 技术上可写任意值且无副作用，但**不进日志也不进响应**，等于没用 |
| `wireToGate.onboardInstanceId`(:324) | GUID，身份 | 不能 |
| `wireToGate.onboardBuildCommit`(:325-327) | 40 位 hex；部署断言等于包 commit（`06-deploy-onboard-hmi.ps1:353-356`） | 不能 |
| `wireToGate.slotModelVersion` / `activeSlotConfigurationVersion`(:333-334) | 只要求非空，随 `WireToGateSessionOptions` 上线（:310-311），服务端可见 | 语义错位，不建议 |
| `environment`(:8) | `"Production"` 打开全部严格检查（:52） | 不能 |

### 5.5 Production 下自动化平面被禁用的矛盾

`OnboardAutomationSettings.Validate` 在 `Enabled && production` 时抛
`InvalidDataException("Production环境禁止启用车载端自动化loopback接口，除非完成独立安全评审。")`
（`src/SQCD.Agv.Infrastructure/Configuration.cs:403-407`，已核对），由 `OnboardSettings.Validate`
调用（:67-71）；`production` 即 `environment` 等于 `"Production"`（:52）。启动侧 `App.xaml.cs:196-211`
捕获 `InvalidDataException`，弹 `软件启动失败` 后 `Shutdown(-1)`。

而 `remote-ops/onboard-hmi/scripts/11-drive-journey.ps1:21-27`（已核对）把「把 `automation.enabled`
改成 `true` 再重启客户端」写成前置条件，只提醒下次 `06-deploy-onboard-hmi.ps1` 会把它改回去，
**没提 Production 会拒绝**。部署渲染的文件来自 `appsettings.Production.template.json`，其
`environment` 镜像 `appsettings.Production.example.json:2` 的 `"Production"`；部署脚本从不碰
`automation.*`（`06-deploy-onboard-hmi.ps1:133-140,240-247,266-267`），平面出厂关闭。启动脚本
`10-start-onboard-stack.ps1` 也只探测 1502 / 58006 与客户端进程（:95,195-197,211），**从不探 58007**。

后果：照脚本做，真车上 58007 起不来。可行的出路只有两条，都不是脚本能替你做的决定：
（a）把车上 `environment` 改成非 Production——代价是 `Configuration.cs:52-93` 的全部生产守卫
（占位符检查、凭据存在性检查）一并失效；（b）改校验器——走 `w2g/*` 分支 + PR，且校验器的错误
文本已经说了「除非完成独立安全评审」，这是一个需要 Kun Wang 点头的安全决定，不是清理。

## 6. slots-simulator

仓库 `slots-simulator`，HEAD `fb5f7c5`。可执行 `src/SQCD_8005AGV_Simulator/SQCD_8005AGV_Simulator.csproj`
（WinExe，WPF，net8.0-windows，:3-5），引用 Core + AutomationHost（:12-13），复制 `simulator.settings.json`
（:14）。HTTP 宿主是 `Microsoft.NET.Sdk.Web` 类库（`src/SQCD_8005AGV_Simulator.AutomationHost/*.csproj:1-3`）
嵌进 WPF 进程。

### 6.1 58006 端点

Kestrel `CreateSlimBuilder`，`Args = []`（`src/SQCD_8005AGV_Simulator.AutomationHost/AutomationHttpServer.cs:44-49`，
:48 不传 CLI 参数，已核对）；`builder.Logging.ClearProviders()`（:50，已核对）；`UseUrls(Endpoint)`（:51），
`Endpoint => $"http://{ListenAddress}:{Port}"`（:31）。地址/端口只来自配置文件：
`AutomationSettings.ListenAddress` 默认 `127.0.0.1`、`Port` 58006
（`src/SQCD_8005AGV_Simulator.Core/Configuration/SimulatorSettings.cs:132-136`，已核对）←
`automation.listenAddress` / `automation.port`（`simulator.settings.json:15-18`）。回环强制：
`Validate()` 拒绝非回环（`SimulatorSettings.cs:43-44`）、拒绝与 modbus 端点相同（:45-49）。
由 `MainViewModel.StartAsync` 启动（`ViewModels/MainViewModel.cs:117-118`），在 `Window_Loaded`
（`MainWindow.xaml.cs:42`）。

路由（`AutomationHost/AutomationApi.cs`，组 `/api/v1` :41；DTO `ApiContracts.cs`）：

| 方法 | 路径 | 位置 | 请求 | 响应 |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/health` | :43 | — | `HealthResponse`（`ApiContracts.cs:24-31`）：`schemaVersion, instanceId, runId, revision, observedAt, status READY/DEGRADED`（:137）、`modbus` |
| GET | `/api/v1/snapshot` | :44 | — | `SnapshotResponse`（:33-45）：+ `doStates[16]`、`diStates[16]`、`slots[]`（:78-87）、`openDoorCount`、`maxOpenDoors`、`faults[]` |
| POST | `/api/v1/reset` | :45-46 | `ResetRequest(RunId, CommandId, ExpectedRevision)`（:3） | `WriteResponse`（:47-57）/ `ErrorResponse`（:59-67） |
| PUT | `/api/v1/slots/{slotNo}/cargo` | :48-55 | `CargoRequest`，`State` EMPTY/OCCUPIED 否则 400 `INVALID_CARGO_STATE` | 同上 |
| POST | `/api/v1/slots/{slotNo}/close-door` | :57-60 | `ResetRequest` 形状 | 同上 |
| PUT | `/api/v1/slots/{slotNo}/lock-feedback-override` | :62-69 | `OverrideRequest`，`Mode` AUTO/FIXED_0/FIXED_1 | 同上 |
| PUT | `/api/v1/slots/{slotNo}/light-curtain-override` | :71-78 | `OverrideRequest` | 同上 |
| PUT | `/api/v1/faults/modbus` | :80-88 | `ModbusFaultRequest`，`Mode` NORMAL/NO_RESPONSE/DISCONNECT/DELAY，`DelayMs` | 同上 |
| GET | `/openapi/v1.json`（不在 `/api/v1` 下） | :90-91 | — | 内嵌 `openapi.json`（:260-270） |

错误映射（:115-120）：`INVALID_SLOT_NO` → 404；`RUN_ID_MISMATCH` / `REVISION_CONFLICT` /
`COMMAND_ID_CONFLICT` / `DOOR_NOT_OPEN` → 409；其余 400。畸形请求 → 400 `INVALID_HTTP_REQUEST`（:12-27）；
未知路径 404 `PATH_NOT_FOUND`，错误动词 405（:28-39）。错误响应仍带 `instanceId` / `runId` / `revision`。
无鉴权（文档承认，`docs/EXTERNAL_AUTOMATION_CONTROL_API.md:161`）。

文档 vs 代码：文档列的八条路由全实现；`/openapi/v1.json` 只在散文里（文档 :19，README :23）；
`openapi.json` 列了八条路径（:12-64），`servers.url` :9。文档说 `delayMs` 范围 1-60000（:152），
API 却把 `DelayMs ?? 0` 直接交给引擎（:84）。文档「实施状态」块（:14-22）已过时。

### 6.2 无日志文件

没有日志库，没有文件 sink。`src/` 下 grep Serilog / NLog / ILogger / AppendAllText：无（只有测试
runner 里的 `Console.WriteLine`）。`ClearProviders()` 之后也没有请求日志——**HTTP 请求不在任何地方
留痕**。唯一机制是三个 `event EventHandler<string> LogEmitted`：`SimulatorEngine.cs:53`（格式 :886
`$"{DateTime.Now:HH:mm:ss.fff}  {message}"`）、`ModbusTcpServer.cs:41`（:414）、
`AutomationHttpServer.cs:34`（:103-104）。汇点是 `MainViewModel` 订阅（:36-38）、UI 线程（:155）、
`ObservableCollection<string> Logs` 500 条封顶（`AppendLog` :207-212）→ `ListBox x:Name="LogList"`
（`MainWindow.xaml:158`），有自动滚动复选框（:152）与清空按钮（:154）。**无导出，仅内存，退出即丢；
本地 `HH:mm:ss.fff`，无日期。**

自动化相关的行：监听（`AutomationHttpServer.cs:62`）、停止（:95）、HTTP reset（`SimulatorEngine.cs:348`）；
cargo / close / override 经 `TransitionOutcome.LogMessage`（:871-872）。README :76-80 描述了窗内日志，
包括 FC01/FC02 轮询抑制（`MainViewModel.LogReadRequests` :86-99；`ModbusTcpServer.cs:334`）。

`runId` / `commandId` / `instanceId` / sublot **没有一个**进日志行。`runId` / `commandId` 只用于控制
（`SimulatorEngine.cs:329-332,371-391`，校验 :807-826）与 HTTP 响应回显（`AutomationApi.cs:102-112,200-213`）。
模拟器没有 sublot / 条码概念。

### 6.3 配置与 `instanceId`

单个 `simulator.settings.json`，位于 `Path.Combine(AppContext.BaseDirectory, ...)`（`MainWindow.xaml.cs:20`）→
`SimulatorSettings.Load`（:18-25），System.Text.Json 大小写不敏感、允许注释与尾逗号（:110-117）。
加载失败 → MessageBox + 重抛（`MainWindow.xaml.cs:25-29`）。**无任何覆盖手段**，路径不可配。命令行：无
（`App.xaml.cs:1-7` 空；`App.xaml:4` `StartupUri`；无 `Main` / `GetCommandLineArgs`；宿主 `Args = []`）。

键树（POCO `SimulatorSettings.cs:7-16,119-159`；值 `simulator.settings.json:1-40`）：`schemaVersion "1.0"`；
`instanceId "agv-slot-simulator-01"`；`agvId "AGV-01"`；
`modbus.{listenAddress 127.0.0.1, port 1502, unitId 255, strictUnitId, doPduBaseAddress 100, diPduBaseAddress 200, doChannelCount 16, diChannelCount 16}`；
`automation.{listenAddress, port 58006}`；
`defaults.{outputMode Level|Pulse, pulseWidthMs 500, unlockFeedbackDelayMs 100, lockFeedbackDelayMs 100, lightCurtainFeedbackDelayMs 50, lockFeedbackModel DoorLatch|FollowOutput, autoPopDoorOnUnlock true, maxOpenDoors 1}`；
`powerOnDoStates int[16]`；
`slots[8]{slotIndex, displayNumber, unlockDoChannel, lockFeedbackDiChannel, lightCurtainDiChannel, unlockActiveLevel, lockedActiveLevel, obstructedActiveLevel}`。

自由字符串：`instanceId`（非空即可，:31-32）→ 每个 HTTP 响应（`src/SQCD_8005AGV_Simulator.Core/Services/SimulatorEngine.cs:450`；
`AutomationApi.cs:103,133,177,207`），不进 UI、不进日志——**最好的载体**。`agvId`（:33-34）只校验、
从不再读——死字段。`schemaVersion` 回显在响应里（:449）。`runId` 不可配：每次 reset（含启动）都是
`$"run-{Guid.NewGuid():N}"`（`SimulatorEngine.cs:789`，已核对；构造 :46），`revision` 重置为 1（:790）。

`remote-ops/onboard-hmi/scripts/08-deploy-slots-simulator.ps1` 只检查文件存在（:113,176），不按现场打补丁。
车上启动（`10-start-onboard-stack.ps1`）：`$SimulatorExe = 'C:\8005\SlotsSimulator\SQCD_8005AGV_Simulator.exe'`（:68），
无参数无环境（`New-ScheduledTaskAction -Execute '$Path' -WorkingDirectory '$workingDirectory'` :127），
工作目录 = exe 目录（:120），Interactive / Limited（:124-128），约 3 秒后注销任务（:137-138）。就绪判定：
端口 1502 与 58006 处于 Listen（`Get-StackState` :94-97；等待 :195-199）；进程名
`SQCD_8005AGV_Simulator`（:88）。`11-drive-journey.ps1` 调 snapshot / cargo / close-door（:15,61,78），
每次调用 `commandId` 为新 GUID（:137,156,165）。

## 7. 协议信封里的关联字段现状

`8005-agv-protocol/schemas/envelope.schema.json`（`$id`
`https://schemas.8005-agv.local/wire-to-gate/v1/envelope.schema.json`，title `ProtocolEnvelope`）：

- `messageId`：:31-33，`$ref` 到 `common/types.schema.json#/$defs/Id`；
- `correlationId`：:34-43，`anyOf [Id, null]`；
- 两者都在 `required` 里（:71-82，`messageId` :77、`correlationId` :78），`additionalProperties: false`（:84）。

信封其余必填：`protocolVersion`（const 1）、`profileId`（const `WIRE_TO_GATE_MVP`）、
`protocolReleaseVersion`、`protocolReleaseManifestSha256`、`messageType`、`agvId`、
`sessionGeneration`（可空）、`sentAt`、`payload`（:6-69）。每个 `schemas/messages/*.schema.json`
都复制同一对字段（例如 `DurableAck.schema.json:32-35,83-84`）。

三个进程的落地情况：

| 进程 | 信封在哪 | `messageId` 进日志 | `correlationId` 进日志 | 落到哪 |
| --- | --- | --- | --- | --- |
| ControlServer.Host | `OnboardMessageProcessor.cs`（解析 `correlationId` :386） | 只有拒收事件 1101 的 `RejectedMessageId` | 否 | SQLite `ProtocolInbox.MessageId` / `ProtocolOutbox.MessageId`、`JourneyRuntimes` 的 `*MessageId` 列 |
| SQCD.Agv.Wpf | `WireToGateProtocol.cs:46-57` | 否（旧路径的 `messageId=` 文本在 W2G 下不走） | 否 | SQLite journal `WireToGateDurableOutbox` 等；58007 的 submit 响应回显 `MessageId` |
| slots-simulator | 不在协议上 | 不适用 | 不适用 | — |

所以今天要把一条 W2G 消息从车载端追到服务端，唯一可用的路径是**两边的 SQLite**，加上 58007
submit 响应里的 `MessageId`；日志文件对这件事零贡献。

## 8. 结论：最小改动建议

按进程，先列**不改代码**就能做的，再列必须走 PR 的。

**ControlServer.Host**（我方仓，直接改）
- 不改代码：lab 起它时加 `Serilog__Properties__RunId=<runId>`（或 `--Serilog:Properties:RunId=`），
  并加一个文件 sink `Serilog__WriteTo__0__Name=File`、`Serilog__WriteTo__0__Args__path=<logRoot>\control-server.ndjson`、
  `Serilog__WriteTo__0__Args__formatter=Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact`
  ——包已在产物里，与安装脚本生成的生产配置同形。L2 编排器今天一个 `Serilog__*` 都没设，这是
  `Invoke-L2Scenario.ps1:258-287` 一处改动。
- 改代码（小）：在 `OnboardMessageProcessor.ProcessAsync` 入口 `LogContext.PushProperty("MessageId", …)` /
  `("CorrelationId", …)` / `("AgvId", …)`，并给**被接受**的信封加一条 Information 事件——今天只记拒收。
  这是让日志参与关联链的唯一办法。

**替身**（我方仓，直接改）
- 不改代码：`--FakeRiot:instanceId=<runId>` / `--FakeMesIngest:instanceId=<runId>` 即可把 run 标识
  带进每个控制面响应；L2 今天传的是常量。
- 改代码（小）：加 `AddJsonConsole` 或请求日志，否则替身 stdout 永远只有 hosting 行；修两处 openapi
  漂移（`NOT_FOUND` 命名、`DemandCommand.demandId`、DELETE 的 400）。

**SQCD.Agv.Wpf**（Kun Wang 的仓，`w2g/*` 分支 + PR）
- 不改代码：**没有**能注入 `runId` 的口。lab 只能从 58007 响应里读它自己生成的 `RunId`，并把
  `agv-{yyyyMMdd}.log` 按时间窗切片。
- 改代码：（a）`FileAppLogger` 行模板加一列（或改 JSON）并让 `WireToGateSessionClient` 在收发时写
  `messageId` / `correlationId`；（b）给 `automation` 节加一个 `runId` 或 `tags` 自由字段回显到响应；
  （c）**先解决第 5.5 节的矛盾**——不解决，真车上的 58007 根本不存在，前两条无从谈起。这一条是
  安全评审级别的决定，PR 描述里要写清楚。

**slots-simulator**（Kun Wang 的仓，`w2g/*` 分支 + PR）
- 不改代码：把 `simulator.settings.json` 的 `instanceId` 写成 run 相关值（部署脚本今天不动这个文件，
  需要 lab 自己在启动前改）。
- 改代码：给 `AutomationHttpServer` / `MainViewModel.AppendLog` 加一个文件 sink，并让 HTTP 请求进
  `LogEmitted`——今天 HTTP 连 UI 日志都不进。

**协议**：不需要动。信封已经有 `messageId` / `correlationId`，缺的是各进程把它们写出来。

## 9. 未能核实的点

- 四份分项调查为只读静态审阅，**没有在任何机器上实跑**验证 `Serilog__Properties__RunId` 在
  CompactJson 里的实际输出形状；结论基于 `ReadFrom.Configuration` 已接线与 `Serilog.Settings.Configuration`
  对 `Properties` 节的既有语义。
- 车载端调查基于分支 `w2g/recovery-entry-mid-session-readiness`（`550dbe9`），不是 Kun Wang 的
  `OnboardHmi_MVP`；两者在日志与自动化平面上的差异未比对。
- 车上实际部署的 `appsettings.json` 未读取（不碰远程机器）；「`environment` 为 `Production`」的判断
  来自模板与 `appsettings.Production.example.json:2`，以及部署脚本的渲染逻辑。
- ControlServer 快照文件里 28 张表的行号取自 `ControlServerDbContextModelSnapshot.cs` 当前版本，
  下一个迁移就会漂移；表名才是稳定引用。
- FakeOnboard 与 ClockSkewProxy 的 openapi 文档未逐条与路由比对（票据只点名了 FakeRiot 与 FakeMesIngest）。
- 模拟器文档「实施状态」块过时的具体条目未逐条列出。
