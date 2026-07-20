# 结果呈现和符号标注机制设计

## 符号增强体系

### 趋势符号规则
```python
# 定义完整的符号映射
SYMBOL_MAPPING = {
    'trend': {
        'strong_positive': '📈',      # 增长 > 10%
        'positive': '↗️',           # 增长 5-10%
        'stable': '➡️',             # 变化 -5% 到 5%
        'negative': '↘️',           # 下降 5-10%
        'strong_negative': '📉',     # 下降 > 10%
    },
    'status': {
        'warning': '⚠️',             # 数据异常或需关注
        'attention': '🚨',           # 重点关注
        'success': '✅',             # 表现良好
        'error': '❌',               # 错误状态
    },
    'business': {
        'growth': '🌱',              # 增长潜力
        'decline': '🍂',             # 下降趋势
        'opportunity': '💡',        # 机会点
        'risk': '⚡',                # 风险点
    }
}
```

### 智能符号分配策略
```python
class SymbolAssigner:
    def __init__(self):
        self.thresholds = {
            'strong_change': 10,       # 10%以上为强变化
            'moderate_change': 5,    # 5-10%为中等变化
            'minor_change': 1        # 1-5%为轻微变化
        }
    
    def assign_trend_symbol(self, change_rate, metric_type=None):
        """根据变化率分配趋势符号"""
        abs_rate = abs(change_rate)
        
        if abs_rate >= self.thresholds['strong_change']:
            return SYMBOL_MAPPING['trend']['strong_positive'] if change_rate > 0 else SYMBOL_MAPPING['trend']['strong_negative']
        elif abs_rate >= self.thresholds['moderate_change']:
            return SYMBOL_MAPPING['trend']['positive'] if change_rate > 0 else SYMBOL_MAPPING['trend']['negative']
        else:
            return SYMBOL_MAPPING['trend']['stable']
    
    def assign_business_symbol(self, metric_context):
        """根据业务上下文分配业务符号"""
        context = metric_context.get('context', {})
        
        # 增长潜力判断
        if context.get('growth_potential', False):
            return SYMBOL_MAPPING['business']['growth']
        
        # 风险点判断
        if context.get('risk_level', 0) > 0.7:
            return SYMBOL_MAPPING['business']['risk']
        
        # 机会点判断
        if context.get('opportunity_score', 0) > 0.8:
            return SYMBOL_MAPPING['business']['opportunity']
        
        return ''
```

## 报告生成模板

### 1. 核心营销运营大盘模板
```python
OVERALL_TEMPLATE = """
## 核心营销运营大盘（{current_week} vs {last_year_week}）

| 指标 | {current_week} | {last_year_week} | 同比变化 | 趋势 |
| :--- | :--- | :--- | :--- | :--- |
{metric_rows}

### 归因总结
{attribution_summary}

### 建议
{suggestions}
"""

def generate_overall_report(metrics_data):
    metric_rows = []
    for metric_key, metric_data in metrics_data.items():
        row = f"| {metric_data['name']} | {format_number(metric_data['current'])} | {format_number(metric_data['last_year'])} | {format_percentage(metric_data['change_rate'])}% | {metric_data['trend']} |"
        metric_rows.append(row)
    
    return OVERALL_TEMPLATE.format(
        current_week=metrics_data['current_week'],
        last_year_week=metrics_data['last_year_week'],
        metric_rows='\n'.join(metric_rows),
        attribution_summary=generate_attribution_summary(metrics_data),
        suggestions=generate_suggestions(metrics_data)
    )
```

### 2. 广告投放效果模板
```python
AD_EFFECTIVENESS_TEMPLATE = """
## 广告投放效果分析

| 指标 | {current_week} | {last_year_week} | 变化 | 趋势 |
| :--- | :--- | :--- | :--- | :--- |
{metric_rows}

### 归因总结
{attribution_summary}

### 建议
{suggestions}
"""

def generate_ad_effectiveness_report(ad_data):
    return AD_EFFECTIVENESS_TEMPLATE.format(
        current_week=ad_data['current_week'],
        last_year_week=ad_data['last_year_week'],
        metric_rows=format_ad_metrics(ad_data),
        attribution_summary=analyze_ad_attribution(ad_data),
        suggestions=generate_ad_suggestions(ad_data)
    )
```

### 3. 多维分析模板（品类/品牌/货盘）
```python
MULTI_DIMENSION_TEMPLATE = """
## {analysis_type}分析

| 排名 | {entity_type}名称 | 周期 | NetGMV | 成交数量 | 子单量 | 成交购买用户数 | 动销商品数 | 广告投放消耗金额 | 广告成交ROI | 使用促销成交金额 | 促销ROI | 前台好评率 | 前台差评率 | 售后满意度 | CPO | QCR |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
{entity_rows}

### 归因总结
{attribution_summary}

### 建议
{suggestions}
"""

def generate_multi_dimension_report(analysis_type, entity_type, top_entities):
    entity_rows = []
    for idx, entity in enumerate(top_entities, 1):
        row = format_entity_row(idx, entity, entity_type)
        entity_rows.append(row)
    
    return MULTI_DIMENSION_TEMPLATE.format(
        analysis_type=analysis_type,
        entity_type=entity_type,
        entity_rows='\n'.join(entity_rows),
        attribution_summary=analyze_dimension_attribution(top_entities),
        suggestions=generate_dimension_suggestions(top_entities)
    )
```

## 归因分析引擎

### 1. 归因规则引擎
```python
class AttributionEngine:
    def __init__(self):
        self.attribution_rules = {
            'market_demand': self.analyze_market_demand,
            'marketing_efficiency': self.analyze_marketing_efficiency,
            'product_quality': self.analyze_product_quality,
            'competitive_landscape': self.analyze_competitive_landscape
        }
    
    def analyze_market_demand(self, data):
        """市场需求归因分析"""
        buyer_change = data.get('buyer_count_change', 0)
        gmv_change = data.get('gmv_change', 0)
        
        if buyer_change > 10 and gmv_change > 15:
            return "📈 市场需求旺盛，用户购买意愿增强"
        elif buyer_change < -5 and gmv_change < -10:
            return "📉 市场需求萎缩，用户购买意愿下降"
        else:
            return "➡️ 市场需求相对稳定"
    
    def analyze_marketing_efficiency(self, data):
        """营销效率归因分析"""
        ad_roi_change = data.get('ad_roi_change', 0)
        promo_roi_change = data.get('promo_roi_change', 0)
        
        if ad_roi_change > 10 and promo_roi_change > 5:
            return "✅ 营销效率显著提升，投资回报改善"
        elif ad_roi_change < -10 or promo_roi_change < -10:
            return "⚠️ 营销效率下降，需要优化投放策略"
        else:
            return "➡️ 营销效率保持稳定"
```

### 2. 建议生成引擎
```python
class RecommendationEngine:
    def __init__(self):
        self.recommendation_templates = {
            'ad_optimization': [
                "建议优化广告投放时段，提高转化效率",
                "考虑调整广告预算分配，聚焦高ROI渠道",
                "测试新的广告创意，提升点击率"
            ],
            'promo_optimization': [
                "优化促销策略组合，提高促销ROI",
                "调整优惠券发放策略，精准触达目标用户",
                "测试不同促销形式，找到最佳转化方案"
            ],
            'product_optimization': [
                "加强商品质量管理，提升用户满意度",
                "优化商品描述和展示，减少售后咨询",
                "根据用户反馈调整商品策略"
            ]
        }
    
    def generate_recommendations(self, analysis_results):
        recommendations = []
        
        # 根据分析结果匹配建议模板
        if analysis_results.get('ad_roi_decline', False):
            recommendations.extend(self.recommendation_templates['ad_optimization'])
        
        if analysis_results.get('promo_roi_decline', False):
            recommendations.extend(self.recommendation_templates['promo_optimization'])
        
        if analysis_results.get('quality_issues', False):
            recommendations.extend(self.recommendation_templates['product_optimization'])
        
        return list(set(recommendations))  # 去重
```

## 结果输出格式

### 1. 结构化JSON输出
```json
{
  "report_metadata": {
    "analysis_period": {
      "current_week": "2024-W15",
      "comparison_week": "2023-W15"
    },
    "generation_time": "2024-04-15T10:30:00Z",
    "report_version": "1.0"
  },
  "sections": {
    "overall_analysis": {
      "metrics": [...],
      "trend_symbols": {...},
      "attribution": "...",
      "recommendations": [...]
    },
    "ad_effectiveness": {...},
    "promo_strategy": {...},
    "marketing_fields": {...},
    "category_analysis": {...},
    "brand_analysis": {...},
    "cargo_analysis": {...}
  },
  "summary": {
    "key_insights": [...],
    "priority_actions": [...]
  }
}
```

### 2. 可视化报告生成
```python
class ReportVisualizer:
    def __init__(self):
        self.visualization_templates = {
            'table': self.generate_table_view,
            'chart': self.generate_chart_view,
            'dashboard': self.generate_dashboard_view
        }
    
    def generate_comprehensive_report(self, analysis_results, format_type='markdown'):
        if format_type == 'markdown':
            return self.generate_markdown_report(analysis_results)
        elif format_type == 'html':
            return self.generate_html_report(analysis_results)
        elif format_type == 'pdf':
            return self.generate_pdf_report(analysis_results)
```

## 自动化报告调度

### 1. 定时任务配置
```python
class ReportScheduler:
    def __init__(self):
        self.schedule_config = {
            'weekly_report': {
                'cron': '0 9 * * 1',  # 每周一上午9点
                'analysis_type': 'full_weekly_analysis'
            },
            'ad_hoc_report': {
                'trigger': 'on_demand',
                'analysis_type': 'custom_period_analysis'
            }
        }
    
    def schedule_weekly_report(self):
        """调度每周报告生成"""
        current_week = self.get_latest_complete_week()
        analysis_agent = MarketingAnalysisAgent()
        return analysis_agent.execute_full_analysis(current_week)
```

### 2. 报告分发机制
```python
class ReportDistributor:
    def __init__(self):
        self.distribution_channels = {
            'email': EmailDistributor(),
            'slack': SlackDistributor(),
            'wechat': WeChatDistributor(),
            'dashboard': DashboardDistributor()
        }
    
    def distribute_report(self, report_data, recipients, channels):
        results = {}
        for channel in channels:
            if channel in self.distribution_channels:
                distributor = self.distribution_channels[channel]
                results[channel] = distributor.send(report_data, recipients)
        return results
```