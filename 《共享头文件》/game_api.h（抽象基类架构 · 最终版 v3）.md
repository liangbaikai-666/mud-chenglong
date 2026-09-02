# game_api.h（抽象基类架构 · 最终版 v3）

cpp

```cpp
#pragma once
#include <string>
#include <vector>
#include <set>
#include <map>

// ==================== 枚举：游戏状态 ====================
enum class GameState { PLAYING, STORY_DEATH, VICTORY, GAME_OVER };
enum class BattleResult { WIN, LOSE, FLEE };
enum class GuideType { BASIC, COMBAT, TALISMAN, STATUS, INVENTORY };

// ==================== 枚举：符咒ID ====================
enum class TalismanID {
    RAT    = 1,  // 鼠符咒：赋予无生命物体生命（悬浮/操控傀儡）
    OX     = 2,  // 牛符咒：力大无穷（物理攻击加成）
    TIGER  = 3,  // 虎符咒：平衡阴阳/灵魂归位（防止混乱状态）
    RABBIT = 4,  // 兔符咒：极速行动（额外行动点/先手）
    DRAGON = 5,  // 龙符咒：爆破火焰（范围伤害）
    SNAKE  = 6,  // 蛇符咒：隐身（闪避率提升/潜行）
    HORSE  = 7,  // 马符咒：治愈/驱散负面状态
    SHEEP  = 8,  // 羊符咒：灵魂出窍/移形换影（位移技能）
    MONKEY = 9,  // 猴符咒：变形（变身敌人/改变属性）
    ROOSTER= 10, // 鸡符咒：悬浮/飞行（无视地形/高空攻击）
    DOG    = 11, // 狗符咒：不死/幸运（免死一次/暴击率提升）
    PIG    = 12  // 猪符咒：镭射眼（远程单体高伤）
};

// ==================== 枚举：状态效果 ====================
enum class StatusID {
    MONKEY = 1,        // 猴符咒：变形状态（属性改变/变身敌人）
    RESIST_UP = 2,     // 抗性提升（减少受到的异常状态持续时间）

    // ---- 预留扩展位（后续按需要新增） ----
    // POISON = 3,      // 中毒：每回合持续扣血
    // BURN = 4,        // 灼烧：每回合扣血+攻击降低
    // STUN = 5,        // 眩晕：跳过行动回合
    // HASTE = 6,       // 加速：额外行动点
    // SLOW = 7,        // 减速：减少行动点
    // INVISIBLE = 8,   // 隐身：闪避率提升
    // SHIELD = 9,      // 护盾：吸收下次伤害
    // WEAKNESS = 10,   // 虚弱：攻击力降低
    // SILENCE = 11,    // 沉默：无法使用技能
};

// ==================== 枚举：道具ID ====================
enum class ItemID {
    POTION_HP = 1,     // 回血药水：恢复目标一定量HP（数值待平衡调整）
    POTION_MP = 2      // 回蓝药水：恢复目标一定量MP（数值待平衡调整）

    // ---- 预留扩展位（后续按需要新增） ----
    // TALISMAN_FRAGMENT = 3,  // 符咒碎片：收集一定数量可合成符咒
    // KEY_NORMAL = 4,         // 普通钥匙：解锁特定关卡门
    // BOMB = 5,               // 炸弹：对敌人造成范围伤害
    // SCROLL = 6,             // 技能卷轴：学习新技能
};

// ==================== 结构体 ====================
struct LevelConfig {
    std::string id;
    int layer;
    std::vector<std::string> prerequisites;
    std::vector<std::vector<class Enemy*>> waves;
    float difficulty_coef;
    struct RewardConfig {
        int money_base;
        float item_drop_rate;
        bool talisman_fragment_reward;
    } rewards;
};

struct DialogueNode {
    std::string speaker;
    std::string text;
    std::vector<std::string> options;
};

// ==================== 前置声明 ====================
class Entity;
class Player;
class Enemy;

// ==================== 模块1：战斗引擎（抽象基类） ====================
class IBattleEngine {
public:
    virtual ~IBattleEngine() = default;

    // 开始战斗，传入敌人列表
    virtual void start_battle(const std::vector<Enemy*>& enemies) = 0;

    // 处理一回合的战斗行动（由外部主循环或关卡系统驱动）
    virtual void process_actions() = 0;

    // 获取当前战斗结果（战斗未结束时返回进行中状态）
    virtual BattleResult get_battle_result() const = 0;

    // 查询战斗是否仍在进行中
    virtual bool is_battle_active() const = 0;

    // ---- 预留扩展位 ----
    // virtual void set_level_manager(ILevelManager* mgr) = 0;
    // virtual void add_player_action(const Action& action) = 0;
    // virtual void force_end_battle() = 0;
};

// ==================== 模块2：符咒管理器（抽象基类） ====================
class ITalismanManager {
public:
    virtual ~ITalismanManager() = default;

    // 添加符咒到背包
    virtual void add_to_backpack(TalismanID id) = 0;

    // 查询是否拥有某个符咒
    virtual bool has_talisman(TalismanID id) const = 0;

    // 触发符咒效果（战斗中调用）
    virtual TalismanID trigger_talisman(TalismanID id, Entity* caster) = 0;

    // 检查是否可以触发某个符咒
    virtual bool can_trigger(TalismanID id) const = 0;

    // 回合结束处理（符咒持续时间/冷却等）
    virtual void on_turn_end() = 0;

    // 战斗开始初始化
    virtual void on_battle_start() = 0;

    // 状态效果被施加时通知（供符咒系统响应）
    virtual void on_status_effect_applied(Entity* target, StatusID id, int duration) = 0;

    // 请求移除状态效果（符咒系统→状态系统）
    virtual void request_effect_removal(Entity* target, StatusID id) = 0;

    // ---- 预留扩展位 ----
    // virtual std::vector<TalismanID> get_owned_talismans() const = 0;
    // virtual void remove_talisman(TalismanID id) = 0;
    // virtual int get_talisman_count(TalismanID id) const = 0;
};

// ==================== 模块3：状态管理器（抽象基类） ====================
class IStatusEffectManager {
public:
    virtual ~IStatusEffectManager() = default;

    // 添加状态到目标
    virtual void add_status(Entity* target, StatusID id, int duration) = 0;

    // 移除目标的指定状态
    virtual void remove_status(Entity* target, StatusID id) = 0;

    // 查询目标是否有指定状态
    virtual bool has_status(Entity* target, StatusID id) const = 0;

    // 获取目标指定状态的剩余持续时间
    virtual int get_status_duration(Entity* target, StatusID id) const = 0;

    // 回合结束处理（持续时间递减/持续效果触发）
    virtual void on_turn_end() = 0;

    // 战斗开始初始化
    virtual void on_battle_start() = 0;

    // 战斗结束清理
    virtual void on_battle_end() = 0;

    // 符咒效果请求（符咒系统→状态系统）
    virtual void on_talisman_effect_request(Entity* caster, TalismanID talisman_id) = 0;

    // 施加剧情状态（关卡系统调用）
    virtual void apply_story_status(Entity* target, StatusID id, int duration) = 0;

    // ---- 预留扩展位 ----
    // virtual void clear_all_statuses(Entity* target) = 0;
    // virtual std::vector<StatusID> get_all_statuses(Entity* target) const = 0;
};

// ==================== 模块4：背包管理器（抽象基类） ====================
class IInventoryManager {
public:
    virtual ~IInventoryManager() = default;

    // 添加道具（返回是否成功）
    virtual bool add_item(ItemID item_id, int count) = 0;

    // 移除道具（返回是否成功）
    virtual bool remove_item(ItemID item_id, int count) = 0;

    // 查询是否拥有指定道具（至少1个）
    virtual bool has_item(ItemID item_id) const = 0;

    // 获取指定道具的持有数量
    virtual int get_item_count(ItemID item_id) const = 0;

    // 增加金钱
    virtual void add_money(int amount) = 0;

    // 消费金钱（返回是否成功）
    virtual bool consume_money(int amount) = 0;

    // 查询当前金钱
    virtual int get_money() const = 0;

    // 使用道具（对目标生效）
    virtual void use_item(ItemID item_id, Entity* target) = 0;

    // ---- 预留扩展位 ----
    // virtual void clear_inventory() = 0;
    // virtual std::vector<std::pair<ItemID, int>> get_all_items() const = 0;
};

// ==================== 模块5：关卡系统（抽象基类） ====================
class ILevelManager {
public:
    virtual ~ILevelManager() = default;

    // 初始化关卡进度
    virtual void init_progress() = 0;

    // 查询是否可以进入指定关卡
    virtual bool can_enter_level(const std::string& level_id) const = 0;

    // 关卡通关回调
    virtual void on_level_cleared(const std::string& level_id) = 0;

    // 自动补全缺失符咒（剧情需要）
    virtual void auto_grant_missing_talismans() = 0;

    // 为指定关卡开始战斗
    virtual void start_battle_for_level(const std::string& level_id) = 0;

    // 战斗结束回调
    virtual void on_battle_end(BattleResult result, const std::vector<Enemy*>& defeated) = 0;

    // 查询当前所在层
    virtual int get_current_layer() const = 0;

    // 查询是否已集齐12符咒
    virtual bool is_all_talismans_collected() const = 0;

    // ---- 预留扩展位 ----
    // virtual std::string get_current_level_id() const = 0;
    // virtual std::set<std::string> get_cleared_levels() const = 0;
};

class IDialogueManager {
public:
    virtual ~IDialogueManager() = default;

    // 获取NPC对话内容
    virtual std::vector<DialogueNode> get_npc_dialogue(const std::string& npc_id) const = 0;

    // 推进剧情阶段
    virtual void advance_stage(const std::string& new_stage) = 0;

    // NPC交互触发
    virtual std::string on_npc_interact(const std::string& npc_id) = 0;

    // 触发引导提示
    virtual std::string trigger_guide_hint(GuideType type) const = 0;

    // ---- 预留扩展位 ----
    // virtual void set_current_npc(const std::string& npc_id) = 0;
};

class IGameStateManager {
public:
    virtual ~IGameStateManager() = default;

    // 获取当前游戏状态
    virtual GameState get_game_state() const = 0;

    // 检查剧情死亡条件
    virtual void check_story_death() = 0;

    // 胜利
    virtual void on_victory() = 0;

    // 游戏结束
    virtual void on_game_over() = 0;

    // ---- 预留扩展位 ----
    // virtual void set_game_state(GameState state) = 0;
};
```