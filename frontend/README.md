# Frontend（管理平台前端）

## 技术栈

- Vue 3 + TypeScript
- Element Plus
- Pinia
- Three.js + Potree
- Axios + WebSocket

## 页面建议

- 设备总览页
- 任务管理页
- 地图可视化页

## 状态管理

- `deviceStore`：设备实时状态、电量、轨迹
- `missionStore`：任务创建、执行、完成状态
- `mapStore`：点云加载、融合进度、成果保存

## 交互建议

- 通过 WebSocket 订阅设备状态和地图融合进度
- 通过 Axios 完成设备配置与任务下发
- 地图中心支持图层开关、点云密度调节、版本切换
