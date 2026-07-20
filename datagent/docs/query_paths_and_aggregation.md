# 查询路径和聚合策略设计

## 第1部分：整体盘面分析查询

### 查询路径
```cypher
MATCH (m_current:weekly_metrics {entity_type: 'total', entity_id: 'all', week_id: $current_week})
MATCH (m_last_year:weekly_metrics {entity_type: 'total', entity_id: 'all', week_id: $last_year_week})
RETURN 
    m_current.net_gmv as current_net_gmv,
    m_last_year.net_gmv as last_year_net_gmv,
    m_current.ad_spend as current_ad_spend,
    m_last_year.ad_spend as last_year_ad_spend,
    m_current.ad_profit as current_ad_profit,
    m_last_year.ad_profit as last_year_ad_profit,
    m_current.ad_roi as current_ad_roi,
    m_last_year.ad_roi as last_year_ad_roi,
    m_current.promo_roi as current_promo_roi,
    m_last_year.promo_roi as last_year_promo_roi,
    m_current.promo_discount_amount as current_promo_discount,
    m_last_year.promo_discount_amount as last_year_promo_discount,
    m_current.promo_orders as current_promo_orders,
    m_last_year.promo_orders as last_year_promo_orders
```

### 聚合计算
```python
def calculate_overall_metrics(current, last_year):
    results = {}
    
    # 计算同比变化率和趋势符号
    metrics_to_compare = [
        'net_gmv', 'ad_spend', 'ad_profit', 'ad_roi',
        'promo_roi', 'promo_discount_amount', 'promo_orders'
    ]
    
    for metric in metrics_to_compare:
        current_val = current.get(f'current_{metric}', 0)
        last_year_val = last_year.get(f'last_year_{metric}', 0)
        
        change_value = current_val - last_year_val
        change_rate = (change_value / last_year_val * 100) if last_year_val != 0 else 0
        
        # 趋势判断
        if abs(change_rate) > 10:
            trend = "📈" if change_rate > 0 else "📉"
        elif abs(change_rate) > 5:
            trend = "↗️" if change_rate > 0 else "↘️"
        else:
            trend = "➡️"
            
        results[metric] = {
            'current': current_val,
            'last_year': last_year_val,
            'change_value': change_value,
            'change_rate': round(change_rate, 2),
            'trend': trend
        }
    
    return results
```

## 第2部分：广告投放效果查询

### 查询路径
```cypher
MATCH (m_current:weekly_metrics {entity_type: 'total', entity_id: 'all', week_id: $current_week})
MATCH (m_last_year:weekly_metrics {entity_type: 'total', entity_id: 'all', week_id: $last_year_week})
RETURN 
    m_current.net_gmv as current_net_gmv,
    m_last_year.net_gmv as last_year_net_gmv,
    m_current.ad_gross_profit as current_ad_gross_profit,
    m_last_year.ad_gross_profit as last_year_ad_gross_profit,
    m_current.ad_gross_margin as current_ad_gross_margin,
    m_last_year.ad_gross_margin as last_year_ad_gross_margin
```

### 转化率计算
```python
def calculate_ad_conversion_metrics(current, last_year):
    # 计算转化率变化
    current_conversion = current.get('current_net_gmv', 0) / max(current.get('current_ad_spend', 1), 1)
    last_year_conversion = last_year.get('last_year_net_gmv', 0) / max(last_year.get('last_year_ad_spend', 1), 1)
    
    conversion_change = current_conversion - last_year_conversion
    conversion_change_rate = (conversion_change / last_year_conversion * 100) if last_year_conversion != 0 else 0
    
    return {
        'current_conversion': round(current_conversion, 4),
        'last_year_conversion': round(last_year_conversion, 4),
        'change': round(conversion_change, 4),
        'change_rate': round(conversion_change_rate, 2),
        'trend': "📈" if conversion_change_rate > 5 else "📉" if conversion_change_rate < -5 else "➡️"
    }
```

## 第3部分：促销优惠策略评估查询

### 查询路径
```cypher
MATCH (m_current:weekly_metrics {entity_type: 'total', entity_id: 'all', week_id: $current_week})
MATCH (m_last_year:weekly_metrics {entity_type: 'total', entity_id: 'all', week_id: $last_year_week})
RETURN 
    m_current.promo_gmv as current_promo_gmv,
    m_last_year.promo_gmv as last_year_promo_gmv,
    m_current.promo_orders as current_promo_orders,
    m_last_year.promo_orders as last_year_promo_orders,
    m_current.coupon_cost as current_coupon_cost,
    m_last_year.coupon_cost as last_year_coupon_cost,
    m_current.coupon_roi as current_coupon_roi,
    m_last_year.coupon_roi as last_year_coupon_roi
```

## 第4部分：营销场域效应分析查询

### 查询路径
```cypher
MATCH (f:marketing_field)-[:field_metrics]->(m_current:weekly_metrics {week_id: $current_week})
MATCH (f)-[:field_metrics]->(m_last_year:weekly_metrics {week_id: $last_year_week})
RETURN 
    f.field_name as field_name,
    m_current.field_gmv as current_gmv,
    m_last_year.field_gmv as last_year_gmv
ORDER BY m_current.field_gmv DESC
```

## 第5-7部分：多维分析查询（品类、品牌、货盘）

### 通用查询模式
```cypher
MATCH (entity)-[:belongs_to*0..3]->(parent)
WHERE entity.entity_type IN ['category', 'brand', 'cargo']
  AND entity.level = $level
MATCH (entity)-[:has_metrics]->(m_current:weekly_metrics {week_id: $current_week})
MATCH (entity)-[:has_metrics]->(m_last_year:weekly_metrics {week_id: $last_year_week})
RETURN 
    entity.entity_id as entity_id,
    entity.entity_name as entity_name,
    parent.entity_name as parent_name,
    m_current.net_gmv as current_net_gmv,
    m_last_year.net_gmv as last_year_net_gmv,
    m_current.order_quantity as current_order_quantity,
    m_last_year.order_quantity as last_year_order_quantity,
    m_current.sub_order_count as current_sub_order_count,
    m_last_year.sub_order_count as last_year_sub_order_count,
    m_current.buyer_count as current_buyer_count,
    m_last_year.buyer_count as last_year_buyer_count,
    m_current.active_sku_count as current_active_sku_count,
    m_last_year.active_sku_count as last_year_active_sku_count,
    m_current.ad_spend as current_ad_spend,
    m_last_year.ad_spend as last_year_ad_spend,
    m_current.ad_roi as current_ad_roi,
    m_last_year.ad_roi as last_year_ad_roi,
    m_current.promo_gmv as current_promo_gmv,
    m_last_year.promo_gmv as last_year_promo_gmv,
    m_current.promo_roi as current_promo_roi,
    m_last_year.promo_roi as last_year_promo_roi,
    m_current.positive_rate as current_positive_rate,
    m_last_year.positive_rate as last_year_positive_rate,
    m_current.negative_rate as current_negative_rate,
    m_last_year.negative_rate as last_year_negative_rate,
    m_current.after_sale_satisfaction as current_after_sale_satisfaction,
    m_last_year.after_sale_satisfaction as last_year_after_sale_satisfaction,
    m_current.cpo as current_cpo,
    m_last_year.cpo as last_year_cpo,
    m_current.qcr as current_qcr,
    m_last_year.qcr as last_year_qcr
ORDER BY m_current.net_gmv DESC
LIMIT $top_n
```

## 趋势判断规则

### 符号增强规则
```python
TREND_SYMBOLS = {
    'strong_positive': '📈',      # 增长 > 10%
    'positive': '↗️',           # 增长 5-10%
    'stable': '➡️',             # 变化 -5% 到 5%
    'negative': '↘️',           # 下降 5-10%
    'strong_negative': '📉',     # 下降 > 10%
    'warning': '⚠️',            # 数据异常或缺失
    'attention': '🚨'           # 需重点关注
}

def determine_trend_symbol(change_rate):
    if change_rate > 10:
        return TREND_SYMBOLS['strong_positive']
    elif change_rate > 5:
        return TREND_SYMBOLS['positive']
    elif change_rate > -5:
        return TREND_SYMBOLS['stable']
    elif change_rate > -10:
        return TREND_SYMBOLS['negative']
    else:
        return TREND_SYMBOLS['strong_negative']
```

## 查询优化策略

### 1. 索引设计
```cypher
# 时间维度索引
CREATE INDEX ON time_week(week_id);
CREATE INDEX ON time_day(date_id);

# 实体索引
CREATE INDEX ON category(category_id);
CREATE INDEX ON brand(brand_id);
CREATE INDEX ON cargo(cargo_id);
CREATE INDEX ON marketing_field(field_id);

# 指标索引
CREATE INDEX ON weekly_metrics(entity_type, entity_id, week_id);
CREATE INDEX ON weekly_metrics(week_id, entity_type);
```

### 2. 查询缓存策略
```python
# 缓存热点查询结果
CACHE_CONFIG = {
    'overall_metrics': 3600,        # 整体指标缓存1小时
    'top_entities': 1800,          # TOP实体缓存30分钟
    'field_analysis': 3600,         # 场域分析缓存1小时
    'entity_analysis': 900          # 实体分析缓存15分钟
}
```

### 3. 并行查询优化
```python
# 并行执行多个分析任务
async def parallel_analysis_tasks():
    tasks = [
        overall_analysis_task(),
        ad_effectiveness_task(),
        promo_strategy_task(),
        marketing_field_task(),
        category_analysis_task(),
        brand_analysis_task(),
        cargo_analysis_task()
    ]
    
    # 并发执行所有任务
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # 处理异常结果
    return process_parallel_results(results)
```