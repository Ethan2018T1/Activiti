# 自动化设备统一管理平台架构方案

## 1. 目标

构建一个可统一接入无人机、机器狗、无人车、机器人、无人船、机械手臂等自动化设备的综合管理平台，覆盖：

- 统一设备抽象
- 实时状态监控
- 巡检任务下发（支持 2D 多边形）
- 3D 地图构建与回传
- 多设备地图融合
- 融合地图可视化与持久化

## 2. 统一设备对象模型

```text
Device
├─ id: String
├─ name: String
├─ type: DeviceType (DRONE / DOG / UGV / USV / ARM / ROBOT)
├─ protocol: DeviceProtocol (MAVLink / ROS2 / OPC-UA / Custom)
├─ capabilities: Set<Capability>
├─ status: DeviceStatus
└─ telemetry: TelemetrySnapshot
```

### 2.1 关键子模型

- `Capability`：MOVE、TAKE_OFF、LAND、SCAN_3D、ARM_CONTROL、PAYLOAD_DROP...
- `DeviceStatus`：ONLINE、OFFLINE、IN_MISSION、CHARGING、ERROR
- `TelemetrySnapshot`：
  - `batteryPercent`
  - `position` (lat/lon/alt + EPSG)
  - `heading/speed`
  - `timestamp`

该模型用于后端服务统一处理、前端统一渲染与任务编排。

## 3. 后端架构（Spring Boot + Spring Cloud）

### 3.1 服务拆分建议

- `device-registry-service`：设备注册、能力管理、协议适配配置
- `telemetry-gateway`：Netty WebSocket 接入设备实时数据
- `mission-service`：巡检任务管理、2D 区域任务编排
- `map-service`：点云入库、地图切片、融合任务调度
- `fusion-worker`：Open3D/PCL 融合计算（异步）
- `api-gateway`：前端统一入口

### 3.2 数据流

1. 设备通过协议适配层上报电量/位置/点云片段。
2. `telemetry-gateway` 使用 Netty WebSocket 接收并归一化。
3. 实时状态写入 Redis；同时将事件投递至 Kafka/RabbitMQ。
4. `mission-service` 消费状态与任务事件，更新任务生命周期。
5. `map-service` 消费点云流并触发融合任务。
6. `fusion-worker` 输出融合结果，写入对象存储 + PostGIS 索引。

### 3.3 存储分层

- PostgreSQL + PostGIS：
  - 设备元数据
  - 任务区域（Polygon）
  - 地图空间索引（bounding box、轨迹）
- Redis：
  - 设备最近状态（TTL）
  - WebSocket 订阅会话映射
- 对象存储（MinIO/S3）：
  - 原始点云（pcd/ply）
  - 融合结果

## 4. 任务管理（2D 多边形）

任务示例：

```json
{
  "taskId": "task-001",
  "taskType": "INSPECTION",
  "deviceIds": ["drone-01", "dog-02"],
  "area": {
    "type": "Polygon",
    "coordinates": [[[121.1, 31.2], [121.2, 31.2], [121.2, 31.3], [121.1, 31.3], [121.1, 31.2]]],
    "srid": 4326
  },
  "scanProfile": {
    "density": "HIGH",
    "overlap": 0.2
  }
}
```

## 5. 3D 地图构建与融合

- 设备侧：SLAM 或激光雷达点云采集
- 平台侧：
  - 点云清洗（去噪、下采样）
  - 坐标系配准（ICP / NDT）
  - 多源融合（按时间窗 + 空间网格）
  - 结果版本化（`map_id + version`）

建议融合策略：

- 准实时：5~10 秒批次窗口
- 大任务离线重融合：任务结束后进行高精融合

## 6. 前端架构（Vue3 + TS + Three.js + Potree）

- 页面模块：
  - 设备总览：在线状态、轨迹、电量告警
  - 任务中心：区域绘制、任务下发、进度跟踪
  - 地图中心：点云加载、融合状态、成果保存
- 状态管理（Pinia）：
  - `deviceStore`：设备/遥测状态
  - `missionStore`：任务生命周期
  - `mapStore`：地图构建与融合状态
- 通信：
  - Axios：配置接口、任务操作
  - WebSocket：实时状态与融合进度推送

## 7. Docker / Kubernetes 部署建议

- 基础组件：PostgreSQL+PostGIS、Redis、Kafka/RabbitMQ、MinIO
- 应用组件：各业务服务 + Fusion Worker（可 GPU 节点调度）
- K8s 关键配置：
  - HPA：按 WebSocket 连接数与消息吞吐扩缩
  - Affinity：Fusion Worker 与高性能节点绑定
  - ConfigMap/Secret：协议参数、设备密钥、对象存储凭据

## 8. MVP 迭代建议

1. **M1**：设备抽象 + 实时状态监控（电量/位置）
2. **M2**：2D 多边形任务管理与执行反馈
3. **M3**：单设备 3D 地图构建与查看
4. **M4**：多设备地图准实时融合与保存

