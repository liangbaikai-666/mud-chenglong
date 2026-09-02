# 模块3：状态效果（Buff/Debuff）管理器（新架构适配版）

## 继承关系

cpp

```
class StatusEffectManager : public IStatusEffectManager { ... };
```

实现 `game_api.h`中定义的 `IStatusEffectManager`全部纯虚方法，并扩展内部实现。

------

## 共享头文件与前置声明

cpp

```
#include "game_api.h"           // 包含 IStatusEffectManager、StatusID、TalismanID 等
#include "character_fwd.h"      // 模块1提供，内含 class Entity; class Player; class Enemy; 前置声明
```

> **变更说明**：原设计引用 `shared_attributes.h`（状态ID字符串常量）。新架构下 `StatusID`已定义为 enum class（MONKEY=1, RESIST_UP=2，其余预留扩展），不再需要字符串ID。`game_api.h`已包含 `Entity`前置声明，`character_fwd.h`仍用于 `Player`/`Enemy`前置声明。

------

## 功能1：状态效果基础数据配置

**参数（共5项/状态）：**

- `id`(StatusID)：枚举值，如 `StatusID::MONKEY`、`StatusID::RESIST_UP`（预留扩展位后续按需添加 POISON、BURN 等）
- `name`(string)：中文名称，如 `"猴化"`、`"抗性提升"`
- `type`(enum)：类型，`BUFF`、`DEBUFF`（已移除 NEUTRAL）
- `duration`(int)：默认持续回合数（0表示瞬时或特殊条件）
- `effect_value`(float)：数值效果系数（如力量减半为0.5）

**实现方法：**

- 定义结构体 `StatusEffectData`封装上述参数（id 类型为 `StatusID`）。
- `load_all_status_effects()`：硬编码或从配置文件读取，存入 `std::map<StatusID, StatusEffectData> all_effects`。
- `get_effect_data(StatusID id)`：返回对应状态的常量引用。

------

## 功能2：状态增删查改管理

**参数（共3项）：**

- `active_statuses`(std::map<StatusID, int>)：实体当前生效状态，键为状态ID（StatusID），值为剩余持续回合。
- `entity`(Entity*)：指向玩家或敌人的指针。
- `status_type_filter`(enum)：用于批量清除的过滤类型（`BUFF`/`DEBUFF`）。

**实现 IBattleEngine 接口的方法：**

- `void add_status(Entity* target, StatusID id, int duration) override`：插入或刷新 `active_statuses[id] = duration`，并触发状态变化回调。
- `void remove_status(Entity* target, StatusID id) override`：从 `active_statuses`中移除。
- `bool has_status(Entity* target, StatusID id) const override`：返回 `active_statuses.count(id) > 0`。
- `int get_status_duration(Entity* target, StatusID id) const override`：返回对应值，不存在返回0。

**模块3内部方法（不放入接口）：**

- `void clear_statuses_by_type(Entity* target, StatusType type)`：遍历并移除所有匹配类型的状态。

------

## 功能3：状态叠加与冲突规则

**参数（共2项）：**

- `same_status_policy`(enum)：同状态处理策略，固定为 `REFRESH_DURATION`（刷新回合）。
- `horse_talisman_purify_list`(std::vector<StatusID>)：马符咒可净化的Debuff ID列表，如 `{StatusID::POISON, StatusID::WEAK, StatusID::WITHER}`（预留，当前未定义这些枚举值，后续添加后生效）。

**实现方法：**

- `apply_status(Entity* target, StatusID id, int duration)`：调用 `add_status`，若已存在则只更新持续回合（不叠加层数）。
- `purify_by_horse_talisman(Entity* target)`：遍历 `active_statuses`，若ID在净化列表中，则移除该Debuff，并添加对应的正面Buff（如 `StatusID::RESIST_UP`）。
- `dog_talisman_prevent_death(Entity* target, int& current_hp, int max_hp)`：若 `current_hp`将降至0，且狗符咒生效，则强制设为1，并移除所有持续扣血Debuff（但不免疫剧情即死）。

------

## 功能4：状态持续与回合倒计时

**参数（共1项）：**

- `tick_interval`(enum)：触发时机，枚举值 `{PLAYER_TURN_END, ENEMY_TURN_END, BATTLE_END}`。

**实现 IBattleEngine 接口的方法：**

- `void on_turn_end() override`：遍历所有实体的 `active_statuses`，将所有值大于0的减1；若减至0，调用 `remove_status`并取消效果。
- `void on_battle_start() override`：战斗开始时初始化状态系统（重置临时状态标记）。
- `void on_battle_end() override`：清除所有非永久状态（持续条件为 `BATTLE`的移除，永久的保留）。

------

## 功能5：状态效果实时计算与属性修正

**参数（共3项）：**

- `base_strength`(int)：基础力量值。
- `base_hit_rate`(float)：基础命中率。
- `current_hp_max`(int)：当前生命上限。

**实现方法（模块内部使用，供战斗引擎查询）：**

- `int modify_strength(int base) const`：若存在 `StatusID::WEAK`（预留），返回 `floor(base * 0.5)`；否则返回 `base`。此函数被模块1 `BattleEngine`在伤害计算前调用。
- `float modify_hit_rate(float base, bool is_self)`：若存在 `StatusID::INVISIBLE`（预留）且 `is_self == false`（敌方视角），返回 `0.2`；否则返回 `base`。
- `int modify_magic_damage(int damage) const`：若存在 `StatusID::RESIST_UP`，返回 `floor(damage / 2)`；否则返回 `damage`。（供模块2符咒伤害计算时调用，检查目标是否有 RESIST_UP 状态）
- `int modify_max_hp(int base_max) const`：若存在 `StatusID::WITHER`（预留），返回 `floor(base_max * 0.9)`；否则返回 `base_max`。

------

## 功能6：特殊状态逻辑处理

**参数（共1项）：**

- `monkey_form_turns`(int)：猴化持续回合，固定为2（跨模块确认）。

**实现方法：**

- `void on_monkey_form_start(Entity* entity)`：设置 `StatusID::MONKEY`状态，降低力量和上限，限制文本交互选项。
- `void on_monkey_form_end(Entity* entity)`：移除状态，恢复原属性。

------

## 功能7：与符咒模块和战斗引擎的交互

**跨模块指针（通过抽象基类，由外部注入）：**

cpp



```
ITalismanManager* talisman_mgr;   // 模块2（查询符咒状态）
IBattleEngine* battle_engine;     // 模块1（通知状态变化）
```

**注入方法（实现类内部，不放入 IStatusEffectManager 接口）：**

- `set_talisman_manager(ITalismanManager* mgr)`
- `set_battle_engine(IBattleEngine* engine)`

**实现 IStatusEffectManager 接口的方法：**

- `void on_talisman_effect_request(Entity* caster, TalismanID talisman_id) override`：符咒模块（模块2）调用，根据符咒ID添加对应状态（如牛符咒加力量Buff，实际由符咒模块直接修改属性，状态模块只记录Buff标志）。**注意参数顺序：caster 在前，talisman_id 在后，与 game_api.h 一致。**

**模块3内部方法（不放入接口）：**

- `void notify_status_change(StatusID id, bool applied)`：状态添加/移除时通知UI或剧情模块（通过回调）。