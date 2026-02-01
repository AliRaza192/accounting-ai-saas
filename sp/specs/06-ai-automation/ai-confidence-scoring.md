# AI Confidence Scoring & Error Handling Specification

## 1. Confidence Score Calculation

### A. Classification Agent Confidence

The classification agent calculates confidence scores using multiple factors:

```python
def calculate_confidence(transaction):
    """
    Calculates confidence score for transaction classification
    Returns float between 0.0 and 1.0
    """
    score = 0.0

    # Factor 1: Historical Match Analysis
    exact_matches = get_exact_historical_matches(transaction.description)
    if exact_matches:
        # Perfect historical match indicates high confidence
        if all(match.account == exact_matches[0].account for match in exact_matches):
            score = 1.0
        else:
            # Mixed historical matches indicate medium confidence
            unique_accounts = len(set(match.account for match in exact_matches))
            score = max(0.85, 1.0 - (unique_accounts * 0.05))

    # Factor 2: Fuzzy Matching
    elif fuzzy_similarity > 0.9:
        # High similarity to known patterns
        score = 0.85 + (fuzzy_similarity - 0.9) * 0.1
    elif fuzzy_similarity > 0.8:
        # Moderate similarity to known patterns
        score = 0.75 + (fuzzy_similarity - 0.8) * 0.1
    elif fuzzy_similarity > 0.7:
        # Low similarity but still recognizable
        score = 0.65 + (fuzzy_similarity - 0.7) * 0.1

    # Factor 3: Keyword Matching
    elif keywords_found:
        # Weighted by number and relevance of keywords
        keyword_score = min(0.8, (num_relevant_keywords / total_relevant_keywords) * 0.7)
        score = keyword_score

        # Boost for high-value keywords
        high_value_keywords = sum(1 for kw in keywords_found if kw.is_high_value)
        if high_value_keywords > 0:
            score += (high_value_keywords * 0.05)

    # Factor 4: AI Model Prediction
    else:
        # Base AI model prediction
        model_score = model.predict_proba(transaction)
        score = model_score

    # Adjustment Factors
    # Amount-based adjustment (higher amounts = lower confidence)
    if transaction.amount > 10000:
        score *= 0.95  # High-value transaction penalty
    elif transaction.amount > 5000:
        score *= 0.98  # Medium-value transaction penalty

    # Vendor-based adjustment
    if is_unrecognized_vendor(transaction.vendor):
        score *= 0.90  # Unknown vendor penalty

    # Category-based adjustment
    if transaction.category in ['uncertain', 'mixed']:
        score *= 0.85  # Uncertain category penalty

    # Time-based decay (recent patterns more reliable)
    if transaction.date > (datetime.now() - timedelta(days=30)):
        score *= 1.02  # Recent pattern bonus

    # Ensure score stays within bounds
    return round(max(0.0, min(1.0, score)), 2)
```

### B. OCR Agent Confidence

The OCR agent calculates confidence based on character recognition accuracy:

```python
def ocr_confidence(extracted_data):
    """
    Calculates confidence score for OCR-extracted data
    Returns float between 0.0 and 1.0
    """
    field_scores = []
    critical_field_penalty = 1.0

    for field in extracted_data.fields:
        # Character-level confidence from OCR engine
        char_confidence = sum(char.confidence for char in field.characters) / len(field.characters) if field.characters else 0.5
        field_scores.append(char_confidence)

        # Field-specific adjustments
        if field.type in ['amount', 'date', 'account_number']:  # Critical fields
            if char_confidence < 0.8:
                critical_field_penalty *= 0.8
        elif field.type in ['description', 'vendor']:  # Important fields
            if char_confidence < 0.7:
                critical_field_penalty *= 0.9

    # Overall confidence = weighted average of all fields
    if not field_scores:
        return 0.0

    overall = sum(field_scores) / len(field_scores)

    # Apply critical field penalty
    overall *= critical_field_penalty

    # Document quality factors
    if extracted_data.image_quality < 0.7:
        overall *= 0.85
    elif extracted_data.image_quality < 0.9:
        overall *= 0.95

    # Font clarity factor
    if extracted_data.font_clarity < 0.8:
        overall *= 0.90

    # Layout complexity factor
    if extracted_data.layout_complexity > 0.7:
        overall *= 0.92

    return round(max(0.0, min(1.0, overall)), 2)
```

### C. Reconciliation Agent Confidence

```python
def reconciliation_confidence(match_candidates):
    """
    Calculates confidence for transaction reconciliation matches
    """
    if len(match_candidates) == 0:
        return 0.0
    elif len(match_candidates) > 1:
        # Multiple candidates indicate lower confidence
        return 0.5 / len(match_candidates)
    else:
        primary_match = match_candidates[0]

        # Match strength factors
        amount_match = 1.0 if abs(primary_match.amount_diff) < 0.01 else 0.8
        date_match = calculate_date_proximity_score(primary_match.date_diff)
        description_similarity = calculate_text_similarity(
            primary_match.description1,
            primary_match.description2
        )

        # Combined confidence
        base_confidence = (amount_match * 0.4 + date_match * 0.3 + description_similarity * 0.3)

        # Additional penalties
        if primary_match.source_system != primary_match.target_system:
            base_confidence *= 0.95

        return round(base_confidence, 2)
```

## 2. Confidence Thresholds Decision Matrix

### A. Threshold Configuration

| Score Range | Action | User Experience | Business Impact |
|-------------|--------|-----------------|-----------------|
| 0.95 - 1.0  | Auto-apply | Seamless automation | High efficiency, minimal review |
| 0.80 - 0.94 | Suggest, require approval | Light review needed | Good efficiency, quality control |
| 0.60 - 0.79 | Flag for manual review | Moderate intervention | Balanced approach |
| 0.40 - 0.59 | Require manual verification | Heavy review needed | Safety first approach |
| 0.00 - 0.39 | Reject, require manual entry | Full manual intervention | Maximum safety |

### B. Dynamic Threshold Adjustment

```python
THRESHOLD_ADJUSTMENTS = {
    'by_risk_level': {
        'low': {'auto_apply': 0.85, 'suggest_approval': 0.70, 'manual_review': 0.50},
        'medium': {'auto_apply': 0.90, 'suggest_approval': 0.75, 'manual_review': 0.55},
        'high': {'auto_apply': 0.95, 'suggest_approval': 0.80, 'manual_review': 0.60}
    },
    'by_transaction_type': {
        'expense': {'auto_apply': 0.88, 'suggest_approval': 0.72, 'manual_review': 0.52},
        'revenue': {'auto_apply': 0.92, 'suggest_approval': 0.78, 'manual_review': 0.60},
        'transfer': {'auto_apply': 0.90, 'suggest_approval': 0.75, 'manual_review': 0.55}
    },
    'by_company_size': {
        'small': {'auto_apply': 0.85, 'suggest_approval': 0.70, 'manual_review': 0.50},
        'medium': {'auto_apply': 0.90, 'suggest_approval': 0.75, 'manual_review': 0.55},
        'enterprise': {'auto_apply': 0.95, 'suggest_approval': 0.80, 'manual_review': 0.60}
    }
}

def get_dynamic_thresholds(company_profile):
    """
    Returns appropriate thresholds based on company risk profile
    """
    risk_level = company_profile.risk_tolerance
    transaction_type = company_profile.primary_transaction_type
    company_size = company_profile.size_category

    base_thresholds = THRESHOLD_ADJUSTMENTS['by_risk_level'][risk_level]

    # Apply transaction type multiplier
    type_multipliers = {
        'auto_apply': THRESHOLD_ADJUSTMENTS['by_transaction_type'][transaction_type]['auto_apply'] / 0.90,
        'suggest_approval': THRESHOLD_ADJUSTMENTS['by_transaction_type'][transaction_type]['suggest_approval'] / 0.75,
        'manual_review': THRESHOLD_ADJUSTMENTS['by_transaction_type'][transaction_type]['manual_review'] / 0.55
    }

    # Apply size adjustment
    size_multipliers = {
        'auto_apply': THRESHOLD_ADJUSTMENTS['by_company_size'][company_size]['auto_apply'] / 0.90,
        'suggest_approval': THRESHOLD_ADJUSTMENTS['by_company_size'][company_size]['suggest_approval'] / 0.75,
        'manual_review': THRESHOLD_ADJUSTMENTS['by_company_size'][company_size]['manual_review'] / 0.55
    }

    # Combine adjustments
    adjusted_thresholds = {}
    for level in base_thresholds:
        multiplier_key = level.replace('_', '')
        combined_multiplier = (type_multipliers[level] + size_multipliers[level]) / 2
        adjusted_thresholds[level] = base_thresholds[level] * combined_multiplier

        # Ensure bounds
        adjusted_thresholds[level] = max(0.0, min(1.0, adjusted_thresholds[level]))

    return adjusted_thresholds
```

## 3. Error Handling Framework

### A. AI Service Unavailability

```python
import asyncio
import logging
from typing import Optional
from enum import Enum

class AIServiceStatus(Enum):
    OPERATIONAL = "operational"
    DEGRADED = "degraded"
    UNAVAILABLE = "unavailable"
    OVERLOADED = "overloaded"

class AIServiceHandler:
    def __init__(self):
        self.fallback_service = RuleBasedClassifier()
        self.health_monitor = HealthMonitor()
        self.circuit_breaker = CircuitBreaker()

    async def classify_with_fallback(self, transaction) -> dict:
        """
        Primary classification with fallback mechanism
        """
        try:
            # Check service health first
            if self.health_monitor.get_status() != AIServiceStatus.OPERATIONAL:
                # Use circuit breaker to prevent cascade failures
                if self.circuit_breaker.should_allow_request():
                    result = await self.ai_service.classify(transaction)

                    # Update service health metrics
                    self.health_monitor.record_success()
                    self.circuit_breaker.record_success()

                    return self._enrich_result(result, "primary_ai")
                else:
                    # Circuit breaker tripped, use fallback immediately
                    logging.warning("Circuit breaker active, using fallback")
                    return await self._use_fallback(transaction)
            else:
                result = await self.ai_service.classify(transaction)
                self.health_monitor.record_success()
                return self._enrich_result(result, "primary_ai")

        except (ServiceUnavailable, ConnectionError) as e:
            logging.error(f"AI service unavailable: {str(e)}")
            self.health_monitor.record_failure()
            self.circuit_breaker.record_failure()

            # Use fallback service
            return await self._use_fallback(transaction)

        except RateLimitExceeded as e:
            logging.warning(f"Rate limit exceeded: {str(e)}")
            self.health_monitor.record_degraded()

            # Use fallback if overloaded
            if self.health_monitor.get_status() == AIServiceStatus.OVERLOADED:
                return await self._use_fallback(transaction)
            else:
                # Wait and retry
                await asyncio.sleep(1)
                return await self.classify_with_fallback(transaction)

    async def _use_fallback(self, transaction) -> dict:
        """
        Fallback to rule-based classification
        """
        result = self.fallback_service.classify(transaction)

        return {
            'suggested_account': result.get('account'),
            'confidence': 0.6,  # Lower confidence for fallback
            'method': 'fallback_rules',
            'requires_review': True,  # Always review fallback results
            'fallback_reason': 'ai_service_unavailable'
        }

    def _enrich_result(self, result: dict, method: str) -> dict:
        """
        Add metadata to AI result
        """
        return {
            **result,
            'method': method,
            'requires_review': result.get('confidence', 0.0) < 0.8,
            'processing_time': result.get('processing_time'),
            'model_version': result.get('model_version')
        }
```

### B. Timeout Handling

```python
import asyncio
import uuid
from datetime import datetime, timedelta
from typing import Dict, Any

class TimeoutHandler:
    def __init__(self):
        self.background_queue = BackgroundProcessingQueue()
        self.timeout_config = {
            'classification': 5.0,  # 5 seconds
            'ocr': 8.0,            # 8 seconds
            'reconciliation': 10.0  # 10 seconds
        }

    async def process_with_timeout(self, transaction: dict, agent_type: str) -> dict:
        """
        Process transaction with timeout handling
        """
        timeout = self.timeout_config.get(agent_type, 5.0)

        try:
            # Create timeout task
            result_task = asyncio.create_task(
                self.ai_service.process(transaction, agent_type)
            )

            # Wait for result with timeout
            result = await asyncio.wait_for(result_task, timeout=timeout)

            return {
                'status': 'completed',
                'result': result,
                'processing_time': timeout - result_task.get_coro().__name__
            }

        except asyncio.TimeoutError:
            # Cancel the timed-out task
            result_task.cancel()

            # Queue for background processing
            job_id = str(uuid.uuid4())
            queue_entry = {
                'job_id': job_id,
                'transaction': transaction,
                'agent_type': agent_type,
                'submitted_at': datetime.now(),
                'retry_count': 0
            }

            await self.background_queue.add(queue_entry)

            return {
                'status': 'queued',
                'job_id': job_id,
                'message': 'Processing in background, will notify when complete',
                'estimated_completion': str(datetime.now() + timedelta(minutes=2))
            }

        except Exception as e:
            logging.error(f"Unexpected error in timeout handler: {str(e)}")
            # Fallback to manual processing
            return {
                'status': 'manual_required',
                'error': str(e),
                'requires_manual_intervention': True
            }

class BackgroundProcessingQueue:
    """
    Manages background processing of timed-out requests
    """
    def __init__(self):
        self.jobs = {}
        self.max_retries = 3
        self.retry_delay = 30  # seconds

    async def add(self, job: dict):
        """
        Add job to background processing queue
        """
        job_id = job['job_id']
        self.jobs[job_id] = job

        # Schedule background processing
        asyncio.create_task(self._process_job(job_id))

    async def _process_job(self, job_id: str):
        """
        Process job in background with retry logic
        """
        job = self.jobs.get(job_id)
        if not job:
            return

        for attempt in range(self.max_retries + 1):
            try:
                result = await self.ai_service.process(
                    job['transaction'],
                    job['agent_type']
                )

                # Notify user of completion
                await self._notify_completion(job_id, result)
                del self.jobs[job_id]
                break

            except Exception as e:
                job['retry_count'] += 1
                if job['retry_count'] <= self.max_retries:
                    await asyncio.sleep(self.retry_delay)
                    continue
                else:
                    # Final failure, notify user
                    await self._notify_failure(job_id, str(e))
                    del self.jobs[job_id]
                    break
```

### C. Invalid Response Handling

```python
import json
import re
from typing import Dict, Any, List

class ResponseValidator:
    def __init__(self):
        self.schema_validators = {
            'classification': self._validate_classification_response,
            'ocr': self._validate_ocr_response,
            'reconciliation': self._validate_reconciliation_response
        }

    def validate_response(self, response: Dict[str, Any], agent_type: str) -> tuple[bool, List[str]]:
        """
        Validates AI response format and content
        Returns (is_valid, list_of_errors)
        """
        if not response:
            return False, ["Response is None or empty"]

        if not isinstance(response, dict):
            return False, ["Response is not a dictionary"]

        validator = self.schema_validators.get(agent_type)
        if not validator:
            return False, [f"No validator for agent type: {agent_type}"]

        return validator(response)

    def _validate_classification_response(self, response: Dict[str, Any]) -> tuple[bool, List[str]]:
        """
        Validates classification response
        """
        errors = []

        # Check required fields
        required_fields = ['suggested_account', 'confidence']
        for field in required_fields:
            if field not in response:
                errors.append(f"Missing required field: {field}")

        # Validate confidence score
        confidence = response.get('confidence')
        if confidence is not None:
            if not isinstance(confidence, (int, float)) or not 0.0 <= confidence <= 1.0:
                errors.append(f"Invalid confidence score: {confidence}. Must be between 0.0 and 1.0")

        # Validate account structure
        suggested_account = response.get('suggested_account')
        if suggested_account:
            if isinstance(suggested_account, dict):
                if 'code' not in suggested_account and 'name' not in suggested_account:
                    errors.append("Account must have either code or name field")
            elif not isinstance(suggested_account, str):
                errors.append("Account must be string or dictionary")

        return len(errors) == 0, errors

    def _validate_ocr_response(self, response: Dict[str, Any]) -> tuple[bool, List[str]]:
        """
        Validates OCR response
        """
        errors = []

        # Check required fields
        if 'extracted_data' not in response:
            errors.append("Missing required field: extracted_data")

        # Validate confidence if present
        confidence = response.get('confidence')
        if confidence is not None:
            if not isinstance(confidence, (int, float)) or not 0.0 <= confidence <= 1.0:
                errors.append(f"Invalid confidence score: {confidence}. Must be between 0.0 and 1.0")

        # Validate extracted data structure
        extracted_data = response.get('extracted_data', {})
        if not isinstance(extracted_data, dict):
            errors.append("Extracted data must be a dictionary")

        return len(errors) == 0, errors

    def _validate_reconciliation_response(self, response: Dict[str, Any]) -> tuple[bool, List[str]]:
        """
        Validates reconciliation response
        """
        errors = []

        # Check required fields
        required_fields = ['matches', 'confidence']
        for field in required_fields:
            if field not in response:
                errors.append(f"Missing required field: {field}")

        # Validate confidence
        confidence = response.get('confidence')
        if confidence is not None:
            if not isinstance(confidence, (int, float)) or not 0.0 <= confidence <= 1.0:
                errors.append(f"Invalid confidence score: {confidence}. Must be between 0.0 and 1.0")

        # Validate matches structure
        matches = response.get('matches', [])
        if not isinstance(matches, list):
            errors.append("Matches must be a list")

        for i, match in enumerate(matches):
            if not isinstance(match, dict):
                errors.append(f"Match at index {i} is not a dictionary")
                continue

            if 'transaction_a' not in match or 'transaction_b' not in match:
                errors.append(f"Match at index {i} missing required transaction fields")

        return len(errors) == 0, errors

def handle_invalid_response(response: Dict[str, Any], agent_type: str) -> Dict[str, Any]:
    """
    Handles invalid AI response and returns safe fallback
    """
    validator = ResponseValidator()
    is_valid, errors = validator.validate_response(response, agent_type)

    if is_valid:
        return response

    # Log the invalid response for debugging
    logging.error(f"Invalid AI response for {agent_type}: {errors}", extra={
        'response': response,
        'errors': errors
    })

    # Return safe fallback response
    fallback_response = {
        'suggested_account': None,
        'confidence': 0.0,
        'requires_manual_review': True,
        'error': f"AI returned invalid response: {', '.join(errors)}",
        'method': 'invalid_response_fallback'
    }

    # Trigger alert for ML team
    alert_ml_team_invalid_response(agent_type, errors, response)

    return fallback_response
```

## 4. User Feedback Loop System

### A. Correction Tracking

```python
import uuid
from datetime import datetime
from typing import Dict, Any

class FeedbackLoopManager:
    def __init__(self):
        self.training_data_store = TrainingDataStore()
        self.feedback_analyzer = FeedbackAnalyzer()

    async def on_user_correction(self, transaction_id: str, original_prediction: Dict[str, Any],
                                corrected_value: Any, user_id: str) -> Dict[str, str]:
        """
        Handles user corrections and adds to training data
        """
        # Retrieve original transaction data
        transaction = await get_transaction_by_id(transaction_id)

        # Create training data entry
        training_entry = {
            'id': str(uuid.uuid4()),
            'input': transaction,
            'ai_output': original_prediction.get('suggested_account'),
            'correct_output': corrected_value,
            'was_correct': False,
            'confidence_at_prediction': original_prediction.get('confidence', 0.0),
            'user_id': user_id,
            'correction_timestamp': datetime.now(),
            'original_method': original_prediction.get('method', 'unknown'),
            'feedback_context': {
                'user_role': await get_user_role(user_id),
                'company_id': transaction.get('company_id'),
                'transaction_type': transaction.get('type')
            }
        }

        # Store for training
        await self.training_data_store.add(training_entry)

        # Analyze feedback pattern
        feedback_analysis = await self.feedback_analyzer.analyze_pattern(training_entry)

        # Update confidence model if significant pattern detected
        if feedback_analysis.get('significant_pattern'):
            await self._update_confidence_model(feedback_analysis)

        # Return success message
        return {
            'status': 'success',
            'message': f"Thanks! AI will learn from this correction. Pattern: {feedback_analysis.get('pattern_type', 'general')}"
        }

    async def on_user_confirmation(self, transaction_id: str, prediction: Dict[str, Any],
                                  user_id: str) -> Dict[str, str]:
        """
        Handles positive feedback when user accepts AI suggestion
        """
        transaction = await get_transaction_by_id(transaction_id)

        training_entry = {
            'id': str(uuid.uuid4()),
            'input': transaction,
            'ai_output': prediction.get('suggested_account'),
            'correct_output': prediction.get('suggested_account'),
            'was_correct': True,
            'confidence_at_prediction': prediction.get('confidence', 0.0),
            'user_id': user_id,
            'confirmation_timestamp': datetime.now(),
            'original_method': prediction.get('method', 'unknown')
        }

        await self.training_data_store.add(training_entry)

        return {
            'status': 'success',
            'message': "Thank you for confirming! This helps improve AI accuracy."
        }

    async def _update_confidence_model(self, analysis: Dict[str, Any]):
        """
        Updates confidence scoring based on feedback patterns
        """
        pattern_type = analysis.get('pattern_type')
        confidence_adjustment = analysis.get('confidence_adjustment', {})

        if pattern_type == 'low_confidence_correct':
            # User frequently confirms low-confidence predictions
            # Consider lowering confidence thresholds
            await adjust_confidence_thresholds('lower')
        elif pattern_type == 'high_confidence_wrong':
            # AI frequently makes wrong predictions with high confidence
            # Need to recalibrate confidence scoring
            await trigger_model_recalibration()
        elif pattern_type == 'keyword_pattern':
            # Specific keyword patterns leading to corrections
            # Update keyword-based scoring
            await update_keyword_scoring(analysis.get('keywords'))

class TrainingDataStore:
    """
    Stores and manages training data from user feedback
    """
    def __init__(self):
        self.data_buffer = []
        self.batch_size = 100
        self.retrain_threshold = 1000  # Retrain when 1000+ new samples

    async def add(self, entry: Dict[str, Any]):
        """
        Adds training data entry to buffer
        """
        self.data_buffer.append(entry)

        # Check if we should trigger retraining
        if len(self.data_buffer) >= self.batch_size:
            await self._process_batch()

        # Check if we've reached retraining threshold
        total_samples = await self.get_total_samples()
        if total_samples >= self.retrain_threshold:
            await trigger_model_retraining()

    async def _process_batch(self):
        """
        Process buffered training data
        """
        batch = self.data_buffer[:self.batch_size]
        self.data_buffer = self.data_buffer[self.batch_size:]

        # Store in persistent storage
        await store_training_data_batch(batch)

        # Update statistics
        await update_training_statistics(batch)
```

### B. Feedback Analysis

```python
from collections import defaultdict, Counter
import numpy as np

class FeedbackAnalyzer:
    def __init__(self):
        self.pattern_detectors = {
            'confidence_calibration': self._detect_confidence_patterns,
            'keyword_corrections': self._detect_keyword_patterns,
            'vendor_specific': self._detect_vendor_patterns,
            'time_based': self._detect_time_patterns
        }

    async def analyze_pattern(self, feedback_entry: Dict[str, Any]) -> Dict[str, Any]:
        """
        Analyzes feedback entry for patterns
        """
        analysis = {
            'patterns_detected': [],
            'confidence_adjustment': {},
            'recommended_actions': []
        }

        for pattern_type, detector in self.pattern_detectors.items():
            pattern_result = await detector(feedback_entry)
            if pattern_result.get('detected'):
                analysis['patterns_detected'].append(pattern_type)
                analysis.update(pattern_result)

        # Determine if pattern is significant enough to act on
        analysis['significant_pattern'] = len(analysis['patterns_detected']) > 0

        return analysis

    async def _detect_confidence_patterns(self, feedback_entry: Dict[str, Any]) -> Dict[str, Any]:
        """
        Detects patterns in confidence scoring
        """
        confidence = feedback_entry['confidence_at_prediction']
        was_correct = feedback_entry['was_correct']

        # Track confidence correctness rates
        confidence_bucket = int(confidence * 10)  # Group into 0.1 buckets
        bucket_stats = await get_confidence_bucket_stats(confidence_bucket)

        # Calculate calibration error
        calibration_error = abs(bucket_stats['actual_accuracy'] - confidence_bucket / 10.0)

        if calibration_error > 0.1:  # 10% calibration error threshold
            return {
                'detected': True,
                'pattern_type': 'confidence_miscalibration',
                'calibration_error': calibration_error,
                'recommended_action': 'recalibrate_confidence_scoring'
            }

        return {'detected': False}

    async def _detect_keyword_patterns(self, feedback_entry: Dict[str, Any]) -> Dict[str, Any]:
        """
        Detects keyword patterns in corrections
        """
        transaction = feedback_entry['input']
        description = transaction.get('description', '').lower()
        original_prediction = feedback_entry['ai_output']
        correct_prediction = feedback_entry['correct_output']

        # Extract keywords from description
        keywords = extract_keywords(description)

        # Check if this represents a common misclassification pattern
        pattern_key = f"{original_prediction}_{correct_prediction}"
        keyword_patterns = await get_keyword_patterns(pattern_key)

        significant_keywords = []
        for keyword in keywords:
            if keyword in keyword_patterns:
                frequency = keyword_patterns[keyword]
                if frequency > 5:  # At least 5 occurrences
                    significant_keywords.append(keyword)

        if significant_keywords:
            return {
                'detected': True,
                'pattern_type': 'keyword_pattern',
                'keywords': significant_keywords,
                'original_prediction': original_prediction,
                'correct_prediction': correct_prediction,
                'recommended_action': 'update_keyword_weights'
            }

        return {'detected': False}
```

## 5. Transparency & Explainability

### A. Detailed Response Format

```json
{
  "suggested_account": {
    "code": "6500",
    "name": "Office Supplies"
  },
  "confidence": 0.88,
  "reasoning": {
    "method": "fuzzy_match",
    "confidence_factors": {
      "historical_matches": 0.92,
      "keyword_relevance": 0.85,
      "vendor_recognition": 0.80,
      "amount_consistency": 0.75
    },
    "matched_transactions": 12,
    "similarity_score": 0.91,
    "keywords_found": ["office", "supplies", "stationery"],
    "supporting_evidence": [
      {
        "type": "historical_match",
        "description": "Similar transaction from last month",
        "confidence": 0.95
      },
      {
        "type": "keyword_match",
        "term": "office supplies",
        "weight": 0.85
      }
    ]
  },
  "alternatives": [
    {
      "account": "6600 - Shipping Expense",
      "confidence": 0.15,
      "reason": "Contains 'shipping' keyword but context suggests office supplies"
    },
    {
      "account": "6400 - Travel Expense",
      "confidence": 0.08,
      "reason": "Low relevance to office supplies context"
    }
  ],
  "requires_review": false,
  "processing_time_ms": 45,
  "model_version": "classification-v1.2.3",
  "explanation": "High confidence due to strong historical pattern and keyword match"
}
```

### B. Explainability Engine

```python
class ExplainabilityEngine:
    def __init__(self):
        self.reasoning_templates = {
            'exact_match': "Exact historical match found for this vendor and amount",
            'fuzzy_match': "Strong similarity to known transaction patterns",
            'keyword_match': "Key terms in description suggest this account",
            'rule_based': "Applied business rules based on transaction characteristics",
            'ml_prediction': "Machine learning model prediction based on multiple factors"
        }

    def generate_explanation(self, prediction_result: Dict[str, Any]) -> Dict[str, Any]:
        """
        Generates human-readable explanation for AI prediction
        """
        confidence = prediction_result.get('confidence', 0.0)
        method = prediction_result.get('method', 'unknown')

        explanation_parts = []

        # Add method-based explanation
        if method in self.reasoning_templates:
            explanation_parts.append(self.reasoning_templates[method])

        # Add confidence-specific explanation
        if confidence >= 0.95:
            explanation_parts.append("Very high confidence - safe to auto-apply")
        elif confidence >= 0.80:
            explanation_parts.append("High confidence - suggested for approval")
        elif confidence >= 0.60:
            explanation_parts.append("Moderate confidence - manual review recommended")
        else:
            explanation_parts.append("Low confidence - requires manual verification")

        # Add supporting details
        reasoning = prediction_result.get('reasoning', {})
        if 'matched_transactions' in reasoning:
            count = reasoning['matched_transactions']
            explanation_parts.append(f"Based on {count} similar historical transactions")

        if 'keywords_found' in reasoning:
            keywords = ', '.join(reasoning['keywords_found'][:3])  # Top 3 keywords
            explanation_parts.append(f"Keywords detected: {keywords}")

        return {
            'explanation': '; '.join(explanation_parts),
            'confidence_level': self._get_confidence_level(confidence),
            'action_recommendation': self._get_action_recommendation(confidence),
            'key_factors': self._extract_key_factors(reasoning)
        }

    def _get_confidence_level(self, confidence: float) -> str:
        """
        Maps confidence score to descriptive level
        """
        if confidence >= 0.95:
            return "very_high"
        elif confidence >= 0.80:
            return "high"
        elif confidence >= 0.60:
            return "moderate"
        elif confidence >= 0.40:
            return "low"
        else:
            return "very_low"

    def _get_action_recommendation(self, confidence: float) -> str:
        """
        Gets recommended action based on confidence
        """
        if confidence >= 0.95:
            return "auto_apply"
        elif confidence >= 0.80:
            return "suggest_approval"
        elif confidence >= 0.60:
            return "manual_review"
        else:
            return "manual_verification"

    def _extract_key_factors(self, reasoning: Dict[str, Any]) -> List[Dict[str, Any]]:
        """
        Extracts key factors that influenced the decision
        """
        factors = []

        if 'confidence_factors' in reasoning:
            for factor, score in reasoning['confidence_factors'].items():
                factors.append({
                    'factor': factor,
                    'score': score,
                    'impact': self._get_factor_impact(score)
                })

        return factors

    def _get_factor_impact(self, score: float) -> str:
        """
        Determines impact level of a factor
        """
        if score >= 0.9:
            return "high"
        elif score >= 0.7:
            return "medium"
        elif score >= 0.5:
            return "low"
        else:
            return "minimal"
```

## 6. Monitoring & Alerting System

### A. Performance Metrics

```python
from datetime import datetime, timedelta
from typing import Dict, List
import asyncio

class AIMonitoringSystem:
    def __init__(self):
        self.metrics_store = MetricsStore()
        self.alert_manager = AlertManager()
        self.dashboard_generator = DashboardGenerator()

    async def collect_metrics(self) -> Dict[str, float]:
        """
        Collects daily AI performance metrics
        """
        yesterday = datetime.now() - timedelta(days=1)

        # Query metrics from database
        daily_stats = await self.metrics_store.get_daily_stats(yesterday)

        metrics = {
            'accuracy': daily_stats.get('accuracy', 0.0),
            'avg_confidence': daily_stats.get('avg_confidence', 0.0),
            'error_rate': daily_stats.get('error_rate', 0.0),
            'throughput': daily_stats.get('transactions_processed', 0),
            'avg_processing_time': daily_stats.get('avg_processing_time', 0.0),
            'fallback_rate': daily_stats.get('fallback_usage_rate', 0.0),
            'user_satisfaction': daily_stats.get('user_acceptance_rate', 0.0),
            'feedback_volume': daily_stats.get('user_feedback_count', 0)
        }

        return metrics

    async def monitor_and_alert(self):
        """
        Monitors metrics and sends alerts for anomalies
        """
        metrics = await self.collect_metrics()

        # Check accuracy threshold
        if metrics['accuracy'] < 0.85:
            await self.alert_manager.send_alert(
                severity='HIGH',
                message=f"AI accuracy dropped to {metrics['accuracy']:.2%}",
                category='accuracy_degradation',
                metrics=metrics
            )

        # Check confidence threshold
        if metrics['avg_confidence'] < 0.75:
            await self.alert_manager.send_alert(
                severity='MEDIUM',
                message=f"Average AI confidence is low: {metrics['avg_confidence']:.2%}",
                category='confidence_issue',
                metrics=metrics
            )

        # Check error rate threshold
        if metrics['error_rate'] > 0.05:  # 5% error rate
            await self.alert_manager.send_alert(
                severity='HIGH',
                message=f"AI error rate elevated to {metrics['error_rate']:.2%}",
                category='service_health',
                metrics=metrics
            )

        # Check throughput anomalies
        historical_avg = await self.metrics_store.get_historical_average('throughput')
        if metrics['throughput'] < historical_avg * 0.7:  # 30% below average
            await self.alert_manager.send_alert(
                severity='MEDIUM',
                message=f"AI throughput significantly reduced: {metrics['throughput']} vs avg {historical_avg}",
                category='performance',
                metrics=metrics
            )

        # Check user satisfaction
        if metrics['user_satisfaction'] < 0.80:  # 80% acceptance rate
            await self.alert_manager.send_alert(
                severity='LOW',
                message=f"User satisfaction with AI suggestions declining: {metrics['user_satisfaction']:.2%}",
                category='user_experience',
                metrics=metrics
            )

class MetricsStore:
    """
    Stores and retrieves AI performance metrics
    """
    async def get_daily_stats(self, date: datetime) -> Dict[str, Any]:
        """
        Gets daily performance statistics
        """
        # SQL query to aggregate daily metrics
        query = """
        SELECT
            AVG(CASE WHEN ai_was_correct THEN 1.0 ELSE 0.0 END) as accuracy,
            AVG(ai_confidence) as avg_confidence,
            COUNT(CASE WHEN ai_error IS NOT NULL THEN 1 END) * 1.0 / COUNT(*) as error_rate,
            COUNT(*) as transactions_processed,
            AVG(ai_processing_time) as avg_processing_time,
            COUNT(CASE WHEN ai_method = 'fallback_rules' THEN 1 END) * 1.0 / COUNT(*) as fallback_usage_rate,
            COUNT(CASE WHEN user_accepted_suggestion THEN 1 END) * 1.0 / COUNT(*) as user_acceptance_rate,
            COUNT(CASE WHEN user_provided_feedback THEN 1 END) as user_feedback_count
        FROM ai_predictions
        WHERE DATE(prediction_timestamp) = %s
        """

        return await execute_query(query, (date.date(),))

    async def get_historical_average(self, metric: str) -> float:
        """
        Gets historical average for a specific metric
        """
        query = f"""
        SELECT AVG({metric}) as avg_value
        FROM ai_daily_metrics
        WHERE metric_date >= NOW() - INTERVAL '30 days'
        """

        result = await execute_query(query)
        return result[0]['avg_value'] if result else 0.0

class AlertManager:
    """
    Manages alert generation and distribution
    """
    def __init__(self):
        self.alert_channels = {
            'ml_team': ['email', 'slack'],
            'product_team': ['email', 'dashboard'],
            'ops_team': ['pagerduty', 'slack'],
            'executives': ['email_weekly_digest']
        }

    async def send_alert(self, severity: str, message: str, category: str, metrics: Dict[str, float]):
        """
        Sends alert to appropriate teams based on severity and category
        """
        # Determine recipients based on severity and category
        recipients = self._get_recipients(severity, category)

        # Send alerts through configured channels
        for recipient_group, channels in recipients.items():
            for channel in channels:
                await self._send_via_channel(channel, recipient_group, severity, message, metrics)

        # Log alert for audit trail
        await self._log_alert(severity, message, category, metrics)

    def _get_recipients(self, severity: str, category: str) -> Dict[str, List[str]]:
        """
        Determines who should receive the alert
        """
        recipients = {}

        if severity in ['HIGH', 'CRITICAL']:
            recipients.update({
                'ml_team': ['email', 'slack'],
                'ops_team': ['pagerduty', 'slack']
            })
        elif severity == 'MEDIUM':
            recipients.update({
                'ml_team': ['email'],
                'product_team': ['email']
            })
        else:  # LOW
            recipients.update({
                'product_team': ['email'],
                'ml_team': ['dashboard']
            })

        return recipients
```

### B. Monitoring Dashboard Specification

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           AI Performance Dashboard                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│  System Health    │  Confidence Metrics    │  Error Tracking    │  User Feedback │
│  ┌─────────────┐  │  ┌─────────────────┐  │  ┌──────────────┐  │  ┌───────────┐  │
│  │ Operational │  │  │ Avg Confidence  │  │  │ Error Rate   │  │  │ Acceptance│  │
│  │ ✓ All Green │  │  │ ████████░░ 82% │  │  │ ███░░░░░░░░ 3%│  │  │███████░░░ │  │
│  │ Response:45ms│  │  │ Threshold:80%   │  │  │ Threshold:5% │  │  │ 78%       │  │
│  └─────────────┘  │  └─────────────────┘  │  └──────────────┘  │  └───────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Daily Trends         │  Model Performance    │  Processing Stats    │  Alerts   │
│  ┌─────────────────┐  │  ┌─────────────────┐  │  ┌────────────────┐  │  ┌───────┐  │
│  │ Accuracy Trend  │  │  │ Classification  │  │  │ Throughput:    │  │  │🔴Crit │  │
│  │ ▲▲▲▲▲▲▲▲▲▲▲▲▲   │  │  │ ████████░░ 92% │  │  │ 1,245/hr       │  │  │🟡Warn │  │
│  │ 95% → 88% → 92% │  │  │ OCR ████████░░  │  │  │ Errors: 34     │  │  │🟢OK   │  │
│  └─────────────────┘  │  │ 85%            │  │  │ Latency: 45ms  │  │  │       │  │
│                       │  │ Reconcile      │  │  └────────────────┘  │  └───────┘  │
│                       │  │ ███████░░░ 78% │  │                       │             │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### C. Dashboard Components

```python
DASHBOARD_COMPONENTS = {
    'health_summary': {
        'component': 'HealthSummary',
        'refresh_interval': 30,  # seconds
        'data_source': 'ai.health_metrics',
        'display_fields': [
            'overall_status',
            'response_time',
            'uptime_percentage',
            'active_alerts'
        ]
    },
    'confidence_tracker': {
        'component': 'ConfidenceTracker',
        'refresh_interval': 60,
        'data_source': 'ai.confidence_metrics',
        'display_fields': [
            'current_avg_confidence',
            'confidence_trend',
            'threshold_status',
            'confidence_distribution'
        ]
    },
    'error_monitor': {
        'component': 'ErrorMonitor',
        'refresh_interval': 30,
        'data_source': 'ai.error_metrics',
        'display_fields': [
            'current_error_rate',
            'error_trend',
            'error_types',
            'error_severity'
        ]
    },
    'performance_trends': {
        'component': 'PerformanceTrends',
        'refresh_interval': 120,
        'data_source': 'ai.performance_history',
        'display_fields': [
            'accuracy_trend',
            'processing_time_trend',
            'throughput_trend',
            'user_satisfaction_trend'
        ]
    },
    'model_comparison': {
        'component': 'ModelComparison',
        'refresh_interval': 300,
        'data_source': 'ai.model_metrics',
        'display_fields': [
            'model_versions',
            'accuracy_comparison',
            'confidence_comparison',
            'performance_comparison'
        ]
    },
    'real_time_processing': {
        'component': 'RealTimeProcessing',
        'refresh_interval': 10,
        'data_source': 'ai.live_processing',
        'display_fields': [
            'current_throughput',
            'queue_depth',
            'active_requests',
            'processing_times'
        ]
    }
}

class DashboardGenerator:
    """
    Generates dashboard data and visualizations
    """
    def __init__(self):
        self.component_handlers = {
            'HealthSummary': self._generate_health_summary,
            'ConfidenceTracker': self._generate_confidence_data,
            'ErrorMonitor': self._generate_error_data,
            'PerformanceTrends': self._generate_trend_data,
            'ModelComparison': self._generate_comparison_data,
            'RealTimeProcessing': self._generate_realtime_data
        }

    async def generate_dashboard_data(self) -> Dict[str, Any]:
        """
        Generates complete dashboard data
        """
        dashboard_data = {}

        for component_name, config in DASHBOARD_COMPONENTS.items():
            handler = self.component_handlers.get(config['component'])
            if handler:
                data = await handler()
                dashboard_data[component_name] = data

        return dashboard_data

    async def _generate_health_summary(self) -> Dict[str, Any]:
        """
        Generates health summary data
        """
        latest_metrics = await get_latest_health_metrics()

        return {
            'overall_status': latest_metrics.get('system_status', 'unknown'),
            'response_time': latest_metrics.get('avg_response_time', 0),
            'uptime_percentage': latest_metrics.get('uptime_24h', 0.0),
            'active_alerts': await count_active_alerts(),
            'last_updated': datetime.now().isoformat()
        }

    async def _generate_confidence_data(self) -> Dict[str, Any]:
        """
        Generates confidence tracking data
        """
        confidence_data = await get_confidence_history(hours=24)

        avg_confidence = sum(c['avg_confidence'] for c in confidence_data) / len(confidence_data) if confidence_data else 0.0
        trend = calculate_trend([c['avg_confidence'] for c in confidence_data])

        return {
            'current_avg_confidence': avg_confidence,
            'confidence_trend': trend,
            'threshold_status': 'ok' if avg_confidence >= 0.8 else 'warning',
            'confidence_distribution': await get_confidence_distribution(),
            'last_updated': datetime.now().isoformat()
        }
```

## 7. User Feedback UI Mockups

### A. Correction Interface

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ Transaction Review                                                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│ Description: OFFICE SUPPLIES - STAPLES                                          │
│ Amount: $45.99                                                                  │
│ Date: 2024-01-15                                                               │
│                                                                                 │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ AI Suggestion:                                                              │ │
│ │ Account: 6500 - Office Supplies                                             │ │
│ │ Confidence: 88% (High - Auto-apply)                                         │ │
│ │ Reasoning: Exact match with 3 similar transactions, keyword "office supplies"│ │
│ │ [✓ Accept] [✎ Edit] [ℹ Details]                                             │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                 │
│ [▼ Show Alternative Suggestions]                                                │
│ • 6600 - Shipping Expense (12%)                                               │
│ • 6400 - Travel Expense (8%)                                                  │
│                                                                                 │
│ [Edit Classification] [Submit for Review] [Skip for Later]                    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### B. Feedback Confirmation

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ Thank You for Your Feedback!                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ ✓ Successfully updated AI model with your correction                          │
│                                                                                 │
│ Original Prediction: 6500 - Office Supplies                                   │
│ Your Correction: 6600 - Shipping Expense                                      │
│ Confidence: 88% → Recalibrating...                                            │
│                                                                                 │
│ 🤖 AI Learning: This helps me recognize shipping expenses better in the future!│
│                                                                                 │
│ [Continue Reviewing] [View AI Insights] [Provide More Feedback]               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### C. AI Insights Panel

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ AI Performance Insights                                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ Today's Performance:                                                            │
│ • 124 transactions processed                                                    │
│ • 89% accuracy rate                                                             │
│ • Average confidence: 84%                                                      │
│ • 12 user corrections received                                                 │
│                                                                                 │
│ Learning Improvements:                                                          │
│ • Updated office supply patterns (+3% accuracy)                                │
│ • Improved shipping expense recognition (+2% accuracy)                        │
│ • Adjusted confidence thresholds for vendor "STAPLES"                          │
│                                                                                 │
│ Next Learning Cycle: In 2 hours (when 50+ new corrections gathered)           │
│                                                                                 │
│ [View Detailed Report] [Adjust Settings] [Provide Feedback]                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 8. Database Schema for Monitoring

```sql
-- Table for storing AI prediction results with confidence scores
CREATE TABLE ai.prediction_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL,
    agent_type VARCHAR(50) NOT NULL,
    suggested_account VARCHAR(50),
    confidence_score DECIMAL(3,2) NOT NULL,
    reasoning JSONB,
    alternatives JSONB,
    processing_time_ms INTEGER,
    method_used VARCHAR(50),
    was_correct BOOLEAN,
    user_feedback BOOLEAN, -- Whether user accepted/confirmed suggestion
    requires_review BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table for tracking user feedback and corrections
CREATE TABLE ai.user_feedback (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prediction_id UUID REFERENCES ai.prediction_results(id),
    transaction_id UUID NOT NULL,
    original_prediction JSONB,
    corrected_value JSONB,
    feedback_type VARCHAR(20) CHECK (feedback_type IN ('correction', 'confirmation', 'disagreement')),
    user_id UUID,
    feedback_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    feedback_details TEXT
);

-- Table for daily AI performance metrics
CREATE TABLE ai.daily_performance (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    metric_date DATE NOT NULL,
    agent_type VARCHAR(50) NOT NULL,
    total_predictions INTEGER DEFAULT 0,
    correct_predictions INTEGER DEFAULT 0,
    accuracy_rate DECIMAL(5,4),
    avg_confidence DECIMAL(3,2),
    error_count INTEGER DEFAULT 0,
    error_rate DECIMAL(5,4),
    avg_processing_time_ms DECIMAL(8,2),
    fallback_count INTEGER DEFAULT 0,
    fallback_rate DECIMAL(5,4),
    user_acceptance_count INTEGER DEFAULT 0,
    user_acceptance_rate DECIMAL(5,4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table for real-time monitoring metrics
CREATE TABLE ai.realtime_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    agent_type VARCHAR(50) NOT NULL,
    requests_per_minute INTEGER,
    avg_response_time_ms DECIMAL(8,2),
    error_rate_current DECIMAL(5,4),
    active_connections INTEGER,
    queue_depth INTEGER
);

-- Indexes for performance
CREATE INDEX idx_prediction_results_transaction ON ai.prediction_results(transaction_id);
CREATE INDEX idx_prediction_results_agent ON ai.prediction_results(agent_type);
CREATE INDEX idx_prediction_results_confidence ON ai.prediction_results(confidence_score);
CREATE INDEX idx_prediction_results_date ON ai.prediction_results(created_at);
CREATE INDEX idx_user_feedback_prediction ON ai.user_feedback(prediction_id);
CREATE INDEX idx_daily_performance_date ON ai.daily_performance(metric_date);
CREATE INDEX idx_daily_performance_agent ON ai.daily_performance(agent_type);
CREATE INDEX idx_realtime_metrics_time ON ai.realtime_metrics(timestamp);
```

## 9. Performance Requirements

### A. Response Time Requirements
- **Confidence Calculation**: < 50ms for single transaction
- **AI Prediction**: < 100ms for classification, < 200ms for OCR
- **Error Handling**: < 10ms overhead for fallback mechanisms
- **Feedback Processing**: < 200ms for correction handling

### B. Availability Requirements
- **Primary AI Service**: 99.5% uptime
- **Fallback Service**: 99.9% uptime
- **Confidence Scoring**: 99.9% uptime (critical path)
- **Monitoring System**: 99.0% uptime

### C. Scalability Requirements
- **Throughput**: 1000+ transactions per minute
- **Concurrent Users**: Support 100+ concurrent users
- **Data Volume**: Handle 1M+ prediction records
- **Storage**: Petabyte-scale feedback data retention