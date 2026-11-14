《咪头咪猫》微信小游戏 — 游戏设计文档 (GDD)
版本：v0.2（立项修订版）
作者：策划+程序联合产出
目标平台：微信小游戏（Cocos Creator 3.8+ / TypeScript）
受众：有1~3年前端/游戏开发经验的程序员
一、游戏概述
1.1 一句话描述
《咪头咪猫》​ = 猫咪主题连连看消除 + 云养猫养成，玩家通过闯关赚取喵币，在小喵屋里养猫，猫会饿、会脏、会生病，需要花钱照顾，形成"闯关→赚钱→养猫→生病→再闯关"的闭环。
1.2 核心卖点
卖点
	
说明


🧩 无限关卡
	
算法生成，每关都有新元素（冰块、锁链、翻转等）


🖼️ 丰富图片库
	
猫咪造型+动作图片，不局限于头像，匹配相同图片即可消除


🏠 云养猫
	
小喵屋实时互动，喂食/洗澡/玩耍，猫有情绪反馈


🏥 医疗系统
	
照顾不当会生病，带去医院治疗消耗喵币


💰 经济循环
	
喵币产出（闯关/英语单词）与消耗（商城/医院/咪窝）动态平衡


🐾 猫咪收留站
	
弃养/流浪猫机制，增加情感约束与二次收养机会
1.3 技术选型（细化）
层级
	
技术/工具
	
说明


游戏引擎​
	
Cocos Creator 3.8.4+
	
使用TypeScript开发，支持微信小游戏一键发布


语言​
	
TypeScript 5.x
	
强类型，便于大型项目管理


构建工具​
	
Cocos Creator内置构建
	
输出微信小游戏格式，自动生成game.js/game.json


后端服务​
	
微信云开发（TCB）
	
云函数（Node.js 16+）、云数据库（MongoDB兼容）、云存储


数据库​
	
微信云数据库
	
JSON文档型，集合：users、cats、cat_types、items、shelter


云函数运行时​
	
Node.js 16
	
用于登录、存档读写、分享回放、支付回调


本地存储​
	
wx.setStorageSync
	
缓存玩家数据，减少云调用频率


美术资源生成​
	
MidJourney v6 / DALL·E 3
	
生成猫咪卡通图，输出1024×1024后裁剪为128×128


图片处理​
	
Photoshop / remove.bg API
	
去背、统一尺寸、压缩为WebP


音效生成​
	
Suno AI / 免费音效库（freesound.org）
	
消除音效、喵叫、UI点击


背景音乐​
	
Suno AI
	
生成轻松可爱风BGM，输出MP3


版本控制​
	
Git + 微信开发者工具
	
代码管理，微信IDE用于真机调试和提审


数据分析​
	
微信小程序数据助手
	
留存、关卡通过率、付费转化


广告接入​
	
微信广告组件（激励视频/插屏）
	
wx.createRewardedVideoAd


CI/CD​
	
微信开发者工具CLI
	
命令行构建+上传，可配合GitHub Actions


UI框架​
	
Cocos Creator自带组件 + 自研Widget
	
无第三方UI库，减少包体
1.4 开发阶段规划
阶段
	
内容
	
说明


第一阶段（MVP）​
	
连连看核心 + 基础养成 + 经济系统 + 医院
	
可玩可留存的最小版本


第二阶段（迭代）​
	
猫咪收集系统 + 猫咪收留站 + 背英语单词赚喵币
	
社交/教育/情感深度
二、核心循环
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌──────────┐    喵币+道具     ┌──────────┐                     │
│   │  连连看   │───────────────▶│  商城/   │                     │
│   │  闯关    │                │  小喵屋  │                     │
│   └──────────┘                └──────────┘                     │
│        ▲                          │                              │
│        │                          ▼                              │
│   ┌──────────┐              ┌──────────┐    生病事件    ┌──────────┐
│   │  喵币    │◀────────────│  猫咪    │────────────▶│  宠物   │
│   │  不足    │              │  属性衰减 │              │  医院   │
│   └──────────┘              └──────────┘              └──────────┘
│        ▲                          │                              │
│        │        治疗消耗           │                              │
│        └──────────────────────────┘                              │
│                                                                  │
│   ┌──────────────┐          ┌──────────────┐                     │
│   │  背单词赚     │────────▶│  喵币补充     │                     │
│   │  喵币(二期)   │          └──────────────┘                     │
│   └──────────────┘                                               │
│                                                                  │
│   ┌──────────────┐          ┌──────────────┐                     │
│   │  弃养猫咪     │────────▶│  猫咪收留站   │──免费收养─────────▶ 养猫 │
│   └──────────────┘          └──────────────┘   (二期)            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
三、连连看玩法规则
3.1 基础规则
项目
	
说明


网格大小
	
初始 6×8（48格），随关卡扩大至 8×10


图片类型
	
丰富图片库：猫咪造型图（睡姿/坐姿/站立/跳跃/打哈欠/伸懒腰等）+ 猫咪动作图（扑球/追尾巴/洗脸/踩奶等），每关随机选6~8种


匹配规则
	
图片完全相同即可消除（不限于同品种，只要图片资源一致）


连线规则
	
两点之间用不超过2个拐点的直线连接，且路径上无遮挡


操作方式
	
点击选中第一张 → 点击第二张 → 判定连通 → 消除


失败条件
	
时间耗尽 或 步数用完（不同关卡不同）


胜利条件
	
消除所有方块
3.2 图片库设计
总图片数量：至少48张（12品种 × 4种造型/动作）
分类：
造型类：睡觉、坐姿、站立、趴着、蜷缩、伸懒腰
动作类：扑球、追尾巴、洗脸、踩奶、跳跃、打哈欠
主题类：戴帽子、穿衣服、节日装扮（后期）
匹配逻辑：图片资源路径完全一致即视为相同（不按品种匹配）
关卡配置：每关从图片库中随机抽取N对（N = 网格总数/2），确保成对出现
3.3 操作反馈
选中：图片放大+发光边框
消除：缩放消失 + 粒子特效 + 连线动画（闪电/爪印轨迹）
错误：摇头动画 + 错误音效
连击：连续消除无间隔 → 连击计数×1.2倍分数
3.4 道具系统（闯关中可用）
道具
	
功能
	
获取方式


提示
	
高亮一对可消除的图
	
每关赠送1次，商城购买


洗牌
	
重新排列剩余方块
	
每关赠送1次，商城购买


炸弹
	
消除周围3×3范围
	
商城购买


加时
	
增加30秒
	
商城购买
四、无限关卡设计思路
4.1 关卡生成算法
根据关卡ID读取配置（网格大小、图片种类数、特殊元素数量）
从图片库中随机抽取N对图片（N = 网格总格数/2）
随机填充网格
根据关卡ID叠加特殊元素（冰块、锁链、翻转、移动）
调用死局检测：确保至少存在一对可消除，否则重新排列
4.2 关卡难度曲线
阶段
	
关卡范围
	
网格
	
特殊元素
	
时间/步数
	
目标


新手期
	
1-10
	
6×8
	
无
	
时间180秒
	
教学+适应


成长期
	
11-30
	
6×8→7×8
	
冰块
	
时间150秒
	
引入障碍


进阶期
	
31-60
	
7×8→8×8
	
冰块+锁链
	
步数限制
	
策略思考


挑战期
	
61-100
	
8×8→8×10
	
全部元素
	
时间+步数双限
	
高难度


无尽期
	
100+
	
8×10
	
随机组合
	
动态难度
	
无限挑战
4.3 特殊元素详解
元素
	
外观
	
规则
	
消除方式


冰块
	
图片被冰层覆盖
	
需点击2次（第一次破冰，第二次消除）
	
正常连线或炸弹


锁链
	
图片被铁链缠绕
	
需先消除相邻方块打破锁链
	
相邻消除


翻转
	
图片朝下
	
点击翻转显示，再点第二张比对
	
记忆+匹配


移动
	
图片带箭头
	
每10秒随机交换位置
	
追踪+快速消除


炸弹
	
图片带引线
	
消除时炸毁3×3范围
	
主动触发
五、养成系统（小喵屋）
5.1 猫咪属性
interface Cat {
    id: string;
    catTypeId: string;       // 关联cat_types表
    name: string;            // 玩家自定义
    level: number;           // 1-100
    exp: number;             // 当前经验
    stats: {
        hunger: number;      // 饱食度 0-100
        cleanliness: number; // 清洁度 0-100
        happiness: number;   // 心情 0-100
        health: HealthState; // 健康状态
    };
    sickness: Sickness | null;
    isStray: boolean;        // 是否为流浪猫（弃养/收留站来的）
    lastUpdateTime: number;
    unlockTime: number;
    homeId: string;          // 所属咪窝ID
}
5.2 属性衰减机制
每5分钟触发一次衰减（基于离线时间计算）
饱食度 -3/5分钟，清洁度 -2/5分钟，心情 -1/5分钟
饱食度<20 且 清洁度<30 时，30%概率生病
衰减速度因品种而异（详见猫咪类型表）
5.3 咪窝系统
项目
	
说明


初始咪窝
	
玩家默认拥有1个咪窝，最多容纳3只猫​


额外咪窝
	
商城购买，每个咪窝最多3只猫


购买限制
	
无上限（但价格递增）


猫的移动
	
可在咪窝间转移猫咪
5.4 互动行为
行为
	
操作
	
效果
	
冷却时间


喂食
	
选择食物→点击猫
	
饱食度+（根据食物）
	
无


洗澡
	
点击浴缸→猫
	
清洁度+80
	
30分钟


吹干
	
洗澡后小游戏
	
清洁度+20，防感冒
	
仅洗澡后


玩耍
	
选择玩具→点击猫
	
心情+（根据玩具）
	
无


就医
	
点击医院→选猫
	
治疗疾病
	
治疗后24h免疫


弃养
	
猫咪详情→弃养按钮
	
猫进入收留站，玩家受处罚
	
24小时冷却
5.5 弃养机制（第二阶段）
弃养操作：玩家在猫咪详情页点击"弃养"，弹出确认框（二次确认）
处罚：
弃养后24小时内无法收养新猫
每次弃养记录计入玩家档案（弃养次数）
弃养3次以上：每日喵币产出减半（持续7天）
收留站：被弃养的猫进入收留站，标记为isStray: true
重新收养：玩家可在收留站无条件免费收养一只猫（不限品种），收养后isStray重置为false
5.6 小喵屋场景
背景：可更换（初始1个，商城购买更多）
家具：猫窝、沙发、猫爬架、喂食碗、玩具箱（装饰+属性加成）
猫咪动画：待机（呼吸/眨眼）、饥饿（肚子叫）、脏（灰尘粒子）、生病（趴地/流泪）、开心（跳跃/转圈）
六、猫咪类型表（cat_types）
第二阶段迭代加入猫咪收集系统后生效。此表为全局配置，每种猫咪品种一条记录。
字段
	
类型
	
说明


typeId
	
string
	
品种唯一ID


name
	
string
	
品种名称


rarity
	
enum
	
稀有度：N / R / SR / SSR


description
	
string
	
品种描述


hungerRate
	
number
	
饱食度衰减系数（默认1.0）


cleanRate
	
number
	
清洁度衰减系数（默认1.0）


happyRate
	
number
	
心情衰减系数（默认1.0）


sicknessResist
	
number
	
生病抗性（0-1，越高越不易生病）


coinBonus
	
number
	
喵币产出加成（0-0.5）


unlockLevel
	
number
	
解锁所需关卡


unlockMethod
	
enum
	
获取方式：levelUp / shelter / activity


imagePrefix
	
string
	
图片资源前缀（如 cat_ragdoll_）


isStray
	
boolean
	
是否为流浪品种（默认false）
品种列表（示例）
品种
	
稀有度
	
解锁关卡
	
衰减系数
	
特殊属性


橘猫
	
N
	
初始
	
1.2/1.0/1.0
	
无


英国短毛猫
	
N
	
初始
	
1.0/1.0/1.0
	
无


布偶猫
	
R
	
5
	
0.8/1.0/1.0
	
清洁度下降慢


暹罗猫
	
R
	
10
	
1.0/1.0/0.8
	
心情下降慢


苏格兰折耳猫
	
R
	
15
	
0.9/1.0/1.0
	
生病概率-10%


波斯猫
	
SR
	
25
	
0.8/0.8/0.8
	
全属性下降-20%


缅因猫
	
SR
	
35
	
1.0/1.0/1.0
	
喵币产出+10%


孟加拉豹猫
	
SR
	
50
	
1.2/1.0/1.0
	
连击分数+20%


加拿大无毛猫
	
SSR
	
80
	
1.5/1.5/1.0
	
清洁度需求极高


三花猫
	
SSR
	
活动
	
1.0/1.0/1.0
	
稀有度光效


异国短毛猫
	
SSR
	
100
	
0.7/0.7/0.7
	
全属性抗性
七、经济系统
7.1 货币类型
货币
	
获取方式
	
消耗途径
	
备注


喵币
	
闯关奖励、每日登录、分享、广告、背单词（二期）​
	
商城购物、医院治疗、咪窝购买
	
主货币


钻石
	
充值、成就奖励
	
购买稀有猫、特殊道具
	
付费货币（可选）
7.2 喵币产出设计
来源
	
单次数量
	
频率限制
	
阶段


关卡通关
	
50-200
	
每关1次
	
一期


连击奖励
	
10×连击数
	
每关上限100
	
一期


剩余步数/时间
	
1:1换算
	
每关上限50
	
一期


每日登录
	
100-300
	
每日1次
	
一期


分享给好友
	
50
	
每日3次
	
一期


看激励视频
	
200
	
每日5次
	
一期


背英语单词
	
10-50/词
	
每日上限500
	
二期​


成就奖励
	
500-2000
	
一次性
	
一期
7.3 喵币消耗设计
项目
	
价格范围
	
说明


猫粮（基础）
	
30
	
+20饱食度


猫粮（高级）
	
80
	
+50饱食度


猫零食
	
50
	
+15心情


猫玩具
	
100-300
	
+20~50心情


家具
	
200-1000
	
装饰+属性加成


咪窝
	
500-2000（递增）
	
每个最多3只猫


医院治疗
	
100-500
	
根据疾病类型


道具（闯关中）
	
50-200
	
提示/洗牌/炸弹
7.4 经济平衡验证
每日产出估算（活跃玩家，30分钟，一期）：
- 闯关10关 × 150平均 = 1500
- 连击奖励 = 100
- 剩余步数 = 50
- 每日登录 = 200
- 分享 = 150
- 广告 = 1000
- 合计 ≈ 3000喵币/日

每日消耗估算（养3只猫，1个咪窝）：
- 喂食3次×50 = 150
- 洗澡 = 0（冷却）
- 玩具 = 200
- 医院（概率）= 200
- 合计 ≈ 550喵币/日（正常）

二期加入背单词后，产出增加约500/日，用于支撑咪窝扩容和更多猫咪养成。
八、商城系统
8.1 商品分类
enum ShopCategory {
    Food = "food",
    Toy = "toy",
    Furniture = "furniture",
    Decoration = "decoration",
    Prop = "prop",
    Home = "home"           // 咪窝
}
8.2 商品列表（示例）
名称
	
分类
	
价格
	
效果


小鱼干
	
食物
	
30
	
饱食度+20


猫罐头
	
食物
	
80
	
饱食度+50


猫薄荷
	
玩具
	
50
	
心情+15


逗猫棒
	
玩具
	
100
	
心情+30


毛线球
	
玩具
	
150
	
心情+40


猫窝（基础）
	
家具
	
300
	
清洁度衰减-10%


猫爬架
	
家具
	
500
	
心情衰减-20%


咪窝（额外）
	
咪窝
	
500/1000/2000递增
	
容纳3只猫


提示道具
	
道具
	
50
	
闯关提示1次


洗牌道具
	
道具
	
100
	
闯关洗牌1次
九、宠物医院系统
9.1 疾病类型
疾病
	
触发条件
	
症状描述
	
治疗费用


感冒
	
洗澡后未吹干
	
打喷嚏、流鼻涕
	
100喵币


皮肤病
	
清洁度<20持续30分钟
	
掉毛、抓挠
	
200喵币


肠胃炎
	
饱食度<10或喂食过量
	
呕吐、腹泻
	
300喵币


骨折
	
连续3关失败
	
跛行、哭泣
	
500喵币


抑郁症
	
心情<10持续1小时
	
不互动、趴着
	
150喵币
9.2 治疗流程
猫咪生病 → 弹窗提醒 → 点击"去治疗" → 医院场景 → 选择生病的猫 → 查看诊断 → 确认治疗 → 扣除喵币 → 恢复健康
9.3 预防机制
洗澡后吹干小游戏：温度条控制，成功不生病，失败50%概率感冒
喂食过量警告：饱食度>100提示，连续3次过量→肠胃炎概率+50%
十、猫咪收留站（第二阶段）
10.1 功能说明
项目
	
说明


入口
	
主界面→小喵屋→收留站按钮


内容
	
显示所有被弃养的猫（品种、等级、弃养原因）


收养规则
	
玩家可免费收养1只（无条件），收养后进入玩家咪窝


弃养记录
	
每只被弃养的猫保留7天，超时后自动消失


处罚公示
	
弃养玩家的昵称（匿名化）和弃养次数在收留站顶部滚动显示
10.2 数据库（shelter集合）
{
  _id: String,
  catId: String,           // 关联cats表
  catTypeId: String,      // 品种
  abandonerOpenId: String, // 弃养者
  abandonTime: Number,     // 弃养时间戳
  reason: String,          // 弃养原因（玩家可选填）
  isAdopted: Boolean,     // 是否已被收养
  adopterOpenId: String | null,
  adoptTime: Number | null,
  expireTime: Number       // 7天后过期
}
十一、背单词赚喵币（第二阶段）
11.1 功能说明
项目
	
说明


入口
	
主界面→"背单词"按钮


模式
	
看英文选中文 / 看中文选英文 / 拼写


词库
	
小学→大学→雅思分级（按玩家选择）


奖励
	
每答对1词 +10~50喵币（按难度）


每日上限
	
500喵币


连对奖励
	
连续答对10题额外+100喵币
11.2 数据表（word_books集合）
{
  _id: String,
  level: String,           // 难度等级
  word: String,           // 英文单词
  meaning: String,        // 中文释义
  example: String,        // 例句
  audioUrl: String        // 发音URL（云存储）
}
十二、UI界面清单
12.1 场景列表
场景名
	
说明
	
关键元素


Loading
	
加载页
	
Logo、进度条


Home
	
主界面
	
开始按钮、设置、商城、小喵屋、排行榜、背单词入口（二期）


LevelSelect
	
关卡选择
	
关卡列表、星级、当前进度


Game
	
连连看主场景
	
网格、图片方块、分数、时间/步数、道具栏、暂停


Pause
	
暂停弹窗
	
继续、重开、设置、退出


Result
	
结算弹窗
	
星级、得分、喵币奖励、下一关


CatSelect
	
选猫界面（二期）
	
随机3只猫选1只


CatRoom
	
小喵屋
	
咪窝列表、猫咪、状态栏、互动按钮


CatDetail
	
猫咪详情
	
属性条、等级、弃养按钮


Shop
	
商城
	
分类标签、商品列表


Hospital
	
医院
	
生病猫列表、诊断、治疗


Shelter
	
收留站（二期）
	
弃养猫列表、收养按钮


WordBook
	
背单词（二期）
	
题目、选项、得分


Inventory
	
背包
	
道具列表


Settings
	
设置
	
音效、音乐、振动
12.2 通用组件
组件
	
说明


TopBar
	
喵币、钻石、设置


BottomNav
	
主页、闯关、小喵屋、商城、医院


CatAvatar
	
猫咪头像（状态角标）


ItemCard
	
商品卡片


StatBar
	
属性条（绿→黄→红）


Dialog
	
弹窗


Toast
	
轻提示
十三、数据存档设计（微信云开发）
13.1 数据库集合
// users 集合
{
  _openid: String,
  coin: Number,
  diamond: Number,
  currentLevel: Number,
  homes: Array,            // 咪窝列表（每个最多3只猫）
  inventory: Array,        // 背包
  abandonCount: Number,    // 弃养次数
  lastAbandonTime: Number, // 上次弃养时间
  wordBookLevel: String,   // 当前词库等级
  wordBookDailyEarn: Number, // 今日背单词已赚喵币
  settings: Object,
  createTime: Date,
  updateTime: Date
}

// cats 集合
{
  _id: String,
  userId: String,
  catTypeId: String,
  name: String,
  level: Number,
  exp: Number,
  stats: Object,
  sickness: String | null,
  isStray: Boolean,        // 是否为流浪猫（弃养/收留站来的），默认false
  homeId: String,          // 所属咪窝ID
  lastUpdateTime: Number,
  unlockTime: Number
}

// cat_types 集合（全局配置，每品种一条记录）
{
  typeId: String,
  name: String,
  rarity: String,          // N/R/SR/SSR
  description: String,
  hungerRate: Number,
  cleanRate: Number,
  happyRate: Number,
  sicknessResist: Number,
  coinBonus: Number,
  unlockLevel: Number,
  unlockMethod: String,
  imagePrefix: String,
  isStray: Boolean          // 是否为流浪品种，默认false
}

// shelter 集合
{
  _id: String,
  catId: String,
  catTypeId: String,
  abandonerOpenId: String,
  abandonTime: Number,
  reason: String,
  isAdopted: Boolean,
  adopterOpenId: String | null,
  adoptTime: Number | null,
  expireTime: Number
}

// shop_items 集合（静态配置）
{
  itemId: String,
  name: String,
  category: String,
  price: Number,
  currency: String,
  effectType: String,
  effectValue: Number,
  description: String,
  unlockLevel: Number
}

// word_books 集合（二期）
{
  _id: String,
  level: String,
  word: String,
  meaning: String,
  example: String,
  audioUrl: String
}
13.2 本地缓存策略
首次启动：从云拉取全量数据 → 存本地Storage
每次操作后：更新本地 → 异步同步云
离线时：仅用本地，上线后同步
十四、开发优先级
优先级
	
模块
	
阶段
	
预计工时


P0
	
连连看核心算法+渲染
	
一期
	
5天


P0
	
关卡生成器
	
一期
	
2天


P0
	
猫咪数据模型+衰减
	
一期
	
2天


P0
	
小喵屋UI+互动
	
一期
	
3天


P0
	
喵币系统+商城
	
一期
	
2天


P0
	
宠物医院逻辑
	
一期
	
1天


P0
	
微信云开发接入
	
一期
	
2天


P1
	
猫咪收集系统（cat_types）
	
二期
	
3天


P1
	
猫咪收留站
	
二期
	
2天


P1
	
背单词赚喵币
	
二期
	
4天


P1
	
咪窝扩容
	
二期
	
1天


P2
	
音效+特效
	
一期
	
2天


P2
	
分享+广告
	
一期
	
1天
十五、附录：AI协作Prompt模板
A. 生成关卡配置JSON
"生成JSON数组，包含关卡1到20的配置：levelId、rows、cols、imageCount（6-8）、iceCount、chainCount、flipCount、timeLimit、stepLimit、rewardCoin。难度递增。"
B. 生成猫咪类型表JSON
"生成12种猫咪品种的JSON数据，字段：typeId、name、rarity(N/R/SR/SSR)、description、hungerRate(0.7-1.5)、cleanRate(0.7-1.5)、happyRate(0.7-1.5)、sicknessResist(0-1)、coinBonus(0-0.5)、unlockLevel(1-100)、unlockMethod(levelUp/shelter/activity)、imagePrefix、isStray(false)。"
C. 生成商城商品JSON
"生成20个商城商品JSON：itemId、name、category(food/toy/furniture/decoration/prop/home)、price、currency(coin/diamond)、effectType、effectValue、description、unlockLevel。"
D. 生成单词表JSON
"生成50个英语单词JSON（小学难度）：word、meaning、example、level(elementary)。"
下一步行动：
确认GDD v0.2内容
开始P0模块：连连看核心算法+渲染
准备图片资源清单（48张猫咪造型/动作图）