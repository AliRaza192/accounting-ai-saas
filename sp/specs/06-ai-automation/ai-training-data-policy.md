# AI Model Training & Version Control Specification

## 1. Training Data Management

### A. Data Collection Sources

The system collects training data from multiple sources:

**User Corrections:**
- When users modify AI suggestions
- Manual overrides of AI decisions
- Explicit feedback on AI accuracy
- Corrections to auto-classified transactions

**Approved Classifications:**
- User-approved AI suggestions
- Manually classified items that match AI suggestions
- Supervised learning examples
- Verified transaction mappings

**Successful Matches:**
- Auto-matched reconciliation items
- Correctly identified patterns
- Validated journal entry suggestions
- Confirmed transaction classifications

**Feedback Ratings:**
- Explicit user satisfaction ratings
- Implicit feedback from user behavior
- Time-to-resolution metrics
- User engagement with suggestions

### B. Data Storage Schema

```sql
CREATE TABLE ai.training_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    agent_type ENUM('classification', 'reconciliation', 'journal', 'compliance', 'insight', 'chat') NOT NULL,
    input_data JSONB NOT NULL,
    expected_output JSONB,
    actual_output JSONB,
    was_correct BOOLEAN,
    user_feedback TEXT,
    confidence_score DECIMAL(3,2),
    feedback_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_training_data_tenant ON ai.training_data(tenant_id);
CREATE INDEX idx_training_data_agent_type ON ai.training_data(agent_type);
CREATE INDEX idx_training_data_created_at ON ai.training_data(created_at);
CREATE INDEX idx_training_data_was_correct ON ai.training_data(was_correct);
```

### C. Privacy Protection Measures

**Tenant Data Isolation:**
- Each tenant's data stored separately
- Cross-tenant access strictly prohibited
- Role-based access controls enforced
- Data residency requirements maintained

**Default Isolation Policy:**
- No cross-tenant learning by default
- Models trained per-tenant or on synthetic data
- Aggregate statistics only shared with explicit consent

**Opt-in Aggregate Learning:**
- Users can opt-in to contribute anonymized data
- Aggregated at country/regional level only
- Individual transaction data never shared

**PII Scrubbing Process:**
```python
def scrub_pii(data):
    """
    Scrubs personally identifiable information from training data
    """
    import re

    # Remove names, addresses, SSNs, phone numbers
    scrubbed_data = re.sub(r'\b[A-Z][a-z]+\s+[A-Z][a-z]+\b', '[NAME]', data)
    scrubbed_data = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '[SSN]', scrubbed_data)
    scrubbed_data = re.sub(r'\b\d{3}-\d{3}-\d{4}\b', '[PHONE]', scrubbed_data)

    # Remove email addresses
    scrubbed_data = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL]', scrubbed_data)

    # Remove credit card numbers
    scrubbed_data = re.sub(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b', '[CARD]', scrubbed_data)

    return scrubbed_data
```

## 2. Model Versioning System

### A. Version Storage Schema

```sql
CREATE TABLE ai.model_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_type VARCHAR(50) NOT NULL,
    version VARCHAR(20) NOT NULL, -- v1.0, v1.1
    trained_on TIMESTAMP NOT NULL,
    training_samples INTEGER,
    validation_samples INTEGER,
    accuracy_score DECIMAL(5,4),
    precision_score DECIMAL(5,4),
    recall_score DECIMAL(5,4),
    f1_score DECIMAL(5,4),
    training_duration INTERVAL,
    is_active BOOLEAN DEFAULT FALSE,
    is_production BOOLEAN DEFAULT FALSE,
    rollback_version UUID REFERENCES ai.model_versions(id),
    model_artifact_path TEXT, -- Path to stored model file
    training_config JSONB, -- Training parameters used
    created_by UUID,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_model_versions_agent_type ON ai.model_versions(agent_type);
CREATE INDEX idx_model_versions_active ON ai.model_versions(is_active);
CREATE INDEX idx_model_versions_production ON ai.model_versions(is_production);
CREATE INDEX idx_model_versions_version ON ai.model_versions(version);
```

### B. Version Control Workflow

**Model Development Cycle:**
1. Collect sufficient training data (>100 new samples)
2. Train candidate model in development environment
3. Validate performance against baseline
4. Deploy to staging for A/B testing
5. Promote to production if performance acceptable
6. Mark previous version as inactive

**Version Promotion Process:**
```python
def promote_model_version(candidate_version_id):
    """
    Promotes a candidate model to production after validation
    """
    # Validate candidate model meets requirements
    candidate_metrics = get_model_metrics(candidate_version_id)
    baseline_metrics = get_current_production_metrics()

    if candidate_metrics['accuracy'] >= baseline_metrics['accuracy'] * 0.98:  # Allow 2% degradation
        # Deactivate current production model
        deactivate_current_production_model()

        # Activate new model
        activate_model(candidate_version_id)

        # Update production flag
        update_model_production_status(candidate_version_id, True)

        return True
    else:
        # Candidate model doesn't meet requirements
        return False
```

## 3. Retraining Schedule

### A. Agent-Specific Schedules

**Classification Agent: Daily**
- New data: Daily accumulation
- Retrain trigger: 50+ new corrections OR 7 days elapsed
- Priority: High (direct user impact)

**Reconciliation Agent: Daily (Conditional)**
- New data: Daily accumulation
- Retrain trigger: 25+ new matches OR 7 days elapsed
- Priority: Medium (process efficiency)

**Insight Agent: Weekly**
- New data: Weekly aggregation
- Retrain trigger: 100+ new insights OR 30 days elapsed
- Priority: Low (analytics improvement)

**Chat Agent: Weekly**
- New data: Weekly aggregation
- Retrain trigger: 50+ new conversations OR 30 days elapsed
- Priority: Medium (user experience)

### B. Minimum Sample Requirements

```python
RETRAINING_SCHEDULE = {
    'classification': {
        'frequency': 'daily',
        'min_samples': 50,
        'max_age_days': 7,
        'validation_split': 0.2
    },
    'reconciliation': {
        'frequency': 'daily_conditional',
        'min_samples': 25,
        'max_age_days': 7,
        'validation_split': 0.2
    },
    'insight': {
        'frequency': 'weekly',
        'min_samples': 100,
        'max_age_days': 30,
        'validation_split': 0.2
    },
    'chat': {
        'frequency': 'weekly',
        'min_samples': 50,
        'max_age_days': 30,
        'validation_split': 0.2
    }
}
```

## 4. A/B Testing Framework

### A. Testing Implementation

```python
import random
from enum import Enum

class TestGroup(Enum):
    CONTROL = "control"
    TREATMENT = "treatment"

def route_prediction_request(input_data, agent_type):
    """
    Routes prediction requests to control or treatment groups
    """
    # Determine allocation probability based on agent type
    test_allocation = get_test_allocation(agent_type)

    if random.random() < test_allocation:
        group = TestGroup.TREATMENT
        model = get_latest_candidate_model(agent_type)
    else:
        group = TestGroup.CONTROL
        model = get_current_production_model(agent_type)

    # Make prediction
    prediction = model.predict(input_data)

    # Log for A/B testing analysis
    log_ab_test_result(input_data, prediction, group, model.version)

    return prediction

def evaluate_ab_test_results(agent_type, test_duration_days=7):
    """
    Evaluates A/B test results after specified duration
    """
    # Get test results
    control_results = get_test_results(agent_type, TestGroup.CONTROL)
    treatment_results = get_test_results(agent_type, TestGroup.TREATMENT)

    # Calculate metrics
    control_accuracy = calculate_accuracy(control_results)
    treatment_accuracy = calculate_accuracy(treatment_results)

    # Statistical significance test
    p_value = perform_statistical_test(control_results, treatment_results)

    # Decision logic
    if treatment_accuracy > control_accuracy and p_value < 0.05:
        return {
            'decision': 'promote',
            'reason': f'Treatment model better (accuracy: {treatment_accuracy:.4f} vs {control_accuracy:.4f})',
            'p_value': p_value
        }
    elif treatment_accuracy < control_accuracy * 0.98:  # Significant degradation
        return {
            'decision': 'reject',
            'reason': f'Treatment model worse (accuracy: {treatment_accuracy:.4f} vs {control_accuracy:.4f})',
            'p_value': p_value
        }
    else:
        return {
            'decision': 'continue_testing',
            'reason': 'No significant difference detected',
            'p_value': p_value
        }
```

### B. A/B Testing Configuration

```python
AB_TEST_CONFIG = {
    'allocation_ratio': 0.1,  # 10% to treatment group
    'minimum_sample_size': 1000,  # Minimum samples per variant
    'test_duration_days': 7,  # Duration before evaluation
    'success_threshold': 0.02,  # Minimum improvement for promotion
    'statistical_power': 0.8,  # Desired statistical power
    'alpha_level': 0.05  # Significance level
}
```

## 5. Performance Monitoring

### A. Metrics Schema

```sql
CREATE TABLE ai.model_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES ai.model_versions(id),
    metric_date DATE NOT NULL,
    total_predictions INTEGER DEFAULT 0,
    correct_predictions INTEGER DEFAULT 0,
    accuracy DECIMAL(5,4),
    precision DECIMAL(5,4),
    recall DECIMAL(5,4),
    f1_score DECIMAL(5,4),
    avg_confidence DECIMAL(3,2),
    user_overrides INTEGER DEFAULT 0,
    avg_response_time_ms DECIMAL(8,2),
    peak_memory_usage_mb INTEGER,
    cpu_utilization_percent DECIMAL(5,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Daily aggregates for performance tracking
CREATE TABLE ai.daily_performance (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_type VARCHAR(50) NOT NULL,
    model_version VARCHAR(20) NOT NULL,
    date DATE NOT NULL,
    accuracy_trend DECIMAL(5,4),
    confidence_stability DECIMAL(5,4),
    user_satisfaction_score DECIMAL(3,2),
    prediction_volume INTEGER,
    error_rate DECIMAL(5,4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_model_metrics_version_date ON ai.model_metrics(model_version_id, metric_date);
CREATE INDEX idx_daily_performance_date ON ai.daily_performance(date);
CREATE INDEX idx_daily_performance_agent ON ai.daily_performance(agent_type);
```

### B. Monitoring Implementation

```python
def update_model_metrics(model_version_id, prediction_result):
    """
    Updates model metrics based on prediction results
    """
    # Calculate metrics for this prediction
    is_correct = prediction_result.get('was_correct', False)
    confidence = prediction_result.get('confidence', 0.0)
    response_time = prediction_result.get('response_time_ms', 0)

    # Update daily metrics
    today = datetime.date.today()

    # Upsert metrics for today
    query = """
    INSERT INTO ai.model_metrics (
        model_version_id, metric_date, total_predictions,
        correct_predictions, accuracy, avg_confidence,
        user_overrides, avg_response_time_ms
    ) VALUES (%s, %s, 1, %s, %s, %s, %s, %s)
    ON CONFLICT (model_version_id, metric_date)
    DO UPDATE SET
        total_predictions = ai.model_metrics.total_predictions + 1,
        correct_predictions = ai.model_metrics.correct_predictions + CASE WHEN %s THEN 1 ELSE 0 END,
        accuracy = (ai.model_metrics.correct_predictions + CASE WHEN %s THEN 1 ELSE 0 END) * 1.0 /
                   (ai.model_metrics.total_predictions + 1),
        avg_confidence = (ai.model_metrics.avg_confidence * ai.model_metrics.total_predictions + %s) /
                         (ai.model_metrics.total_predictions + 1),
        user_overrides = ai.model_metrics.user_overrides + CASE WHEN %s THEN 1 ELSE 0 END,
        avg_response_time_ms = (ai.model_metrics.avg_response_time_ms * ai.model_metrics.total_predictions + %s) /
                               (ai.model_metrics.total_predictions + 1)
    """

    execute_query(query, (
        model_version_id, today, is_correct,
        1.0 if is_correct else 0.0, confidence,
        1 if prediction_result.get('was_overridden', False) else 0,
        response_time,
        is_correct, is_correct, confidence,
        1 if prediction_result.get('was_overridden', False) else 0,
        response_time
    ))
```

## 6. Rollback Policy

### A. Auto-Rollback Triggers

```python
def check_model_rollback_conditions(model_version_id):
    """
    Checks if model should be rolled back based on performance metrics
    """
    current_metrics = get_recent_model_metrics(model_version_id, days=7)
    baseline_metrics = get_baseline_metrics(model_version_id)

    rollback_triggers = []

    # Accuracy drop > 5%
    if current_metrics['accuracy'] < baseline_metrics['accuracy'] * 0.95:
        rollback_triggers.append({
            'trigger': 'accuracy_drop',
            'current': current_metrics['accuracy'],
            'baseline': baseline_metrics['accuracy'],
            'threshold': 0.95
        })

    # User override rate > 20%
    override_rate = current_metrics['user_overrides'] / current_metrics['total_predictions']
    if override_rate > 0.20:
        rollback_triggers.append({
            'trigger': 'high_override_rate',
            'current': override_rate,
            'threshold': 0.20
        })

    # Response time degradation > 50%
    if current_metrics['avg_response_time_ms'] > baseline_metrics['avg_response_time_ms'] * 1.5:
        rollback_triggers.append({
            'trigger': 'performance_degradation',
            'current': current_metrics['avg_response_time_ms'],
            'baseline': baseline_metrics['avg_response_time_ms'],
            'threshold': 1.5
        })

    if rollback_triggers:
        trigger_rollback(model_version_id, rollback_triggers)
        return True

    return False
```

### B. Rollback Process

```python
def trigger_rollback(model_version_id, triggers):
    """
    Executes model rollback process
    """
    # Alert ML team
    alert_ml_team(f"Rollback triggered for model {model_version_id}", triggers)

    # Get previous stable version
    previous_version = get_previous_stable_model(get_agent_type(model_version_id))

    if previous_version:
        # Deactivate current problematic model
        deactivate_model(model_version_id)

        # Activate previous stable model
        activate_model(previous_version['id'])

        # Log rollback event
        log_rollback_event(model_version_id, previous_version['id'], triggers)

        # Require investigation
        create_investigation_ticket(model_version_id, triggers)
    else:
        # No previous version available - emergency mode
        alert_operations_team("No previous model available for rollback")
```

## 7. Audit Trail System

### A. Prediction Logging Schema

```sql
CREATE TABLE ai.prediction_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES ai.model_versions(id),
    agent_type VARCHAR(50) NOT NULL,
    tenant_id UUID NOT NULL,
    input_data JSONB NOT NULL,
    predicted_output JSONB,
    actual_output JSONB,
    confidence_score DECIMAL(3,2),
    was_correct BOOLEAN,
    user_feedback TEXT,
    user_override BOOLEAN DEFAULT FALSE,
    override_reason TEXT,
    prediction_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    prediction_duration_ms DECIMAL(8,2),
    is_reproducible BOOLEAN DEFAULT TRUE
);

-- Indexes for audit trail
CREATE INDEX idx_prediction_log_tenant ON ai.prediction_log(tenant_id);
CREATE INDEX idx_prediction_log_model ON ai.prediction_log(model_version_id);
CREATE INDEX idx_prediction_log_timestamp ON ai.prediction_log(prediction_timestamp);
CREATE INDEX idx_prediction_log_agent ON ai.prediction_log(agent_type);
```

### B. Reproducibility Guarantee

```python
def log_prediction_with_reproducibility(input_data, model_version_id, prediction_result):
    """
    Logs prediction with full reproducibility information
    """
    # Generate unique prediction ID for reproducibility
    prediction_id = str(uuid.uuid4())

    # Store with full context for reproduction
    log_entry = {
        'prediction_id': prediction_id,
        'model_version_id': model_version_id,
        'input_data': input_data,
        'predicted_output': prediction_result['output'],
        'confidence_score': prediction_result['confidence'],
        'prediction_timestamp': datetime.utcnow().isoformat(),
        'model_parameters': get_model_parameters(model_version_id),
        'feature_extraction_params': get_feature_params(model_version_id),
        'environment_metadata': get_environment_metadata()
    }

    # Insert into audit table
    insert_prediction_log(log_entry)

    return prediction_id

def reproduce_prediction(prediction_id):
    """
    Reproduces a prediction using the exact same conditions
    """
    log_entry = get_prediction_log(prediction_id)

    # Recreate model with exact parameters
    model = recreate_model_from_version(
        log_entry['model_version_id'],
        log_entry['model_parameters']
    )

    # Apply exact same feature extraction
    processed_input = apply_feature_extraction(
        log_entry['input_data'],
        log_entry['feature_extraction_params']
    )

    # Make prediction
    reproduced_prediction = model.predict(processed_input)

    return {
        'original': log_entry['predicted_output'],
        'reproduced': reproduced_prediction,
        'is_identical': reproduced_prediction == log_entry['predicted_output']
    }
```

## 8. Training Pipeline Architecture

### A. Pipeline Components

```
Data Collection → Data Processing → Model Training → Validation → Deployment → Monitoring
       ↓              ↓                ↓            ↓          ↓           ↓
   Raw Data    →  Cleaned Data  →  Trained Model → Metrics → Production → Performance
   (Training)      (Features)       (Artifacts)    (Scores)   (Models)   (Tracking)
```

### B. Pipeline Implementation

```python
class TrainingPipeline:
    def __init__(self, agent_type):
        self.agent_type = agent_type
        self.config = RETRAINING_SCHEDULE[agent_type]

    def collect_new_training_data(self):
        """
        Collects new training data since last training cycle
        """
        last_training = get_last_training_time(self.agent_type)

        new_data = query_new_training_data(
            agent_type=self.agent_type,
            since=last_training
        )

        return new_data

    def preprocess_data(self, raw_data):
        """
        Preprocesses raw training data
        """
        processed_data = []

        for record in raw_data:
            # Scrub PII
            scrubbed_input = scrub_pii(record['input_data'])

            # Extract features
            features = extract_features(scrubbed_input, self.agent_type)

            processed_record = {
                'features': features,
                'expected_output': record['expected_output'],
                'was_correct': record['was_correct']
            }

            processed_data.append(processed_record)

        return processed_data

    def train_model(self, training_data):
        """
        Trains new model with provided data
        """
        # Split data
        train_set, val_set = split_data(training_data, test_size=0.2)

        # Initialize model
        model = initialize_model(self.agent_type)

        # Train model
        start_time = datetime.now()
        model.train(train_set['features'], train_set['expected_output'])
        training_duration = datetime.now() - start_time

        # Validate model
        val_predictions = model.predict(val_set['features'])
        validation_metrics = calculate_metrics(val_set['expected_output'], val_predictions)

        # Save model artifact
        model_artifact_path = save_model_artifact(model, self.agent_type)

        # Create version record
        version_record = {
            'agent_type': self.agent_type,
            'version': generate_next_version(self.agent_type),
            'trained_on': datetime.now(),
            'training_samples': len(train_set),
            'validation_samples': len(val_set),
            'accuracy_score': validation_metrics['accuracy'],
            'precision_score': validation_metrics['precision'],
            'recall_score': validation_metrics['recall'],
            'f1_score': validation_metrics['f1'],
            'training_duration': training_duration,
            'model_artifact_path': model_artifact_path,
            'training_config': self.config
        }

        version_id = create_model_version(version_record)

        return version_id, model

    def deploy_candidate_model(self, version_id):
        """
        Deploys candidate model for A/B testing
        """
        # Deploy to staging environment
        deploy_to_staging(version_id)

        # Start A/B testing
        start_ab_test(version_id, self.agent_type)

        return True
```

## 9. Data Privacy Controls

### A. Privacy Implementation

```python
class PrivacyManager:
    def __init__(self):
        self.pii_detectors = self.initialize_pii_detectors()

    def scrub_training_data(self, data_batch):
        """
        Scrubs PII from training data batch
        """
        scrubbed_batch = []

        for record in data_batch:
            scrubbed_record = {}

            # Process input data
            if isinstance(record.get('input_data'), dict):
                scrubbed_record['input_data'] = self.scrub_dict_pii(record['input_data'])
            elif isinstance(record.get('input_data'), str):
                scrubbed_record['input_data'] = self.scrub_text_pii(record['input_data'])
            else:
                scrubbed_record['input_data'] = record['input_data']

            # Process output data
            if isinstance(record.get('expected_output'), dict):
                scrubbed_record['expected_output'] = self.scrub_dict_pii(record['expected_output'])
            elif isinstance(record.get('expected_output'), str):
                scrubbed_record['expected_output'] = self.scrub_text_pii(record['expected_output'])
            else:
                scrubbed_record['expected_output'] = record['expected_output']

            scrubbed_batch.append(scrubbed_record)

        return scrubbed_batch

    def scrub_dict_pii(self, data_dict):
        """
        Scrubs PII from dictionary structure
        """
        scrubbed = {}

        for key, value in data_dict.items():
            if isinstance(value, str):
                scrubbed[key] = self.scrub_text_pii(value)
            elif isinstance(value, dict):
                scrubbed[key] = self.scrub_dict_pii(value)
            elif isinstance(value, list):
                scrubbed[key] = [self.scrub_text_pii(item) if isinstance(item, str) else item for item in value]
            else:
                scrubbed[key] = value

        return scrubbed

    def enforce_tenant_isolation(self, query_params):
        """
        Enforces tenant isolation in data access
        """
        if not query_params.get('tenant_id'):
            raise ValueError("Tenant ID required for data access")

        # Validate user has access to tenant
        if not user_has_tenant_access(query_params['user_id'], query_params['tenant_id']):
            raise PermissionError("User does not have access to this tenant")

        # Add tenant filter to query
        query_params['filters']['tenant_id'] = query_params['tenant_id']

        return query_params
```

## 10. Monitoring Dashboard Specification

### A. Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                     AI Model Performance Dashboard             │
├─────────────────────────────────────────────────────────────────┤
│  Current Models    │  Performance Trends    │  Alerts & Events │
│  ┌──────────────┐  │  ┌──────────────────┐  │  ┌─────────────┐  │
│  │ v1.2.3       │  │  │ Accuracy Chart   │  │  │ Critical    │  │
│  │ Active       │  │  │ Response Time    │  │  │ Warnings    │  │
│  │ 95.2% Acc    │  │  │ Confidence       │  │  │             │  │
│  └──────────────┘  │  └──────────────────┘  │  └─────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  Training Data    │  Model Versions       │  Resource Usage   │
│  ┌──────────────┐  │  ┌──────────────────┐  │  ┌─────────────┐  │
│  │ Daily Stats  │  │  │ Version History  │  │  │ CPU/Mem     │  │
│  │ New Samples  │  │  │ Rollback Events  │  │  │ Disk Space  │  │
│  │ Quality      │  │  │ A/B Test Status  │  │  │ Network IO  │  │
│  └──────────────┘  │  └──────────────────┘  │  └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### B. Dashboard Components

**Model Health Metrics:**
- Current accuracy scores
- Performance degradation alerts
- User satisfaction metrics
- Error rate monitoring

**Training Data Pipeline:**
- Daily sample collection
- Data quality metrics
- Privacy compliance status
- Cross-tenant isolation status

**Version Management:**
- Active model versions
- A/B test status
- Rollback events
- Performance comparisons

**Resource Utilization:**
- Model serving infrastructure
- Training pipeline resources
- Storage utilization
- Network bandwidth

### C. Real-time Monitoring

```python
DASHBOARD_WIDGETS = {
    'model_health': {
        'component': 'HealthMetrics',
        'refresh_interval': 30,
        'data_source': 'ai.model_metrics',
        'filters': {'metric_date': 'today'},
        'display_fields': ['accuracy', 'precision', 'recall', 'f1_score']
    },
    'training_data': {
        'component': 'TrainingDataMetrics',
        'refresh_interval': 60,
        'data_source': 'ai.training_data',
        'filters': {'created_at': 'today'},
        'display_fields': ['total_samples', 'privacy_compliance', 'data_quality']
    },
    'version_management': {
        'component': 'VersionControl',
        'refresh_interval': 120,
        'data_source': 'ai.model_versions',
        'filters': {'is_active': True},
        'display_fields': ['version', 'accuracy_score', 'is_production', 'trained_on']
    },
    'alerts': {
        'component': 'AlertPanel',
        'refresh_interval': 15,
        'data_source': 'ai.alerts',
        'filters': {'status': 'active'},
        'display_fields': ['alert_type', 'severity', 'timestamp', 'resolved_by']
    }
}
```

## 11. Performance Requirements

### A. Training Pipeline Performance
- **Data Collection**: Complete within 1 hour of trigger
- **Model Training**: Complete within 4 hours for 10K samples
- **Validation**: Complete within 30 minutes
- **Deployment**: Complete within 15 minutes

### B. Monitoring Performance
- **Metric Updates**: Real-time with <5 second delay
- **Alert Generation**: Within 1 minute of threshold breach
- **Dashboard Refresh**: Every 30 seconds for critical metrics
- **Historical Data**: 5-year retention for compliance

### C. Scalability Requirements
- **Concurrent Training Jobs**: Support 10 simultaneous jobs
- **Data Volume**: Handle 1M+ training samples per agent
- **Prediction Throughput**: 1000+ predictions per second
- **Storage**: Petabyte-scale training data storage