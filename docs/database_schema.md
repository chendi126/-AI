# 傻毛宠物 - 数据库表结构设计

> 文档版本：v1.0  
> 撰写日期：2025-06-23  
> 数据库类型：MySQL 8.0 + Redis（缓存） + Elasticsearch（搜索）  
> 字符集：UTF-8mb4（支持 Emoji 和特殊字符）  
> 时区：Asia/Shanghai（UTC+8）

---

## 一、设计原则

### 1.1 核心设计原则

| 原则 | 说明 |
|------|------|
| **业务优先** | 表结构设计优先满足业务需求，不过度设计 |
| **可扩展性** | 预留扩展字段，支持未来业务迭代 |
| **性能导向** | 高频查询加索引，热点数据加缓存 |
| **数据安全** | 敏感数据加密存储，操作日志完整记录 |
| **软删除** | 所有业务表使用 `deleted_at` 软删除，保留数据完整性 |
| **分库分表预留** | 用户表、内容表预留分表键，支持未来水平扩展 |

### 1.2 命名规范

| 规范 | 说明 | 示例 |
|------|------|------|
| 表名 | 小写 + 下划线，复数形式 | `users`, `pet_profiles` |
| 字段名 | 小写 + 下划线 | `user_id`, `created_at` |
| 主键 | 统一使用 `id`，BIGINT UNSIGNED，自增 | `id` |
| 外键 | 表名 + `_id` | `user_id`, `breed_id` |
| 时间字段 | `created_at`, `updated_at`, `deleted_at` | 统一使用 TIMESTAMP |
| 状态字段 | 使用 TINYINT 或 ENUM | `status` |
| 布尔字段 | 使用 TINYINT(1)，0/1 | `is_verified`, `is_featured` |

### 1.3 技术架构

```
┌─────────────────────────────────────────────────────┐
│                      应用层                          │
├─────────────────────────────────────────────────────┤
│  业务服务层 (Node.js / Python / Go)                   │
├─────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Redis   │  │  MySQL   │  │  ES      │           │
│  │  缓存    │  │  主从库   │  │ 搜索引擎  │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│           │            │            │                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ 对象存储  │  │ 向量数据库 │  │ 消息队列  │           │
│  │  OSS    │  │  Milvus  │  │  RabbitMQ│           │
│  └──────────┘  └──────────┘  └──────────┘           │
└─────────────────────────────────────────────────────┘
```

---

## 二、ER 图概述

```
┌──────────┐       ┌─────────────┐       ┌──────────┐
│  users   │───────│ pet_profiles│───────│  breeds  │
│  用户表   │  1:N  │  宠物档案表  │  N:1  │ 品种表   │
└──────────┘       └─────────────┘       └──────────┘
     │                                       │
     │ 1:N                                   │ 1:N
     │                                       │
     ▼                                       ▼
┌──────────┐       ┌─────────────┐       ┌──────────┐
│  posts   │───────│post_comments │       │breed_diet│
│ 帖子表   │  1:N  │  评论表      │       │饮食指南表 │
└──────────┘       └─────────────┘       └──────────┘
     │                                       │
     │ 1:N                                   ├─ breed_health
     │                                       │   健康档案表
     ▼                                       ├─ breed_care
┌──────────┐                               │   护理指南表
│post_likes│                               ├─ breed_diseases
│点赞表    │                               │   疾病表
└──────────┘                               └──────────┘

┌──────────┐       ┌─────────────┐       ┌──────────┐
│ products │───────│order_items  │───────│  orders  │
│ 商品表   │  1:N  │ 订单商品表   │  N:1  │ 订单表   │
└──────────┘       └─────────────┘       └──────────┘
     │
     │ 1:N
     ▼
┌──────────┐
│product_  │
│breed_tags│
│品种标签表 │
└──────────┘
```

---

## 三、核心表结构详细设计

### 3.1 用户系统

---

#### 表 3.1.1：users（用户表）

```sql
CREATE TABLE `users` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT '用户ID',
    `union_id`              VARCHAR(64) NOT NULL DEFAULT '' COMMENT '全局唯一标识（跨平台统一）',
    `phone`                 VARCHAR(20) NOT NULL DEFAULT '' COMMENT '手机号（AES加密存储）',
    `phone_hash`            VARCHAR(64) NOT NULL COMMENT '手机号哈希（用于查询）',
    `email`                 VARCHAR(100) NOT NULL DEFAULT '' COMMENT '邮箱',
    `password`              VARCHAR(255) NOT NULL DEFAULT '' COMMENT '密码（bcrypt哈希）',
    `nickname`              VARCHAR(50) NOT NULL DEFAULT '' COMMENT '昵称',
    `avatar_url`            VARCHAR(500) NOT NULL DEFAULT '' COMMENT '头像URL',
    `gender`                TINYINT NOT NULL DEFAULT 0 COMMENT '性别：0未知 1男 2女',
    `birth_date`            DATE DEFAULT NULL COMMENT '生日',
    `province`              VARCHAR(50) NOT NULL DEFAULT '' COMMENT '省份',
    `city`                  VARCHAR(50) NOT NULL DEFAULT '' COMMENT '城市',
    `district`              VARCHAR(50) NOT NULL DEFAULT '' COMMENT '区县',
    `status`                TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1正常 2注销',
    `user_level`            TINYINT NOT NULL DEFAULT 1 COMMENT '用户等级：1-6',
    `total_points`          INT NOT NULL DEFAULT 0 COMMENT '总积分',
    `available_points`      INT NOT NULL DEFAULT 0 COMMENT '可用积分',
    `follower_count`        INT NOT NULL DEFAULT 0 COMMENT '粉丝数',
    `following_count`       INT NOT NULL DEFAULT 0 COMMENT '关注数',
    `post_count`            INT NOT NULL DEFAULT 0 COMMENT '发帖数',
    `liked_count`           INT NOT NULL DEFAULT 0 COMMENT '获赞数',
    `is_verified`           TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否认证：0否 1是',
    `verified_type`         TINYINT NOT NULL DEFAULT 0 COMMENT '认证类型：0无 1达人 2专家 3机构',
    `verified_info`         JSON DEFAULT NULL COMMENT '认证信息（头衔、领域、证书等）',
    `last_login_at`         TIMESTAMP DEFAULT NULL COMMENT '最后登录时间',
    `last_login_ip`           VARCHAR(50) NOT NULL DEFAULT '' COMMENT '最后登录IP',
    `register_source`       VARCHAR(50) NOT NULL DEFAULT '' COMMENT '注册来源：wechat/apple/phone',
    `register_channel`      VARCHAR(50) NOT NULL DEFAULT '' COMMENT '注册渠道：organic/invite/xiaohongshu',
    `app_version`           VARCHAR(20) NOT NULL DEFAULT '' COMMENT '最后使用App版本',
    `device_id`             VARCHAR(100) NOT NULL DEFAULT '' COMMENT '设备ID',
    `push_token`            VARCHAR(255) NOT NULL DEFAULT '' COMMENT '推送Token',
    `is_push_enabled`       TINYINT(1) NOT NULL DEFAULT 1 COMMENT '是否开启推送',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    `deleted_at`            TIMESTAMP NULL DEFAULT NULL COMMENT '删除时间（软删除）',
    
    UNIQUE KEY `uk_phone_hash` (`phone_hash`),
    UNIQUE KEY `uk_union_id` (`union_id`),
    INDEX `idx_status_level` (`status`, `user_level`),
    INDEX `idx_created_at` (`created_at`),
    INDEX `idx_register_channel` (`register_channel`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';
```

---

#### 表 3.1.2：user_auth（第三方登录表）

```sql
CREATE TABLE `user_auth` (
    `id`            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`       BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `auth_type`     VARCHAR(20) NOT NULL COMMENT '登录类型：wechat_miniprogram/wechat_app/apple',
    `auth_unionid`  VARCHAR(100) NOT NULL COMMENT '第三方平台UnionID',
    `auth_openid`   VARCHAR(100) NOT NULL COMMENT '第三方平台OpenID',
    `auth_nickname` VARCHAR(100) NOT NULL DEFAULT '' COMMENT '第三方昵称',
    `auth_avatar`   VARCHAR(500) NOT NULL DEFAULT '' COMMENT '第三方头像',
    `extra`         JSON DEFAULT NULL COMMENT '扩展信息',
    `created_at`    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_auth` (`auth_type`, `auth_unionid`),
    INDEX `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='第三方登录表';
```

---

#### 表 3.1.3：pet_profiles（宠物档案表）

```sql
CREATE TABLE `pet_profiles` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`               BIGINT UNSIGNED NOT NULL COMMENT '主人用户ID',
    `name`                  VARCHAR(50) NOT NULL COMMENT '宠物昵称',
    `avatar_url`            VARCHAR(500) NOT NULL DEFAULT '' COMMENT '宠物头像',
    `breed_id`              BIGINT UNSIGNED NOT NULL COMMENT '品种ID',
    `breed_name`            VARCHAR(100) NOT NULL DEFAULT '' COMMENT '品种名（冗余，方便展示）',
    `gender`                TINYINT NOT NULL DEFAULT 0 COMMENT '性别：0未知 1公 2母',
    `birth_date`            DATE DEFAULT NULL COMMENT '出生日期',
    `age_months`            INT NOT NULL DEFAULT 0 COMMENT '月龄（自动计算）',
    `weight_kg`             DECIMAL(5,2) DEFAULT NULL COMMENT '体重（公斤）',
    `is_neutered`           TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否绝育：0否 1是',
    `color`                 VARCHAR(50) NOT NULL DEFAULT '' COMMENT '毛色',
    `microchip_id`          VARCHAR(50) NOT NULL DEFAULT '' COMMENT '芯片号',
    `allergies`             JSON DEFAULT NULL COMMENT '过敏食物列表',
    `current_food`          VARCHAR(200) NOT NULL DEFAULT '' COMMENT '当前主粮品牌',
    `food_preferences`      JSON DEFAULT NULL COMMENT '饮食偏好',
    `personality_tags`      JSON DEFAULT NULL COMMENT '性格标签',
    `special_needs`         TEXT COMMENT '特殊需求备注',
    `status`                TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0已故 1健在 2送养',
    `is_default`            TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否默认宠物：0否 1是',
    `display_order`         INT NOT NULL DEFAULT 0 COMMENT '显示顺序',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at`            TIMESTAMP NULL DEFAULT NULL,
    
    INDEX `idx_user_id` (`user_id`, `display_order`),
    INDEX `idx_breed_id` (`breed_id`),
    INDEX `idx_birth_date` (`birth_date`),
    INDEX `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='宠物档案表';
```

---

#### 表 3.1.4：user_relations（用户关系表：关注）

```sql
CREATE TABLE `user_relations` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `follower_id`       BIGINT UNSIGNED NOT NULL COMMENT '关注者ID',
    `following_id`      BIGINT UNSIGNED NOT NULL COMMENT '被关注者ID',
    `relation_type`     TINYINT NOT NULL DEFAULT 1 COMMENT '关系类型：1关注 2好友 3拉黑',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_relation` (`follower_id`, `following_id`),
    INDEX `idx_follower` (`follower_id`, `created_at`),
    INDEX `idx_following` (`following_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户关系表';
```

---

#### 表 3.1.5：user_points_records（积分记录表）

```sql
CREATE TABLE `user_points_records` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `change_type`       TINYINT NOT NULL COMMENT '变动类型：1收入 2支出',
    `points`            INT NOT NULL COMMENT '变动积分（正数）',
    `balance`           INT NOT NULL COMMENT '变动后余额',
    `action_type`       VARCHAR(50) NOT NULL COMMENT '行为类型：signin/post/like/invite/consume等',
    `action_id`         BIGINT UNSIGNED DEFAULT NULL COMMENT '关联业务ID',
    `description`       VARCHAR(255) NOT NULL DEFAULT '' COMMENT '变动说明',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX `idx_user_id` (`user_id`, `created_at`),
    INDEX `idx_action` (`action_type`, `action_id`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='积分记录表';
```

---

### 3.2 品种知识库系统

---

#### 表 3.2.1：breed_categories（品种分类表）

```sql
CREATE TABLE `breed_categories` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `parent_id`         BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '父分类ID：0为顶级',
    `name`              VARCHAR(50) NOT NULL COMMENT '分类名称',
    `name_en`           VARCHAR(100) NOT NULL DEFAULT '' COMMENT '英文名称',
    `icon_url`          VARCHAR(500) NOT NULL DEFAULT '' COMMENT '分类图标',
    `description`       VARCHAR(255) NOT NULL DEFAULT '' COMMENT '分类描述',
    `sort_order`        INT NOT NULL DEFAULT 0 COMMENT '排序',
    `level`             TINYINT NOT NULL DEFAULT 1 COMMENT '层级：1大类 2中类 3小类',
    `status`            TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1正常',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_parent_id` (`parent_id`, `sort_order`),
    INDEX `idx_level` (`level`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='品种分类表';
```

**初始数据示例**：

| id | parent_id | name | level |
|----|-----------|------|-------|
| 1 | 0 | 犬类 | 1 |
| 2 | 0 | 猫类 | 1 |
| 3 | 0 | 异宠 | 1 |
| 4 | 1 | 大型犬 | 2 |
| 5 | 1 | 中型犬 | 2 |
| 6 | 1 | 小型犬 | 2 |
| 7 | 3 | 小宠 | 2 |
| 8 | 3 | 鸟类 | 2 |
| 9 | 3 | 爬宠 | 2 |
| 10 | 3 | 水族 | 2 |

---

#### 表 3.2.2：breeds（品种主表）

```sql
CREATE TABLE `breeds` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `category_id`           BIGINT UNSIGNED NOT NULL COMMENT '分类ID',
    `name`                  VARCHAR(100) NOT NULL COMMENT '品种中文名',
    `name_en`               VARCHAR(100) NOT NULL DEFAULT '' COMMENT '品种英文名',
    `name_alias`            JSON DEFAULT NULL COMMENT '别名列表（如金毛/金毛寻回犬）',
    `origin_country`        VARCHAR(100) NOT NULL DEFAULT '' COMMENT '原产地',
    `origin_country_en`     VARCHAR(100) NOT NULL DEFAULT '' COMMENT '原产地英文',
    `fci_group`             VARCHAR(50) NOT NULL DEFAULT '' COMMENT 'FCI分组',
    `cfa_group`             VARCHAR(50) NOT NULL DEFAULT '' COMMENT 'CFA分组',
    `life_span_min`         INT DEFAULT NULL COMMENT '寿命下限（年）',
    `life_span_max`         INT DEFAULT NULL COMMENT '寿命上限（年）',
    `weight_min_kg`         DECIMAL(5,2) DEFAULT NULL COMMENT '体重下限（kg）',
    `weight_max_kg`         DECIMAL(5,2) DEFAULT NULL COMMENT '体重上限（kg）',
    `height_min_cm`         DECIMAL(5,2) DEFAULT NULL COMMENT '肩高下限（cm）',
    `height_max_cm`         DECIMAL(5,2) DEFAULT NULL COMMENT '肩高上限（cm）',
    `size_type`             TINYINT DEFAULT NULL COMMENT '体型类型：1超小型 2小型 3中型 4大型 5巨型',
    `coat_type`             TINYINT DEFAULT NULL COMMENT '毛发类型：1短毛 2长毛 3卷毛 4硬毛 5无毛',
    `coat_colors`           JSON DEFAULT NULL COMMENT '常见毛色列表',
    `shedding_level`        TINYINT DEFAULT NULL COMMENT '掉毛程度：1-5',
    `grooming_level`        TINYINT DEFAULT NULL COMMENT '护理难度：1-5',
    `exercise_level`        TINYINT DEFAULT NULL COMMENT '运动量需求：1-5',
    `training_level`        TINYINT DEFAULT NULL COMMENT '训练难度：1-5（1最难）',
    `intelligence_level`    TINYINT DEFAULT NULL COMMENT '智商等级：1-5',
    `friendliness_child`    TINYINT DEFAULT NULL COMMENT '与儿童友好度：1-5',
    `friendliness_pet`      TINYINT DEFAULT NULL COMMENT '与其他宠物友好度：1-5',
    `friendliness_stranger` TINYINT DEFAULT NULL COMMENT '与陌生人友好度：1-5',
    `apartment_suitable`    TINYINT DEFAULT NULL COMMENT '公寓适合度：1-5',
    `noise_level`           TINYINT DEFAULT NULL COMMENT '吠叫程度：1-5',
    `care_difficulty`       TINYINT DEFAULT NULL COMMENT '饲养难度：1-5',
    `care_cost_level`       TINYINT DEFAULT NULL COMMENT '饲养成本：1-5',
    `suitable_for`          JSON DEFAULT NULL COMMENT '适合人群标签',
    `personality_tags`        JSON DEFAULT NULL COMMENT '性格标签',
    `brief_intro`           VARCHAR(500) NOT NULL DEFAULT '' COMMENT '一句话简介',
    `full_description`      TEXT COMMENT '品种详细描述',
    `history`               TEXT COMMENT '品种历史',
    `cover_images`          JSON DEFAULT NULL COMMENT '封面图片列表（URL数组）',
    `status`                TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0草稿 1已发布 2下架',
    `content_score`         INT NOT NULL DEFAULT 0 COMMENT '内容完整度评分（0-100）',
    `view_count`            INT NOT NULL DEFAULT 0 COMMENT '浏览次数',
    `favorite_count`        INT NOT NULL DEFAULT 0 COMMENT '收藏次数',
    `share_count`           INT NOT NULL DEFAULT 0 COMMENT '分享次数',
    `created_by`            BIGINT UNSIGNED NOT NULL COMMENT '创建者ID',
    `reviewed_by`           BIGINT UNSIGNED DEFAULT NULL COMMENT '审核者ID（兽医）',
    `reviewed_at`           TIMESTAMP NULL DEFAULT NULL COMMENT '审核时间',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at`            TIMESTAMP NULL DEFAULT NULL,
    
    INDEX `idx_category_id` (`category_id`, `status`),
    INDEX `idx_name` (`name`),
    INDEX `idx_size_type` (`size_type`, `status`),
    INDEX `idx_coat_type` (`coat_type`, `status`),
    INDEX `idx_care_difficulty` (`care_difficulty`, `status`),
    INDEX `idx_status_score` (`status`, `content_score`),
    FULLTEXT INDEX `ft_name` (`name`, `name_en`),
    FULLTEXT INDEX `ft_intro` (`brief_intro`, `full_description`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='品种主表';
```

---

#### 表 3.2.3：breed_diet（饮食指南表）

```sql
CREATE TABLE `breed_diet` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `breed_id`              BIGINT UNSIGNED NOT NULL COMMENT '品种ID',
    `age_stage`             TINYINT NOT NULL COMMENT '阶段：1幼年期 2成年期 3老年期 4全阶段',
    `weight_range`          VARCHAR(50) NOT NULL DEFAULT '' COMMENT '适用体重范围（如"5-10kg"）',
    `daily_food_amount`     VARCHAR(100) NOT NULL DEFAULT '' COMMENT '每日食量建议',
    `feeding_frequency`     VARCHAR(50) NOT NULL DEFAULT '' COMMENT '喂食频率（如"每天3-4次"）',
    `recommended_food_types` JSON DEFAULT NULL COMMENT '推荐食物类型',
    `recommended_foods`     JSON DEFAULT NULL COMMENT '推荐食物列表（商品ID关联）',
    `forbidden_foods`       JSON DEFAULT NULL COMMENT '禁忌食物列表（含毒性说明）',
    `special_dietary_needs` TEXT COMMENT '特殊饮食需求',
    `supplement_suggestions` JSON DEFAULT NULL COMMENT '营养补充建议',
    `water_needs`           VARCHAR(100) NOT NULL DEFAULT '' COMMENT '饮水量建议',
    `transition_tips`       TEXT COMMENT '换粮过渡建议',
    `created_by`            BIGINT UNSIGNED NOT NULL COMMENT '创建者ID',
    `reviewed_by`           BIGINT UNSIGNED DEFAULT NULL COMMENT '审核者ID',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_breed_age` (`breed_id`, `age_stage`),
    INDEX `idx_breed_id` (`breed_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='饮食指南表';
```

---

#### 表 3.2.4：breed_health（健康档案表）

```sql
CREATE TABLE `breed_health` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `breed_id`              BIGINT UNSIGNED NOT NULL COMMENT '品种ID',
    `health_category`       TINYINT NOT NULL COMMENT '健康分类：1遗传病 2常见疾病 3疫苗 4体检 5急救',
    `title`                 VARCHAR(200) NOT NULL COMMENT '标题',
    `content`               TEXT NOT NULL COMMENT '内容',
    `severity_level`        TINYINT DEFAULT NULL COMMENT '严重程度：1轻微 2中等 3严重 4紧急',
    `prevention_tips`       TEXT COMMENT '预防建议',
    `symptoms`              JSON DEFAULT NULL COMMENT '症状列表',
    `treatment_overview`    TEXT COMMENT '治疗概述（不给出具体用药剂量）',
    `when_to_see_vet`       TEXT COMMENT '何时需要就医',
    `related_vaccines`      JSON DEFAULT NULL COMMENT '相关疫苗（疫苗ID数组）',
    `related_products`      JSON DEFAULT NULL COMMENT '相关推荐商品',
    `sort_order`            INT NOT NULL DEFAULT 0 COMMENT '排序',
    `created_by`            BIGINT UNSIGNED NOT NULL COMMENT '创建者ID',
    `reviewed_by`           BIGINT UNSIGNED DEFAULT NULL COMMENT '审核者ID（兽医）',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_breed_category` (`breed_id`, `health_category`),
    INDEX `idx_severity` (`severity_level`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='健康档案表';
```

---

#### 表 3.2.5：breed_care（护理指南表）

```sql
CREATE TABLE `breed_care` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `breed_id`          BIGINT UNSIGNED NOT NULL COMMENT '品种ID',
    `care_category`     TINYINT NOT NULL COMMENT '护理分类：1毛发护理 2运动锻炼 3训练 4居家环境 5气候适应 6社交',
    `title`             VARCHAR(200) NOT NULL COMMENT '标题',
    `content`           TEXT NOT NULL COMMENT '内容',
    `frequency`         VARCHAR(100) NOT NULL DEFAULT '' COMMENT '频率建议（如"每天1次""每周2-3次"）',
    `tools_needed`      JSON DEFAULT NULL COMMENT '所需工具列表',
    `difficulty_level`  TINYINT DEFAULT NULL COMMENT '难度：1-5',
    `time_needed`       VARCHAR(50) NOT NULL DEFAULT '' COMMENT '所需时间',
    `tips`              TEXT COMMENT '小贴士',
    `related_videos`    JSON DEFAULT NULL COMMENT '相关视频链接',
    `sort_order`        INT NOT NULL DEFAULT 0 COMMENT '排序',
    `created_by`        BIGINT UNSIGNED NOT NULL COMMENT '创建者ID',
    `reviewed_by`       BIGINT UNSIGNED DEFAULT NULL COMMENT '审核者ID',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_breed_category` (`breed_id`, `care_category`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='护理指南表';
```

---

### 3.3 内容社区系统

---

#### 表 3.3.1：posts（帖子表）

```sql
CREATE TABLE `posts` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '作者ID',
    `author_nickname`   VARCHAR(50) NOT NULL DEFAULT '' COMMENT '作者昵称（冗余）',
    `author_avatar`     VARCHAR(500) NOT NULL DEFAULT '' COMMENT '作者头像（冗余）',
    `author_level`      TINYINT NOT NULL DEFAULT 1 COMMENT '作者等级（冗余）',
    `author_verified`   TINYINT(1) NOT NULL DEFAULT 0 COMMENT '作者是否认证（冗余）',
    `post_type`         TINYINT NOT NULL DEFAULT 1 COMMENT '类型：1图文 2视频 3图文+视频',
    `title`             VARCHAR(200) NOT NULL DEFAULT '' COMMENT '标题',
    `content`           TEXT COMMENT '正文内容',
    `content_plain`     TEXT COMMENT '纯文本内容（用于搜索）',
    `cover_url`         VARCHAR(500) NOT NULL DEFAULT '' COMMENT '封面图/视频封面',
    `media_count`       TINYINT NOT NULL DEFAULT 0 COMMENT '媒体数量',
    `breed_tags`        JSON DEFAULT NULL COMMENT '关联品种标签（品种ID数组）',
    `topic_tags`        JSON DEFAULT NULL COMMENT '话题标签',
    `location`          VARCHAR(200) NOT NULL DEFAULT '' COMMENT '位置信息',
    `location_poi`      VARCHAR(100) NOT NULL DEFAULT '' COMMENT 'POI ID',
    `latitude`          DECIMAL(10,7) DEFAULT NULL COMMENT '纬度',
    `longitude`         DECIMAL(10,7) DEFAULT NULL COMMENT '经度',
    `visibility`        TINYINT NOT NULL DEFAULT 1 COMMENT '可见范围：1公开 2粉丝 3私密',
    `status`            TINYINT NOT NULL DEFAULT 0 COMMENT '状态：0审核中 1已发布 2拒绝 3下架',
    `reject_reason`     VARCHAR(255) NOT NULL DEFAULT '' COMMENT '拒绝原因',
    `is_featured`       TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否精选：0否 1是',
    `is_pinned`         TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否置顶：0否 1是',
    `view_count`        INT NOT NULL DEFAULT 0 COMMENT '浏览数',
    `like_count`        INT NOT NULL DEFAULT 0 COMMENT '点赞数',
    `comment_count`     INT NOT NULL DEFAULT 0 COMMENT '评论数',
    `share_count`       INT NOT NULL DEFAULT 0 COMMENT '分享数',
    `collect_count`     INT NOT NULL DEFAULT 0 COMMENT '收藏数',
    `hot_score`         INT NOT NULL DEFAULT 0 COMMENT '热度分（用于排序）',
    `published_at`      TIMESTAMP NULL DEFAULT NULL COMMENT '发布时间',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at`        TIMESTAMP NULL DEFAULT NULL,
    
    INDEX `idx_user_id` (`user_id`, `created_at`),
    INDEX `idx_status_published` (`status`, `published_at`),
    INDEX `idx_breed_tags` ((CAST(`breed_tags` AS CHAR(255) ARRAY))), -- 需MySQL 8.0.17+多值索引
    INDEX `idx_hot_score` (`status`, `hot_score`),
    INDEX `idx_is_featured` (`is_featured`, `status`),
    FULLTEXT INDEX `ft_content` (`title`, `content_plain`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='帖子表';
```

> **分表建议**：当帖子量超过 1000 万时，按 `user_id` 取模分 16 张表。

---

#### 表 3.3.2：post_media（帖子媒体表）

```sql
CREATE TABLE `post_media` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `post_id`           BIGINT UNSIGNED NOT NULL COMMENT '帖子ID',
    `media_type`        TINYINT NOT NULL COMMENT '媒体类型：1图片 2视频',
    `media_url`         VARCHAR(500) NOT NULL COMMENT '媒体URL',
    `thumbnail_url`     VARCHAR(500) NOT NULL DEFAULT '' COMMENT '缩略图URL',
    `width`             INT DEFAULT NULL COMMENT '宽度',
    `height`            INT DEFAULT NULL COMMENT '高度',
    `duration`          INT DEFAULT NULL COMMENT '视频时长（秒）',
    `file_size`         INT DEFAULT NULL COMMENT '文件大小（字节）',
    `sort_order`        INT NOT NULL DEFAULT 0 COMMENT '排序',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX `idx_post_id` (`post_id`, `sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='帖子媒体表';
```

---

#### 表 3.3.3：post_comments（评论表）

```sql
CREATE TABLE `post_comments` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `post_id`           BIGINT UNSIGNED NOT NULL COMMENT '帖子ID',
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '评论者ID',
    `user_nickname`     VARCHAR(50) NOT NULL DEFAULT '' COMMENT '评论者昵称（冗余）',
    `user_avatar`       VARCHAR(500) NOT NULL DEFAULT '' COMMENT '评论者头像（冗余）',
    `parent_id`         BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '父评论ID：0为顶级评论',
    `root_id`           BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '根评论ID（用于快速查询所有回复）',
    `content`           TEXT NOT NULL COMMENT '评论内容',
    `like_count`        INT NOT NULL DEFAULT 0 COMMENT '点赞数',
    `is_hot`            TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否热评：0否 1是',
    `status`            TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0隐藏 1显示 2删除',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at`        TIMESTAMP NULL DEFAULT NULL,
    
    INDEX `idx_post_id` (`post_id`, `created_at`),
    INDEX `idx_root_id` (`root_id`, `created_at`),
    INDEX `idx_parent_id` (`parent_id`),
    INDEX `idx_user_id` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='评论表';
```

---

#### 表 3.3.4：post_likes（点赞表）

```sql
CREATE TABLE `post_likes` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `post_id`           BIGINT UNSIGNED NOT NULL COMMENT '帖子ID',
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_post_user` (`post_id`, `user_id`),
    INDEX `idx_user_id` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='点赞表';
```

---

#### 表 3.3.5：user_collections（用户收藏表）

```sql
CREATE TABLE `user_collections` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `collect_type`      TINYINT NOT NULL COMMENT '收藏类型：1帖子 2品种 3商品',
    `target_id`         BIGINT UNSIGNED NOT NULL COMMENT '目标ID',
    `folder_id`         BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '收藏夹ID：0为默认',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_user_type_target` (`user_id`, `collect_type`, `target_id`),
    INDEX `idx_user_folder` (`user_id`, `folder_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户收藏表';
```

---

### 3.4 商城系统

---

#### 表 3.4.1：product_categories（商品分类表）

```sql
CREATE TABLE `product_categories` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `parent_id`         BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '父分类ID',
    `name`              VARCHAR(50) NOT NULL COMMENT '分类名称',
    `icon_url`          VARCHAR(500) NOT NULL DEFAULT '' COMMENT '分类图标',
    `description`       VARCHAR(255) NOT NULL DEFAULT '' COMMENT '描述',
    `sort_order`        INT NOT NULL DEFAULT 0 COMMENT '排序',
    `level`             TINYINT NOT NULL DEFAULT 1 COMMENT '层级',
    `status`            TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1正常',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_parent_id` (`parent_id`, `sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='商品分类表';
```

---

#### 表 3.4.2：products（商品表）

```sql
CREATE TABLE `products` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `category_id`           BIGINT UNSIGNED NOT NULL COMMENT '分类ID',
    `spu_id`                BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'SPU ID（同系列商品）',
    `name`                  VARCHAR(200) NOT NULL COMMENT '商品名称',
    `subtitle`              VARCHAR(255) NOT NULL DEFAULT '' COMMENT '副标题',
    `description`           TEXT COMMENT '商品描述',
    `main_image`            VARCHAR(500) NOT NULL COMMENT '主图',
    `gallery_images`        JSON DEFAULT NULL COMMENT '轮播图列表',
    `detail_images`         JSON DEFAULT NULL COMMENT '详情图列表',
    `brand_id`              BIGINT UNSIGNED DEFAULT NULL COMMENT '品牌ID',
    `brand_name`            VARCHAR(100) NOT NULL DEFAULT '' COMMENT '品牌名（冗余）',
    `source_type`           TINYINT NOT NULL DEFAULT 1 COMMENT '来源：1自营 2联盟电商 3第三方',
    `source_url`            VARCHAR(500) NOT NULL DEFAULT '' COMMENT '联盟商品原始链接',
    `source_platform`       VARCHAR(50) NOT NULL DEFAULT '' COMMENT '来源平台：taobao/jd',
    `source_item_id`        VARCHAR(100) NOT NULL DEFAULT '' COMMENT '来源平台商品ID',
    `price`                 DECIMAL(10,2) NOT NULL COMMENT '售价',
    `original_price`        DECIMAL(10,2) DEFAULT NULL COMMENT '原价',
    `cost_price`            DECIMAL(10,2) DEFAULT NULL COMMENT '成本价',
    `commission_rate`       DECIMAL(5,2) DEFAULT NULL COMMENT '佣金率（%）',
    `commission_amount`     DECIMAL(10,2) DEFAULT NULL COMMENT '佣金金额',
    `stock`                 INT NOT NULL DEFAULT 0 COMMENT '库存（自营有效）',
    `unit`                  VARCHAR(20) NOT NULL DEFAULT '件' COMMENT '单位',
    `specifications`        JSON DEFAULT NULL COMMENT '规格参数（重量/尺寸等）',
    `suitable_breeds`       JSON DEFAULT NULL COMMENT '适用品种ID列表',
    `suitable_ages`         JSON DEFAULT NULL COMMENT '适用年龄段：1幼年期 2成年期 3老年期',
    `suitable_sizes`        JSON DEFAULT NULL COMMENT '适用体型：1-5',
    `suitable_pet_types`    JSON DEFAULT NULL COMMENT '适用宠物类型：dog/cat/small_pet等',
    `labels`                JSON DEFAULT NULL COMMENT '商品标签：hot/new/organic等',
    `rating`                DECIMAL(2,1) NOT NULL DEFAULT 5.0 COMMENT '评分',
    `rating_count`          INT NOT NULL DEFAULT 0 COMMENT '评价数',
    `sold_count`            INT NOT NULL DEFAULT 0 COMMENT '销量',
    `status`                TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0下架 1上架 2售罄',
    `sort_order`            INT NOT NULL DEFAULT 0 COMMENT '排序',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at`            TIMESTAMP NULL DEFAULT NULL,
    
    INDEX `idx_category_id` (`category_id`, `status`),
    INDEX `idx_source` (`source_platform`, `source_item_id`),
    INDEX `idx_price` (`status`, `price`),
    INDEX `idx_sold` (`status`, `sold_count`),
    INDEX `idx_suitable_breeds` ((CAST(`suitable_breeds` AS CHAR(255) ARRAY))),
    FULLTEXT INDEX `ft_name` (`name`, `subtitle`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='商品表';
```

---

#### 表 3.4.3：product_breed_tags（商品品种关联表）

```sql
CREATE TABLE `product_breed_tags` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `product_id`        BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `breed_id`          BIGINT UNSIGNED NOT NULL COMMENT '品种ID',
    `recommend_level`   TINYINT NOT NULL DEFAULT 3 COMMENT '推荐等级：1强烈 2推荐 3一般 4不推荐',
    `recommend_reason`  VARCHAR(255) NOT NULL DEFAULT '' COMMENT '推荐理由',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_product_breed` (`product_id`, `breed_id`),
    INDEX `idx_breed_id` (`breed_id`, `recommend_level`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='商品品种关联表';
```

---

#### 表 3.4.4：orders（订单表）

```sql
CREATE TABLE `orders` (
    `id`                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `order_no`              VARCHAR(50) NOT NULL COMMENT '订单编号',
    `user_id`               BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `order_type`            TINYINT NOT NULL DEFAULT 1 COMMENT '订单类型：1自营 2联盟',
    `status`                TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1待付款 2已付款 3已发货 4已完成 5已取消 6退款中 7已退款',
    `total_amount`          DECIMAL(10,2) NOT NULL COMMENT '订单总金额',
    `discount_amount`       DECIMAL(10,2) NOT NULL DEFAULT 0 COMMENT '优惠金额',
    `pay_amount`            DECIMAL(10,2) NOT NULL COMMENT '实付金额',
    `pay_method`            TINYINT DEFAULT NULL COMMENT '支付方式：1微信 2支付宝',
    `pay_time`              TIMESTAMP NULL DEFAULT NULL COMMENT '支付时间',
    `pay_no`                VARCHAR(100) NOT NULL DEFAULT '' COMMENT '第三方支付单号',
    `receiver_name`         VARCHAR(50) NOT NULL DEFAULT '' COMMENT '收货人',
    `receiver_phone`        VARCHAR(20) NOT NULL DEFAULT '' COMMENT '收货电话',
    `receiver_address`      VARCHAR(500) NOT NULL DEFAULT '' COMMENT '收货地址',
    `logistics_company`     VARCHAR(50) NOT NULL DEFAULT '' COMMENT '物流公司',
    `logistics_no`          VARCHAR(100) NOT NULL DEFAULT '' COMMENT '物流单号',
    `deliver_time`          TIMESTAMP NULL DEFAULT NULL COMMENT '发货时间',
    `receive_time`          TIMESTAMP NULL DEFAULT NULL COMMENT '收货时间',
    `remark`                VARCHAR(500) NOT NULL DEFAULT '' COMMENT '用户备注',
    `source`                VARCHAR(50) NOT NULL DEFAULT '' COMMENT '订单来源',
    `commission_total`      DECIMAL(10,2) NOT NULL DEFAULT 0 COMMENT '佣金总额',
    `created_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at`            TIMESTAMP NULL DEFAULT NULL,
    
    UNIQUE KEY `uk_order_no` (`order_no`),
    INDEX `idx_user_id` (`user_id`, `created_at`),
    INDEX `idx_status` (`status`, `created_at`),
    INDEX `idx_pay_time` (`pay_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单表';
```

---

#### 表 3.4.5：order_items（订单商品表）

```sql
CREATE TABLE `order_items` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `order_id`          BIGINT UNSIGNED NOT NULL COMMENT '订单ID',
    `product_id`        BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `product_name`      VARCHAR(200) NOT NULL COMMENT '商品名称（快照）',
    `product_image`     VARCHAR(500) NOT NULL COMMENT '商品图片（快照）',
    `spec_info`         JSON DEFAULT NULL COMMENT '规格信息（快照）',
    `quantity`          INT NOT NULL COMMENT '数量',
    `unit_price`        DECIMAL(10,2) NOT NULL COMMENT '单价',
    `total_price`       DECIMAL(10,2) NOT NULL COMMENT '小计',
    `commission`        DECIMAL(10,2) NOT NULL DEFAULT 0 COMMENT '佣金',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX `idx_order_id` (`order_id`),
    INDEX `idx_product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单商品表';
```

---

### 3.5 AI 系统

---

#### 表 3.5.1：ai_conversations（AI 对话表）

```sql
CREATE TABLE `ai_conversations` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `session_id`        VARCHAR(64) NOT NULL COMMENT '会话ID',
    `conversation_type` TINYINT NOT NULL DEFAULT 1 COMMENT '对话类型：1普通问答 2商品推荐 3品种咨询',
    `status`            TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1进行中 2已完成 3超时',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_user_session` (`user_id`, `session_id`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='AI对话表';
```

---

#### 表 3.5.2：ai_messages（AI 消息表）

```sql
CREATE TABLE `ai_messages` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `conversation_id`   BIGINT UNSIGNED NOT NULL COMMENT '对话ID',
    `role`              TINYINT NOT NULL COMMENT '角色：1用户 2AI 3系统',
    `content`           TEXT NOT NULL COMMENT '消息内容',
    `content_type`      TINYINT NOT NULL DEFAULT 1 COMMENT '内容类型：1文本 2图片 3商品卡片 4品种卡片',
    `extra_data`        JSON DEFAULT NULL COMMENT '附加数据（商品ID、品种ID等）',
    `tokens_used`       INT DEFAULT NULL COMMENT '使用的Token数',
    `model`             VARCHAR(50) NOT NULL DEFAULT '' COMMENT '使用的大模型',
    `latency_ms`        INT DEFAULT NULL COMMENT '响应延迟（毫秒）',
    `user_feedback`     TINYINT DEFAULT NULL COMMENT '用户反馈：1满意 2不满意',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX `idx_conversation_id` (`conversation_id`, `created_at`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='AI消息表';
```

---

#### 表 3.5.3：knowledge_chunks（知识库分片表）

```sql
CREATE TABLE `knowledge_chunks` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `chunk_id`          VARCHAR(64) NOT NULL COMMENT '向量ID（对应向量数据库）',
    `source_type`       TINYINT NOT NULL COMMENT '来源类型：1品种百科 2饮食指南 3健康档案 4护理指南 5UGC帖子',
    `source_id`         BIGINT UNSIGNED NOT NULL COMMENT '来源ID',
    `source_table`      VARCHAR(50) NOT NULL COMMENT '来源表名',
    `breed_id`          BIGINT UNSIGNED DEFAULT NULL COMMENT '关联品种ID',
    `category`          VARCHAR(50) NOT NULL DEFAULT '' COMMENT '知识分类',
    `content`           TEXT NOT NULL COMMENT '文本内容',
    `content_hash`      VARCHAR(64) NOT NULL COMMENT '内容哈希',
    `embedding`         JSON DEFAULT NULL COMMENT '向量数据（MySQL备份，主存在Milvus）',
    `metadata`          JSON DEFAULT NULL COMMENT '元数据（标题、URL等）',
    `is_active`         TINYINT(1) NOT NULL DEFAULT 1 COMMENT '是否启用',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_chunk_id` (`chunk_id`),
    UNIQUE KEY `uk_source` (`source_type`, `source_id`),
    INDEX `idx_breed_id` (`breed_id`, `is_active`),
    INDEX `idx_category` (`category`, `is_active`),
    FULLTEXT INDEX `ft_content` (`content`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='知识库分片表';
```

---

### 3.6 通知系统

---

#### 表 3.6.1：notifications（通知表）

```sql
CREATE TABLE `notifications` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '接收者ID',
    `sender_id`         BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '发送者ID：0为系统',
    `notification_type` TINYINT NOT NULL COMMENT '通知类型：1点赞 2评论 3关注 4系统 5活动',
    `target_type`       TINYINT NOT NULL COMMENT '目标类型：1帖子 2评论 3用户 4品种 5商品',
    `target_id`         BIGINT UNSIGNED NOT NULL COMMENT '目标ID',
    `title`             VARCHAR(100) NOT NULL COMMENT '标题',
    `content`           VARCHAR(500) NOT NULL COMMENT '内容',
    `extra_data`        JSON DEFAULT NULL COMMENT '扩展数据',
    `is_read`           TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否已读：0否 1是',
    `read_at`           TIMESTAMP NULL DEFAULT NULL COMMENT '阅读时间',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX `idx_user_id` (`user_id`, `is_read`, `created_at`),
    INDEX `idx_type` (`notification_type`, `created_at`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='通知表';
```

---

### 3.7 运营系统

---

#### 表 3.7.1：banners（轮播图表）

```sql
CREATE TABLE `banners` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `title`             VARCHAR(100) NOT NULL COMMENT '标题',
    `image_url`         VARCHAR(500) NOT NULL COMMENT '图片URL',
    `link_type`         TINYINT NOT NULL COMMENT '链接类型：1品种详情 2帖子详情 3商品详情 4外部链接 5活动页',
    `link_target`       VARCHAR(500) NOT NULL COMMENT '链接目标',
    `position`          VARCHAR(50) NOT NULL COMMENT '展示位置：home/feed/breed/mall',
    `sort_order`        INT NOT NULL DEFAULT 0 COMMENT '排序',
    `start_time`        TIMESTAMP NOT NULL COMMENT '开始时间',
    `end_time`          TIMESTAMP NOT NULL COMMENT '结束时间',
    `click_count`       INT NOT NULL DEFAULT 0 COMMENT '点击次数',
    `status`            TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1启用',
    `created_by`        BIGINT UNSIGNED NOT NULL COMMENT '创建者',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_position` (`position`, `status`, `sort_order`),
    INDEX `idx_time` (`status`, `start_time`, `end_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='轮播图表';
```

---

#### 表 3.7.2：activities（活动表）

```sql
CREATE TABLE `activities` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `title`             VARCHAR(200) NOT NULL COMMENT '活动标题',
    `subtitle`          VARCHAR(255) NOT NULL DEFAULT '' COMMENT '副标题',
    `description`       TEXT COMMENT '活动描述',
    `activity_type`     TINYINT NOT NULL COMMENT '活动类型：1发布激励 2邀请裂变 3知识挑战 4晒宠活动',
    `cover_image`       VARCHAR(500) NOT NULL DEFAULT '' COMMENT '封面图',
    `banner_image`      VARCHAR(500) NOT NULL DEFAULT '' COMMENT '横幅图',
    `rules`             JSON DEFAULT NULL COMMENT '活动规则',
    `rewards`           JSON DEFAULT NULL COMMENT '奖励配置',
    `start_time`        TIMESTAMP NOT NULL COMMENT '开始时间',
    `end_time`          TIMESTAMP NOT NULL COMMENT '结束时间',
    `participant_count` INT NOT NULL DEFAULT 0 COMMENT '参与人数',
    `status`            TINYINT NOT NULL DEFAULT 0 COMMENT '状态：0未开始 1进行中 2已结束 3已取消',
    `created_by`        BIGINT UNSIGNED NOT NULL COMMENT '创建者',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX `idx_status_time` (`status`, `start_time`, `end_time`),
    INDEX `idx_type` (`activity_type`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='活动表';
```

---

#### 表 3.7.3：user_activity_records（用户活动参与记录表）

```sql
CREATE TABLE `user_activity_records` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `activity_id`       BIGINT UNSIGNED NOT NULL COMMENT '活动ID',
    `user_id`           BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `progress`          JSON DEFAULT NULL COMMENT '进度数据',
    `reward_status`     TINYINT NOT NULL DEFAULT 0 COMMENT '奖励状态：0未达标 1已达标 2已发放',
    `reward_data`       JSON DEFAULT NULL COMMENT '发放的奖励',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_activity_user` (`activity_id`, `user_id`),
    INDEX `idx_user_id` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户活动参与记录表';
```

---

### 3.8 搜索与标签系统

---

#### 表 3.8.1：search_keywords（搜索关键词表）

```sql
CREATE TABLE `search_keywords` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `keyword`           VARCHAR(100) NOT NULL COMMENT '关键词',
    `keyword_type`      TINYINT NOT NULL COMMENT '类型：1品种 2内容 3商品 4用户',
    `search_count`      INT NOT NULL DEFAULT 1 COMMENT '搜索次数',
    `result_count`      INT NOT NULL DEFAULT 0 COMMENT '结果数',
    `is_hot`            TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否热门：0否 1是',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_keyword_type` (`keyword`, `keyword_type`),
    INDEX `idx_hot` (`is_hot`, `search_count`),
    INDEX `idx_count` (`search_count`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='搜索关键词表';
```

---

#### 表 3.8.2：topics（话题标签表）

```sql
CREATE TABLE `topics` (
    `id`                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `name`              VARCHAR(100) NOT NULL COMMENT '话题名称',
    `description`       VARCHAR(255) NOT NULL DEFAULT '' COMMENT '话题描述',
    `cover_url`         VARCHAR(500) NOT NULL DEFAULT '' COMMENT '话题封面',
    `post_count`        INT NOT NULL DEFAULT 0 COMMENT '帖子数',
    `follow_count`      INT NOT NULL DEFAULT 0 COMMENT '关注数',
    `view_count`        INT NOT NULL DEFAULT 0 COMMENT '浏览数',
    `is_hot`            TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否热门：0否 1是',
    `status`            TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用 1正常',
    `created_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY `uk_name` (`name`),
    INDEX `idx_hot` (`is_hot`, `post_count`),
    INDEX `idx_status` (`status`, `post_count`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='话题标签表';
```

---

## 四、数据字典汇总

### 4.1 通用状态枚举

| 状态类型 | 值 | 含义 |
|----------|-----|------|
| `status`（通用） | 0 | 禁用/下架/草稿/未开始 |
| | 1 | 正常/启用/已发布/进行中 |
| | 2 | 审核中/已结束/已售罄 |
| | 3 | 已拒绝/已取消 |
| | 4 | 退款中 |
| | 5 | 已退款 |

### 4.2 用户相关枚举

| 字段 | 值 | 含义 |
|------|-----|------|
| `gender` | 0 | 未知 |
| | 1 | 男 |
| | 2 | 女 |
| `user_level` | 1 | 萌新铲屎官 |
| | 2 | 初级铲屎官 |
| | 3 | 进阶铲屎官 |
| | 4 | 资深铲屎官 |
| | 5 | 养宠达人 |
| | 6 | 宠物专家 |
| `verified_type` | 0 | 无认证 |
| | 1 | 达人认证 |
| | 2 | 专家认证 |
| | 3 | 机构认证 |
| `relation_type` | 1 | 关注 |
| | 2 | 好友 |
| | 3 | 拉黑 |

### 4.3 宠物相关枚举

| 字段 | 值 | 含义 |
|------|-----|------|
| `pet_status` | 0 | 已故 |
| | 1 | 健在 |
| | 2 | 送养 |
| `size_type` | 1 | 超小型（<4kg） |
| | 2 | 小型（4-10kg） |
| | 3 | 中型（10-25kg） |
| | 4 | 大型（25-45kg） |
| | 5 | 巨型（>45kg） |
| `coat_type` | 1 | 短毛 |
| | 2 | 长毛 |
| | 3 | 卷毛 |
| | 4 | 硬毛 |
| | 5 | 无毛 |
| `age_stage` | 1 | 幼年期 |
| | 2 | 成年期 |
| | 3 | 老年期 |
| | 4 | 全阶段 |

### 4.4 内容相关枚举

| 字段 | 值 | 含义 |
|------|-----|------|
| `post_type` | 1 | 图文 |
| | 2 | 视频 |
| | 3 | 图文+视频 |
| `post_status` | 0 | 审核中 |
| | 1 | 已发布 |
| | 2 | 拒绝 |
| | 3 | 下架 |
| `visibility` | 1 | 公开 |
| | 2 | 粉丝可见 |
| | 3 | 私密 |

---

## 五、索引设计原则

### 5.1 索引策略

| 场景 | 策略 | 示例 |
|------|------|------|
| 等值查询 | 单列索引 | `WHERE user_id = ?` → `INDEX(user_id)` |
| 范围查询 | 组合索引（范围列在后） | `WHERE status = 1 AND created_at > ?` → `INDEX(status, created_at)` |
| 排序查询 | 组合索引包含排序列 | `ORDER BY hot_score DESC` → `INDEX(status, hot_score)` |
| 多列筛选 | 组合索引 | `WHERE breed_id = ? AND age_stage = ?` → `UNIQUE(breed_id, age_stage)` |
| 全文搜索 | FULLTEXT 索引 | `FULLTEXT INDEX ft_content(title, content)` |
| 多值查询 | 多值索引（MySQL 8.0.17+） | `INDEX(breed_tags)` 对 JSON 数组 |
| 向量检索 | 向量数据库（Milvus） | 不建 MySQL 索引，用专用向量库 |

### 5.2 分表策略

| 表名 | 分表键 | 分表数 | 触发条件 |
|------|--------|--------|----------|
| `users` | `id` 取模 | 16 | 用户量 > 1000 万 |
| `posts` | `user_id` 取模 | 16 | 帖子量 > 1000 万 |
| `post_comments` | `post_id` 取模 | 16 | 评论量 > 5000 万 |
| `post_likes` | `post_id` 取模 | 16 | 点赞量 > 1 亿 |
| `notifications` | `user_id` 取模 | 16 | 通知量 > 5000 万 |
| `orders` | `user_id` 取模 | 16 | 订单量 > 1000 万 |
| `ai_messages` | `conversation_id` 取模 | 8 | 消息量 > 5000 万 |

---

## 六、缓存设计

### 6.1 Redis 缓存策略

| 缓存Key | 数据 | TTL | 说明 |
|---------|------|-----|------|
| `user:profile:{user_id}` | 用户基本信息 | 1 小时 | 用户信息缓存 |
| `user:pet_profiles:{user_id}` | 用户宠物列表 | 30 分钟 | 宠物档案缓存 |
| `breed:detail:{breed_id}` | 品种详情 | 24 小时 | 品种信息缓存（变动少） |
| `breed:hot_list` | 热门品种列表 | 1 小时 | 热门品种排行榜 |
| `post:detail:{post_id}` | 帖子详情 | 30 分钟 | 帖子内容缓存 |
| `post:hot_list:{breed_id}` | 热门帖子 | 30 分钟 | 按品种的热门帖子 |
| `feed:home:{user_id}` | 首页Feed | 5 分钟 | 个性化推荐Feed（短TTL保证新鲜度） |
| `search:hot_keywords` | 热搜词 | 10 分钟 | 搜索热词 |
| `product:detail:{product_id}` | 商品详情 | 1 小时 | 商品信息缓存 |
| `product:recommend:{breed_id}` | 品种推荐商品 | 1 小时 | 按品种推荐商品 |
| `ai:session:{session_id}` | AI对话上下文 | 30 分钟 | 对话历史缓存 |
| `rate_limit:ai:{user_id}` | AI限流计数 | 1 分钟 | 用户AI问答频率限制 |
| `counter:post_view:{post_id}` | 帖子浏览计数 | 5 分钟 | 先写Redis，定期刷到MySQL |
| `counter:post_like:{post_id}` | 帖子点赞计数 | 5 分钟 | 同上 |

### 6.2 缓存更新策略

| 场景 | 策略 | 说明 |
|------|------|------|
| 写操作 | Cache-Aside（旁路缓存） | 先写数据库，再删缓存，下次读时重建 |
| 读操作 | 缓存穿透保护 | 缓存未命中查DB，DB也未命中则缓存空值（TTL 5分钟） |
| 热点数据 | 预热 + 本地缓存 | 启动时预热，应用层加 Caffeine 本地缓存 |
| 计数器 | 延迟写入 | 先写Redis，定时任务批量写入MySQL |

---

## 七、数据安全设计

### 7.1 敏感数据处理

| 数据类型 | 处理方式 | 说明 |
|----------|----------|------|
| 手机号 | AES-256 加密存储 + 哈希索引 | 数据库中存储密文，查询用哈希 |
| 密码 | bcrypt 哈希（cost=12） | 不可逆存储 |
| 身份证号 | 不存储（如必须则AES加密） | 本业务大概率不需要 |
| 宠物健康记录 | 行级加密（AES） | 涉及隐私的医疗数据 |
| 收货地址 | 部分字段加密（姓名、电话） | 订单中加密存储 |
| 日志中的敏感信息 | 脱敏处理 | 手机号显示为 138****1234 |

### 7.2 权限控制

| 数据 | 访问控制 |
|------|----------|
| 用户基本信息 | 本人 + 管理员 |
| 宠物档案 | 本人（公开部分可配置） |
| 订单信息 | 本人 + 商家 + 管理员 |
| 用户行为数据 | 脱敏后供分析使用 |
| AI 对话记录 | 本人 + 管理员（审核用） |

---

## 八、数据库扩展性规划

### 8.1 读写分离

```
Master（写） ──→ Slave-1（读）
           ├──→ Slave-2（读）
           └──→ Slave-3（读，报表）
```

- 写操作：Master
- 读操作：Slave（按负载均衡）
- 报表查询：专用 Slave，不影响业务

### 8.2 水平扩展路径

| 阶段 | 用户量 | 数据量 | 架构方案 |
|------|--------|--------|----------|
| 初期 | < 100 万 | < 100 GB | 单库 + 读写分离 |
| 中期 | 100-500 万 | 100 GB - 1 TB | 分库分表（按用户ID） |
| 后期 | 500 万+ | 1 TB+ | 分布式数据库（TiDB/PolarDB） |

### 8.3 大数据归档

| 数据类型 | 归档策略 | 保留周期 |
|----------|----------|----------|
| 通知记录 | 3 个月后归档到冷存储 | 热库 3 个月，冷库 1 年 |
| AI 消息记录 | 6 个月后归档 | 热库 6 个月，冷库 2 年 |
| 用户行为日志 | 实时写入 ClickHouse | 永久保留（压缩存储） |
| 订单数据 | 2 年后归档 | 热库 2 年，冷库 7 年（税务要求） |
| 已删除内容 | 软删除后 1 年物理删除 | 软删除 1 年 |

---

> **本数据库设计为初始版本，随着业务发展需要持续迭代。每季度进行一次数据库健康检查，包括索引优化、慢查询分析、容量规划等。**
