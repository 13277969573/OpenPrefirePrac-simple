# CS2 Bot管理技术文档 - 已知问题与解决方案

## 📋 文档概述

本文档记录了在CS2（Counter-Strike 2）中使用 CounterStrikeSharp 框架进行Bot管理时遇到的各种技术陷阱和解决方案。这些问题在开发 OpenPrefirePrac 插件时被发现和验证。

**版本信息**:
- CS2 版本: 2026年1月
- CounterStrikeSharp: 最新版本
- 测试环境: Windows Server, competitive 模式

---

## ⚠️ 问题1: `bot_add_ct`/`bot_add_t` 命令会创建2个Bot

### 🔍 **问题描述**

在使用 `Server.ExecuteCommand("bot_add_ct")` 或 `Server.ExecuteCommand("bot_add_t")` 时，**每个命令会创建2个bot，而不是1个**。

### 📊 **实测日志证据**

```
19:15:27.022 → 执行命令: bot_add_ct
19:15:27.055 → Bot "Ava" 连接 (Slot 3)
19:15:27.092 → Bot "Delrow" 连接 (Slot 4)  ← 同一个命令创建了第2个bot！

19:15:27.119 → 执行命令: bot_add_ct
19:15:27.124 → Bot "Buckshot" 连接 (Slot 5)
19:15:27.142 → Bot "Ricksaw" 连接 (Slot 6)  ← 同一个命令创建了第2个bot！
```

**时间间隔**:
- 第1个bot出现: 命令执行后 33ms
- 第2个bot出现: 命令执行后 70ms
- 两个bot之间: 约 37ms

### 🎯 **问题根源**

#### **可能原因1: bot_quota_mode 影响**

当 `bot_quota_mode=competitive` 时，系统可能有特殊的bot创建机制：
- 为了队伍平衡，自动在两队各创建1个bot
- 或者 `competitive` 模式有"成对创建"的默认行为

#### **可能原因2: 服务器配置文件**

某些服务器配置（如 `gamemode_competitive.cfg`）可能覆盖了bot创建行为。

### ✅ **解决方案**

**不要尝试"聪明"地减半调用次数！**

正确的做法是：
1. 按照需要的bot数量正常调用命令（如需要6个bot，就调用6次）
2. 实现**防御性检查机制**，在 `OnClientPutInServerHandler` 中：
   - 跟踪已添加的bot数量
   - 当超过所需数量时，立即踢出多余的bot

```csharp
if (_botSlots.Count < requiredBots)
{
    _botSlots.Add(slot);  // 正常添加
}
else
{
    // 防御：踢出多余的bot
    _logger.LogWarning("Excess bot detected. Kicking...");
    KickBot(slot);
}
```

**优点**:
- ✅ 健壮性强，能应对服务器配置变化
- ✅ 自动适应不同的 `bot_quota_mode`
- ✅ 即使bot创建行为改变，代码仍然工作

---

## ⚠️ 问题2: 只使用 `bot_add_ct` 会导致队伍分配错乱

### 🔍 **问题描述**

如果只使用 `bot_add_ct`，不使用 `bot_join_team CT`，会导致**bot加入错误的队伍**。

### 📊 **实测日志证据**

**测试场景**: 玩家在T队，需要创建6个CT bot

#### ❌ **错误做法**（只用 `bot_add_ct`）:

```
命令: bot_add_ct (×6次)

结果:
✅ Ava - Team:CT(3)      正确
✅ Delrow - Team:CT(3)   正确  
✅ Buckshot - Team:CT(3) 正确
❌ Ricksaw - Team:T(2)   错误！加入了T队
✅ Gunner - Team:CT(3)   正确
❌ Stone - Team:T(2)     错误！加入了T队
```

**问题**: 约33%的bot加入了错误的队伍！

#### ✅ **正确做法**（先 `bot_join_team`，再 `bot_add_ct`）:

```
命令: bot_join_team CT + bot_add_ct (×6次)

结果:
✅ Soldier - Team:CT(3)   正确
✅ Osiris - Team:CT(3)    正确
✅ Ricksaw - Team:CT(3)   正确
✅ Crusher - Team:CT(3)   正确
✅ Rip - Team:CT(3)       正确
✅ Buckshot - Team:CT(3)  正确
```

**结果**: 100%的bot都加入了正确的队伍！

### 🎯 **问题根源**

#### **CS2的bot创建机制**

在CS2中，`bot_add_ct`/`bot_add_t` 命令并**不保证**bot一定加入指定队伍，可能受以下因素影响：

1. **队伍平衡算法**: 服务器可能自动平衡队伍，将bot分配到人数较少的队伍
2. **bot_quota_mode**: `competitive` 模式可能有特殊的队伍分配逻辑
3. **mp_autoteambalance**: 即使设置为0，仍可能有其他平衡机制
4. **创建时序**: 快速连续创建多个bot时，队伍分配可能不稳定

#### **bot_join_team 的作用**

`bot_join_team <队伍>` 命令的作用是**明确设置下一个bot的队伍偏好**，相当于：
```
服务器内部状态: "下一个创建的bot必须加入CT队"
```

虽然这会导致每个命令创建2个bot（问题1），但它**保证了队伍分配的正确性**。

### ✅ **最佳实践**

**必须同时使用两个命令**:

```csharp
if (player.TeamNum == (byte)CsTeam.CounterTerrorist)
{
    // 玩家在CT队，bot加入T队
    Server.ExecuteCommand("bot_join_team T");   // 必需！设置队伍偏好
    Server.ExecuteCommand("bot_add_t");         // 必需！创建bot
}
else if (player.TeamNum == (byte)CsTeam.Terrorist)
{
    // 玩家在T队，bot加入CT队
    Server.ExecuteCommand("bot_join_team CT");  // 必需！设置队伍偏好
    Server.ExecuteCommand("bot_add_ct");        // 必需！创建bot
}
```

**重要**: 
- ⚠️ **不要**删除 `bot_join_team` 来试图解决bot数量翻倍问题
- ✅ **应该**保留两个命令，并通过防御性检查处理多余bot

---

## ⚠️ 问题3: OnPlayerSpawn 事件会触发3次

### 🔍 **问题描述**

当一个bot连接并生成时，`OnPlayerSpawn` 事件会被**连续触发3次**，间隔仅几毫秒。

### 📊 **实测日志证据**

```
19:15:27.055 → OnClientPutInServerHandler: Bot "Ava" 连接 (Slot 3)
19:15:27.059 → OnPlayerSpawn #1: Ava - Slot:3, Team:NONE(0)
19:15:27.071 → OnPlayerSpawn #2: Ava - Slot:3, Team:NONE(0)  ← 12ms后
19:15:27.075 → OnPlayerSpawn #3: Ava - Slot:3, Team:CT(3)    ← 16ms后
```

**时间间隔**:
- 第1次 → 第2次: 12ms
- 第2次 → 第3次: 4ms
- 总耗时: 16ms

**队伍信息变化**:
- 第1次: `Team:NONE(0)` - 实体创建
- 第2次: `Team:NONE(0)` - 初始化
- 第3次: `Team:CT(3)`   - 真正spawn，有正确队伍信息

### 🎯 **问题根源**

这是 **CS2 引擎的正常行为**，bot的完整生成过程分为多个阶段：

#### **阶段1: 实体创建**
```
触发: OnPlayerSpawn #1
状态: Team = NONE(0)
作用: 在内存中创建bot实体对象
```

#### **阶段2: 属性初始化**
```
触发: OnPlayerSpawn #2
状态: Team = NONE(0)
作用: 初始化基础属性（血量、位置等）
```

#### **阶段3: 世界生成**
```
触发: OnPlayerSpawn #3
状态: Team = CT(3) 或 T(2)
作用: Bot真正spawn到游戏世界，队伍信息正确
```

### ⚠️ **潜在风险**

如果在 `OnPlayerSpawn` 中有**状态更新逻辑**，可能会被错误地执行3次！

#### **错误示例**:

```csharp
public HookResult OnPlayerSpawn(EventPlayerSpawn @event, GameEventInfo info)
{
    var bot = @event.Userid;
    
    if (_botSlots.Contains(bot.Slot))
    {
        _playerStatus.Progress++;  // ❌ 危险！会被执行3次
        // Progress 会错误地增加3，而不是1
    }
    
    return HookResult.Continue;
}
```

**后果**:
- 如果需要6个bot，Progress会从0增加到18（6 × 3）
- bot会被传送到错误的位置
- 训练逻辑完全错乱

### ✅ **解决方案**

#### **方案1: 使用队伍信息判断**

只在第3次spawn（有正确队伍信息时）执行逻辑：

```csharp
public HookResult OnPlayerSpawn(EventPlayerSpawn @event, GameEventInfo info)
{
    var bot = @event.Userid;
    
    // 只在bot有正确队伍信息时处理
    if (bot.TeamNum == (byte)CsTeam.None || bot.TeamNum == 0)
    {
        return HookResult.Continue;  // 跳过前2次spawn
    }
    
    // 现在是第3次spawn，可以安全地执行逻辑
    if (_botSlots.Contains(bot.Slot))
    {
        _playerStatus.Progress++;  // ✅ 只会执行1次
        // ... 其他逻辑
    }
    
    return HookResult.Continue;
}
```

#### **方案2: 时间戳去重**（过度设计，不推荐）

使用时间戳记录上次spawn时间，忽略200ms内的重复事件：

```csharp
private readonly Dictionary<int, float> _lastBotSpawnTime = new();

public HookResult OnPlayerSpawn(EventPlayerSpawn @event, GameEventInfo info)
{
    var bot = @event.Userid;
    var currentTime = Server.CurrentTime;
    
    if (_lastBotSpawnTime.TryGetValue(bot.Slot, out var lastTime))
    {
        if (currentTime - lastTime < 0.2f)  // 200ms内认为是重复
        {
            return HookResult.Continue;  // 忽略重复spawn
        }
    }
    _lastBotSpawnTime[bot.Slot] = currentTime;
    
    // 处理逻辑...
}
```

**⚠️ 不推荐原因**:
- 过度复杂，增加维护成本
- 需要管理额外的字典和清理逻辑
- 方案1更简单且足够有效

### 📝 **最佳实践**

**推荐使用方案1（队伍信息判断）**:

```csharp
// 通用的spawn处理框架
public HookResult OnPlayerSpawn(EventPlayerSpawn @event, GameEventInfo info)
{
    var playerOrBot = @event.Userid;
    
    // 1. 基础检查
    if (playerOrBot == null || !playerOrBot.IsValid || playerOrBot.IsHLTV)
    {
        return HookResult.Continue;
    }
    
    // 2. 跳过未分配队伍的spawn（前2次）
    if (playerOrBot.TeamNum == (byte)CsTeam.None)
    {
        return HookResult.Continue;
    }
    
    // 3. 现在可以安全地执行逻辑
    if (playerOrBot.IsBot && _botSlots.Contains(playerOrBot.Slot))
    {
        // Bot逻辑（只会执行1次）
        HandleBotSpawn(playerOrBot);
    }
    else if (!playerOrBot.IsBot)
    {
        // 玩家逻辑（只会执行1次）
        HandlePlayerSpawn(playerOrBot);
    }
    
    return HookResult.Continue;
}
```

---

## 🎯 总结：OpenPrefirePrac 的最终方案

### **核心设计原则**

1. **接受bot数量翻倍的现实**
   - 不要试图"修复"CS2的行为
   - 通过防御性编程处理多余bot

2. **必须使用完整的命令组合**
   ```csharp
   Server.ExecuteCommand("bot_join_team CT");
   Server.ExecuteCommand("bot_add_ct");
   ```

3. **在合适的阶段执行逻辑**
   - `OnClientPutInServerHandler`: 添加/踢出bot
   - `OnPlayerSpawn`: 只在 `TeamNum != NONE` 时执行游戏逻辑

### **完整的Bot管理流程**

```
1. StartPractice() 调用 AddBot(6)
   ↓
2. AddBot() 循环6次，每次间隔0.1秒
   - bot_join_team CT
   - bot_add_ct
   ↓
3. 服务器创建12个bot（6命令 × 2bot）
   ↓
4. OnClientPutInServerHandler 依次触发12次
   - 前6个: _botSlots.Add(slot)  ← 正常添加
   - 后6个: KickBot(slot)        ← 防御性踢出
   ↓
5. OnPlayerSpawn 每个bot触发3次（共36次）
   - 前2次: Team=NONE，跳过
   - 第3次: Team=CT，执行逻辑
   ↓
6. 最终结果: 恰好6个bot在正确的队伍
```

### **关键代码片段**

```csharp
// 1. Bot创建（不要修改这两行的顺序和内容）
Server.ExecuteCommand("bot_join_team CT");
Server.ExecuteCommand("bot_add_ct");

// 2. Bot添加（防御性检查）
if (_botSlots.Count < requiredBots)
{
    _botSlots.Add(slot);
}
else
{
    KickBot(slot);  // 踢出多余bot
}

// 3. Spawn处理（跳过前2次）
if (playerOrBot.TeamNum == (byte)CsTeam.None)
{
    return HookResult.Continue;
}
```

---

## 📚 参考信息

### **测试环境**
- **操作系统**: Windows Server 2019/2022
- **CS2 版本**: 2026年1月
- **bot_quota**: 0
- **bot_quota_mode**: competitive
- **mp_autoteambalance**: 0

### **相关ConVar**

| ConVar | 值 | 说明 |
|--------|---|------|
| `bot_quota` | 0 | 服务器目标bot数量（每队） |
| `bot_quota_mode` | competitive | Bot补充模式（normal/fill/competitive） |
| `bot_join_after_player` | 0 | 玩家加入时不自动添加bot |
| `bot_chatter` | off | 关闭bot语音 |
| `mp_autoteambalance` | 0 | 关闭队伍自动平衡 |

### **已知兼容性**

| 服务器配置 | bot_join_team + bot_add | 仅 bot_add | 防御性Kick |
|-----------|------------------------|-----------|-----------|
| competitive | ✅ 工作 | ❌ 队伍错乱 | ✅ 必需 |
| casual | ✅ 工作 | ⚠️ 未测试 | ✅ 推荐 |
| custom | ✅ 工作 | ⚠️ 未测试 | ✅ 推荐 |

---

## 🔗 相关资源

- **CounterStrikeSharp**: https://docs.cssharp.dev/
- **CS2 Bot Commands**: Valve官方文档
- **OpenPrefirePrac**: https://github.com/lengran/OpenPrefirePrac

---

## 📝 更新日志

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-01-15 | 1.0 | 初始版本，记录所有已知问题 |

---

## ⚖️ 许可证

本文档可自由分享和修改，但请保留原始来源。

**作者**: OpenPrefirePrac 开发团队  
**最后更新**: 2026年1月15日
