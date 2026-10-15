# 现有业务 YAML 接入 UBSocketV2 裸设备修改指南

由旧版《现有业务 YAML 接入 UBSE 设备修改指南》另存改写。适用于 Docker Compose 业务，手动选择一个非 bonding UDMA 及其 EID；不运行设备插件，不调用 UBSE 或 `agg expose`。

## 1. 确认设备名和 EID 索引

在宿主机查看 `urma_admin show --all`，选择现场已配置可用、确认未分配给其他业务的“设备名 + EID 索引”。例如设备 `udmac0d1e2` 的 `eid0`，索引填写 `0`，不是完整 EID 或表格行号。以下值仅为示例，须替换为现场结果。

## 2. 合并到原业务 YAML

将下面的 `app` 换成原服务名，只合并 `devices` 和 `environment`。已有同名字段追加或更新内容，不重复创建 YAML 键；原环境变量及其他业务配置保留。

```yaml
services:
  app:
    devices:
      - "/dev/uburma/udmac0d1e2:/dev/uburma/udmac0d1e2:rw"
    environment:
      UBSOCKET_UB_DEV: "udmac0d1e2"
      UBSOCKET_UB_EID_IDX: "0"
      UBSOCKET_UB_BOUNDING_DEV: "0"
      UBSOCKET_UB_BUSY_POLLING: "0"
```

- 设备路径和 `UBSOCKET_UB_DEV` 必须对应同一个裸 UDMA；只映射选中的节点。
- 当前业务库 `urma_setting.cpp` 读取的是 `UBSOCKET_UB_BOUNDING_DEV`（带 `BOUNDING`），不要照设备插件示例误写为 `UBSOCKET_UB_BONDING_DEV`。
- 若原 `environment` 使用列表形式，改成相应的 `- UBSOCKET_UB_DEV=udmac0d1e2` 等条目，不能与映射形式混用。

## 3. 删除旧版接入配置

若原 YAML 已按 UBSE 指南修改：删除旧 VFE/bonding 的设备映射、`/run/ubse-start` tmpfs 和等待放行的 `entrypoint`，将 `entrypoint`、`command`、`restart` 恢复为接入 UBSE 前的原业务配置。原指南为 UBSE 增加的辅助设备映射按 V2 实际依赖确认，业务本来需要的挂载保留。

保留业务镜像、库加载方式、配置卷、网络、端口和运行用户，不新增等待容器或手动放行步骤。按原 Compose 项目参数校验配置并重建目标服务；仅重启旧容器不会应用新增设备和环境变量。

## 适用边界

- 业务镜像须具备兼容的 UBSocketV2/URMA 库，所选设备和 EID 已在目标运行环境可用；本方案不创建 EID、不改变其网络命名空间归属。
- 同一 UDMA 的多个 EID 共享设备节点，环境变量仅选择 EID，不提供 EID 访问隔离。手工接入需自行记录占用关系。
- `urma_admin show --all` 可能显示全量设备，不能据此判断容器获配范围；以 YAML 设备映射、环境变量和实际业务通信结果核验。
- 本片段适用于 Compose；Kubernetes YAML 应通过 V2 设备插件申请资源，不能直接添加 Compose 的 `devices` 字段。

本文是 YAML 修改模板，未验证现场设备通信。
