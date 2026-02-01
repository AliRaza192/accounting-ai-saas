# AI Insight & Analytics Agent Specification

## 1. Agent Purpose

The AI Insight & Analytics Agent is responsible for:
- Generating actionable financial insights from accounting data
- Performing variance analysis between periods
- Detecting trends and patterns in financial metrics
- Creating forecasts for revenue, cash flow, and expenses
- Explaining anomalies and unusual fluctuations

## 2. Analysis Types

### A. Variance Analysis

**Input:** P&L for Jan 2024 vs Jan 2023

**Insight Output:**
```
Revenue increased 15% ($150K → $172.5K) driven by:

New customers: +$12.5K
Price increase: +$10K

However, Operating Expenses jumped 25% ($80K → $100K):

Marketing spend doubled ($10K → $20K)
New hires added $15K in salaries

Net Margin decreased from 46% to 42%.
Recommendation: Monitor marketing ROI closely.
```

**Algorithm:**
```python
def variance_analysis(current_period, comparison_period):
    """
    Performs detailed variance analysis between two periods
    """
    variances = {}

    # Calculate percentage changes
    for account, current_val in current_period.items():
        comp_val = comparison_period.get(account, 0)

        if comp_val != 0:
            percent_change = ((current_val - comp_val) / comp_val) * 100
        else:
            percent_change = float('inf') if current_val > 0 else 0

        variances[account] = {
            'current': current_val,
            'comparison': comp_val,
            'change': current_val - comp_val,
            'percent_change': percent_change
        }

    return variances
```

### B. Trend Detection

**Example Analysis:**
```
Cash balance declining consistently:

Dec: $50K
Jan: $42K (-16%)
Feb: $35K (-17%)

Projection: Will reach $20K by April.
Alert: Consider credit line or reduce expenses.
```

**Algorithm:**
```python
def detect_trends(time_series_data, min_periods=3):
    """
    Detects trends in financial metrics over time
    """
    if len(time_series_data) < min_periods:
        return None

    # Calculate trend using linear regression
    x_values = list(range(len(time_series_data)))
    y_values = [float(val) for val in time_series_data.values()]

    # Simple linear regression
    n = len(x_values)
    sum_x = sum(x_values)
    sum_y = sum(y_values)
    sum_xy = sum(x * y for x, y in zip(x_values, y_values))
    sum_x_squared = sum(x * x for x in x_values)

    slope = (n * sum_xy - sum_x * sum_y) / (n * sum_x_squared - sum_x * sum_x)
    intercept = (sum_y - slope * sum_x) / n

    trend_direction = "increasing" if slope > 0 else "decreasing" if slope < 0 else "stable"

    return {
        'slope': slope,
        'direction': trend_direction,
        'magnitude': abs(slope),
        'significance': abs(slope) / max(abs(val) for val in y_values) if y_values else 0
    }
```

### C. Anomaly Explanation

**Example Analysis:**
```
Detected: Utilities expense $5,000 (usual: $500)
Analysis:

10x normal amount
Transaction date: Jan 15
Vendor: Power Company
Likely cause: Annual pre-payment or error

Recommendation: Verify with vendor, may need accrual adjustment.
```

**Algorithm:**
```python
def explain_anomaly(account_data, threshold_multiplier=3):
    """
    Explains anomalies in account balances
    """
    historical_avg = calculate_historical_average(account_data)
    historical_std = calculate_historical_std_dev(account_data)

    current_value = account_data[-1]  # Most recent value

    if historical_avg and historical_std:
        z_score = abs(current_value - historical_avg) / historical_std

        if z_score > threshold_multiplier:
            return {
                'is_anomaly': True,
                'magnitude': z_score,
                'expected_range': [historical_avg - 2*historical_std, historical_avg + 2*historical_std],
                'actual_value': current_value,
                'expected_value': historical_avg,
                'deviation': current_value - historical_avg
            }

    return {'is_anomaly': False}
```

## 3. Forecasting Models

### A. Revenue Prediction
```python
def forecast_revenue(historical_data, periods=3):
    """
    Predicts revenue for next periods using trend analysis
    """
    import numpy as np
    from sklearn.linear_model import LinearRegression

    # Prepare data for forecasting
    X = np.array(range(len(historical_data))).reshape(-1, 1)
    y = np.array(list(historical_data.values()))

    # Fit model
    model = LinearRegression()
    model.fit(X, y)

    # Predict future values
    future_X = np.array(range(len(historical_data), len(historical_data) + periods)).reshape(-1, 1)
    predictions = model.predict(future_X)

    return {
        'predictions': predictions.tolist(),
        'trend_slope': model.coef_[0],
        'confidence_interval': calculate_confidence_interval(model, X, y, future_X)
    }
```

### B. Cash Flow Forecast
```python
def forecast_cash_flow(inflow_data, outflow_data, periods=3):
    """
    Forecasts cash flow considering inflows and outflows
    """
    # Separate forecasting for inflows and outflows
    inflow_forecast = forecast_revenue(inflow_data, periods)
    outflow_forecast = forecast_revenue(outflow_data, periods)

    # Calculate net cash flow
    net_forecast = [inflow - outflow for inflow, outflow in
                   zip(inflow_forecast['predictions'], outflow_forecast['predictions'])]

    return {
        'inflow_predictions': inflow_forecast['predictions'],
        'outflow_predictions': outflow_forecast['predictions'],
        'net_cash_flow': net_forecast,
        'cash_position': calculate_projected_cash_position(net_forecast)
    }
```

### C. Seasonality Detection
```python
def detect_seasonality(time_series_data, period_length=12):
    """
    Detects seasonal patterns in financial data
    """
    import numpy as np

    # Convert to numpy array for easier manipulation
    data = np.array(list(time_series_data.values()))

    # Calculate seasonal indices
    seasonal_indices = []
    for i in range(period_length):
        season_data = data[i::period_length]  # Every nth element
        if len(season_data) > 0:
            seasonal_indices.append(np.mean(season_data))

    # Normalize seasonal indices
    overall_mean = np.mean(data)
    normalized_indices = [idx / overall_mean for idx in seasonal_indices]

    return {
        'seasonal_indices': normalized_indices,
        'has_seasonality': max(normalized_indices) / min(normalized_indices) > 1.2,
        'peak_months': [i for i, idx in enumerate(normalized_indices) if idx > 1.1]
    }
```

## 4. Comparative Metrics

### A. Industry Benchmarks
```python
def compare_to_benchmarks(company_metrics, industry_data):
    """
    Compares company metrics to industry averages
    """
    comparisons = {}

    for metric, company_value in company_metrics.items():
        industry_avg = industry_data.get(metric, 0)

        if industry_avg != 0:
            ratio = company_value / industry_avg
            comparison = {
                'company': company_value,
                'industry': industry_avg,
                'ratio': ratio,
                'performance': 'above' if ratio > 1.1 else 'below' if ratio < 0.9 else 'similar'
            }
        else:
            comparison = {
                'company': company_value,
                'industry': industry_avg,
                'ratio': 'N/A',
                'performance': 'no_comparison_available'
            }

        comparisons[metric] = comparison

    return comparisons
```

### B. Year-over-Year Growth
```python
def calculate_yoy_growth(current_year_data, previous_year_data):
    """
    Calculates year-over-year growth percentages
    """
    growth_rates = {}

    for account, current_val in current_year_data.items():
        prev_val = previous_year_data.get(account, 0)

        if prev_val != 0:
            growth_rate = ((current_val - prev_val) / prev_val) * 100
        else:
            growth_rate = float('inf') if current_val > 0 else 0

        growth_rates[account] = {
            'current': current_val,
            'previous': prev_val,
            'growth_rate': growth_rate,
            'growth_amount': current_val - prev_val
        }

    return growth_rates
```

## 5. Agent Boundaries

### CAN Do
- Analyze financial data and generate insights
- Create detailed reports and summaries
- Detect patterns and trends in data
- Forecast future financial performance
- Explain anomalies and unusual fluctuations

### CANNOT Do
- Modify or alter financial data in any way
- Post adjustments or make accounting entries
- Override user decisions or system configurations

## 6. Natural Language Insights

### Example Interaction:
**User:** "Why did my profit drop last month?"

**Agent Response:**
```
Your profit decreased by $15K (30%) in January because:

Revenue was flat at $50K (no growth)
Operating expenses increased by $15K:

New software subscriptions: $8K
Marketing campaign: $5K
Legal fees: $2K

Your profit margin dropped from 40% to 10%.
To improve next month:

Follow up on 5 overdue invoices ($12K)
Consider pausing non-essential subscriptions
Marketing ROI should show results in 60 days
```

### Natural Language Processing Pipeline
```python
def process_natural_query(query, financial_data):
    """
    Processes natural language queries about financial data
    """
    # Parse query intent
    intent = classify_query_intent(query)

    # Generate appropriate analysis based on intent
    if intent == 'variance_analysis':
        return generate_variance_insights(financial_data)
    elif intent == 'trend_analysis':
        return generate_trend_insights(financial_data)
    elif intent == 'anomaly_explanation':
        return explain_anomalies(financial_data)
    elif intent == 'forecasting':
        return generate_forecasts(financial_data)
    else:
        return generate_general_insights(financial_data)

def classify_query_intent(query):
    """
    Classifies the intent behind a natural language query
    """
    query_lower = query.lower()

    if any(word in query_lower for word in ['why', 'drop', 'increase', 'change', 'difference']):
        return 'variance_analysis'
    elif any(word in query_lower for word in ['trend', 'declining', 'growing', 'pattern']):
        return 'trend_analysis'
    elif any(word in query_lower for word in ['unusual', 'anomaly', 'strange', 'high', 'low']):
        return 'anomaly_explanation'
    elif any(word in query_lower for word in ['forecast', 'predict', 'future', 'next']):
        return 'forecasting'
    else:
        return 'general_insights'
```

## 7. Scheduled Insights

### A. Weekly Cash Flow Summary
- Current cash position
- Inflow/outflow analysis
- Upcoming obligations
- Short-term projections

### B. Monthly Performance Review
- Revenue and expense variances
- Key metric trends
- Budget vs. actual performance
- Action items for improvement

### C. Quarterly Forecast Update
- Updated revenue projections
- Cash flow outlook
- Seasonal adjustments
- Strategic recommendations

### D. Annual Budget vs. Actual
- Full-year performance summary
- Budget variance analysis
- Trend continuation assessment
- Next year planning insights

## 8. API Integration

### Endpoint: POST /api/v1/ai/insights

#### Request Body
```json
{
  "analysis_type": "variance",
  "report_type": "profit_loss",
  "period_current": "2024-01",
  "period_compare": "2023-01",
  "filters": {
    "accounts": ["revenue", "expenses"],
    "departments": ["sales", "marketing"],
    "entities": ["parent_company"]
  },
  "user_context": {
    "user_id": "usr_123",
    "company_size": "mid_market",
    "industry": "technology"
  }
}
```

#### Response Schema
```json
{
  "insights": [
    {
      "category": "revenue",
      "finding": "Revenue increased 15%",
      "impact": "positive",
      "drivers": [
        {
          "factor": "new_customers",
          "contribution": 12500,
          "percentage": 25
        },
        {
          "factor": "price_increase",
          "contribution": 10000,
          "percentage": 20
        }
      ],
      "recommendations": [
        "Continue customer acquisition efforts",
        "Monitor price sensitivity"
      ],
      "confidence": 0.92
    }
  ],
  "summary": "Overall performance improved with strong revenue growth, though expenses increased significantly.",
  "confidence": 0.88,
  "metadata": {
    "analysis_date": "2024-02-01T10:30:00Z",
    "data_coverage": 0.95,
    "completeness": "complete"
  }
}
```

#### Alternative Request Types

**Trend Analysis Request:**
```json
{
  "analysis_type": "trend",
  "metric": "cash_balance",
  "time_period": "last_6_months",
  "forecast_periods": 3
}
```

**Forecast Request:**
```json
{
  "analysis_type": "forecast",
  "metrics": ["revenue", "expenses", "cash_flow"],
  "forecast_horizon": "next_quarter",
  "confidence_level": 0.95
}
```

## 9. Claude API Prompts for Analysis

### Variance Analysis Prompt
```
You are a financial analyst examining the following P&L data:

Current Period: {current_data}
Previous Period: {previous_data}

Perform a detailed variance analysis focusing on:
1. Percentage changes in major categories
2. Key drivers of changes
3. Impact on profitability
4. Actionable recommendations

Structure your response in clear sections with dollar amounts and percentages.
Use professional but accessible language.
```

### Trend Analysis Prompt
```
Analyze the following time series data for {metric_name}:

{time_series_data}

Identify:
1. Overall trend direction and magnitude
2. Seasonal patterns if present
3. Rate of change
4. Projected future values
5. Potential risks or opportunities

Provide specific figures and actionable insights.
```

### Anomaly Explanation Prompt
```
Explain the following anomaly in {account_name}:

Current Value: ${current_value}
Historical Average: ${historical_avg}
Standard Deviation: ${std_dev}

Possible causes and recommended actions:
1. Business explanation
2. Verification steps
3. Accounting implications
4. Follow-up actions

Be specific and practical in your recommendations.
```

### Natural Language Query Prompt
```
A user has asked: "{user_question}"

Using the following financial data: {financial_context}

Provide a clear, concise answer that addresses their specific concern.
Include relevant figures and actionable recommendations.
Format your response in a conversational tone suitable for non-accountants.
```

## 10. Statistical Algorithms

### A. Moving Average Calculation
```python
def calculate_moving_average(data, window_size=3):
    """
    Calculates moving average for trend smoothing
    """
    if len(data) < window_size:
        return data

    moving_averages = []
    for i in range(len(data) - window_size + 1):
        window = data[i:i + window_size]
        avg = sum(window) / window_size
        moving_averages.append(avg)

    return moving_averages
```

### B. Confidence Interval Calculation
```python
def calculate_confidence_interval(model, X_train, y_train, X_predict, confidence=0.95):
    """
    Calculates confidence intervals for predictions
    """
    from scipy import stats
    import numpy as np

    # Calculate residuals
    y_pred_train = model.predict(X_train)
    residuals = y_train - y_pred_train

    # Calculate standard error
    mse = np.mean(residuals ** 2)
    std_error = np.sqrt(mse)

    # Calculate t-value for confidence interval
    df = len(y_train) - 2
    t_value = stats.t.ppf((1 + confidence) / 2, df)

    # Calculate confidence intervals
    intervals = []
    for x in X_predict:
        pred = model.predict([x])[0]
        margin = t_value * std_error
        intervals.append((pred - margin, pred + margin))

    return intervals
```

### C. Correlation Analysis
```python
def calculate_correlations(financial_data):
    """
    Calculates correlations between different financial metrics
    """
    import numpy as np

    correlations = {}
    metrics = list(financial_data.keys())

    for i, metric1 in enumerate(metrics):
        for j, metric2 in enumerate(metrics[i+1:], i+1):
            data1 = np.array(list(financial_data[metric1].values()))
            data2 = np.array(list(financial_data[metric2].values()))

            # Calculate correlation coefficient
            correlation = np.corrcoef(data1, data2)[0, 1]

            correlations[f"{metric1}_vs_{metric2}"] = {
                'correlation': correlation,
                'strength': 'strong' if abs(correlation) > 0.7 else 'moderate' if abs(correlation) > 0.3 else 'weak',
                'direction': 'positive' if correlation > 0 else 'negative'
            }

    return correlations
```

## 11. Natural Language Generation Templates

### Variance Analysis Template
```
{metric_name} {change_direction} {percentage_change}% ({amount_change}) from {previous_period} to {current_period}.

Primary drivers:
{%- for driver in drivers %}
- {{ driver.factor }}: {{ driver.contribution }}
{%- endfor %}

Impact: {impact_assessment}
Recommendation: {actionable_recommendation}
```

### Trend Analysis Template
```
{metric_name} has been {trend_direction} over the past {time_period}.

Rate of change: {change_rate} per period
Pattern: {pattern_type} {%- if seasonal %} with {seasonal_characteristics} {% endif %}

Projection: Expected to {future_expectation} in the next {forecast_period}.
Risk Level: {risk_assessment}
```

### Anomaly Explanation Template
```
Anomaly detected in {account_name} for {date}:
- Current Value: ${current_value}
- Expected Range: ${expected_min} - ${expected_max}
- Deviation: {deviation_percentage}%

Likely Causes:
{%- for cause in potential_causes %}
- {{ cause }}
{%- endfor %}

Recommended Actions:
{%- for action in recommended_actions %}
- {{ action }}
{%- endfor %}
```

## 12. Performance Requirements

### Accuracy Standards
- **Variance Analysis**: >95% accuracy in identifying major changes
- **Forecasting**: 85% accuracy within 10% of actual results
- **Anomaly Detection**: <5% false positive rate
- **Natural Language**: Human-like explanation quality

### Performance Metrics
- **Response Time**: < 2 seconds for standard analyses
- **Data Processing**: Handle datasets up to 10,000 records
- **Concurrency**: Support 100 simultaneous analysis requests
- **Uptime**: 99.5% availability for insight generation

### Scalability
- Horizontal scaling for high-volume periods
- Caching for frequently requested analyses
- Batch processing for scheduled insights
- Asynchronous processing for complex analyses