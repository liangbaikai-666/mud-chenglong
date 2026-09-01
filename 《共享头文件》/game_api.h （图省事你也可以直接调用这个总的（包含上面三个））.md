



~~~cpp
game_api.h共享头文件最终版。

以下是汇总所有模块依赖的枚举、结构体、前置声明、常量、跨模块函数签名的完整共享头文件：

#ifndef GAME_API_H
#define GAME_API_H

#include <string>
#include <vector>

// ==================== 前置声明 ====================
class Entity;
class Player;
class Enemy;

// ==================== 枚举定义 ====================

/**
 * 游戏全局状态
 */
enum class GameState {
    PLAYING,        // 正常游玩
    STORY_DEATH,    // 剧情即死（如缺少关键道具/符咒）
    VICTORY,        // 通关胜利
    GAME_OVER       // 游戏结束（HP归零且无复活）
};

/**
 * 战斗结果
 */
enum class BattleResult {
    WIN,            // 胜利
    LOSE,           // 失败
    FLEE            // 逃跑
};

/**
 * 小玉引导提示类型
 */
enum class GuideType {
    TALISMAN_USAGE,     // 符咒使用提示
    ITEM_USAGE,         // 道具使用提示
    LEVEL_ENTRANCE,     // 关卡入口提示
    BOSS_WARNING,       // Boss战警告
    STORY_HINT          // 剧情提示
};

// ==================== 结构体定义 ====================

/**
 * 关卡配置
 */
struct LevelConfig {
    std::string id;                  // 关卡ID
    int layer;                       // 所属层（1~4）
    std::vector<std::string> prerequisites; // 前置关卡ID列表
    int waves;                       // 敌人波次数
    float difficulty_coef;           // 难度系数（影响敌人属性和奖励）
    RewardConfig rewards;            // 通关奖励
};

/**
 * 奖励配置
 */
struct RewardConfig {
    int money_base;                  // 金钱基数
    float item_drop_rate;            // 道具掉落率（0.0~1.0）
    int talisman_fragment_reward;    // 符咒碎片奖励数量
};

/**
 * 对话节点
 */
struct DialogueNode {
    std::string speaker;             // 说话者名称
    std::string text;                // 对话文本
    std::vector<std::string> options; // 可选分支（空=无选项，自动继续）
};

// ==================== 常量定义 ====================

/**
 * 符咒ID常量（1~12）
 */
namespace TalismanID {
    const int RAT    = 1;   // 鼠符咒
    const int OX     = 2;   // 牛符咒
    const int TIGER  = 3;   // 虎符咒
    const int RABBIT = 4;   // 兔符咒
    const int DRAGON = 5;   // 龙符咒
    const int SNAKE  = 6;   // 蛇符咒
    const int HORSE  = 7;   // 马符咒
    const int GOAT   = 8;   // 羊符咒
    const int MONKEY = 9;   // 猴符咒
    const int ROOSTER = 10; // 鸡符咒
    const int DOG    = 11;  // 狗符咒
    const int PIG    = 12;  // 猪符咒
}

/**
 * 状态ID常量
 */
namespace StatusID {
    const std::string MONKEY     = "MONKEY";      // 猴化（Debuff，固定2回合）
    const std::string RESIST_UP  = "RESIST_UP";   // 抵抗提升（Buff）
    // ... 其他状态ID按需扩展
}

/**
 * 道具ID常量
 */
namespace ItemID {
    const std::string POTION_HP  = "POTION_HP";   // 生命药水
    const std::string POTION_MP  = "POTION_MP";   // 魔力药水
    // ... 其他道具ID按需扩展
}

// ==================== 跨模块接口声明（各模块公开API） ====================

// ---- 模块1：战斗引擎 ----
class BattleEngine {
public:
    void start_battle(const std::vector<Enemy*>& enemy_list);
    void process_actions();
    // 内部调用模块2：int trigger_talisman(int id, Entity* caster);
    // 内部调用模块2：void grant_extra_action(); // 限玩家回合
};

// ---- 模块2：符咒管理器 ----
class TalismanManager {
public:
    void add_to_backpack(int id);
    int trigger_talisman(int id, Entity* caster);
    bool can_trigger(int id) const;
    void on_turn_end();
    void on_battle_start();
    // 供模块3调用：
    void on_status_effect_applied(int effect_id, int duration);
    void request_effect_removal(int effect_id);
};

// ---- 模块3：状态效果管理器 ----
class StatusEffectManager {
public:
    void add_status(Entity* target, const std::string& id, int duration);
    void remove_status(Entity* target, const std::string& id);
    bool has_status(Entity* target, const std::string& id) const;
    int get_status_duration(Entity* target, const std::string& id) const;
    void on_turn_end();
    void on_battle_start();
    void on_battle_end();
    // 供模块2调用：
    void on_talisman_effect_request(int talisman_id, Entity* caster);
    // 供模块5调用：
    void apply_story_status(const std::string& id, Entity* target);
};

// ---- 模块4：背包管理器 ----
class InventoryManager {
public:
    bool add_item(const std::string& item_id, int count);
    bool remove_item(const std::string& item_id, int count);
    bool has_item(const std::string& item_id, int count) const;
    void add_money(int amount);
    bool consume_money(int amount);
    int get_money() const;
    bool use_item(const std::string& item_id, Player* user);
};

// ---- 模块5：关卡管理器 ----
class LevelManager {
public:
    void init_progress();
    bool can_enter_level(const std::string& level_id) const;
    void on_level_cleared(const std::string& level_id);
    void auto_grant_missing_talismans();
    void start_battle_for_level(const std::string& level_id);
    void on_battle_end(BattleResult result, const std::vector<Enemy*>& defeated);
};

// ---- 模块5：对话管理器 ----
class DialogueManager {
public:
    std::vector<DialogueNode> get_npc_dialogue(const std::string& npc_id) const;
    void advance_stage(const std::string& new_stage);
    std::string on_npc_interact(const std::string& npc_id);
    std::string trigger_guide_hint(GuideType type) const;
};

// ---- 模块5：游戏状态管理器 ----
class GameStateManager {
public:
    GameState get_game_state() const;
    void check_story_death();
    void on_victory();
    void on_game_over();
};

#endif // GAME_API_H
~~~

