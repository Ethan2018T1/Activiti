# Backend（管理平台服务端）

## 模块建议

- `services/device-registry-service`
- `services/telemetry-gateway`
- `services/mission-service`
- `services/map-service`
- `workers/fusion-worker`

## 关键接口

- `POST /api/devices`：设备注册
- `GET /api/devices/{id}/state`：设备最新状态
- `POST /api/missions`：下发巡检任务（含 Polygon）
- `GET /api/maps/{id}`：获取融合地图元数据
- `POST /ws/telemetry`：设备实时数据接入（WebSocket）

## 数据与中间件

- PostgreSQL + PostGIS（空间任务与轨迹）
- Redis（实时状态）
- Kafka / RabbitMQ（遥测与地图任务流）

## 地图处理链路

采集 -> 预处理 -> 配准 -> 融合 -> 存储 -> 前端可视化
