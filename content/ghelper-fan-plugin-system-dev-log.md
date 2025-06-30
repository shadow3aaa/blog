---
title: 为 Asus 笔记本添加 PID 风扇控制
date: 2025-06-30
math: true
---

正值夏日，AMD 酷热难当，好似室内暖机，惹得风扇轰鸣。

## PID 算法简介

PID（比例-积分-微分）控制器是一种广泛应用于工业控制系统的反馈回路机制。<!--more-->它通过计算误差（设定值与实际值之间的差异），并根据比例、积分和微分三个部分来调整输出，从而实现对系统的精确控制。
PID 控制器的基本公式为：

$$
u(t) = K_p e(t) + K_i \int_0^t e(\tau) d\tau + K_d \frac{de(t)}{dt}
$$

其中：

- \\( u(t) \\) 是控制输出
- \\( e(t) \\) 是误差（设定值与实际值的差异）
- \\( K_p \\)、\\( K_i \\)、\\( K_d \\) 分别是比例、积分和微分增益系数

## 探寻如何设置风扇转速

我使用的是 windows，有一精妙软件`GHelper`可用，盖其底层使用 ACPI 设置风扇曲线。则设一平坦曲线，或可实现风扇转速的手动控制。

当即 fork 它，思之直接写死，颇为不雅，应加入一插件系统，接受传感器数据，返回转速，如此甚便。

## 设计插件系统

为了实现最大的灵活性，我决定使用[Lua](https://www.lua.org/)作为插件的脚本语言。Lua 轻量、快速且易于嵌入到 .NET 应用中。通过 `NLua` 这个库，C#可以方便地调用 Lua 脚本。

插件的核心 API 非常简单，只包含一个必须实现的全局函数：`update`。

```lua
function update(sensors, dt)
    -- 你的逻辑代码在这里

    return {
        cpu_fan = 50, -- CPU 风扇转速 (0-100)
        gpu_fan = 50, -- GPU 风扇转速 (0-100)
    }
end
```

- `sensors`: 一个包含所有硬件传感器数据的 `table`，例如 `sensors.cpu_temp`。
- `dt`: 距离上次调用的时间差（秒），对于 PID 算法中的微分和积分计算至关重要。
- **返回值**: 一个包含风扇转速目标的 `table`。

## 实现核心逻辑

为了管理插件的生命周期，我创建了 `FanPluginManager.cs`。它负责：

1.  **发现插件**: 在 `Plugins/fans/` 目录下查找所有 `.lua` 文件。
2.  **加载插件**: 读取并执行选定的 Lua 脚本。
3.  **运行插件**: 定期调用脚本中的 `update` 函数，并将返回的风扇转速应用到系统中。

```csharp
// GHelper/app/Plugins/FanPluginManager.cs

public Dictionary<string, int> RunPlugin(Dictionary<string, float> sensorData)
{
    // ...
    try
    {
        LuaFunction updateFunction = _luaState["update"] as LuaFunction;
        // ...
        double dt = AppConfig.Get("sensor_timer", 1000) / 1000.0;
        var result = updateFunction.Call(sensorTable, dt);

        if (result != null && result.Length > 0 && result[0] is LuaTable resultTable)
        {
            var fanSpeeds = new Dictionary<string, int>();
            foreach (var key in resultTable.Keys)
            {
                fanSpeeds[key.ToString()] = Convert.ToInt32(resultTable[key]);
            }
            return fanSpeeds;
        }
    }
    //...
}
```

同时，我修改了主循环，将传感器刷新率提高到 `100ms`，以便插件能够更实时地响应温度变化，并定期调用 `AutoFans()` 方法来执行插件逻辑。

## 编写默认的 PID 控制器

为了提供一个开箱即用的高级示例，我编写了一个名为 `default.lua` 的 PID 控制器插件。

```lua
-- GHelper/app/Plugins/fans/default.lua

local cpu_pid = {
  target_temperature = 93,
  Kp = 12.0,
  Ki = 0.8,
  Kd = 1.2,
  integral = 0,
  previous_error = 0
}

function calculate_fan_speed(pid_controller, current_temperature, dt)
  local error = current_temperature - pid_controller.target_temperature

  pid_controller.integral = pid_controller.integral + error * dt
  pid_controller.integral = math.max(integral_min, math.min(integral_max, pid_controller.integral))

  local derivative = (error - pid_controller.previous_error) / dt

  local pid_output = (pid_controller.Kp * error) + (pid_controller.Ki * pid_controller.integral) + (pid_controller.Kd * derivative)

  pid_controller.previous_error = error

  local base_fan_speed = 30
  local fan_speed = base_fan_speed + pid_output

  return math.max(base_fan_speed, math.min(100, fan_speed))
end

function update(sensors, dt)
  local fan_speeds = {}
  if sensors.cpu_temp then
    fan_speeds.cpu_fan = calculate_fan_speed(cpu_pid, sensors.cpu_temp, dt)
  end
  -- ... for gpu_fan
  return fan_speeds
end
```

用户可以直接在脚本中调整 `target_temperature`（目标温度）以及 `Kp`, `Ki`, `Kd` 这三个 PID 核心参数，以满足个性化的散热需求。
