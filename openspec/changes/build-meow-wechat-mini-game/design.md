# 设计

## 背景

参见 [proposal.md](proposal.md) 了解动机与范围。当前仓库只有 [meow_plan.md](../../../meow_plan.md)，没有 Cocos Creator 工程、游戏源码、云函数或测试；以下为新建产品的目标设计。设计约束包括微信小游戏运行环境、手机触控、离线期间可继续操作，以及服务端对货币与奖励的最终校验。

## 目标 / 非目标

**目标：**
- 将玩法规则与 Cocos 场景表现分离，使连线规则、经济规则和猫咪状态可独立验证。
- 以配置驱动关卡、猫咪品种、商店商品、奖励和词库，便于后续调参与扩充内容。
- 以云端已确认状态作为玩家经济和拥有物的权威记录，同时支持本地缓存和离线操作队列。
- 以可运行的核心关卡作为第一阶段基础，再逐步接入养成、云同步和后续收集玩法。

**非目标：**
- 本轮仅完善规划及生成提示词，不实现代码或生成游戏资源；后续实现需显式进入 apply。不扩展支付、排行榜或社交排名规则。
- 不要求所有二期能力在首个可玩版本中上线；按下文阶段实施。
- 不假定设计文档中的云开发运行时版本当前仍受支持。

## 决策

### 客户端按表现、领域和平台适配分层

Cocos Creator 场景负责 UI、动画和触控；领域层负责路径匹配、棋盘状态、关卡结果、属性衰减、奖励资格与商品规则；微信平台适配层封装登录、生命周期、分享、广告和云调用。领域规则不直接依赖场景节点或微信 API，便于使用确定性输入输出进行测试。

**替代方案：** 将规则直接写在场景脚本中实现更快，但会令关卡逻辑与 UI 难以分别测试，也会增加微信 API 替换和离线测试的成本。

### 关卡内容由配置驱动，并在进入游戏前验证

关卡配置描述棋盘大小、图片种类、时间/步数、特殊方块和奖励。生成器按配置成对放置图片；每关 6×8 初始棋盘使用 6 至 8 种图片组成 24 对。生成后执行合法移动检测，不合格则重新排列或重新生成；局内无合法匹配时可洗牌恢复移动。特殊方块的组合先作为显式关卡配置支持，避免未定义组合静默改变基础规则。

**替代方案：** 完全随机生成后只在玩家卡住时提示洗牌，较简单，但可能频繁打断关卡；预先验证能减少不可玩棋盘。复杂特殊方块的全局求解成本需要通过原型测试评估。

### 云端为货币与奖励权威，本地作为缓存和待确认操作队列

本地存档保存最近一次云端确认基线，并单独保存离线待处理操作。每个涉及货币、商品所有权、关卡奖励、收养或每日限额的操作带唯一操作标识；云端校验资格并去重。通过的操作更新权威状态，拒绝的操作不入账，客户端从最新确认基线重算后续预测状态。客户端整份快照不得覆盖云端状态。

**替代方案：** 重新联网时以本地快照覆盖云端，操作简单但可能重复发奖、丢失另一设备进度或制造负余额；仅允许在线操作则更易保持一致，但不符合离线游玩需求。

### 猫咪属性与收容期限使用服务端时间

每只猫记录最后一次成功喂食的服务端时间。普通需求衰减仍按访问时惰性补算；自动收容则由服务端到期扫描处理，并在收容所列表查询时补偿扫描延迟。扫描以“当前仍归原主人所有且距最后喂食已满 30 天”为条件，使用幂等状态转换，因此重复扫描不会重复入所。客户端时间只用于显示倒计时，不能延后或提前改变入所资格。

**替代方案：** 仅在玩家或猫咪再次访问时检查会导致离线玩家的猫长期不进入收容列表；固定间隔扫描增加调度依赖，但符合自动入所行为。

### 收容价格与所有权转移由云端配置和交易保证

每个猫咪品种或获取来源必须能解析出赎回基准价；免费获得的猫也必须有明确配置基准价。收容低价作为独立配置项。入所未满一年时，服务端仅接受原主人以基准价两倍赎回；满一年后原主人资格终止并开放其他玩家按收容低价购买。每笔交易在服务端校验时间边界、买家余额、咪窝空位和猫咪当前状态，并原子化扣款及所有权转移。

**替代方案：** 在客户端计算价格或直接改写猫咪所有者字段实现简单，但不能可靠执行一年资格边界、并发抢购和原子扣款。

### 采用渐进式上线顺序

1. 先完成连连看核心、关卡生成与结算，形成不依赖云端的可玩闭环。
2. 接入玩家本地存档、云端校验和货币/商店，再实现猫咪照护与医院。
3. 接入品种收集、奖励渠道及活动能力。
4. 最后加入收留站和词汇练习；它们依赖玩家、货币和云端数据能力。

各阶段共享数据契约，但未启用的后续入口不展示为可用功能。

### 云函数运行时必须以当前平台支持为准

设计文档中的 Node.js 16 只视为历史建议。建立云函数前先核对微信云开发当前可选运行时、SDK 和部署限制，再锁定依赖与版本；若目标运行时无法兼容客户端需求，应在实现前更新设计与规格。

**替代方案：** 直接采用文档中的旧版本会减少初始决策工作，但可能引入无法部署或停止维护的依赖风险。

### 后台身份、服务边界与数据库选型

首版按微信云开发作为部署边界评估，使用云函数承接客户端请求，并从可信调用上下文取得微信身份；客户端不能通过请求体指定或覆盖 `playerId`。玩家资料只保存游戏需要的信息，不把 OpenID 返回客户端；昵称和头像为可选资料。云端是货币、奖励、库存、猫咪所有权及布局的权威状态，本地只保存已确认快照和待确认操作。

由于产品要求 SQL 表结构，以下采用 **MySQL 8 兼容的关系型参考 DDL**表达实体、关系、约束和索引；这不是对微信云开发提供原生 MySQL 的断言。任务 1.2 必须先验证当前云函数能否以受支持的方式连接关系型数据库，并确认事务、唯一约束、并发更新和网络配置。如果该路径不受支持，则在实现前明确选用受支持的关系型服务，或把同一逻辑模型映射到云开发文档数据库集合，并保留等价的事务及唯一性校验；不得把下述 DDL 原样当作可部署的云开发脚本。

**替代方案：** 将玩家进度、钱包流水和堆叠/实例物品拆成更多单用途表会便于独立扩展，但首版的一对一玩家进度和统一背包读取会增加不必要的关联；将所有状态塞进单一 JSON 存档则难以可靠地约束余额、并发交易和物品实例唯一性。

### 关系型参考 Schema

字段命名使用 `snake_case`；所有时间由服务端以 UTC 写入。货币和库存使用整数计量，金额不使用浮点数。玩家进度直接并入 `players`；每关一条的明细仍单独保存在 `player_level_progress`。`player_currency_ledger` 记录所有玩家货币流水，按 `currency` 区分喵币和钻石。`player_inventory` 同时承载堆叠物资和唯一耐用品实例：堆叠商品每玩家/SKU 至多一行，耐用品每个实例一行；布置坐标单独保存，布局版本由咪窝记录。

```sql
CREATE TABLE players (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  openid VARCHAR(64) NOT NULL,
  nickname VARCHAR(64) NULL,
  avatar_url VARCHAR(512) NULL,
  highest_unlocked_level INT UNSIGNED NOT NULL DEFAULT 1,
  settings_json JSON NULL,
  schema_version INT UNSIGNED NOT NULL DEFAULT 1,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_players_openid (openid)
);

CREATE TABLE homes (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  player_id BIGINT UNSIGNED NOT NULL,
  home_no SMALLINT UNSIGNED NOT NULL,
  name VARCHAR(64) NOT NULL,
  layout_version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_home_player_no (player_id, home_no),
  CONSTRAINT fk_homes_player FOREIGN KEY (player_id) REFERENCES players(id)
);

CREATE TABLE cat_breeds (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  breed_key VARCHAR(64) NOT NULL,
  name VARCHAR(64) NOT NULL,
  rarity VARCHAR(24) NOT NULL,
  base_price BIGINT UNSIGNED NOT NULL,
  shelter_price BIGINT UNSIGNED NOT NULL,
  config_json JSON NOT NULL,
  config_version INT UNSIGNED NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_breed_key (breed_key)
);

CREATE TABLE cats (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  owner_player_id BIGINT UNSIGNED NULL,
  original_owner_player_id BIGINT UNSIGNED NULL,
  home_id BIGINT UNSIGNED NULL,
  breed_id BIGINT UNSIGNED NOT NULL,
  name VARCHAR(64) NOT NULL,
  status VARCHAR(16) NOT NULL DEFAULT 'owned',
  level INT UNSIGNED NOT NULL DEFAULT 1,
  experience BIGINT UNSIGNED NOT NULL DEFAULT 0,
  hunger TINYINT UNSIGNED NOT NULL DEFAULT 100,
  cleanliness TINYINT UNSIGNED NOT NULL DEFAULT 100,
  mood TINYINT UNSIGNED NOT NULL DEFAULT 100,
  trust TINYINT UNSIGNED NOT NULL DEFAULT 0,
  energy TINYINT UNSIGNED NOT NULL DEFAULT 100,
  health TINYINT UNSIGNED NOT NULL DEFAULT 100,
  preferences_json JSON NULL,
  activity_state_json JSON NULL,
  state_updated_at DATETIME(3) NOT NULL,
  last_fed_at DATETIME(3) NULL,
  shelter_at DATETIME(3) NULL,
  redeem_until DATETIME(3) NULL,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  KEY ix_cats_owner_home (owner_player_id, home_id, status),
  KEY ix_cats_shelter_due (status, last_fed_at),
  CONSTRAINT fk_cats_owner FOREIGN KEY (owner_player_id) REFERENCES players(id),
  CONSTRAINT fk_cats_original_owner FOREIGN KEY (original_owner_player_id) REFERENCES players(id),
  CONSTRAINT fk_cats_home FOREIGN KEY (home_id) REFERENCES homes(id),
  CONSTRAINT fk_cats_breed FOREIGN KEY (breed_id) REFERENCES cat_breeds(id),
  CONSTRAINT ck_cat_stats CHECK (
    hunger <= 100 AND cleanliness <= 100 AND mood <= 100
    AND trust <= 100 AND energy <= 100 AND health <= 100
  )
);

CREATE TABLE player_breed_collection (
  player_id BIGINT UNSIGNED NOT NULL,
  breed_id BIGINT UNSIGNED NOT NULL,
  unlocked_at DATETIME(3) NOT NULL,
  source VARCHAR(32) NOT NULL,
  PRIMARY KEY (player_id, breed_id),
  CONSTRAINT fk_breed_collection_player FOREIGN KEY (player_id) REFERENCES players(id),
  CONSTRAINT fk_breed_collection_breed FOREIGN KEY (breed_id) REFERENCES cat_breeds(id)
);

CREATE TABLE wallets (
  player_id BIGINT UNSIGNED NOT NULL,
  currency VARCHAR(16) NOT NULL,
  balance BIGINT NOT NULL DEFAULT 0,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (player_id, currency),
  CONSTRAINT fk_wallets_player FOREIGN KEY (player_id) REFERENCES players(id),
  CONSTRAINT ck_wallet_balance CHECK (balance >= 0)
);

CREATE TABLE player_currency_ledger (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  player_id BIGINT UNSIGNED NOT NULL,
  currency VARCHAR(16) NOT NULL,
  delta BIGINT NOT NULL,
  balance_after BIGINT NOT NULL,
  reason VARCHAR(32) NOT NULL,
  reference_id VARCHAR(64) NOT NULL,
  operation_id CHAR(36) NOT NULL,
  created_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_currency_ledger_operation (player_id, operation_id, currency),
  KEY ix_currency_ledger_player_time (player_id, created_at),
  CONSTRAINT fk_currency_ledger_player FOREIGN KEY (player_id) REFERENCES players(id)
);

CREATE TABLE shop_products (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  sku VARCHAR(64) NOT NULL,
  name VARCHAR(128) NOT NULL,
  category VARCHAR(24) NOT NULL,
  item_kind VARCHAR(24) NOT NULL,
  currency VARCHAR(16) NOT NULL,
  price BIGINT UNSIGNED NOT NULL,
  item_config_json JSON NOT NULL,
  catalog_version INT UNSIGNED NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_shop_sku (sku)
);

CREATE TABLE player_inventory (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  player_id BIGINT UNSIGNED NOT NULL,
  sku VARCHAR(64) NOT NULL,
  entry_type VARCHAR(16) NOT NULL,
  quantity BIGINT UNSIGNED NOT NULL DEFAULT 1,
  stack_sku VARCHAR(64) GENERATED ALWAYS AS (
    CASE WHEN entry_type = 'stack' THEN sku ELSE NULL END
  ) STORED,
  location VARCHAR(16) NOT NULL DEFAULT 'bag',
  home_id BIGINT UNSIGNED NULL,
  item_state JSON NULL,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  acquired_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_inventory_stack (player_id, stack_sku),
  KEY ix_inventory_player_location (player_id, location, home_id),
  CONSTRAINT fk_inventory_player FOREIGN KEY (player_id) REFERENCES players(id),
  CONSTRAINT fk_inventory_sku FOREIGN KEY (sku) REFERENCES shop_products(sku),
  CONSTRAINT fk_inventory_home FOREIGN KEY (home_id) REFERENCES homes(id),
  CONSTRAINT ck_inventory_entry CHECK (
    (entry_type = 'stack' AND quantity > 0)
    OR (entry_type = 'instance' AND quantity = 1)
  ),
  CONSTRAINT ck_inventory_location CHECK (
    (location = 'bag' AND home_id IS NULL)
    OR (location = 'home' AND home_id IS NOT NULL)
  )
);

CREATE TABLE home_layout_items (
  inventory_item_id BIGINT UNSIGNED NOT NULL,
  x SMALLINT NOT NULL,
  y SMALLINT NOT NULL,
  rotation SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (inventory_item_id),
  CONSTRAINT fk_layout_inventory_item
    FOREIGN KEY (inventory_item_id) REFERENCES player_inventory(id)
);

CREATE TABLE litter_box_state (
  home_id BIGINT UNSIGNED NOT NULL,
  sand_amount INT UNSIGNED NOT NULL DEFAULT 0,
  sand_capacity INT UNSIGNED NOT NULL,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  last_simulated_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (home_id),
  CONSTRAINT fk_litter_home FOREIGN KEY (home_id) REFERENCES homes(id),
  CONSTRAINT ck_litter_capacity CHECK (sand_amount <= sand_capacity)
);

CREATE TABLE litter_clumps (
  id CHAR(36) NOT NULL,
  home_id BIGINT UNSIGNED NOT NULL,
  cat_id BIGINT UNSIGNED NOT NULL,
  status VARCHAR(16) NOT NULL DEFAULT 'active',
  created_at DATETIME(3) NOT NULL,
  cleared_at DATETIME(3) NULL,
  clear_operation_id CHAR(36) NULL,
  PRIMARY KEY (id),
  KEY ix_clumps_home_status (home_id, status, created_at),
  UNIQUE KEY uk_clumps_clear_operation (clear_operation_id),
  CONSTRAINT fk_clumps_home FOREIGN KEY (home_id) REFERENCES homes(id),
  CONSTRAINT fk_clumps_cat FOREIGN KEY (cat_id) REFERENCES cats(id)
);

CREATE TABLE player_level_progress (
  player_id BIGINT UNSIGNED NOT NULL,
  level_id INT UNSIGNED NOT NULL,
  best_score BIGINT UNSIGNED NOT NULL DEFAULT 0,
  stars TINYINT UNSIGNED NOT NULL DEFAULT 0,
  first_cleared_at DATETIME(3) NULL,
  first_reward_claimed_at DATETIME(3) NULL,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  PRIMARY KEY (player_id, level_id),
  CONSTRAINT fk_level_progress_player FOREIGN KEY (player_id) REFERENCES players(id)
);

CREATE TABLE level_attempts (
  id CHAR(36) NOT NULL,
  player_id BIGINT UNSIGNED NOT NULL,
  level_id INT UNSIGNED NOT NULL,
  result VARCHAR(16) NOT NULL,
  score BIGINT UNSIGNED NOT NULL DEFAULT 0,
  duration_ms BIGINT UNSIGNED NOT NULL DEFAULT 0,
  server_verified BOOLEAN NOT NULL DEFAULT FALSE,
  operation_id CHAR(36) NOT NULL,
  created_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_attempt_operation (player_id, operation_id),
  KEY ix_attempt_player_level_time (player_id, level_id, created_at),
  CONSTRAINT fk_attempt_player FOREIGN KEY (player_id) REFERENCES players(id)
);

CREATE TABLE care_sessions (
  id CHAR(36) NOT NULL,
  player_id BIGINT UNSIGNED NOT NULL,
  cat_id BIGINT UNSIGNED NOT NULL,
  home_id BIGINT UNSIGNED NOT NULL,
  care_type VARCHAR(24) NOT NULL,
  status VARCHAR(16) NOT NULL,
  progress_json JSON NOT NULL,
  version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  started_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  KEY ix_care_player_status (player_id, status),
  CONSTRAINT fk_care_player FOREIGN KEY (player_id) REFERENCES players(id),
  CONSTRAINT fk_care_cat FOREIGN KEY (cat_id) REFERENCES cats(id),
  CONSTRAINT fk_care_home FOREIGN KEY (home_id) REFERENCES homes(id)
);

CREATE TABLE reward_claims (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  player_id BIGINT UNSIGNED NOT NULL,
  source VARCHAR(32) NOT NULL,
  period_key VARCHAR(64) NOT NULL,
  operation_id CHAR(36) NOT NULL,
  reward_json JSON NOT NULL,
  created_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_reward_period (player_id, source, period_key),
  UNIQUE KEY uk_reward_operation (player_id, operation_id),
  CONSTRAINT fk_reward_player FOREIGN KEY (player_id) REFERENCES players(id)
);

CREATE TABLE operation_receipts (
  player_id BIGINT UNSIGNED NOT NULL,
  operation_id CHAR(36) NOT NULL,
  operation_type VARCHAR(40) NOT NULL,
  request_hash BINARY(32) NOT NULL,
  status VARCHAR(16) NOT NULL,
  response_json JSON NULL,
  created_at DATETIME(3) NOT NULL,
  completed_at DATETIME(3) NULL,
  PRIMARY KEY (player_id, operation_id),
  CONSTRAINT fk_receipt_player FOREIGN KEY (player_id) REFERENCES players(id)
);
```

设计约束：

- `players` 的 `openid` 唯一且只在服务端使用；进度字段不再另设一对一的 `player_progress` 表。`player_level_progress` 是每玩家每关一行的明细，仍单独保存。
- `wallets` 是当前余额快照，`player_currency_ledger` 是全部货币的追加流水。余额更新、流水写入、物品交付及操作回执必须在同一事务内完成。
- `player_inventory.entry_type='stack'` 时按玩家/SKU 聚合数量；`entry_type='instance'` 时每行主键代表一个耐用品实例且数量固定为 1。类型必须与商品目录的持有模型一致。
- `home_layout_items` 只保存唯一库存实例的坐标和方向；`homes.layout_version` 是并发保存布局的版本。服务端校验物品归属、可摆属性、房间范围、重叠和必要通路。
- `operation_receipts` 将幂等键限定为玩家范围。相同操作 ID 和相同请求返回原结果；请求哈希不匹配必须拒绝，不可复用回执。
- JSON 只承载可演进的配置/会话属性；余额、所有权、数量、状态、版本和关系使用显式列、索引及约束。
- 收容时保留 `original_owner_player_id`，将当前所有者清空并转成 sheltered 状态；购买或赎回时在事务内设置新所有者、目标咪窝和交易记录。公开查询不得返回 OpenID。

### 云函数 API 契约

客户端通过微信云函数调用下表中的业务操作。身份从可信云函数上下文取得，不在 Request 中传 `playerId`。所有写请求使用 UUID `operationId`；带版本字段的更新必须进行乐观并发校验。ID 在 JSON 中按字符串传输，避免 JavaScript 数值精度损失。

统一成功包络：

```json
{
  "ok": true,
  "requestId": "req-uuid",
  "serverTime": "2026-11-12T09:30:00.000Z",
  "data": {},
  "versions": { "player": 12 }
}
```

统一失败包络：

```json
{
  "ok": false,
  "requestId": "req-uuid",
  "serverTime": "2026-11-12T09:30:00.000Z",
  "error": {
    "code": "VERSION_CONFLICT",
    "message": "咪窝数据已在其他设备更新。",
    "retryable": false,
    "details": { "currentVersion": 8 }
  }
}
```

错误码至少包括 `AUTH_REQUIRED`、`INVALID_ARGUMENT`、`NOT_FOUND`、`INSUFFICIENT_FUNDS`、`INSUFFICIENT_ITEMS`、`VERSION_CONFLICT`、`OPERATION_REJECTED`、`OPERATION_ID_REUSED`、`DEPENDENCY_REJECTED`、`RATE_LIMITED` 和 `TEMPORARY_UNAVAILABLE`。错误不得伪装成成功响应。

| 云函数 | 用途 | 幂等/并发约束 |
|---|---|---|
| `bootstrap` | 首次创建账号、初始钱包和咪窝；已有玩家时返回当前快照 | 按服务端身份唯一创建 |
| `getPlayerSnapshot` | 返回玩家摘要、钱包、关卡、猫、咪窝、库存及版本 | 只读 |
| `getLevelMap` / `startLevel` | 获取关卡地图；开始关卡时由服务端创建 attempt 并返回棋盘种子/配置版本 | 校验解锁状态；attempt 唯一 |
| `getShopCatalog` | 按目录版本读取可购买商品 | 只读；价格以服务端目录为准 |
| `purchaseItem` | 扣币并交付堆叠物或唯一实例 | `operationId` 去重，单事务 |
| `saveHomeLayout` | 保存咪窝布置和版本 | 校验 `expectedLayoutVersion` |
| `feedCat` | 扣食物并结算喂食 | 操作去重并校验 `expectedCatVersion` |
| `startCare` / `commitCareProgress` | 创建或推进洗浴、梳理、猫砂维护会话 | 操作去重、会话版本和有效阶段校验 |
| `clearLitterClump` / `refillLitter` | 清理唯一结团或按实际填入量补砂 | 盆版本校验，物资与状态原子更新 |
| `syncOperations` | 有序提交离线操作及依赖 | 逐条回执；拒绝依赖链不保留预测结果 |
| `submitLevelResult` | 校验对局并结算关卡奖励 | 以 attempt/operation ID 去重，奖励按服务端配置计算 |
| `claimReward` | 领取登录、分享、视频或成就奖励 | 服务端判定有效性、周期和限额 |
| `listShelterCats` | 返回当前玩家可赎回或可购买的收容猫 | 不泄露原主人身份 |
| `redeemShelterCat` / `adoptShelterCat` | 赎回或购买收容猫并转移所有权 | 检查资格/期限/余额/咪窝容量，单事务 |

关卡地图和开始接口：

`getLevelMap` Request 为 `{ "cursor": null }`；Response `data` 返回已解锁关卡的 ID、状态、星级、最佳成绩、奖励预览和下一页游标。`startLevel` Request 为 `{ "operationId": "uuid", "levelId": 1 }`；成功 Response `data` 返回 `{ "attemptId": "uuid", "levelId": 1, "seed": 123456, "configVersion": 4, "startedAt": "2026-11-12T09:30:00.000Z" }`。服务端拒绝未解锁关卡并保存 attempt，提交结果时只能引用有效 attempt。

照护会话接口：

`startCare` Request 为 `{ "operationId": "uuid", "catId": "3001", "homeId": "101", "careType": "bath", "expectedCatVersion": 5 }`；Response 返回服务端 `sessionId`、会话版本、有效阶段和 `startedAt`。`commitCareProgress` Request 为 `{ "operationId": "uuid", "sessionId": "uuid", "expectedSessionVersion": 2, "completedSegments": ["left_side"], "action": "continue" }`；Response 返回确认的阶段、猫状态及新版本。服务端只结算首次有效的区段；重试、取消和中断恢复不得重复扣物或发放效果。

猫砂维护接口：

`clearLitterClump` Request 为 `{ "operationId": "uuid", "homeId": "101", "clumpId": "uuid", "expectedLitterVersion": 4 }`；Response 返回该结团已清理、猫砂盆新版本和服务端确认时间。`refillLitter` Request 为 `{ "operationId": "uuid", "homeId": "101", "amount": 20, "expectedLitterVersion": 5 }`；Response 返回实际补入量、剩余猫砂库存、盆内砂量及新版本。盆容量不足、库存不足或版本过期时整项拒绝，不做部分扣除。

#### 玩家初始化与快照

`bootstrap` Request：

```json
{ "clientBuild": "0.1.0", "clientSchemaVersion": 1 }
```

`bootstrap` Response `data`：

```json
{
  "player": { "displayName": "玩家", "highestUnlockedLevel": 1, "version": 1 },
  "wallets": [
    { "currency": "meow_coin", "balance": 1000, "version": 1 },
    { "currency": "diamond", "balance": 0, "version": 1 }
  ],
  "homes": [{ "homeId": "101", "name": "咪窝 1", "layoutVersion": 1 }],
  "cats": [],
  "inventory": [],
  "catalogVersion": 1,
  "snapshotVersion": 1
}
```

示例余额和物品数量仅用于说明响应形状；新玩家初始资产由服务端配置决定，不由该示例固定。

重复 bootstrap 不重复发放初始货币或物资。`getPlayerSnapshot` 使用空 Request 并返回同一玩家范围的完整同步快照，服务端根据调用身份确定玩家。

#### 商城目录与购买

`getShopCatalog` Request：

```json
{ "category": "furniture", "ifCatalogVersion": 3 }
```

Response `data`：

```json
{
  "catalogVersion": 3,
  "products": [{
    "sku": "cat_bed_basic",
    "name": "软软猫床",
    "category": "furniture",
    "entryType": "instance",
    "price": { "currency": "meow_coin", "amount": 200 },
    "placement": {
      "allowed": true, "homeTypes": ["home"],
      "width": 2, "height": 2, "rotations": [0, 90],
      "use": "rest_anchor"
    }
  }]
}
```

`purchaseItem` Request：

```json
{
  "operationId": "8ffb335c-7c5e-45bd-a26c-735bde291a51",
  "sku": "cat_bed_basic",
  "quantity": 1,
  "expectedCatalogVersion": 3
}
```

Response `data`：

```json
{
  "operationId": "8ffb335c-7c5e-45bd-a26c-735bde291a51",
  "status": "confirmed",
  "charged": { "currency": "meow_coin", "amount": 200 },
  "wallet": { "currency": "meow_coin", "balance": 800, "version": 2 },
  "granted": {
    "entryType": "instance",
    "items": [{ "inventoryItemId": "9001", "sku": "cat_bed_basic", "location": "bag" }]
  }
}
```

服务端忽略客户端价格；在一个事务中校验商品及余额、扣款、写入流水、交付物品和记录回执。

#### 保存咪窝布置

`saveHomeLayout` Request：

```json
{
  "operationId": "724efafb-3212-4c45-bcef-9a98b8e3fed6",
  "homeId": "101",
  "expectedLayoutVersion": 7,
  "placements": [
    { "inventoryItemId": "9001", "x": 2, "y": 1, "rotation": 0 }
  ]
}
```

Response `data`：

```json
{
  "operationId": "724efafb-3212-4c45-bcef-9a98b8e3fed6",
  "status": "confirmed",
  "homeId": "101",
  "layoutVersion": 8,
  "placements": [
    { "inventoryItemId": "9001", "x": 2, "y": 1, "rotation": 0 }
  ]
}
```

版本冲突时不写入任何布局，返回 `VERSION_CONFLICT`、当前版本和可恢复快照。服务端同时校验每个实例归属、摆放类型、房间适用性、占地、重叠及必要设施通路。

#### 喂食

`feedCat` Request：

```json
{
  "operationId": "8378b8fb-bb59-4570-92c7-8367687c5fdd",
  "catId": "3001",
  "foodSku": "food_main",
  "quantity": 1,
  "expectedCatVersion": 5
}
```

Response `data`：

```json
{
  "operationId": "8378b8fb-bb59-4570-92c7-8367687c5fdd",
  "status": "confirmed",
  "inventory": { "sku": "food_main", "quantity": 4 },
  "cat": {
    "catId": "3001", "hunger": 82,
    "lastFedAt": "2026-11-12T09:31:00.000Z", "version": 6
  }
}
```

同一操作重试返回原结果，不重复扣食物，不刷新 `lastFedAt`。

#### 离线操作同步

`syncOperations` Request：

```json
{
  "clientSnapshotVersion": 12,
  "operations": [
    {
      "operationId": "8ffb335c-7c5e-45bd-a26c-735bde291a51",
      "type": "purchase",
      "dependsOn": [],
      "payload": { "sku": "cat_bed_basic", "quantity": 1 }
    },
    {
      "operationId": "724efafb-3212-4c45-bcef-9a98b8e3fed6",
      "type": "save_home_layout",
      "dependsOn": ["8ffb335c-7c5e-45bd-a26c-735bde291a51"],
      "payload": {
        "homeId": "101",
        "expectedLayoutVersion": 7,
        "placements": [{ "inventoryItemId": "9001", "x": 2, "y": 1, "rotation": 0 }]
      }
    }
  ]
}
```

Response `data`：

```json
{
  "results": [
    {
      "operationId": "8ffb335c-7c5e-45bd-a26c-735bde291a51",
      "status": "confirmed",
      "result": { "chargedAmount": 200, "inventoryItemIds": ["9001"] }
    },
    {
      "operationId": "724efafb-3212-4c45-bcef-9a98b8e3fed6",
      "status": "confirmed",
      "result": { "layoutVersion": 8 }
    }
  ],
  "snapshotVersion": 13
}
```

每项也可返回 `rejected` 和错误码；前置操作拒绝时，依赖项返回 `DEPENDENCY_REJECTED`。队列按顺序处理，不能把客户端快照整体写回服务端。

#### 关卡结果与奖励领取

`submitLevelResult` Request：

```json
{
  "operationId": "c801a589-7910-494d-81ed-74db67251727",
  "attemptId": "ab021f1c-d0d1-4578-8afe-c9e6d11f74a9",
  "levelId": 1,
  "result": "won",
  "score": 1200,
  "durationMs": 85000,
  "replay": { "seed": 123456, "actions": [] }
}
```

Response `data`：

```json
{
  "operationId": "c801a589-7910-494d-81ed-74db67251727",
  "status": "confirmed",
  "verified": true,
  "score": 1200,
  "reward": { "currency": "meow_coin", "amount": 100 },
  "wallet": { "currency": "meow_coin", "balance": 900, "version": 3 },
  "progress": { "levelId": 1, "stars": 3, "firstClear": true }
}
```

奖励金额由服务端配置决定，不接受客户端提交奖励数值。离线关卡奖励只在服务端验证流程/资格后入账；若验证方案不可行，任务实现前必须定义离线奖励的限制策略。

`claimReward` Request 与 Response：

```json
{
  "operationId": "ce51d1ed-bfb3-41cb-922a-4a2d28516d63",
  "source": "daily_login"
}
```

```json
{
  "operationId": "ce51d1ed-bfb3-41cb-922a-4a2d28516d63",
  "status": "confirmed",
  "reward": { "currency": "meow_coin", "amount": 50 },
  "wallet": { "currency": "meow_coin", "balance": 950, "version": 4 },
  "claimedAt": "2026-11-12T09:35:00.000Z"
}
```

奖励周期和有效凭据由服务端判定；客户端不得指定日期以绕过次数上限。分享/视频奖励只有在平台能力支持并验证有效凭据后才启用。
平台奖励需要回传的校验凭据由对应平台能力验证任务确定；每日登录不需要客户端凭据。

#### 收容猫查询与所有权交易

`listShelterCats` Request：

```json
{ "cursor": null, "limit": 20 }
```

Response `data`：

```json
{
  "items": [{
    "catId": "3001", "name": "团团", "breed": "橘猫",
    "level": 3, "shelterAt": "2026-01-01T00:00:00.000Z",
    "action": "redeem", "price": { "currency": "meow_coin", "amount": 2000 }
  }],
  "nextCursor": null
}
```

`redeemShelterCat` / `adoptShelterCat` 使用如下请求结构，按操作选择云函数：

```json
{
  "operationId": "5b3509f9-1aeb-4be6-b302-7432d5a040b9",
  "catId": "3001",
  "targetHomeId": "101",
  "expectedCatVersion": 9
}
```

Response `data`：

```json
{
  "operationId": "5b3509f9-1aeb-4be6-b302-7432d5a040b9",
  "status": "confirmed",
  "cat": { "catId": "3001", "owner": "self", "homeId": "101", "version": 10 },
  "charged": { "currency": "meow_coin", "amount": 2000 },
  "wallet": { "currency": "meow_coin", "balance": 3000, "version": 5 }
}
```

服务端决定玩家是否为原主人、交易是否处于有效期限、猫是否仍可交易、咪窝是否有容量；扣款、流水、猫咪主人/咪窝变更和回执必须原子提交。公共列表不能包含 OpenID 或原主人身份。

## 风险 / 权衡

- [风险] 离线奖励和消费可能被篡改 → [缓解] 云端校验奖励资格、限额、余额和唯一操作标识；客户端待确认结果不视为最终入账。
- [风险] 微信云开发的数据库类型、事务能力或外网连接方式不符合 MySQL 参考 DDL → [缓解] 在 1.2 明确验证；不支持时在实现前选择受支持的关系型数据库或等价文档模型，并通过契约测试保证原子购买、奖励及所有权转移。
- [风险] 将物品堆叠与独立实例放在一张表时，客户端或后台错误地混用数量语义 → [缓解] 商品目录声明唯一持有类型，数据库约束和服务端校验共同拒绝不匹配数据。
- [风险] 特殊方块组合使棋盘生成或死局检测成本过高 → [缓解] 按关卡启用机制并用固定种子测试生成耗时；先保证至少存在合法操作，复杂机制分批开放。
- [风险] 猫咪离线衰减与云端时间发生冲突 → [缓解] 将玩家本地状态视为预测值，使用云端确认时间重算；为异常或超长间隔定义可配置补算边界。
- [风险] 自动收容扫描延迟或重复执行 → [缓解] 扫描与列表查询使用同一服务端资格条件，并以幂等状态转换保证最终入所且不重复处理。
- [风险] 免费猫没有原始售价，或玩家并发购买导致重复所有权 → [缓解] 发布前为所有猫来源配置赎回基准价与收容低价；由云端原子交易保证一只猫只转移一次。
- [风险] 完整路线图范围过大且美术资源未提供 → [缓解] 按阶段开发，先以占位资源验证触控和棋盘布局，发布前替换为正式素材。
- [风险] 微信云运行时或广告/分享能力与计划不符 → [缓解] 在对应阶段开始前做平台能力验证；将平台 API 隔离在适配层并让非平台逻辑可本地运行。

## 迁移计划

这是新建小游戏，不迁移既有生产数据。先完成平台/数据库能力验证，再以版本化 SQL migration 建立表、约束和索引；若落地为文档数据库，则同步维护逻辑集合映射、唯一性与事务策略。按阶段使用本地和云端测试数据验证。发布前验证客户端与云函数契约兼容；回滚在线写入时保留最近已确认数据，不将待同步队列整份覆盖云端。配置和 schema 更新均记录版本并提供兼容/回退步骤。

## 待决问题

- 排行榜的范围、排名周期和计分规则尚未定义；在另行确认前不纳入实现任务。
- 钻石充值、广告/分享接入与平台审核要求需在商业化阶段按当时的微信平台政策确认。
- 正式美术、音效素材和资源授权来源需由产品/美术流程提供；不影响当前领域规格与工程分层。

## 猫咪生活空间与互动完整方案

### 1. 方案状态与体验目标

本节是已经同步至本变更提案、规格与任务的目标设计，不代表已经实现。行为契约见 [猫咪照护规格](specs/cat-care/spec.md)、[生活空间规格](specs/cat-home/spec.md)、[经济规格](specs/game-economy/spec.md) 和 [存档规格](specs/player-save/spec.md)。仓库仍无游戏源码或测试，本轮只完善规划。

**玩家已确认的方向：**
- 轻拟真、偏治愈：猫咪会拒绝或躲开，玩家可以等待、换方式和安抚。
- 咪窝是猫咪生活的房间；喂食、玩耍等日常互动发生在房间内。
- 洗澡、吹干和猫砂盆位于另一个场景，即本方案所称的护理区。
- 猫砂是日常必需消耗品，玩家从商店购买，并定期铲屎。
- 玩家能添置家具和玩具，购置并摆放的物品显示在房间画面中。
- 喂食成功立即扣食物并增加饱食度，后续动画不再结算；不采用余粮延迟增加属性的方案。
- 玩具按商品类型区分耐用型和消耗型。
- 可协助擦干，吹干操作失误或协助完成不触发感冒；其他疾病规则保留。

**本轮纳入规划的设计默认值：**
- 每个咪窝关联一个护理区，同窝猫咪共享该区；护理区不是新的猫咪归属。
- 抚摸、梳毛和局部擦爪留在房间；完整洗澡和吹干进入护理区。
- 初始咪窝配备基础食盆、水碗、休息位置、猫砂盆、猫砂铲、梳理工具和少量猫砂，新增猫咪配对应命名食盆；后续补充猫砂需要购买。
- 家具和耐用玩具购买后可重复使用；食物在成功喂食时扣除，消耗型玩具在首次有效使用时扣除，猫砂按实际填入量扣除。
- 增加精力、即时情绪、信任与个体偏好，精力按玩耍消耗和休息恢复；不增加口渴付费循环。
- 猫砂盆先采用游戏化共享容量，不模拟现实养猫所需盆数，也不将本方案作为现实护理指南。

以上默认值是为实现完整方案记录的设计选择，并非额外声称得到逐项产品确认；它们已作为本变更的目标行为纳入规格和任务。价格、周期、容量及美术参数可后续调参，分阶段上线不改变已定义的验收边界。三个已知规则冲突均已按本轮用户选择解决。

**体验目标：** 玩家观察猫咪的信号，尝试互动，得到回应，再调整行为；猫咪会使用房间、记住偏好并主动邀请玩家。结束或暂停不等于失败。

**非目标：** 不设计强制每日打卡、精确喂养克数、排泄细节展示、复杂家具物理模拟，或通过猫咪痛苦强迫消费的机制。

### 2. 双场景空间与导航

```text
+---------------------------+       +---------------------------+
| ROOM                      |       | CARE AREA                 |
| Food / Water / Rest       |       | Litter box / Supplies     |
| Pet / Play / Brush        | <---> | Bath / Towel / Dry        |
| Furniture / Placed toys   |       | Scoop / Refill / Replace  |
+---------------------------+       +---------------------------+
             |
             v
+---------------------------+
| SHOP / INVENTORY          |
| Food / Litter / Furniture |
| Toys / Care supplies      |
+---------------------------+
```

**咪窝房间布局：**
- 上方为当前咪窝名称、猫咪头像和可展开的需求摘要，不在猫身上堆满进度条。
- 中部为猫咪活动画面，包含食水区、休息区、玩耍区、家具和通往护理区的门。
- 下方保留商店、背包、布置和护理区入口；互动时收起不相关按钮。
- 选中猫咪时以名字和柔和轮廓明确对象。近景是房间内镜头变化，不加载新的日常互动场景。
- 食物不足、猫砂不足等提示附着在对应物品或护理区入口，不连续弹出强制窗口。

**护理区布局：**
- 猫砂区与洗浴区在同一场景，但分成不同位置，不在浴盆旁堆放排泄物。
- 猫砂区显示猫砂盆、铲子和库存入口；洗浴区显示浴盆、毛巾与吹干台。
- 同时只展开一个操作面板；猫砂盆被使用时不得清理，洗浴过程中不得切去铲屎。
- 进入护理区不要求先选猫；铲屎维护的是盆，洗澡才需要选择当前咪窝的猫。
- 返回房间保留原咪窝、选中猫和镜头位置，不把一次护理当作转移咪窝。

**猫咪详情与图鉴的入口边界：**
- 玩家拥有的单只猫是独立实例；从咪窝的猫咪头像条选中猫后，可通过“详情”入口或猫咪上下文菜单打开该猫的详情页。
- 猫咪详情页展示该实例的当前状态、等级、已发现偏好、当前活动和所属咪窝；返回时恢复原咪窝、选中猫和视图状态。
- 图鉴以猫咪品种为对象，展示品种资料、收集状态和解锁条件，不作为查看或管理某只已拥有猫咪当前状态的入口。
- 从图鉴选择已拥有品种时仍查看该品种资料；若要查看具体猫咪实例，使用明确的“前往咪窝”入口，再在咪窝选择目标猫。相同品种拥有多只猫时，不由图鉴猜选其中一只。

**替代方案与理由：** 将每个动作做成独立小游戏容易复用模板，但会使家成为菜单。采用两个持续空间，日常操作就地发生，护理动作集中到护理区，既保持生活感，也避免一个画面过于拥挤。

### 3. 猫咪需求、情绪、关系与自主行为

| 层次 | 内容 | 表现与作用 |
|---|---|---|
| 生活需求 | 饱食度、清洁度、健康及精力 | 决定是否接受喂食、是否愿意玩、是否需要休息 |
| 即时情绪 | 好奇、兴奋、放松、紧张、烦躁 | 决定当下姿势、靠近距离和接受程度 |
| 长期关系 | 与玩家的信任 | 改变主动接近、手喂与亲密动作的出现条件 |
| 个体偏好 | 食物、玩具、抚摸位置、声音接受度 | 使不同猫对同一操作产生可理解的差异 |
| 环境状态 | 猫砂盆可用性、家具占用、可达路径 | 决定能够使用什么，而不是直接增加好感 |

保留已有“心情”作为长期舒适度摘要，即时情绪单独计算，不直接把“心情 80”翻译为永远开心。偏好按个体记录，不将品种与性格完全绑定。

**信号设计：**
- 愿意接近：竖尾、向玩家走来、头顶手。
- 有兴趣：耳朵转向目标、盯住玩具、伏低身体。
- 放松：眯眼、身体松弛、安静休息；呼噜不是每次必播。
- 想暂停：甩尾、转身、爪子轻挡或走开。
- 对首次或难理解的信号给一句提示，如“它有点累了，休息一下吧”；不只靠颜色或声音传达。

**自主行为优先级：** 身体不适或必要护理状态优先，其次如厕、已确认喂食的进食表现和休息，再到自由玩耍与亲密邀请。已在如厕、吃饭、睡觉的猫不被普通点击强制打断。自主吃饭表现不消耗新食物或再次增加属性。

猫咪可以在房间巡视、跳上猫爬架、钻纸箱、抓抓板、玩地上的球。行为取决于可达目标和当前需求，随机性仅用于同样合适的动作选择。购买更多家具不直接保证更高信任。

**关系收益：** 接受边界、合适的陪伴和照护可产生小幅、限额的正向积累。反复滑动不无限刷收益，未回应邀请和一次操作失误不降低长期信任。

**替代方案与理由：** 单一心情值实现简单，但无法表现“吃饱却不想被摸”；分层状态能让回应有原因，同时避免为每种情绪添加新的显式任务。

### 4. 房间互动一：喂食与手喂

**入口：** 点击食盆，或回应猫咪在碗旁的叫声。多猫时每只猫对应有名字的食盆，避免猫咪争抢造成库存和归属混乱。

**交互流程：**
1. 展开食物列表，显示库存、份数和已发现的口味偏好。
2. 拖动包装到碗上或点“喂食一份”，先校验库存、饱食度、意愿和活动占用；未接受时不提交成功。
3. 猫走近闻食物，愿意时成功提交一份，立即扣食物、增加饱食度并记录唯一操作标识，再播放进食动画；拒绝时犹豫或离开且不扣物品。
4. 玩家可以观察，也可以返回房间普通视角；猫继续进食。
5. 吃完后猫舔嘴、洗脸或偶尔蹭手，画面展示已结算效果，不再因吃完结算一次。

**回应与调整：**
- 饥饿且接受口味：积极进食。
- 不熟悉口味：先观察，玩家可以退开等待。
- 已饱：转头离开，提示“它已经吃饱了，稍后再喂吧”，不扣食物也不更新最后喂食时间。
- 对近距离手喂尚不适应：后退，不消耗未吃的零食。
- 偏好影响兴趣，但基础主粮不会因随机挑食而长期无法满足必要需求。

**手喂：** 关系允许时，选零食并把手停在猫前方，等猫主动靠近；接受并首次吃下时作为成功喂食，原子扣一份并增加属性。成功前取消不扣，成功后退出不回滚已确认效果。

**状态与中断：** 成功喂食在单次操作中扣库存并增加属性，食盆中可见食物只是已结算动画进度，不是能再次喂食、返还或换口味获益的余粮。动画期间不接受第二次喂食；退出或重启保留操作确认及表现进度，恢复或跳过动画均不重复发放。饱食度不超过 100。云端首次确认该成功操作时记录服务端最后喂食时间，重试不以重试时间重置期限。

**替代方案：** 随实际进食逐段结算更拟真，但增加余粮、时间补算和重复收益边界。本轮用户明确选择立即结算，故不建立可延后消费的盆内食物库存。

**水碗：** 作为基础可见设施，猫可以自主喝水；首版建议不新增口渴值或另一个付费消耗循环。

### 5. 房间互动二：逗猫棒与地面玩具

**逗猫棒流程：**
1. 从房间玩具或工具栏选择逗猫棒，显示地面可活动区域。
2. 拖动玩具，猫先转头追踪；慢移时伏低，停顿时准备扑击。
3. 猫扑出后玩家轻移或停住，让它有机会抓住。
4. 抓住后保留抱住、轻踢的满足动画，再释放继续。
5. 猫疲劳后趴下，玩家可收起玩具或留在房间陪伴。

```text
[Watch] --> [Stalk] --> [Pounce] --> [Catch] --> [Release]
   ^                                               |
   +-----------------------------------------------+
Any stage --> [Rest] --> [Resume / End]
```

- 拖动过快时猫停下来找目标，给“慢一点”的提示，不显示失败。
- 一直不让抓到时兴趣降低；玩家停顿后仍有恢复机会。
- 手指松开时玩具停在合法位置；超出区域时停在边界，界面说明活动范围。
- 精力不足或身体不适时不进入追逐，提供“让它休息”选项。
- 耐用玩具不按场次消耗；消耗型玩具首次有效使用扣一次，接受前取消或拒绝不扣。心情收益按有效互动结算并递减，空拖和快速重复开启不计收益。

**地面玩具：** 球和不倒翁摆放后留在房间。玩家轻点可滚动或摇动，猫决定是否追逐；玩具活动限制在可达区域，不穿墙或挡住食盆。

**结束：** 不用星级和操作得分评价陪伴。可显示“它玩得很满足”或“它想休息了”，不每次强制弹结算窗。

### 6. 房间互动三：抚摸、梳毛和擦爪

**抚摸：**
- 点选猫进入房间内近景；睡眠或不愿靠近时保留观察，不强制起身。
- 在脸颊、下巴或背部短距离缓慢滑动，每次有效滑动对应一次抚摸。
- 喜欢时眯眼、靠向手；不喜欢时转身或轻挡；快速连续滑动时出现暂停信号。
- 松手立即停止，猫走开结束本次互动。翻肚皮不自动视为允许摸肚子。
- 已发现的偏好可在详情中查看，不预先把所有喜好展示成答案。

**梳毛：**
- 从工具栏拿出梳子，沿背部方向缓慢滑动，毛发逐渐整齐，出现少量浮毛。
- 猫愿意时转身露出另一侧；坐起或走开时可以结束。
- 每段有效梳理保存小幅清洁效果，已干净部位不重复计收益，不要求覆盖每个像素。

**擦爪：**
- 猫回到房间留下脚印时可拿毛巾，点选它愿意伸出的爪子进行局部擦拭。
- 轻擦污渍逐渐消失；猫缩爪时暂停，重新伸出后再继续。
- 猫不配合时可以稍后处理，不拉住爪子、不惩罚退出。

基础梳子和擦拭毛巾建议可重复使用，避免日常亲密互动全部依赖购买物品。

### 7. 护理区互动一：猫砂盆日常循环

**物品职责：**
- 猫砂盆：长期设施，有可见砂层、使用状态和容量。
- 猫砂：商店购买的消耗品，库存与盆内余量分别记录。
- 猫砂铲：可重复使用工具，清除污物和结团时不另收喵币。

**每日需求不等于每日签到：** 猫咪按经过时间及配置产生如厕需求，玩家根据盆的实际状态定期清理、补砂；不要求自然日跨零点就重置或强制换整盆。

**猫咪自主如厕：**
1. 有需求且盆可用时，猫走向护理区；玩家留在房间时不自动切镜头。
2. 玩家进入护理区可看到猫进盆、刨砂、遮挡处理和离开，不展示排泄细节。
3. 完成后增加可清理的结团或图标，减少盆内可用砂量。
4. 猫离开后盆恢复可维护状态，入口出现轻提示。

**铲屎操作：**
1. 点击盆查看“可用砂量、待清理数量、是否正在使用”。
2. 选择铲子，点击或短拖铲起一个结团；两种手势效果一致。
3. 将铲子移向废弃容器，进入范围后吸附，松手完成一次清理。
4. 中途松手且未到容器时结团留在盆内，不先扣状态。
5. 清理完显示“清理好了”；砂量不足时提示补砂，不自动购买。

**补砂操作：**
1. 从库存选择猫砂，预览本次填入量、库存扣除和剩余容量。
2. 拖到盆上并确认添加，提交成功后同时减少库存、增加盆内可用砂量。
3. 按实际填入量扣除；包装允许保留剩余量，不因只缺少一点砂就浪费整包。
4. 盆满时拒绝添加并说明原因；库存不足时显示可补数量和商店入口。

**整盆更换：** 作为单独的明确操作，先展示旧砂将被处理和所需新砂。只有新砂充足且玩家确认时才原子完成倒旧砂、清洗和填新砂；不足或取消不先清空旧盆。

**状态反馈：**

| 盆状态 | 画面 | 玩家下一步 |
|---|---|---|
| 可用且干净 | 砂层平整 | 不必操作 |
| 有待清理结团 | 少量符号化结团 | 铲屎 |
| 砂量偏低 | 砂层变薄 | 补砂 |
| 无砂或污物达到容量 | 提示暂不可用 | 清理并补砂 |
| 猫正在使用 | 使用中标记 | 等待，不展开维护动作 |

**照护反馈建议：** 不可用时猫在入口犹豫，降低环境舒适度并给可解释的提醒；恢复可用后影响消退。不新增“少铲一次就立即生病”的概率，也不改变已存在的疾病条件。

**离线补算：** 按经过周期惰性补算，污物累积到盆的配置容量，不无限生成需要数百次点击的结团。游标推进至已处理时间，超容量区间不留待清理后重放。超出容量的需求不重复制造无限惩罚；长离线回来能通过有限清理和补砂恢复。离线累计规则与现有猫咪属性衰减分别处理。

**多猫建议：** 同窝猫共享盆的容量，使用时串行占用，其他猫等待；三只猫的需求合并计算，不让多个动画同时写入盆。基础盆的容量与经济配置需要覆盖同窝上限，升级盆主要改善便利和外观，不强迫多猫玩家付费。

**替代方案与理由：** 直接点“清洁”最省成本，但缺乏照护参与感；精确铲砂物理又易变成劳动。采用结团级吸附操作，保留铲起和倒掉的仪式，避免精度考核。

### 8. 护理区互动二：洗澡与吹干

洗澡保留已有冷却约束；日常清洁优先通过梳毛、擦拭完成，明显脏污时再引导进入护理区。一次只护理一只猫。

| 阶段 | 玩家操作 | 猫咪与画面回应 |
|---|---|---|
| 准备 | 选猫，查看冷却和状态，确认开始 | 愿意时靠近，不安时允许返回或改局部擦拭 |
| 湿润 | 在身体可操作区域缓慢拖动花洒 | 毛发变湿；面部不是操作目标 |
| 清洗 | 小范围滑动涂抹泡沫 | 污渍淡化，猫保持或调整姿势 |
| 冲净 | 移动花洒冲去泡沫 | 分区完成，不要求逐像素覆盖 |
| 擦干 | 使用毛巾宽范围滑动 | 湿润程度降低 |
| 吹干 | 移动工具并选择较低档位 | 毛发逐渐蓬松，对声音不安时停下 |

**双向反馈：** 猫缩身、转头或后退时，工具松手即停；可以移开工具、暂停安抚。安抚降低紧张但不保证立刻服从，不使用快速连点或限时条。

**中断安全：**
- 弄湿前直接取消，不计已完成洗澡。
- 已弄湿时提供“结束清洗并协助擦干”，不把湿猫直接丢回房间。
- 切后台或重启保留护理阶段、湿润程度和已完成分区，恢复时进入继续护理或协助擦干入口。
- 完成清洗冲净阶段时记录一次洗澡完成时间并开始冷却；在此之前取消不算完整洗澡。恢复会话不得重置该时间。
- 协助擦干只消除湿润并结算已完成护理阶段，不赠送未完成的清洁区段收益。

**疾病规则边界：** 按本轮确认取消“吹干操作失误触发感冒”，手动或协助完成均不触发该判定。饱食度低于 20 且清洁度低于 30 的既有疾病判定、治疗费用和 24 小时免疫保持不变。

### 9. 家具、玩具购买与可视化布置

**商店分类：**

| 分类 | 商品示例 | 交付形态 |
|---|---|---|
| 食物 | 主粮、零食 | 消耗库存 |
| 猫砂 | 基础猫砂、不同外观或便利特性的猫砂 | 可拆分计量的消耗库存 |
| 家具 | 猫床、猫爬架、纸箱、抓板 | 独立拥有物实例 |
| 玩具 | 逗猫棒、球、不倒翁 | 耐用工具或可摆放实例 |
| 护理设施 | 猫砂盆外观、护理区收纳装饰 | 对应场景的设施或装饰 |

**首版建议商品目录：**

| 场景 | 商品 | 提供的用途 | 建议价格（喵币） |
|---|---|---|---:|
| 咪窝 | 猫床款式 | 休息位置、躺卧锚点 | 200–300 |
| 咪窝 | 纸箱 | 钻入和探头的互动锚点 | 100–150 |
| 咪窝 | 抓板 | 抓挠互动锚点 | 150–250 |
| 咪窝 | 猫爬架 | 登高站位和跳跃锚点 | 400–600 |
| 咪窝 | 球、不倒翁 | 玩家或猫咪触发的地面玩具 | 100–200 |
| 咪窝 | 窗帘、地毯、墙饰 | 改变房间外观 | 50–200 |
| 护理区 | 猫砂盆外观款式 | 改变基础猫砂盆外观，不替代必要设施 | 200–300 |
| 护理区 | 毛巾架、收纳柜、浴垫 | 改变场景外观和陈设 | 50–250 |
| 护理区 | 浴盆外观款式 | 改变洗浴区外观 | 200–350 |

价格是用于首轮经济配置和原型验证的建议区间，可调，不作为固定承诺。基础食水设施、猫砂盆和基本照护工具免费提供；不将完成基础照护所需的设施设为付费门槛。家具和装饰只提供外观、休息位置或互动锚点，不提供饱食度、清洁度、心情、信任、精力等属性加成，也不改变属性衰减速度。与猫互动后产生的效果仍按既有互动规则和收益上限结算，不因家具价格或数量提高。

每件商品展示价格、货币、用途、适用场景、占地和已拥有数量；明确区分“外观”“休息点”“互动锚点”和消耗类型。基础猫砂建议以喵币购买，不将必需品限于钻石或广告。

**购买闭环：** 商品详情预览房间效果 -> 确认购买 -> 原子扣款并交付 -> 选择“去布置”或“放背包”。买到不等于已经放置，未摆放家具不冒充房间已有物。

**布置模式：**
1. 点击“布置”，暂停当前房间的互动与家具使用动画，不改动经过时间计算。
2. 从拥有物列表选家具，拖到地面；显示占地与有效/无效标识。
3. 按支持的方向旋转；合法时确认，非法时显示具体原因。
4. 可移动或收回已摆放物，最后保存整个布局；取消恢复进入模式前布局。
5. 保存后猫重新选择可达目标，逐步探索新物品，不强制所有猫同时围观。

**放置约束：**
- 初版建议采用隐藏网格和吸附，不做自由物理摆放；逻辑网格可先按咪窝 8×6 格、护理区 6×4 格作为调参起点，再依据场景透视和屏幕比例校准，不代表固定像素尺寸。
- 小型地面物件和装饰可从 1×1 格起配，猫床、纸箱和抓板可从 2×2 格起配，猫爬架可从 2×2 或 2×3 格起配；每件商品的占地、锚点和可旋转方向由配置定义，家具只在支持的方向旋转。
- 不重叠、不挡门、不遮断食盆和护理入口的可达路径。
- 墙饰只占墙面槽位；猫砂盆只放护理区，不能拖进房间。
- 基础必要设施保持至少一套可用，替换时验证新设施，不允许误收回最后一套。
- 同一家具实例只能在一个位置；跨咪窝移动是移动拥有物，不复制商品。
- 正在被猫使用的家具不能直接搬走；提示先等待，或在进入布置时将猫安全移到空闲站位。

**物品不是背景贴图：**
- 猫床支持躺卧；猫爬架提供可达站位和跳跃锚点；抓板支持抓挠；纸箱支持进入和探头。
- 地面玩具可由玩家轻点激活，也可由猫自主使用。
- 装饰类物品只改变外观；家具提供外观、休息点或互动锚点，不售卖任何被动属性加成或衰减减免。
- 多个同类家具只增加可选择的位置，不提高属性、信任或互动收益；使用行为本身仍遵循既有互动资格与收益上限。

**替代方案与理由：** 固定装饰槽最简单但个性化有限；完全自由摆放会增加寻路、遮挡和触控成本。隐藏网格结合少量锚点能兼顾可见布置与可靠的猫咪使用动画。

### 10. 多猫、空间占用和生命周期

保留每个咪窝最多三只猫的已有规则。玩家切换选中对象不暂停其他猫的生活需求，但近景期间其他猫不抢当前手喂或逗猫棒的输入。

- 猫床、抓板、猫砂盆等有独立占用状态；只有预留成功的猫才能开始使用。
- 同一只猫同一时间只能处于一种主要活动，不能同时如厕和洗澡。
- 若猫在护理区，房间显示其当前位置，不制造第二只实体；返回后恢复同一只猫。
- 玩家不在护理区时如厕可以由逻辑补算，不常驻两个完整渲染场景。
- 转移咪窝前先完成或退出必要护理，释放原空间占用；转移成功后按新咪窝的设施继续生活。
- 猫离开咪窝不带走共享家具或猫砂，不把新咪窝的盆状态复制为原盆状态。

### 11. 商品消耗、经济平衡与提醒

**消耗边界：**
- 买猫砂扣货币；补砂扣库存；铲屎不扣额外货币。
- 耐用玩具和家具购入一次，使用不重复扣除；明确标为消耗品的商品另按实际效果扣除。
- 非法操作、未开始的互动、取消布置不扣物品，不以播放动画作为扣款成功依据。
- 不默认把铲屎转化为无限喵币来源，照护主要回馈环境恢复和关系体验。

**价格调参方法：** 先测一只和三只猫的日常主粮及猫砂消耗，再比较常规游玩所得喵币；基础必需消耗不应要求每天完成大量关卡。单猫日常成本按“主粮份数 × 单价 + 猫砂实际用量 × 单位价”估算，多猫按实际使用计量，不按每次进入护理区收费。

价格、容量、如厕周期、互动收益上限和恢复速度作为配置，本文不捏造正式数值。建议原型中让一次日常互动约半分钟至一分钟可自然结束，清理一个盆只需少量操作；这是节奏测试起点，不是强制计时规则。

**提醒策略：** 房间入口汇总“食物待补、猫砂待清理”等实际需求；可以忽略并稍后处理。库存不足显示准确原因和商店入口，余额不足保留原状态；不强制广告、不反复弹窗。

### 12. 数据模型与领域边界

在原有表现、领域和平台适配分层上补充以下概念，不在本节生成代码或数据库迁移：

| 概念 | 需保存的信息 |
|---|---|
| 猫咪生活状态 | 原有属性、精力、偏好、信任、即时情绪、当前逻辑活动、最后补算时间 |
| 咪窝布局 | 咪窝标识、布局版本、家具实例及坐标/方向、可用设施 |
| 护理区 | 所属咪窝、猫砂盆实例、护理设施布局 |
| 猫砂盆 | 盆内砂量、结团记录、容量、最后补算时间、状态版本 |
| 食盆表现 | 对应猫咪、已结算喂食操作标识、食物外观及动画阶段；不维护可再次获益的余粮 |
| 拥有物与库存 | 家具/玩具实例、消耗品数量及猫砂剩余计量 |
| 护理会话 | 猫咪、阶段、湿润程度、已完成区域、会话标识 |
| 待确认操作 | 操作标识、对象、依赖、顺序、预期版本和确认状态 |

场景负责手势、镜头和动画；领域层负责有效互动、状态推进、容量、占用和扣除条件；云端确认货币、库存、拥有物及关键状态变化。

**操作粒度：** 不上传每一帧手指坐标。将拖动归纳为成功喂食、玩耍会话、梳理区段、结团清除、补砂和布局保存；服务端按操作标识、资格和配置收益上限校验，客户端动画不是奖励权威。

**原子与幂等：**
- 补砂同时减少库存并增加盆内余量；重复提交返回原结果。
- 清理绑定结团标识和盆版本，同一结团最多被清除一次。
- 布置保存验证物品所有权、唯一位置和可达性，并一次提交布局。
- 如厕补算与玩家清理基于同一版本规则，避免清理完又被旧快照覆盖。
- 喂食即时结算与最后喂食时间绑定同一幂等记录；猫砂使用和需求补算共享可信时间策略，不让修改本机时钟制造收益或重置需求。

**离线与冲突：** 继续使用已确认基线加有序预测操作。待确认购买的家具可以预览，但带待确认标记；购买失败后依赖它的摆放撤销并明确提示。补砂失败时不保留预测盆内增量；清理或布局保存冲突时保留已确认部分，重新基于最新版本处理后续操作，不整份覆盖云端。

### 13. 资源与技术实现建议

- 咪窝和护理区采用可复用模板，每个咪窝加载自己的布局及猫咪状态。
- 家具配置包含占地、绘制层级、可放区域、交互类型和使用锚点；缺少必要配置时阻止上线该商品并报告错误。
- 猫咪至少准备观察、走动、休息、吃饭、扑击、抓住、蹭手、拒绝、梳毛配合和护理紧张等状态；先用占位资源验证动作衔接。
- 猫砂盆需要空、可用、少砂、有结团和使用中等视觉状态；排泄表现符号化。
- 布置碰撞和可达性在确认前验证，不能依赖猫走不过去后静默瞬移。
- 不在每帧访问云端，状态补算和经济提交按明确操作边界进行。
- 音效可关闭；扩大交互命中区域，提供点选替代拖动，不把精细手势作为必需护理的唯一通路。

### 14. 体验验收场景

以下已映射到规格与任务，实施时须记录真实验证结果，本轮尚未执行游戏测试：

| 场景 | 预期结果 |
|---|---|
| 猫已饱，玩家尝试喂食 | 拒绝并解释，不扣食物、不改变属性或最后喂食时间 |
| 喂食成功后动画中重启 | 即时结算保留，恢复动画不重复扣物品或增加属性 |
| 手喂被拒绝，玩家退出 | 未吃的零食不消耗，不扣信任 |
| 猫疲劳，玩家继续拖逗猫棒 | 猫休息，有原因提示，不继续刷心情 |
| 抚摸出现暂停信号，玩家松手 | 动作停止，猫可继续留在附近或离开 |
| 部分梳毛后返回房间 | 有效进度保留，重复梳同一区域不无限获益 |
| 猫正在如厕，玩家点击铲子 | 提示使用中，不清除污物、不扣物品 |
| 一次结团清理请求重复提交 | 只清除一次，不重复改变状态 |
| 盆只缺少部分猫砂 | 只扣实际填入量，包装余量保留 |
| 猫砂不足，尝试整盆更换 | 不先清空旧盆，提示缺少数量 |
| 长时间离线后回到护理区 | 累积有容量边界，有限操作即可恢复，不无限追加结团 |
| 三只猫先后使用同一个盆 | 串行占用，消耗正确合并，不出现重复猫或负砂量 |
| 湿润后切后台或重启 | 恢复护理状态，可继续或协助擦干 |
| 家具放在门口或不可达位置 | 拒绝保存并说明原因，旧布局不丢失 |
| 收回正在被使用的猫床 | 等待或先安全释放占用，不移除猫仍引用的锚点 |
| 将护理区猫砂盆拖进房间 | 拒绝放置，显示适用场景 |
| 待确认家具购买被云端拒绝 | 依赖摆放撤销，有错误提示，不仅扣款或保留复制物 |
| 退出布置或重启 | 取消恢复旧布局；已保存布局可恢复 |
| 猫主动叼来玩具，玩家不回应 | 猫转去其他活动，不扣信任、不弹强制任务 |

### 15. 分阶段验证与风险

在原有整体上线顺序中细化养猫阶段，以下是建议顺序，不是首发范围承诺：
1. 房间和护理区导航、选猫、基础设施及存档。
2. 房间喂食、逗猫棒、抚摸与情绪回应，验证双向互动。
3. 猫砂商店商品、如厕补算、铲屎、补砂和多猫占用，形成必需照护闭环。
4. 家具购买、背包、可视化布置及猫咪自主使用。
5. 梳毛、擦爪、完整洗浴与中断恢复，再扩展主动邀请和个体偏好。

- [风险] 家具很多但猫只会随机走动 -> [缓解] 首批家具必须有实际使用锚点和行为，不以装饰数量替代互动质量。
- [风险] 清理猫砂变成重复劳动 -> [缓解] 结团吸附、离线容量边界和少量操作；通过一只与三只猫的体验分别调参。
- [风险] 必需品成本压迫玩家 -> [缓解] 用日常成本与常规收益测算平衡，提供初始基本物资，必需品不只用钻石购买。
- [风险] 猫的拒绝不可理解 -> [缓解] 状态驱动回应并配合首次提示，基础需求提供可行替代方式。
- [风险] 两场景产生重复猫或占用残留 -> [缓解] 统一逻辑猫实体，场景仅绑定表现，切换和转移显式释放占用。
- [风险] 关闭养猫功能后留下未完成洗浴 -> [缓解] 保留会话恢复和协助擦干入口，不因关闭新入口丢弃关键状态。

### 16. 规划同步与决定记录

本轮继续完善现有变更，不新建重叠变更；已将目标设计同步到提案、规格、任务和下方生成提示词。原有未受影响玩法保留。规划完成不等于代码已经实现：

| 关联文件 | 同步内容 |
|---|---|
| [proposal.md](proposal.md) | 房间/护理区、猫砂必需品和可视化布置的范围说明 |
| [cat-care/spec.md](specs/cat-care/spec.md) | 双向互动、如厕清理补砂、情绪与关系、多猫护理及中断规则 |
| [cat-home/spec.md](specs/cat-home/spec.md) | 房间/护理区导航、安全区 UI、可见布置、可达性和自主使用 |
| [game-economy/spec.md](specs/game-economy/spec.md) | 猫砂计量、耐用玩具与家具实例、购买和实际使用的扣除边界 |
| [player-save/spec.md](specs/player-save/spec.md) | 猫砂盆内余量、喂食表现、布局、护理会话和离线依赖操作 |
| [tasks.md](tasks.md) | 新交互、设施、布局与对应验收任务 |

**本轮明确确认的决定：**
- 喂食保留旧规则：操作成功即扣库存并增加饱食度。否决延迟进食结算，不保存可再次获益的余粮；动画仅作表现。
- 玩具改为按商品类型区分：耐用品不扣实例，消耗品首次有效使用扣一次；拒绝或成功前取消不扣。
- 吹干采用治愈方案：可协助擦干，不因吹干操作失误触发感冒；仅结算已完成护理效果。

这些决定已反映在规格中，实施者不得再次使用前文对话里的旧草案。其他疾病、洗澡冷却和连续 30 天未喂食的收容规则仍有效，猫砂和家具不重置最后喂食时间，也不新增收容触发条件。首次成功喂食的云端确认时间更新期限，拒绝或同一操作重试不重置。

**可延后调参的事项：** 商品价格、砂量单位显示、盆容量、如厕周期、互动节奏、提示出现时机、美术主题和音效组合。调整不得绕过以上边界或把基础照护变成强制付费。

## 全局 UI 页面与导航设计

### 页面清单与边界

首版规划 11 个主要界面。这里的“界面”表示玩家能到达的页面，不要求每个界面都是独立 Creator 场景；加载、地图、结算和详情等可由同一场景中的页面状态或 Prefab 实现。设置、暂停、购买确认、猫咪动作菜单和操作反馈使用浮层，不计入主界面数量。

| # | 主界面 | 主要职责 | 主要入口与返回 |
|---|---|---|---|
| 1 | 加载与恢复 | 初始化资源、读取本地状态并恢复可继续的进度；失败时给出重试 | 启动进入；成功后进入大厅 |
| 2 | 游戏大厅 | 展示玩家身份、货币和核心玩法入口；分流至闯关或咪窝 | 启动后的主入口 |
| 3 | 关卡地图 | 展示关卡路线、进度、解锁状态和奖励预览 | 大厅进入；开始挑战进入对局 |
| 4 | 连连看对局 | 展示棋盘、当前目标、计时/步数和局内道具 | 地图进入；暂停浮层可继续、重开或退出 |
| 5 | 关卡结算 | 展示胜负、成绩和奖励，提供下一步选择 | 对局结束进入；返回地图或开始下一关 |
| 6 | 咪窝房间 | 展示猫咪日常生活、选猫、互动、商店/背包/布置和护理入口 | 大厅进入；护理区和猫咪详情由此进入 |
| 7 | 护理区 | 在同一护理场景中分开展示猫砂维护和洗浴 | 咪窝进入；返回原咪窝并恢复视图 |
| 8 | 商店 | 分类浏览、预览并购买商品 | 大厅或咪窝进入；关闭后返回来源页面 |
| 9 | 背包与布置 | 按家具、玩具、日用品、食物查看持有物；进入布置时只选可摆放物 | 主要从咪窝进入；取消或保存后返回原空间 |
| 10 | 猫咪图鉴 | 展示品种资料、收集状态和解锁条件 | 大厅底部导航进入；不直接管理某只猫实例 |
| 11 | 猫咪详情 | 展示所选猫实例的状态、等级、偏好和活动 | 从咪窝选猫后进入；返回原咪窝及所选猫 |

首版底部主导航固定为“大厅 / 闯关 / 咪窝 / 图鉴”。商店从大厅或咪窝的次级入口进入；背包与布置主要从咪窝进入。对局、结算、护理区和详情等沉浸页面使用返回入口，不重复显示完整底部导航。收留站和词汇练习属于后续阶段，不占首版主导航位置。

### 全局视觉与布局规则

- 沿用暖色手绘、轻俯视、治愈感风格：奶油白、浅木色、杏橙、鼠尾草绿和深棕文字；圆角卡片、柔和阴影、轻量按压反馈。
- 大厅、咪窝、护理区和图鉴以场景插画/猫咪为主视觉；连连看对局减少背景装饰、提高棋盘与状态信息对比度。
- 页面顶部优先呈现标题、返回入口和必要状态；主操作集中在安全区域内的底部或明确的操作卡片。遵循本设计的竖屏适配、安全区、微信胶囊和最小触控面积规则。
- 页面跳转尽量保留来源上下文；关闭浮层返回原页面，护理区返回原咪窝，猫咪详情返回进入前咪窝并恢复当前选中猫。
- 加载或数据恢复失败必须有明确错误状态和可用操作，不以空白页或看似成功的默认数据代替。

### 11 个主界面线框

#### 1. 加载与恢复

```text
+----------------------------------+
|                                  |
|            咪头咪猫              |
|       [猫咪与房间主题插画]        |
|                                  |
|          正在整理猫窝...          |
|          [====------]            |
|                                  |
|       失败时显示：[重试]          |
+----------------------------------+
```

展示品牌、主题插画和加载状态。资源或存档恢复失败时显示原因及重试入口；成功后进入游戏大厅。

#### 2. 游戏大厅

```text
+----------------------------------+
| [头像] 玩家名         喵币  设置 |
|----------------------------------|
|                                  |
|      [猫咪在大厅场景中活动]       |
|                                  |
|   +--------------+ +-----------+ |
|   | 继续闯关     | | 前往咪窝  | |
|   +--------------+ +-----------+ |
|                                  |
|   [活动/奖励入口]  [新内容提示]  |
|----------------------------------|
| 大厅       闯关       咪窝   图鉴 |
+----------------------------------+
```

大厅突出“继续闯关”和“前往咪窝”两个主操作；玩家资料、货币、活动和奖励为次级信息，不堆叠成管理仪表盘。

#### 3. 关卡地图

```text
+----------------------------------+
| < 大厅          关卡地图    喵币 |
|----------------------------------|
|                 [关卡 05]         |
|                    |              |
|          [关卡 04]                 |
|              |                    |
|   [关卡 03]--+                    |
|       |                          |
|   [关卡 02]                      |
|       |                          |
|   [关卡 01] 已通关                |
|----------------------------------|
| 奖励预览              [开始挑战] |
+----------------------------------+
```

以纵向路线表达关卡进度；当前关卡、已完成和未解锁状态有不同视觉样式。选中关卡后显示奖励预览和开始入口。

#### 4. 连连看对局

```text
+----------------------------------+
| < 暂停         第 05 关    01:32 |
|----------------------------------|
| 目标：清除所有方块       步数 18 |
|                                  |
|      +--------------------+      |
|      | 方块 方块 方块 方块 |      |
|      | 方块 方块 方块 方块 |      |
|      | 方块 方块 方块 方块 |      |
|      | 方块 方块 方块 方块 |      |
|      |       ...          |      |
|      +--------------------+      |
|                                  |
|   [提示]       [洗牌]       [道具]|
+----------------------------------+
```

棋盘为页面视觉中心；顶部显示关卡、目标及时间/步数，底部提供局内操作。暂停使用浮层，提供继续、重新开始和退出关卡。

#### 5. 关卡结算

```text
+----------------------------------+
| < 返回地图                       |
|                                  |
|          [胜利 / 失败]            |
|            本关完成               |
|          成绩 / 星级               |
|                                  |
|          获得：+120 喵币          |
|                                  |
|   [重试] [返回地图] [下一关]      |
+----------------------------------+
```

胜败共用结算结构，文案、奖励和按钮按结果切换。胜利时突出下一关；失败时突出重试。显示奖励确认状态，避免把待云端确认奖励冒充已到账。

#### 6. 咪窝房间

```text
+----------------------------------+
| 咪窝 01       喵币/钻石      设置|
| [猫1] [猫2] [猫3]                |
|----------------------------------|
| 窗户       猫爬架        护理区门|
|                                  |
| 纸箱       猫咪          抓板     |
|                                  |
| 猫床   食盆/水碗       活动空地   |
|----------------------------------|
| 商店       背包       布置  护理区|
| 大厅       闯关       咪窝   图鉴 |
+----------------------------------+
```

房间是日常照护主场景。猫头像用于选择实例；选中后提供详情入口及针对该猫的互动。主导航与房间操作栏分层，交互中收起无关工具。

#### 7. 护理区

```text
+----------------------------------+
| < 返回咪窝          护理区        |
|----------------------------------|
| 猫砂维护区          洗浴区        |
|                                  |
| [猫砂盆]          [浴盆]          |
| [猫砂铲]          [毛巾]          |
| [废弃容器]        [吹干台]        |
|                                  |
|       中间保留通行空间             |
|----------------------------------|
| 选择设施后显示对应操作面板         |
+----------------------------------+
```

猫砂维护和洗浴区域清晰分隔；进入护理区不要求预选猫，开始洗澡时才选择猫。一次只打开一个操作面板。

#### 8. 商店

```text
+----------------------------------+
| < 返回          商店         喵币 |
|----------------------------------|
| 食物 | 猫砂 | 家具 | 玩具         |
|----------------------------------|
| [图] 猫床款式       200 喵币      |
|      休息位置 / 咪窝 / 占地 2x2   |
|      [预览]             [购买]    |
|----------------------------------|
| [图] 纸箱           100 喵币      |
|      互动锚点 / 咪窝 / 占地 2x2   |
|      [预览]             [购买]    |
+----------------------------------+
```

商品卡片展示名称、价格、效果/用途、数量、适用空间和占地；家具只展示外观、休息点或互动锚点，不宣称数值优势。购买确认作为浮层。

#### 9. 背包与布置

```text
+----------------------------------+
| < 返回咪窝       背包 / 布置      |
|----------------------------------|
| 家具 | 玩具 | 日用品 | 食物       |
|----------------------------------|
| 拥有物列表     |     当前空间      |
| [猫床 x1]      |  +------------+  |
| [纸箱 x1]      |  | 吸附网格   |  |
| [抓板 x1]      |  |   [预览物] |  |
|                |  +------------+  |
|----------------------------------|
| [旋转] [收回]       [取消] [保存] |
+----------------------------------+
```

普通背包状态按四类展示物品，并区分堆叠数量与唯一实例。进入布置状态后只列出家具及配置为可摆放的玩具；食物和日用品不能作为摆放目标。取消恢复原布局，保存后才提交。

#### 10. 猫咪图鉴

```text
+----------------------------------+
| 猫咪图鉴                 搜索/筛选|
|----------------------------------|
| 全部 | 已拥有 | 未解锁            |
|----------------------------------|
| [品种卡] [品种卡] [品种卡]        |
| 品种 A   品种 B   ???             |
| 已拥有   已拥有   解锁条件         |
|                                  |
| [品种卡] [品种卡] [品种卡]        |
|----------------------------------|
| 大厅       闯关       咪窝   图鉴  |
+----------------------------------+
```

图鉴以品种和收集状态为对象，展示品种资料及解锁条件；不直接打开或管理某只猫咪实例的当前状态。

#### 11. 猫咪详情

```text
+----------------------------------+
| < 返回咪窝          猫咪详情      |
|----------------------------------|
|          [猫咪立绘/动作]          |
|               米米                |
|       品种：橘猫    等级：03       |
|----------------------------------|
| 饱食 [======---]   清洁 [=====--] |
| 心情 [=======--]   精力 [====---] |
| 健康 [==========]                 |
|----------------------------------|
| 偏好：喜欢逗猫棒 / 喜欢摸下巴      |
| 当前活动：在猫床休息               |
|             [回到咪窝互动]         |
+----------------------------------+
```

详情针对从咪窝选中的猫咪实例，展示当前状态、品种、等级、已发现偏好、活动及所属咪窝；活动不允许的操作不可显示为可执行操作。返回恢复原咪窝和选中猫。

### 页面跳转与主流程

```text
启动 -> 加载/恢复 -> 游戏大厅
                         |
          +--------------+---------------+
          v                              v
       关卡地图                         咪窝房间
          |                         /     |      \
          v                        v      v       v
       连连看对局 <-> 结算       护理区   商店   背包/布置
          |                        \      |       /
          +------ 返回地图/大厅       +---- 返回房间

游戏大厅 -> 猫咪图鉴
咪窝选择猫 -> 猫咪详情 -> 返回原咪窝与该猫
```

- 首次流程：加载/恢复 -> 大厅 -> 第一关引导 -> 对局 -> 结算 -> 返回地图或大厅；引导分段出现，不连续堆叠弹窗。
- 闯关流程：大厅 -> 地图 -> 对局 -> 结算 -> 下一关/重试/返回地图；退出对局需经暂停确认。
- 养猫流程：大厅 -> 咪窝 -> 选猫并互动；需要洗浴或猫砂维护时进入护理区，再返回原房间状态。
- 物品流程：商店预览 -> 购买确认 -> 放入背包或去布置 -> 保存/取消 -> 返回发起购买或布置的页面。
- 图鉴流程只浏览品种资料和解锁进度；查看拥有猫咪的实例状态必须从咪窝选择具体猫后进入详情。

## 微信小游戏 UI 与交互生成提示词

**用途：** 以下“提示词开始”至“提示词结束”之间的内容可以整体复制给 AI，作为 Cocos Creator 微信小游戏场景与功能的生成输入。它替代对话中未保存的手机网页原型提示词，不以 React、DOM、CSS 或 localStorage 为实现目标。本次只保存提示词，不生成工程、代码或资源。

### 提示词开始

你是一名游戏 UI 设计师、交互设计师和 Cocos Creator 工程师。请按照以下要求设计并制作《咪头咪猫》的猫咪生活与互动模块，目标为 **Cocos Creator 3.8 系列、TypeScript、竖屏微信小游戏**，不是微信小程序页面，也不是网页应用。

#### A. 工作范围与决策边界

1. 先检查已有工程、锁定的 Creator 版本、场景和资源，沿用项目规范。无工程时使用可实际获得的 3.8 系列版本建立工程，并报告具体版本及操作条件；不能用普通网页代替 Creator 工程。
2. 生成房间、护理区、猫咪互动、商店、背包和布置相关内容；不扩展支付、广告、排行榜、医院或收容所，不破坏已有模块。
3. 不只给效果图或文字说明：在具备开发权限的实施阶段，应提供可加载的场景、Prefab、脚本、资源绑定和实际功能。缺少编辑器或美术资源时如实说明，不把未运行的结果称为已完成。
4. 当前处于探索或文档阶段时，仅输出设计，不执行上述实现；生成提示词本身不构成实施授权。
5. 以下三项已明确确认并同步本变更规格，不再作为待选项：
   - 喂食操作成功即扣食物并增加饱食度，后续进食动画不再结算或留下可再次获益的余粮；
   - 耐用玩具重复使用，消耗型玩具首次有效使用扣一次；
   - 洗浴可协助擦干，不因吹干操作失误或协助完成触发感冒。
6. 保留其他疾病与收容规则；开发时读取本变更的 cat-care、cat-home、game-economy 和 player-save 规格，不按旧草案自行覆盖已确认行为。
7. 下文标为“建议”的布局、美术和数值为可调参数；精力、关系、共享护理区和基础设施已纳入规划，不作为省略功能的理由。

#### B. 产品气质与美术规范

游戏为轻拟真、偏治愈的养猫体验。猫咪会拒绝、会疲劳、会主动接近，玩家可以等待、调整操作和尊重边界。互动不按星级评价，不设置操作失误重罚或必须回应的倒计时。

**建议美术方向：**
- 暖色手绘感 2D 轻俯视空间，能看到后墙、局部侧墙和地板，不做后台管理界面。
- 奶油白 #FFF8EE、浅木色 #DDBB95、杏橙 #F2A477、鼠尾草绿 #A9BEA0、深棕文字 #51443C。
- 圆角九宫格面板、柔和接地阴影、小幅按压反馈；重要文字维持足够对比。
- 猫咪是画面主角，表情、姿势、尾巴和移动均能传达状态。
- 猫、家具、道具比例和描边统一，不用 emoji 或杂乱网络图片代替主要美术。
- 正式美术优先使用可授权且可被目标引擎正确导入的贴图图集；资源不足时允许以 Graphics 绘制统一占位形象，并明确占位资源清单。不要假设运行时可直接加载任意 SVG。
- 猫咪至少具有头、耳朵、身体、尾巴、前爪的可控表现层，或提供具有等效动作的序列帧资源。不得仅用一个静态头像加文字模拟全部互动。

#### C. 设计尺寸、适配与层级

以 **750 × 1334 设计单位**作为建议竖屏基准，适配不同长宽比，不把设计单位当成设备物理像素。

- 使用 Canvas、UITransform、Widget、Layout 等 Creator UI 能力进行布局。
- 根据设备安全区域和微信胶囊实际边界布局 HUD；通过平台适配层读取信息，不假定所有手机有相同顶部留白。
- 主交互区域至少约 88 × 88 设计单位，并在目标设备上验证实际触达大小。
- 顶部固定 HUD，底部固定工具栏，中间空间由场景占据；短屏优先保证动作按钮和猫咪可见。
- 顶部包含咪窝名、货币和设置，下方是最多三只猫的头像条。
- 点击头像选择当前猫；在选中状态提供清晰的“详情”入口，查看单只猫咪实例信息。
- 普通界面以场景为主，不常驻大量属性条；详情点击展开。
- 动作面板为底部抽屉，通常不超过安全可用高度的 40%；商店和背包可使用更高的滚动面板。
- 动作进行时隐藏无关工具按钮，保留明确的停止或返回入口。
- 切换场景可短暂淡入淡出，不用高频闪烁或夸张晃动；支持关闭声音和减少动态效果。

建议节点组织如下，节点名可按项目规范调整，但各层责任必须保持清晰：

```text
CatHome
  Canvas
    SpaceRoot
      Background
      Floor
      FurnitureLayer
      CatLayer
      Foreground
      WorldEffects
    HUD
      SafeTop
        RoomTitle
        Currency
        SettingsButton
        CatSelector
      SafeBottom
        MainToolbar
    InteractionLayer
      TargetIndicator
      GestureTool
      ContextActions
    OverlayLayer
      BottomSheetHost
      ModalHost
      ToastHost
      TransitionBlocker
```

所有节点通过 Prefab 和类型明确的组件绑定，不把节点名搜索散落在业务代码中。弹窗开启时拦截底层触摸；只有场景过渡或真实提交期间阻止重复操作，不让遮罩永久挡住输入。

#### D. 咪窝房间主画面

房间是日常生活主场景，喂食、逗猫棒、抚摸、梳毛和擦爪均在此发生。近景只是同一场景中的视图变化，不另开五个小游戏场景。

**猫咪详情入口：** 点击猫咪头像先选中目标，再通过头像旁“详情”入口或猫咪上下文菜单进入其详情页。详情页以单只猫咪实例为对象；从详情返回后恢复咪窝和该只猫的选中状态。图鉴中的品种卡不直接打开此实例详情。

**建议构图，坐标以归一化画面位置说明，不作为硬编码像素：**
- 后墙窗户位于上方中间；猫爬架预览区在右后方。
- 猫床位于左侧中景；纸箱在左后方或中景。
- 中央至前景留出猫咪走动、扑击和互动空间。
- 食盆和水碗位于左前方，护理区门位于右后方或右侧墙。
- 抓板放在不挡动线的侧边；可摆地面玩具的活动区优先安排在中央开阔地带。
- 位置可以通过布置改变，但门、固定背景与必要设施锚点受约束。
- 空房间也必须完整好看，家具不能成为填满屏幕的商品卡片。

**界面与点击：**
- 底部工具栏：“商店”“背包”“布置”“护理区”。
- 点击猫：选中并显示小型动作菜单，“摸摸”“玩耍”“梳毛”“擦爪”。
- 点击食盆：展开该猫的食物面板；多猫食盆标明名字。
- 点击地面玩具：激活可互动表现，不直接弹出商城。
- 点击家具：空闲时可邀请猫使用，不能强制中断睡眠、如厕或护理。
- 点击护理门进入护理区，返回保留咪窝、选中猫和视图状态。
- 点击空地关闭菜单；普通模式不允许拖走猫或移动家具。

**空闲生活：** 猫会看窗外、短距离走动、在床上休息、使用抓板或玩球。目标必须可达，不随机穿墙或快速往返。偶尔主动叼玩具或靠近邀请抚摸，忽略邀请不扣信任。

#### E. 猫咪状态与动画

分别维护需求、即时情绪、长期信任和个体偏好，避免“心情高就永远接受操作”。

- 需求包含饱食度、清洁度、心情、健康和精力；玩耍消耗精力，休息按配置恢复。
- 即时情绪：放松、好奇、兴奋、紧张、烦躁。
- 当前活动：空闲、走动、进食、玩耍、休息、如厕、洗浴。
- 个体偏好：食物、玩具、抚摸位置，逐步发现，不完全绑定品种。
- 平时显示“想玩一会儿”等短语，头像详情才显示属性。

最低表现要求：

| 状态 | 姿态与反馈 |
|---|---|
| 放松 | 眯眼、身体松弛、慢摆尾 |
| 好奇 | 耳朵朝向目标、转头观察 |
| 准备扑击 | 身体伏低、视线盯住玩具 |
| 抓到玩具 | 前爪抱住，短暂轻踢 |
| 接受抚摸 | 靠向手、轻蹭 |
| 想暂停 | 甩尾、转身、爪子轻挡 |
| 疲劳 | 趴下，减少动作 |
| 进食 | 低头，间歇舔嘴 |
| 护理紧张 | 缩身、回看工具或后退 |

拒绝必须能追溯到疲劳、偏好、情绪或活动占用，不能随机失败。第一次出现边界信号时配一句短提示，例如“它想歇一会儿”。声音、颜色均不能成为唯一信息来源。

#### F. 喂食与手喂界面

食盆面板包含当前喂食表现状态、食物列表、库存、效果和“喂食一份”，保留关闭入口。

1. 选食物后，拖动包装到碗上，或点击等价按钮。
2. 命中碗范围时高亮，松手尝试一份；检查库存、饱食度、意愿和活动占用，长按不无限扣除。
3. 猫靠近闻食物，愿意时成功执行，立即扣食物并增加饱食度；已饱或拒绝时不扣并解释。
4. 碗中贴图展示已结算食物的进食动画，动画减少或完成不再增加属性。
5. UI 回显本次即时结算结果，动画进行中禁止重复喂食。

成功前取消不扣物品，库存不足显示原因和商店入口。退出和重启保留已结算操作与动画阶段，不把动画中的食物视为可再次喂食的余粮，不能返还或换口味重复获益。云端首次成功确认时记录最后喂食时间，同一请求重试不刷新期限。

手喂时玩家将零食停在猫前方，猫主动靠近。接受并首次吃下即成功扣除及增加属性，未接受且取消不扣；愿意程度由关系与状态决定，不允许强行把手贴到嘴上。

#### G. 逗猫棒与地面玩具界面

启动后收起普通工具栏，显示“收起玩具”，地面活动边界用低干扰提示表现。

- 触摸拖动控制玩具头，猫观察并靠近；慢移后短暂停顿可触发扑击。
- 猫抓到后停留抱住动作，再释放，不能玩具每次都逃走。
- 拖动太快时猫停下来寻找，提示“慢一点，它在找玩具”。
- 疲劳时趴下，不继续刷心情；玩家可结束并陪伴休息。
- 松手工具停在合法范围，超界限制位置，不穿墙。
- 地面球点击可滚动，猫自主决定追逐；玩具停留于可达区域。
- 不设置计时评分；收益按有效会话限额，不按每帧拖动发放。
- 耐用玩具不扣实例，消耗型在首次有效使用时扣一次，取消或拒绝不扣，重复动画不能再次扣除。

#### H. 抚摸、梳毛与擦爪界面

抚摸进入房间内近景，猫仍有地面和房间背景。标明返回，不大面积覆盖猫。

- 脸颊、下巴、背部分别作为命中区域。
- 缓慢滑动触发抚摸，喜欢则眯眼靠近，不喜欢则转身或轻挡。
- 高频快速滑动出现暂停信号，松手停止。
- 猫走开即可结束，不弹失败、不强拉回来。
- 提供点按式轻抚作为等价操作；翻肚皮不自动允许摸肚子。

梳毛选择工具后沿毛向缓慢滑动，毛发逐渐整齐；区段完成后不重复刷清洁值。猫坐起时允许结束，部分效果保留。

擦爪时显示可擦污渍，宽容判断有效区域，不要求逐像素擦除；缩爪暂停，等猫主动重新伸出。工具退出时解除输入锁。

#### I. 护理区主画面

护理区是与咪窝相连的另一个场景：
- 左侧划为猫砂维护区，放置猫砂盆、铲子、猫砂袋和废弃容器。
- 右侧划为洗浴区，放置浴盆；毛巾架、收纳柜和吹干台靠后墙布置。
- 猫砂区与洗浴区在同一场景中分隔，不把污物堆在浴盆边。
- 中间保留从入口到两侧设施的通行区域；收纳装饰不得阻断猫咪或玩家操作路径。
- 基础猫砂盆、铲子、浴盆、毛巾和吹干工具作为必要照护设施提供；可购买的同类款式只替换外观，不提高照护效率或数值收益。
- 顶部显示“护理区”和返回入口；进入时不要求选猫。
- 铲屎维护盆，洗澡需要选择当前咪窝的一只猫。
- 同时只开启一个操作面板，护理过程中不能切换去铲屎。
- 同一只猫在房间和护理区只对应一个逻辑实体，不能复制出现。

#### J. 猫砂盆操作界面

点击盆出现底部面板：“可用砂量”“待清理数量”“铲屎”“补砂”“整盆更换”。

盆需要空、干净、少砂、有结团和使用中等可见状态，排泄表现符号化。

**铲屎：**
1. 选择铲子，点击或短拖选中一个结团。
2. 拖到废弃容器，进入范围时吸附并高亮。
3. 松手后才清除结团；未投到容器则放回。
4. 提供“点击选中 -> 点击丢弃”的等价操作。
5. 清理结束提示“清理好了”，砂不足时建议补砂，不自动购买。

**补砂：**
- 显示库存、缺少量和本次实际填入量，确认后同时扣库存、增盆内余量。
- 猫砂允许按实际用量消耗，包装余量保留。
- 盆满、无库存、正在使用时说明原因。
- 猫砂从商店购买，铲子可重复使用，铲屎不再收取额外喵币。

**整盆更换：** 确认新砂充足后一次完成旧砂处理和新砂填入，取消或不足时不得先清空盆。

**日常生活：**
- 猫自主前往如厕，刨砂与遮挡动画后留下可清理结团。
- 盆被占用时禁止维护，多猫串行使用。
- 玩家不在护理区时逻辑继续，不强制切镜头。
- 按经过时间积累，不以跨零点强制换盆；离线容量有边界，不生成数百个待清理物。
- 盆不可用给温和提示，不自行新增疾病、收容或付费惩罚规则。

#### K. 洗澡与吹干界面

顶部展示阶段，底部只显示相关工具，中央保留足够大的猫咪操作区域。

流程：准备 -> 湿润 -> 清洗 -> 冲净 -> 毛巾擦干 -> 吹干 -> 返回房间。

- 准备阶段选择猫、检查洗澡冷却，未满足条件说明剩余时间。
- 花洒仅在身体有效区域工作，面部不作为操作目标。
- 小幅滑动清洗产生泡沫，冲洗后泡沫按区段消失。
- 毛巾宽范围滑动降低湿润程度。
- 吹干工具可移动、暂停并选择支持的档位；表现毛发逐渐恢复。
- 猫紧张时缩身或后退，松手即停，可以暂停安抚，不要求快速连点。
- 未弄湿前可以取消；已弄湿后返回需选择继续或协助擦干，协助清除湿润并只结算已完成阶段，不直接丢弃状态。
- 切后台、重启后能继续当前护理阶段。
- 吹干操作失误及协助完成不触发感冒，其他疾病判定不变；首次完成清洗冲净时记录冷却，恢复不重复起算，之前取消不算完整洗澡。

#### L. 商店、背包与可见布置

商店以底部高面板展示“食物”“猫砂”“家具”“玩具”，使用 ScrollView，卡片包括图标、名字、价格、数量、作用、消耗类型和适用场景。

- 购买时先检查资格与余额，成功同时扣款和交付。
- 余额不足显示“喵币不足”，不改余额和库存。
- 家具买入后选择“去布置”或“放背包”，不默认已放进房间。
- 背包固定分为“家具”“玩具”“日用品”“食物”四类：家具包含猫床、猫爬架、纸箱、抓板及装饰；玩具包含耐用工具和消耗型玩具；日用品包含猫砂等按量或按次消耗的护理物资；食物包含主粮和零食。
- 背包按商品的持有模型展示数量或独立实例：计量/堆叠消耗品显示可用量，耐用家具和工具显示唯一实例及所在咪窝/护理区；猫砂铲、梳理工具等初始提供的可重复使用工具不计作消耗品。
- 布置模式只列出家具和商品配置明确标记为可摆放的玩具；食物、猫砂及其他日用品不出现在场景摆放清单中。耐用或消耗是玩具的持有/使用属性，不自动决定其是否可摆放。
- 家具预览必须体现空间外观、休息用途或互动锚点，不展示被动属性加成；商品详情说明家具不改变猫咪照护数值。

布置模式：
1. 显示与当前空间对应的隐藏吸附网格和“保存”“取消”，保留顶部场景身份。
2. 从“家具”分类或可摆放玩具筛选中拖出物品，显示占地、适用空间、有效状态与原因。
3. 支持合法位置移动、可用方向旋转和收回。
4. 保存整份布局；取消恢复进入前布局。
5. 保存后猫重新寻找可达目标，逐步探索新物品。

约束：
- 不重叠、不挡门，不阻断必要设施路径。
- 猫砂盆只能放护理区；墙饰只能放支持的墙面位置。
- 不允许同一实例重复摆放，不允许收回最后一套必要设施。
- 使用中的家具先等待或安全释放占用。
- 画面按位置处理前后遮挡与接地阴影，物品不会浮在界面上。
- 猫床可躺、猫爬架可登高、纸箱可进入、抓板可抓挠、小球可追逐；装饰明确只影响外观。
- 家具坐标、方向、占地、可用锚点和适用场景通过配置维护，不写死在背景图里；家具配置不包含被动照护属性加成。

#### M. 工程组织与平台适配

建议使用房间和护理区两个场景或等效空间容器，共享 UI 与逻辑状态。猫咪、食盆、家具、猫砂盆、商店卡片和工具面板使用可复用 Prefab。

保持三层：
- 表现层：节点、触控、动画、镜头、UI。
- 领域层：状态推进、行为资格、占用、交易和布局验证。
- 平台层：微信存储、生命周期、云调用和设备信息。

输入与坐标：
- 使用 Creator 的触摸事件及目标版本支持的坐标转换 API；不要混用屏幕、Canvas 和家具局部坐标。
- 记录活动触点，防止第二根手指触发重复消耗；正确处理 TOUCH_CANCEL、拖出区域与面板关闭。
- 场景退出时解除监听、调度和 Tween，不重复注册、重复补算或留下不可点击遮罩。
- 近景缩放后仍应正确命中猫咪部位，不能靠固定屏幕矩形判断。

表现资源：
- 使用 Sprite、Label、Button、ScrollView、ProgressBar、Mask、Animation 或 Tween 等目标版本能力。
- Cocos 编辑器可正确打开与导入资源，引用和元数据通过支持的流程生成，不手写虚假资源 UUID。
- 大资源按实际微信平台限制规划分包或 Asset Bundle，发布前核对当前限制，不引用未经验证的固定包体上限。
- 预加载当前必需资源；加载失败给重试和返回选项，不静默使用成功形态的空图。

#### N. 存档、经济与云端边界

- 正式小游戏使用微信存储适配，不用 localStorage。编辑器预览使用明确标记的预览适配，不伪装成微信云端。
- 持久化猫咪状态、最后更新时间、已结算喂食表现进度、猫砂余量及结团、库存、家具实例、布局、设置和护理会话。
- 保持云端已确认基线与有序待确认操作，关键交易不以客户端快照作为权威。
- 云端未接入时可以使用显式本地演示模式验证体验，但不得标注“云同步成功”或将演示状态写入正式存档。
- 每次购买、补砂、清理和布局提交带幂等标识；重复提交不重复扣款、交付或清理。
- 不逐帧上传拖动坐标；按有效区段或会话提交，并验证配置收益上限。
- 切后台保存必要进度，恢复按可信时间补算；不让系统时间回拨产生负衰减或额外收益。
- 如厕与玩家维护共用版本和占用校验，不出现负砂量或清理后被旧状态覆盖。
- 提交失败明确通知并恢复一致状态，不宽泛捕获后继续显示成功。
- 存档损坏需显示恢复选择，不静默清空存档。

#### O. 演示数据与验收

建议准备一只橘白猫“团团”，以及两个可由测试工具加入的猫咪，覆盖咪窝三只上限。演示初始数据可为喵币 1000、主粮 5 份、零食 3 份、猫砂 100 单位，盆内砂量 60/100、结团 2 个；均标为测试配置，不作为生产经济数值。

仅开发构建开放测试面板：设置饥饿、疲劳和紧张，新增结团、触发如厕、模拟经过时间、增加测试货币、解除测试冷却和重置测试存档。正式构建禁用，测试标识不能绕过服务端校验。

逐项验证并记录真实结果：
1. 商店购买主粮，喂食成功立即扣库存和增加饱食度，进食动画及其重启不重复结算；拒绝或成功前取消不扣。
2. 逗猫棒慢移、停顿时猫扑击，抓到有动作，疲劳能休息。
3. 不同位置和节奏抚摸产生不同回应，停止及时生效。
4. 梳毛或擦爪中途退出保留有效效果，不重复刷值。
5. 购买猫砂、清理、补砂、整盆更换形成闭环；库存不足不清空旧盆。
6. 三只猫不同时占盆，猫在两场景没有重复实体。
7. 买家具、入背包、摆放保存后画面可见，猫能实际使用。
8. 非法布局拒绝保存，取消布置恢复旧布局。
9. 洗澡、擦干、吹干及中断恢复正确，协助完成不触发感冒，冷却与已完成阶段不重复结算。
10. 重启后余额、布局、食盆和盆状态一致；重复操作不重复扣款。
11. 不同长宽比、刘海屏和微信胶囊下没有遮挡，触控坐标准确。
12. 编辑器预览、微信开发者工具和至少一台真机分别验证。未具备的环境标注“未验证”，不可拿浏览器预览替代微信真机结论。

#### P. 交付格式

具备实施授权并完成规划同步后，交付：
- 明确 Creator 版本的工程及启动、预览、微信构建说明。
- 房间与护理区场景、公共 Prefab、配置、脚本及正确资源引用。
- 各界面正常态、操作态、拒绝态、库存不足态和中断恢复的截图或录屏。
- 原创或授权资源来源、占位资源与正式替换清单。
- 实际执行的验证记录，以及阻塞项和未完成项。

若当前工具不能生成某项资源或不能运行 Creator，应说明限制并交付可检查的部分，不编造编辑器操作、构建成功或真机验证。

最终体验应是：玩家在一个可布置的咪窝里观察和陪伴猫咪，在相连护理区维护生活设施，猫咪会回应、会休息、会主动使用玩家购置的物品。不能用无响应按钮、仅改文本的假互动或网页界面冒充微信小游戏实现。

### 提示词结束