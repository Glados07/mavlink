# `fuel_cell.xml` 协议说明

本文说明 [`fuel_cell.xml`](../../message_definitions/v1.0/fuel_cell.xml) 中的燃料电池 MAVLink 定义。该方言包含 `common.xml`，并由 `all.xml` 引入。消息 ID `61000`～`61002` 属于项目私有分配，尚非 MAVLink 官方分配；这些 ID 需要 MAVLink 2。通信各端应使用同一版本的 XML 生成代码。

## 消息总览

| ID | 消息 | 方向与用途 |
| --- | --- | --- |
| `61000` | `FUEL_CELL_CONTROL` | 向飞控提交燃料电池控制器的持续启停命令，或请求一次故障复位脉冲。 |
| `61001` | `FUEL_CELL_STATUS` | 飞控上报控制器连接、设备状态、当前命令及控制请求处理结果。 |
| `61002` | `FUEL_CELL_TELEMETRY` | 待补充。 |

## `FUEL_CELL_CONTROL`（61000）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `request_id` | `uint32_t` | 发送方生成的非零请求编号。重试同一请求时，编号、`control_command` 和 `reset_fault` 必须保持一致。飞控仅缓存最近一次请求，缓存键包含发送方 system/component ID 和请求编号。 |
| `target_system` | `uint8_t` | 目标飞行器的 system ID；`0` 表示向所有系统广播。 |
| `target_component` | `uint8_t` | 目标飞控的 component ID；`0` 表示向所有组件广播。 |
| `control_command` | `uint8_t` | 持续保存并发送的控制命令，取值见 `FUEL_CELL_CONTROL_COMMAND`。 |
| `reset_fault` | `uint8_t` | `0` 为普通启停请求；`1` 请求故障复位。复位会将持续命令切为 `OFF`，由飞控在三个 CAN 控制帧周期内置位独立复位位，随后清除。紧急停机优先，可取消正在进行的复位脉冲。 |

母线电压、母线电流和请求功率由飞控决定，不能通过本消息修改。新请求通常要求 CAN 设备连接健康；例外是紧急停机抢占正在进行的复位脉冲。重复请求使用原处理结果，不再次执行命令；若同一缓存请求编号配合不同控制字段使用，则请求被拒绝。处理结果通过 `FUEL_CELL_STATUS` 返回。

### `FUEL_CELL_CONTROL_COMMAND`

| 值 | 枚举 | 含义 |
| ---: | --- | --- |
| `0` | `FUEL_CELL_CONTROL_COMMAND_OFF` | 关闭控制器。 |
| `1` | `FUEL_CELL_CONTROL_COMMAND_ON` | 启动控制器。 |
| `2` | `FUEL_CELL_CONTROL_COMMAND_EMERGENCY_STOP` | 请求紧急停机。 |

## `FUEL_CELL_STATUS`（61001）

飞控每 `200 ms` 发布一次状态，即使 CAN 设备尚未连接；连接状态变化或控制请求处理后还会立即发布。接收端若连续 `1 s` 未收到本消息，应将这路状态视为不可用。

| 字段 | 类型 | 单位或枚举 | 说明 |
| --- | --- | --- | --- |
| `response_request_id` | `uint32_t` | — | `control_result` 对应的请求编号；`0` 表示尚未处理请求。 |
| `requested_power` | `uint16_t` | `W` | 飞控当前用于 CAN 控制帧的请求功率。只读，不代表设备已确认该功率。 |
| `soc` | `uint16_t` | `c%`（`0.01%`） | 最近一次有效的设备锂电池荷电状态；仅在 `connection_state == OK` 时有效。 |
| `dcdc_output_current` | `uint16_t` | `cA`（`0.01 A`） | 最近一次有效的设备 DC/DC 输出电流。原始值 `0`～`65535` 均合法；与 PMU 母线电流无关。断连后保留旧值，但仅在连接正常时有效。 |
| `device_state` | `uint8_t` | `FUEL_CELL_DEVICE_STATE` | 最近一次有效的 4 位设备状态。`0`～`13` 已定义，未知值 `14`、`15` 原样转发；仅在连接正常时有效。 |
| `control_command` | `uint8_t` | `FUEL_CELL_CONTROL_COMMAND` | 飞控已保存、正向设备发送的命令；设备实际运行状态应看 `device_state`。 |
| `control_result` | `uint8_t` | `MAV_RESULT` | `response_request_id` 对应控制请求的处理结果。 |
| `connection_state` | `uint8_t` | `FUEL_CELL_CONNECTION_STATE` | 当前控制器连接状态。 |

`control_result` 的相关取值：

| `MAV_RESULT` | 在此消息中的含义 |
| --- | --- |
| `ACCEPTED` | 飞控已保存命令或已启动复位脉冲；不表示设备已完成操作或清除故障。 |
| `TEMPORARILY_REJECTED` | CAN 设备未连接，或复位脉冲期间收到普通启停请求。 |
| `DENIED` | 缓存的请求编号被用于不同的控制字段。 |
| `UNSUPPORTED` | 控制字段值无效。 |

### `FUEL_CELL_CONNECTION_STATE`

| 值 | 枚举 | 含义 |
| ---: | --- | --- |
| `0` | `FUEL_CELL_CONNECTION_STATE_NEVER_CONNECTED` | 尚未收到有效设备状态帧。 |
| `1` | `FUEL_CELL_CONNECTION_STATE_OK` | 设备通信正常。 |
| `2` | `FUEL_CELL_CONNECTION_STATE_ERROR_OCCURRED` | 设备曾连接，但当前通信已丢失。 |

### `FUEL_CELL_DEVICE_STATE`

| 值 | 枚举 | 含义 |
| ---: | --- | --- |
| `0` | `FUEL_CELL_DEVICE_STATE_POWERING_ON` | 上电中。 |
| `1` | `FUEL_CELL_DEVICE_STATE_POWER_ON_COMPLETE` | 上电完成。 |
| `2` | `FUEL_CELL_DEVICE_STATE_ENTERING_STANDBY` | 进入待机中。 |
| `3` | `FUEL_CELL_DEVICE_STATE_STANDBY` | 待机完成。 |
| `4` | `FUEL_CELL_DEVICE_STATE_STARTING` | 启动中。 |
| `5` | `FUEL_CELL_DEVICE_STATE_START_COMPLETE` | 启动完成。 |
| `6` | `FUEL_CELL_DEVICE_STATE_RUNNING` | 运行中。 |
| `7` | `FUEL_CELL_DEVICE_STATE_STOPPING` | 正常停机中。 |
| `8` | `FUEL_CELL_DEVICE_STATE_STOP_COMPLETE` | 正常停机完成。 |
| `9` | `FUEL_CELL_DEVICE_STATE_FAULT_STOPPING` | 故障停机中。 |
| `10` | `FUEL_CELL_DEVICE_STATE_FAULT_STOP_COMPLETE` | 故障停机完成。 |
| `11` | `FUEL_CELL_DEVICE_STATE_EMERGENCY_STOPPING` | 紧急停机中。 |
| `12` | `FUEL_CELL_DEVICE_STATE_EMERGENCY_STOP_COMPLETE` | 紧急停机完成。 |
| `13` | `FUEL_CELL_DEVICE_STATE_OUTPUT_CURRENT_LIMITED` | 输出电流受限。 |

设备状态是原始 4 位值。`14`、`15` 未定义，但接收端不应仅因收到这两个值就判定连接异常或更改控制权限。

## `FUEL_CELL_TELEMETRY`（61002）

> 待补充：具体说明由负责该消息的合作者编写。
