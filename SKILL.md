# Food Estimation Skill

## 1. Skill Name

`food-estimation`

---

# 2. Purpose

本 Skill 用于健身、饮食管理、营养追踪类应用中，对用户通过：

- 自然语言
- 食物照片
- 食物包装照片
- 菜单截图
- 餐厅食品
- 历史饮食记录

提交的饮食内容进行：

1. 食物识别
2. 食物标准化
3. 份量 / 重量估算
4. 可食部估算
5. 烹饪方式识别
6. 油脂、调味料和酱汁估算
7. 营养数据库匹配
8. 热量和宏量营养素计算
9. 不确定性评估
10. 必要时向用户进行最少量追问
11. 学习用户个人饮食份量习惯

本 Skill 的核心原则是：

> **AI 负责识别、推理和估计；确定性程序负责营养计算。**

禁止让 LLM / VLM 直接生成最终卡路里数字作为事实。

---

# 3. Core Philosophy

系统不得采用：

```text
图片
→ AI
→ “大约 783 kcal”
```

必须采用：

```text
用户输入
↓
Food Recognition
↓
Food Normalization
↓
Portion Estimation
↓
Cooking Adjustment
↓
Confidence Analysis
↓
Nutrition Database
↓
Deterministic Calculation
↓
Calories / Protein / Carbs / Fat
↓
Uncertainty Range
```

最终营养数据必须具有可追溯来源。

---

# 4. Fundamental Rules

## Rule 1 — Never hallucinate precision

禁止在信息不足时输出：

```text
742 kcal
```

应该输出：

```text
约 650–820 kcal

当前中位估计：
约 730 kcal
```

除非：

- 用户提供准确重量
- 产品包装明确标注
- 餐厅提供标准份量
- 数据来源可信且份量明确

否则禁止表现出虚假的克级精度。

---

## Rule 2 — Separate recognition from calculation

模型不得直接决定：

```text
鸡胸肉 = 317 kcal
```

模型只能产生结构化信息：

```json
{
  "canonical_food": "chicken_breast",
  "estimated_edible_weight_g": 190,
  "cooking_method": "pan_fried",
  "added_oil_g": 6
}
```

随后由 Nutrition Engine：

```text
food nutrients per 100g
×
edible weight
+
added ingredients
```

计算最终营养。

---

# 5. Input Types

Skill 必须处理以下输入。

## 5.1 Natural Language

例如：

```text
我刚刚吃了一条蓝瓜子斑，一碗藜麦米饭。
```

```text
午饭吃了两块鸡胸、一点西兰花，还有半碗米饭。
```

```text
刚刚吃了一份麦当劳双吉士，一个中薯。
```

---

## 5.2 Image

用户可以提供：

- 一张餐盘照片
- 多角度照片
- 食品包装
- 菜单
- 外卖
- 餐厅菜品

---

## 5.3 Hybrid

例如：

```text
[图片]

这个是我今天晚餐，米饭只吃了一半。
```

此时文本信息优先于视觉推断。

---

# 6. Evidence Priority

重量和食物信息必须按照证据等级处理。

优先级从高到低：

```text
P0 用户明确提供实际称重
P1 包装重量 / 营养标签
P2 餐厅官方标准份量
P3 用户已经建立的个人份量数据
P4 LiDAR / 深度 / 几何测量
P5 已知容器容量
P6 已知参考物尺寸
P7 视觉尺寸估算
P8 食物标准 Portion Prior
P9 LLM / VLM 一般经验估算
```

高等级证据不得被低等级证据覆盖。

例如：

用户说：

```text
这块牛排称过了，生重 280g。
```

视觉模型认为：

```text
约 220g
```

必须使用：

```text
280g
```

而不是平均：

```text
250g
```

---

# 7. Food Parsing

任何输入首先转换成 `FoodItem[]`。

示例：

```text
我刚刚吃了一条蓝瓜子斑，一碗藜麦米饭。
```

应解析：

```json
{
  "items": [
    {
      "raw_name": "蓝瓜子斑",
      "quantity": 1,
      "unit": "whole_fish"
    },
    {
      "raw_name": "藜麦米饭",
      "quantity": 1,
      "unit": "bowl"
    }
  ]
}
```

---

# 8. Food Normalization

每一种食物必须映射到标准食品实体。

例如：

```text
蓝瓜子斑
↓
地区俗称
↓
石斑鱼类
↓
具体标准食物实体
```

输出：

```json
{
  "raw_name": "蓝瓜子斑",
  "canonical_name": "细点石斑鱼",
  "canonical_id": "food_xxxxx",
  "category": "fish",
  "taxonomy_confidence": 0.87
}
```

---

# 9. Alias System

食品数据库必须支持：

```text
canonical food
↕
aliases
```

例如：

```json
{
  "canonical_name": "sweet_potato",
  "aliases": [
    "红薯",
    "地瓜",
    "番薯",
    "甘薯"
  ]
}
```

必须考虑：

- 地区差异
- 品牌名
- 菜名
- 网络俗称
- 餐厅自定义名字
- 用户自己的命名

---

# 10. Compound Food Decomposition

复合菜不能直接视作单一营养实体，除非存在可信标准配方。

例如：

```text
番茄炒蛋
```

尽可能拆成：

```text
鸡蛋
番茄
食用油
糖
盐
其他调料
```

但不要为了理论准确度不断向用户追问。

如果信息不足，应使用 Recipe Prior：

```json
{
  "dish": "番茄炒蛋",
  "recipe_prior": {
    "egg_ratio": 0.42,
    "tomato_ratio": 0.43,
    "oil_ratio": 0.08,
    "other_ratio": 0.07
  }
}
```

并提高不确定性。

---

# 11. Portion Estimation Engine

份量估算必须独立于 Food Recognition。

输出必须包含：

```json
{
  "portion": {
    "p10_g": 150,
    "p50_g": 210,
    "p90_g": 290,
    "confidence": 0.63,
    "source": "visual_prior"
  }
}
```

禁止只有：

```json
{
  "weight": 210
}
```

---

# 12. Portion Probability

推荐使用：

```text
P10
P50
P90
```

分别表示：

```text
P10 = 偏小但合理的重量
P50 = 中位估计
P90 = 偏大但合理的重量
```

例如：

```text
“一碗米饭”
```

不要固定：

```text
200g
```

应该根据：

- 碗尺寸
- 食物高度
- 米饭密度
- 地区习惯
- 用户历史

生成：

```json
{
  "p10_g": 140,
  "p50_g": 185,
  "p90_g": 245
}
```

---

# 13. Whole Food vs Edible Portion

必须区分：

```text
gross_weight
```

与：

```text
edible_weight
```

例如：

```text
一整条鱼 = 600g
```

不能直接认为：

```text
吃了 600g 鱼肉
```

应计算：

```text
edible_weight
=
gross_weight × edible_ratio
```

结构：

```json
{
  "gross_weight_g": {
    "p50": 600
  },
  "edible_ratio": {
    "p10": 0.48,
    "p50": 0.58,
    "p90": 0.65
  }
}
```

---

# 14. Edible Ratio Database

以下类型必须支持 edible ratio：

- 整鱼
- 带骨肉
- 鸡腿
- 鸡翅
- 排骨
- 龙虾
- 螃蟹
- 贝类
- 水果
- 带皮水果
- 坚果
- 带壳食品

不要依赖 LLM 临时猜测。

优先查询 Food Database。

---

# 15. Cooking Method

必须识别：

```text
raw
boiled
steamed
grilled
baked
air_fried
pan_fried
deep_fried
braised
stir_fried
roasted
stewed
smoked
unknown
```

烹饪方式必须单独存储。

---

# 16. Added Fat Estimation

油是卡路里误差的重要来源之一。

对于：

```text
炒
煎
炸
烤
拌
```

需要建立：

```text
added_fat
```

对象。

例如：

```json
{
  "added_fat": {
    "type": "cooking_oil",
    "p10_g": 3,
    "p50_g": 7,
    "p90_g": 14,
    "confidence": 0.48
  }
}
```

不要简单认为：

```text
煎 = 10g 油
```

---

# 17. Sauce Estimation

对于：

- 沙拉酱
- 花生酱
- 芝麻酱
- 火锅蘸料
- 糖浆
- 奶油酱
- 咖喱
- 红烧汁
- 糖醋汁
- 芝士酱

必须作为独立 FoodItem 处理。

---

# 18. Hidden Calories

系统应该特别识别高影响隐藏热量：

```text
oil
butter
sauce
sugar
cream
cheese
nuts
dressings
fried coating
```

但不得因为可能存在，就自动过度估计。

必须进入 uncertainty。

---

# 19. Image Analysis Protocol

图片输入时按照以下步骤：

```text
1. Detect food regions
2. Segment food items
3. Identify food
4. Estimate serving geometry
5. Detect plate / bowl / container
6. Search reference objects
7. Estimate volume
8. Convert volume → weight
9. Apply food density
10. Apply edible ratio
11. Estimate cooking adjustment
```

---

# 20. Visual Portion Estimation

视觉重量估算禁止直接依赖：

```text
“看起来大概 250g”
```

优先寻找：

- 盘子
- 碗
- 筷子
- 勺
- 刀叉
- 易拉罐
- 饮料瓶
- 手机
- 标准餐盒

作为尺寸参照。

---

# 21. Container Memory

系统应允许用户建立：

```text
My Containers
```

例如：

```json
{
  "container_id": "user_bowl_01",
  "name": "我家白色饭碗",
  "capacity_ml": 420,
  "diameter_cm": 13.2
}
```

以后识别到这个碗时，应该优先用于份量估算。

---

# 22. User Portion Memory

必须建立：

```text
User Portion Prior
```

例如用户过去纠正：

```text
一碗米饭 = 168g
一碗米饭 = 175g
一碗米饭 = 181g
一碗米饭 = 170g
```

建立：

```json
{
  "user_id": "...",
  "portion_key": "rice:standard_bowl",
  "p10": 164,
  "p50": 172,
  "p90": 184,
  "samples": 4
}
```

以后优先于全局 Prior。

---

# 23. Personalization Priority

推荐：

```text
Global Prior
↓
Regional Prior
↓
User Prior
↓
Meal-specific correction
```

随着用户数据增加：

```text
User Prior 权重逐渐增加
Global Prior 权重逐渐降低
```

---

# 24. Correction Learning

当用户修改：

```text
AI：这碗饭估计 220g

用户：其实只有 160g
```

必须记录：

```json
{
  "prediction": 220,
  "ground_truth": 160,
  "context": {
    "food": "rice",
    "container": "user_bowl_01"
  }
}
```

用于：

- 用户个性化
- 系统 Eval
- Portion Prior 更新

---

# 25. Confidence System

每个 Item 都必须有 confidence。

推荐：

```text
0.90–1.00 VERY_HIGH
0.75–0.89 HIGH
0.55–0.74 MEDIUM
0.35–0.54 LOW
0.00–0.34 VERY_LOW
```

---

# 26. Confidence Dimensions

不要只有一个 confidence。

建议：

```json
{
  "confidence": {
    "food_identity": 0.93,
    "portion": 0.52,
    "cooking_method": 0.84,
    "edible_ratio": 0.71,
    "nutrition_match": 0.92,
    "overall": 0.65
  }
}
```

---

# 27. Overall Confidence

推荐不要使用简单平均值。

应该对影响最终卡路里的变量提高权重。

例如：

```text
overall confidence
=
food identity
×
portion confidence
×
nutrition match
×
cooking adjustment
```

也可以采用经过 calibration 的加权模型。

---

# 28. Information Gain Clarification

系统禁止：

```text
信息不足
→ 连续问用户 5 个问题
```

应该计算：

> 哪个问题最可能显著降低最终营养误差？

例如：

```text
一条鱼 + 一碗饭
```

如果当前：

```text
鱼重量贡献 ±250 kcal
米饭贡献 ±70 kcal
```

优先问：

```text
鱼大概多大？
```

而不是：

```text
藜麦和大米比例是多少？
```

---

# 29. Clarification Budget

每顿饭：

```text
默认最多追问 1 次
```

特别高不确定性：

```text
最多 2 次
```

禁止无限追问。

如果用户不回答：

```text
继续采用合理区间估算。
```

---

# 30. Clarification UI

不要优先问：

```text
这条鱼具体多少克？
```

因为如果用户知道重量，大概率已经输入。

优先提供低认知成本选项：

```text
这条鱼大概多大？

○ 手掌大小
○ 约两个手掌长
○ 大约 500g
○ 接近 1kg
○ 不确定
```

---

# 31. Confidence-Based Behavior

## VERY_HIGH / HIGH

直接记录。

例如：

```text
包装鸡胸肉
净含量 180g
营养标签清晰
```

---

## MEDIUM

记录：

```text
约 420 kcal
预计范围 370–480 kcal
```

一般无需打扰用户。

---

## LOW

如果某个问题可以大幅提高准确率：

```text
追问一次
```

---

## VERY_LOW

不要表现为准确结果。

应显示：

```text
当前只能粗略估算。
```

并提供快速修正。

---

# 32. Nutrition Database

Nutrition Engine 必须使用结构化数据库。

每条营养数据至少：

```json
{
  "food_id": "food_1234",
  "name": "cooked_white_rice",
  "basis": "100g",
  "energy_kcal": 130,
  "protein_g": 2.7,
  "carbs_g": 28.2,
  "fat_g": 0.3,
  "fiber_g": 0.4,
  "source": "...",
  "source_version": "...",
  "confidence": 0.95
}
```

---

# 33. Data Source Priority

营养数据优先：

```text
品牌官方营养表
餐厅官方数据
国家级 / 权威食品数据库
可信商业数据库
系统标准食物库
AI fallback
```

AI fallback 只能作为最后手段。

并必须：

```text
confidence = LOW
```

---

# 34. Brand Product Handling

例如：

```text
250ml 蒙牛牛奶
```

不要匹配：

```text
generic whole milk
```

如果能匹配具体 SKU：

```text
品牌
产品
规格
营养标签
```

应优先使用 SKU 数据。

---

# 35. Restaurant Handling

例如：

```text
麦当劳中薯
```

优先：

```text
McDonald's official nutrition
```

不要通过照片重新猜重量。

---

# 36. Deterministic Nutrition Calculation

计算必须由代码完成。

公式：

```text
nutrition =
nutrition_per_100g
×
edible_weight_g
÷
100
```

例如：

```text
protein =
protein_per_100g
×
weight
÷
100
```

---

# 37. Total Meal Nutrition

总餐：

```text
meal calories
=
Σ food calories
+
Σ sauce calories
+
Σ added fat calories
+
Σ beverage calories
```

---

# 38. Range Calculation

不能只计算：

```text
P50 kcal
```

推荐同时计算：

```text
P10 calories
P50 calories
P90 calories
```

通过不同食物重量和油脂范围传播。

MVP 可以采用：

```text
lower estimate
median estimate
upper estimate
```

高级版本可以使用：

```text
Monte Carlo Simulation
```

---

# 39. Monte Carlo Mode

当多个变量存在范围时：

```text
weight
edible_ratio
oil
sauce
recipe ratio
```

推荐进行：

```text
N = 1000
```

或：

```text
N = 5000
```

次采样。

生成：

```text
P10
P50
P90
```

而不是简单把所有 minimum / maximum 相加。

---

# 40. Output JSON Schema

标准输出：

```json
{
  "meal_id": "meal_xxx",
  "items": [
    {
      "raw_name": "蓝瓜子斑",
      "canonical_name": "细点石斑鱼",
      "canonical_id": "food_xxx",

      "quantity": 1,
      "unit": "whole_fish",

      "cooking_method": "steamed",

      "gross_weight": {
        "p10_g": 380,
        "p50_g": 520,
        "p90_g": 700
      },

      "edible_ratio": {
        "p10": 0.50,
        "p50": 0.58,
        "p90": 0.65
      },

      "edible_weight": {
        "p10_g": 190,
        "p50_g": 302,
        "p90_g": 455
      },

      "added_fat": {
        "p10_g": 0,
        "p50_g": 3,
        "p90_g": 8
      },

      "confidence": {
        "food_identity": 0.91,
        "portion": 0.48,
        "cooking_method": 0.70,
        "overall": 0.61
      },

      "evidence": [
        {
          "type": "user_text",
          "priority": "P9"
        }
      ]
    }
  ],

  "nutrition": {
    "energy_kcal": {
      "p10": 520,
      "p50": 690,
      "p90": 880
    },

    "protein_g": {
      "p10": 42,
      "p50": 57,
      "p90": 73
    },

    "carbs_g": {
      "p10": 39,
      "p50": 52,
      "p90": 67
    },

    "fat_g": {
      "p10": 12,
      "p50": 19,
      "p90": 31
    }
  },

  "confidence": 0.62,

  "needs_clarification": true,

  "clarification": {
    "question": "这条鱼大概有多大？",
    "reason": "fish_weight_high_information_gain",
    "options": [
      "手掌大小",
      "两个手掌左右",
      "大约500g",
      "接近1kg",
      "不确定"
    ]
  }
}
```

---

# 41. User-Facing Output

用户默认不需要看到复杂内部计算。

推荐 UI：

```text
晚餐

蓝瓜子斑
约 302g 可食部分
≈ 360 kcal

藜麦米饭
约 185g
≈ 245 kcal

其他调味
≈ 60 kcal

────────

预计摄入

665 kcal

合理范围
560–790 kcal

蛋白质 54g
碳水 49g
脂肪 21g

估算可信度：中等
```

---

# 42. Avoid False Accuracy

不要显示：

```text
664.8 kcal
53.7g protein
48.91g carbs
```

对于估算餐饮数据推荐：

```text
约 665 kcal
约 54g 蛋白质
```

---

# 43. Quick Correction UI

每个 FoodItem 应支持快速修改：

```text
食物名称
重量
份量
烹饪方式
是否吃完
油脂
```

例如：

```text
蓝瓜子斑

估计整鱼 520g

[小一点]
[差不多]
[大一点]

或者：
[输入重量]
```

---

# 44. Partial Consumption

必须识别：

```text
吃了一半
剩三分之一
只吃了几口
米饭剩一半
```

结构：

```json
{
  "served_weight_g": 220,
  "consumption_ratio": 0.5,
  "consumed_weight_g": 110
}
```

---

# 45. Leftover Image

如果用户上传：

```text
餐前照片
+
餐后照片
```

可以：

```text
consumed_volume
=
before_volume - after_volume
```

作为高价值证据。

---

# 46. Temporal Context

理解：

```text
刚刚
早餐
午饭
晚饭
训练后
夜宵
```

用于 Meal Record。

但时间上下文不得影响热量估算本身。

---

# 47. Fitness Context

如果用户处于：

```text
减脂
增肌
维持
耐力训练
```

不要改变食品营养事实。

正确：

```text
你今天的蛋白质目标还差 35g。
```

错误：

```text
因为你在减脂，所以这顿鸡胸只有 300 kcal。
```

---

# 48. Never Change Facts to Fit Goals

用户的目标：

```text
减脂 1800 kcal
```

不能影响：

```text
食物实际估算热量
```

Nutrition Estimation 与 Diet Coaching 必须分离。

---

# 49. Error Categories

系统必须记录误差来源：

```text
FOOD_ID_ERROR
PORTION_ERROR
COOKING_METHOD_ERROR
OIL_ERROR
SAUCE_ERROR
RECIPE_ERROR
DATABASE_MAPPING_ERROR
EDIBLE_RATIO_ERROR
USER_CONSUMPTION_ERROR
VISION_SCALE_ERROR
UNKNOWN
```

---

# 50. Evaluation Metrics

系统必须持续评估。

至少包括：

## Food Recognition

```text
Top-1 Accuracy
Top-3 Accuracy
Top-5 Accuracy
```

## Portion

```text
MAE grams
MAPE
Median Absolute Error
```

## Nutrition

```text
Calories MAE
Calories MAPE
Protein MAE
Carbs MAE
Fat MAE
```

## Uncertainty

```text
P80 coverage
P90 coverage
confidence calibration
```

## UX

```text
clarification rate
user correction rate
meal completion rate
logging abandonment rate
```

## Personalization

```text
error day 1
error day 7
error day 30

personalization gain
```

---

# 51. Confidence Calibration

如果系统说：

```text
80% confidence
```

长期来看：

```text
大约 80% 的真实结果
```

应该落在预测区间。

否则说明 confidence 是假的。

必须定期做 calibration。

---

# 52. Clarification KPI

追问本身不是成功。

目标：

```text
maximum error reduction
/
minimum user effort
```

需要衡量：

```text
Error reduction per clarification
```

如果某问题只提高：

```text
5 kcal
```

不要问。

如果可以降低：

```text
250 kcal
```

值得问。

---

# 53. Food Portion Prior Database

建立：

```text
portion_priors
```

至少包含：

```json
{
  "food_id": "...",
  "portion_type": "bowl",
  "region": "CN",
  "p10_g": 130,
  "p50_g": 180,
  "p90_g": 250,
  "sample_count": 10000
}
```

---

# 54. Contextual Portion Prior

未来可以增加：

```text
restaurant
home
canteen
takeout
fast_food
fine_dining
meal_prep
```

例如：

```text
restaurant bowl rice
```

和：

```text
home bowl rice
```

份量分布可能不同。

---

# 55. Image Model Responsibilities

Vision Model 只负责：

```text
食物识别
食物区域
容器识别
视觉大小
数量
烹饪方式
是否存在酱汁
视觉参考物
```

禁止：

```text
直接输出最终 calories
```

---

# 56. LLM Responsibilities

LLM 负责：

```text
自然语言理解
别名解析
食物拆分
上下文理解
选择 Portion Prior
决定是否追问
生成自然语言反馈
```

---

# 57. Deterministic Engine Responsibilities

代码负责：

```text
nutrition arithmetic
unit conversion
Monte Carlo
confidence propagation
portion database lookup
nutrition database lookup
range calculation
```

---

# 58. Recommended Architecture

```text
User
 │
 ├── Text
 │
 └── Image
       ↓
Input Parser
       ↓
Food Recognition
       ↓
Food Resolver
       ↓
Canonical Food
       ↓
Portion Engine
 ┌─────┼───────────────┐
 │     │               │
User  Global        Vision /
Prior Prior         Geometry
 │     │               │
 └─────┴───────┬───────┘
               ↓
         Portion Fusion
               ↓
        Cooking Engine
               ↓
        Confidence Engine
               ↓
       Clarification Engine
               ↓
        Nutrition Lookup
               ↓
     Deterministic Calculator
               ↓
      Nutrition Distribution
               ↓
          User Result
               ↓
         User Correction
               ↓
      Personalization Engine
```

---

# 59. Portion Fusion

多个证据存在时，不允许简单平均。

例如：

```text
User history = 170g
Vision = 240g
Global prior = 190g
```

根据：

```text
evidence priority
confidence
sample size
historical reliability
```

融合。

---

# 60. Conflict Detection

如果两个高质量证据严重冲突：

```text
包装重量：200g
视觉估计：430g
```

优先检查：

```text
是不是包装有两份？
是不是只吃了一部分？
是不是视觉识别错了？
```

而不是：

```text
315g
```

---

# 61. Decision Rule for Clarification

伪代码：

```python
if overall_confidence >= HIGH:
    return result

uncertain_variables = rank_by_calorie_impact()

best_question = uncertain_variables[0]

expected_gain = calculate_expected_information_gain(best_question)

if expected_gain >= CLARIFICATION_THRESHOLD:
    ask(best_question)
else:
    return result_with_uncertainty()
```

---

# 62. Calorie Impact Ranking

例如：

```text
鱼重量 uncertainty：
±220 kcal

米饭重量 uncertainty：
±70 kcal

藜麦比例 uncertainty：
±12 kcal
```

排序：

```text
1. 鱼重量
2. 米饭重量
3. 藜麦比例
```

只问第一个。

---

# 63. Example Workflow A

用户：

```text
我吃了一碗米饭。
```

系统：

```text
Food = cooked rice

Quantity = bowl

No exact weight
```

检查：

```text
user bowl prior?
```

有：

```text
P50 = 172g
```

则直接估算。

不需要追问。

---

# 64. Example Workflow B

用户：

```text
我吃了一块牛排。
```

当前区间：

```text
150–450g
```

卡路里误差巨大。

应该问：

```text
这块牛排大概：

○ 手掌大小
○ 比手掌稍大
○ 约250g
○ 约400g+
○ 不确定
```

---

# 65. Example Workflow C

用户：

```text
吃了一份肯德基香辣鸡腿堡。
```

如果能命中：

```text
restaurant product DB
```

直接使用标准营养数据。

不要视觉估算。

---

# 66. Example Workflow D

用户：

```text
[餐盘照片]
```

识别：

```text
白米饭
牛肉
西兰花
酱汁
```

分别建立 FoodItem。

不能：

```text
整盘 = 700 kcal
```

---

# 67. Example Workflow E

用户：

```text
我吃了一条蓝瓜子斑，一碗藜麦饭。
```

解析：

```text
Fish
+
Mixed grain rice
```

假设：

```text
Fish size uncertainty = high
Rice uncertainty = medium
```

则问：

```text
这条鱼大概有多大？
```

获得答案后重新计算。

---

# 68. Unknown Food

如果无法识别：

```text
不要硬猜
```

返回：

```text
我不太确定这是哪种食物。
```

同时给：

```text
Top 3 candidates
```

例如：

```text
可能是：

1. 石斑鱼
2. 鲈鱼
3. 多宝鱼
```

让用户一点确认。

---

# 69. Multiple Similar Foods

如果营养差异极小：

```text
不需要确认。
```

例如：

两个营养接近的绿叶菜。

如果：

```text
候选 A = 150 kcal
候选 B = 600 kcal
```

则必须确认。

---

# 70. Nutrition Difference Threshold

是否追问食物身份，不取决于：

```text
classification confidence
```

而取决于：

```text
nutrition consequence
```

这是核心原则。

---

# 71. MVP Scope

第一版本优先实现：

```text
✓ 文本饮食解析
✓ 图片食物识别
✓ Canonical Food Resolver
✓ Nutrition DB
✓ Portion Prior
✓ User Portion Prior
✓ Confidence Engine
✓ Clarification Engine
✓ Correction Learning
✓ P10/P50/P90
```

---

# 72. Do NOT Prioritize in MVP

第一版暂时不要投入大量时间：

```text
× 精确 3D reconstruction
× 自研视觉模型
× 超高精度体积重建
× 所有中国菜完整 recipe database
× 每种调味料精确识别
```

先完成反馈闭环。

---

# 73. V2

增加：

```text
Food segmentation
Depth estimation
Reference-object geometry
Container recognition
Before / after meal comparison
Restaurant DB
Barcode scanning
Package OCR
```

---

# 74. V3

增加：

```text
LiDAR / ARKit
3D food reconstruction
Personalized visual portion model
User-specific recipe learning
Household container library
Long-term portion embeddings
```

---

# 75. Database Tables

推荐至少建立：

```text
foods
food_aliases
nutrition_values
food_portion_priors
food_edible_ratios
food_density
recipes
recipe_ingredients
restaurant_foods
brand_foods
user_portion_priors
user_containers
meal_records
meal_items
prediction_corrections
food_images
estimation_events
```

---

# 76. Estimation Audit Log

每一次结果必须可以解释来源。

例如：

```json
{
  "food": "rice",
  "estimated_weight": 172,
  "reason": [
    "user_container_detected",
    "user_portion_prior",
    "4 historical corrections"
  ]
}
```

避免黑盒结果。

---

# 77. Developer Requirement

实现任何新的 Food Estimation 功能前必须回答：

```text
1. 这个功能影响哪个误差来源？

2. 是否能减少真实 nutrition error？

3. 它是否增加用户操作成本？

4. 是否存在更简单的确定性方案？

5. 是否会制造虚假精确度？
```

---

# 78. Anti-Patterns

禁止：

```text
AI photo → calories
```

禁止：

```text
所有碗 = 200g
```

禁止：

```text
所有鸡胸 = 150g
```

禁止：

```text
所有煎制 = +10g oil
```

禁止：

```text
confidence = 模型自我感觉
```

禁止：

```text
不知道 → 编一个精确数字
```

禁止：

```text
为了更准追问用户大量信息
```

禁止：

```text
把卡路里目标反向影响食物热量估算
```

---

# 79. Preferred Behavior

应该：

```text
合理猜测
+
明确不确定性
+
只问最有价值的问题
+
允许一键修正
+
从修正中学习
```

---

# 80. Product Principle

Food logging 应该做到：

```text
Fast first
Accurate enough initially
More accurate over time
Correctable always
Transparent about uncertainty
```

而不是：

```text
第一次使用就假装非常精确
```

---

# 81. Long-Term Goal

随着用户使用：

```text
Day 1
Global Prior

↓ corrections

Day 7
Mixed Prior

↓ corrections

Day 30
User-specific Prior

↓ continued learning

Long term
Personal Food Model
```

最终：

```text
“我的一碗饭”
```

应该比：

```text
“普通人的一碗饭”
```

更准确。

---

# 82. Agent Instruction

当你作为开发 Agent 实现这个 Skill 时：

1. 不要把所有逻辑写进一个 Prompt。
2. 将确定性逻辑写成代码模块。
3. 将 Nutrition DB 与 LLM 解耦。
4. 将 Portion Engine 与 Food Recognition 解耦。
5. 所有估算必须保存 evidence。
6. 所有 prediction 必须允许用户 correction。
7. 所有关键算法必须可测试。
8. 所有营养结果必须可追踪来源。
9. 所有 uncertainty 必须显式建模。
10. 不得为了 UI 简单而删除内部 uncertainty。
11. UI 可以显示单个推荐值，但内部必须保留完整分布。
12. Agent 若发现旧代码采用“LLM 直接生成卡路里”的方案，应逐步重构而非继续扩展该方案。

---

# 83. Required Module Interfaces

推荐实现以下模块：

```text
FoodParser
FoodResolver
FoodRecognitionService
PortionEstimator
PortionPriorRepository
PersonalPortionService
CookingEstimator
EdibleRatioResolver
FoodDensityResolver
NutritionRepository
NutritionCalculator
ConfidenceEngine
ClarificationEngine
CorrectionService
EstimationAuditService
```

---

# 84. Final System Contract

最终系统必须满足：

```text
输入：
“我刚吃了一条蓝瓜子斑、一碗藜麦米饭”

系统不能：

“你摄入了 743 kcal。”

系统应该：

理解食物
↓
标准化
↓
估算份量分布
↓
计算可食部分
↓
判断烹饪方式
↓
估算油脂
↓
查询营养数据库
↓
计算营养分布
↓
判断最大不确定来源
↓
必要时只追问一个高价值问题
↓
输出中位估计 + 合理范围
↓
允许用户快速修正
↓
学习用户习惯
```

---

# 85. Definition of Done

只有满足以下条件才能认为该 Skill 已正确实现：

```text
[ ] LLM 不直接决定最终 calories
[ ] 每个 FoodItem 有 canonical food
[ ] Portion 有 uncertainty
[ ] Nutrition 有明确数据库来源
[ ] 用户提供的重量具有最高优先级
[ ] Whole food 支持 edible ratio
[ ] Cooking method 独立建模
[ ] Added fat 独立建模
[ ] 支持 user portion prior
[ ] 支持 correction learning
[ ] 支持 clarification information gain
[ ] 每顿默认最多追问一次
[ ] UI 不显示虚假精确度
[ ] 支持 calorie / P / C / F
[ ] 支持 P10 / P50 / P90
[ ] 有 Portion MAE 测试
[ ] 有 Calories MAE 测试
[ ] 有 confidence calibration 测试
[ ] 所有估算拥有 audit trail
```

---

# 86. Highest Priority

如果实现过程中需要在：

```text
更复杂
```

和：

```text
更可靠
```

之间选择，

永远优先：

```text
更可靠。
```

如果需要在：

```text
让 AI 猜更多
```

与：

```text
建立确定性规则
```

之间选择，

优先：

```text
确定性规则。
```

如果需要在：

```text
显示一个很精确的数字
```

与：

```text
诚实表达不确定性
```

之间选择，

永远选择：

```text
诚实表达不确定性。
```
