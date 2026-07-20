# Nebula Graph 营销分析图数据模型

## 节点类型设计

### 时间维度节点
```cypher
# 时间周节点
CREATE TAG time_week (
    week_id string,           # 格式: YYYY-WW
    year int,
    week_number int,
    start_date date,
    end_date date,
    is_complete bool         # 是否为完整自然周
)

# 时间日节点
CREATE TAG time_day (
    date_id string,           # 格式: YYYY-MM-DD
    date date,
    week_id string,           # 关联到周
    month string,
    quarter string
)
```

### 业务实体节点
```cypher
# 品类节点
CREATE TAG category (
    category_id string,
    category_name string,
    level int,                # 1-一级, 2-二级, 3-三级
    parent_category_id string
)

# 品牌节点
CREATE TAG brand (
    brand_id string,
    brand_name string
)

# 货盘节点
CREATE TAG cargo (
    cargo_id string,
    cargo_name string,
    cargo_type string       # 如: 爆款、常规、长尾
)

# 营销场域节点
CREATE TAG marketing_field (
    field_id string,
    field_name string,      # 如: 百亿补贴、便宜包邮、大秒杀
    field_type string       # 业务类型
)
```

### 指标聚合节点
```cypher
# 周度指标聚合节点
CREATE TAG weekly_metrics (
    metrics_id string,      # 格式: entity_type-entity_id-week_id
    entity_type string,     # category/brand/cargo/field/total
    entity_id string,
    week_id string,
    
    -- 核心指标
    net_gmv double,
    ad_spend double,
    ad_profit double,
    ad_roi double,
    promo_roi double,
    promo_discount_amount double,
    promo_orders int,
    
    -- 广告效果指标
    ad_gross_profit double,
    ad_gross_margin double,
    
    -- 促销效果指标
    promo_gmv double,
    coupon_cost double,
    coupon_roi double,
    
    -- 场域效果指标
    field_gmv double,
    
    -- 多维分析指标
    order_quantity int,
    sub_order_count int,
    buyer_count int,
    active_sku_count int,
    positive_rate double,
    negative_rate double,
    after_sale_satisfaction double,
    cpo double,             # 每百万订单投诉量
    qcr double              # 服务质量指标
)
```

## 边类型设计

### 实体关系边
```cypher
# 品类层级关系
CREATE EDGE belongs_to ()

# 品牌与品类关系
CREATE EDGE brand_category ()

# 货盘与品类关系
CREATE EDGE cargo_category ()

# 场域与指标关系
CREATE EDGE field_metrics ()
```

### 时间关系边
```cypher
# 周与日关系
CREATE EDGE week_contains_day ()

# 同比关系
CREATE EDGE year_over_year (
    similarity double       # 相似度权重
)
```

### 指标衍生关系
```cypher
# 指标计算依赖关系
CREATE EDGE metrics_dependency (
    dependency_type string  # parent/child/sibling
)
```