#### 模块3：状态效果（Buff/Debuff）管理器（修补版）

**共享头文件与前置声明**

- `#include "shared_attributes.h"`（统一包含，虽本模块主要使用字符串ID，但保持项目结构一致）
- `#include "character_fwd.h"`（内含 `class Entity;`前置声明，供函数参数使用）

#### 功能1：状态效果基础数据配置

**参数（共5项/状态）：**

- `id`(string)：英文ID，如 `"poison"`（**跨模块修补：状态ID统一为string**）
- `name`(string)：中文名称，如 `"中毒"`
- `type`(enum)：类型，`BUFF`、`DEBUFF`（**已移除 NEUTRAL**）
- `duration`(int)：默认持续回合数（0表示瞬时或特殊条件）
- `effect_value`(float)：数值效果系数（如力量减半为0.5，凋零降上限10%为0.1）

**实现方法：**

- 定义结构体 `StatusEffectData`封装上述参数。
- `load_all_status_effects()`：从配置文件或常量加载所有状态数据，存入 `std::map<std::string, StatusEffectData> all_effects`。
- `get_effect_data(const std::string& id)`：返回对应状态的常量引用。

#### 功能2：状态增删查改管理

**参数（共3项）：**

- `active_statuses`(map<string, int>)：实体当前生效状态，键为状态ID（string），值为剩余持续回合。
- `entity`(Entity*)：指向玩家或敌人的指针（**已通过 character_fwd.h前置声明**）。
- `status_type_filter`(enum)：用于批量清除的过滤类型（`BUFF`/`DEBUFF`）。

**实现方法：**

- `void add_status(const std::string& id, int duration)`：**与 game_api.h对齐**，插入或刷新 `active_statuses[id] = duration`。
- `void remove_status(const std::string& id)`：从 `active_statuses`中移除。
- `bool has_status(const std::string& id) const`：**与 game_api.h对齐**，返回 `active_statuses.count(id) > 0`。
- `int get_remaining_turns(const std::string& id)`：返回对应值，不存在返回0。
- `void clear_statuses_by_type(StatusType type)`：**与 game_api.h对齐**，遍历并移除所有匹配类型的状态。

#### 功能3：状态叠加与冲突规则

**参数（共2项）：**

- `same_status_policy`(enum)：同状态处理策略，固定为 `REFRESH_DURATION`（刷新回合）。
- `horse_talisman_purify_list`(vector<string>)：马符咒可净化的Debuff ID列表，如 `{"poison", "weak", "wither"}`。

**实现方法：**

- `apply_status(const std::string& id, int duration)`：调用 `add_status`，若已存在则只更新持续回合（不叠加层数）。
- `purify_by_horse_talisman()`：遍历 `active_statuses`，若ID在净化列表中，则移除该Debuff，并添加对应的正面Buff（如 `"anti_poison"`）。
- `dog_talisman_prevent_death(int& current_hp, int max_hp)`：若 `current_hp`将降至0，且狗符咒生效，则强制设为1，并移除所有持续扣血Debuff（但不免疫剧情即死）。

#### 功能4：状态持续与回合倒计时

**参数（共1项）：**

- `tick_interval`(enum)：触发时机，枚举值 `{PLAYER_TURN_END, ENEMY_TURN_END, BATTLE_END}`（**与 game_api.h中 TurnEndType一致**）。

**实现方法：**

- `on_turn_end(TurnEndType type)`：遍历 `active_statuses`，将所有值大于0的减1；若减至0，调用 `remove_status`并取消效果。
- `on_battle_end()`：清除所有非永久状态（持续条件为 `BATTLE`的移除，永久的保留）。

#### 功能5：状态效果实时计算与属性修正

**参数（共3项，已移除 base_magic_resist）：**

- `base_strength`(int)：基础力量值。
- `base_hit_rate`(float)：基础命中率。
- `current_hp_max`(int)：当前生命上限。

**实现方法：**

- `int modify_strength(int base) const`：**与 game_api.h对齐**，若存在 `weak`，返回 `floor(base * 0.5)`；否则返回 `base`。此函数被模块1 `BattleEngine`在伤害计算前调用。
- `modify_hit_rate(float base)`：若存在 `invisible`（自身），返回 `base`（自身隐身不影响自己命中，但敌人打你时命中率降为20%，此逻辑在敌人侧计算）；若存在 `invisible`且是敌方视角，返回 `0.2`。
- `modify_magic_damage(int damage)`：若存在 `resist_up`，返回 `floor(damage / 2)`；否则返回 `damage`。（**注：此函数供模块2符咒伤害计算时调用，检查目标是否有 resist_up状态**）
- `modify_max_hp(int base_max)`：若存在 `wither`，返回 `floor(base_max * 0.9)`；否则返回 `base_max`。

#### 功能6：特殊状态逻辑处理

**参数（共1项，已移除 fall_dmg_amount 和 wither_max_hp_reduction）：**

- `monkey_form_turns`(int)：猴化持续回合，**固定为2**（跨模块修补确认）。

**实现方法：**

- `on_monkey_form_start(Entity* entity)`：设置 `monkey_form`状态，降低力量和上限，限制文本交互选项。
- `on_monkey_form_end(Entity* entity)`：移除状态，恢复原属性。

#### 功能7：与符咒模块和战斗引擎的交互

**参数（无独立存储，主要为事件回调）：**

- 回调函数指针或观察者接口。

**实现方法：**

- `void on_talisman_effect_request(const std::string& talisman_id, Entity* caster)`：**与 game_api.h对齐**，符咒模块（模块2）调用，根据符咒ID（string）添加对应状态（如牛符咒加力量Buff，实际由符咒模块直接修改属性，状态模块只记录Buff标志）。
- `void on_battle_engine_turn_end(TurnEndType type)`：战斗引擎（模块1）调用，触发 `on_turn_end`。
- `void register_status_change_callback(std::function<void(const std::string&, bool)> callback)`：注册回调，当状态添加/移除时通知UI或剧情模块。