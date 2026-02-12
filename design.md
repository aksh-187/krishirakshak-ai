# Design Document: KrishiRakshak AI

## Overview

KrishiRakshak AI is a serverless, district-level crop disease intelligence network built on AWS infrastructure. The system transforms rice crop protection in Andhra Pradesh from reactive treatment to predictive, community-coordinated disease management.

The architecture follows an event-driven, microservices pattern using AWS Lambda for compute, DynamoDB for data persistence, S3 for image storage, and API Gateway for client communication. The design prioritizes offline-first mobile operation, cost efficiency, and scalability to 10,000+ concurrent users while maintaining sub-5-second response times on 2G/3G networks.

The system serves two distinct user personas: rice farmers requiring accessible disease detection through mobile devices, and agricultural officers needing comprehensive outbreak intelligence through web dashboards. All farmer-facing interfaces support Telugu language with voice capabilities, optimized for low-literacy users and low-end devices.

Key architectural decisions include: (1) serverless compute for automatic scaling and cost optimization, (2) single-table DynamoDB design for efficient queries, (3) progressive web app (PWA) for offline-first mobile experience, (4) CloudFront CDN for low-latency content delivery, and (5) event-driven processing for real-time aggregation and alerting.

## Architecture

### System Components

**Image_Processor**
- Receives crop images from Farmer_Dashboard via API Gateway
- Validates image format, size, and quality
- Compresses images to <500KB while preserving diagnostic features
- Stores original and processed images in S3 with lifecycle policies
- Triggers Disease_Detection_System via SNS event

**Disease_Detection_System**
- Achieves minimum 85% F1-score on validation dataset representative of Andhra Pradesh rice field conditions
- Model retraining pipeline scheduled quarterly using newly collected labeled field data
- Model performance monitored continuously using real-world confidence distribution tracking
- Identifies rice diseases from 15-class taxonomy (Bacterial Blight, Blast, Brown Spot, etc.)
- Returns disease classification with confidence scores
- Supports multi-disease detection per image
- Responds within 5-second SLA

**Geo_Logger**
- Captures GPS coordinates from mobile device
- Validates coordinates within Andhra Pradesh boundaries
- Reverse geocodes to district/mandal/village using cached mapping data
- Creates Detection_Record with anonymized farmer_id
- Stores records in DynamoDB Detection table
- Publishes detection event to EventBridge for downstream processing

**Community_Radar**
- Subscribes to detection events via EventBridge
- Aggregates detections by district and mandal in 15-minute windows
- Clustering executed on 15-minute district-level detection windows to ensure Lambda execution efficiency and avoid full-dataset scans
- Calculates detection density per 5km radius
- Identifies clusters exceeding 3 standard deviations above baseline
- Stores aggregation results in DynamoDB Aggregation table
- Uses density-based clustering (DBSCAN-inspired logic) optimized for Lambda time constraints
Clustering uses epsilon = 5km, min_samples = 3 to detect outbreak clusters.

**Risk_Scoring_Engine**
- Executes every 30 minutes via EventBridge scheduled rule
- Retrieves current detection density, outbreak clusters, predictions, and historical trends
- Weight parameters configurable via administrative settings without requiring code redeployment
- Default weighting: 40% current detection density + 30% outbreak clusters + 20% forecast probability + 10% historical trend
- Normalizes scores to 0-100 scale
- Stores risk scores in DynamoDB RiskScore table
- Triggers Alert_System when score changes exceed threshold

**Prediction_Engine**
- Integrates with external weather data provider (e.g., AWS Weather Data Exchange or OpenWeatherMap API) for 7-day forecasts
- Retrieves temperature, humidity, rainfall predictions by district
- Applies disease-climate correlation models trained on historical data
- Generates outbreak probability for each disease type
- Updates forecasts twice daily (6 AM, 6 PM IST)
- Stores predictions in DynamoDB Forecast table
Uses LSTM-based ensemble with rolling window training to predict disease outbreaks.

**Alert_System**
- Monitors risk score changes and outbreak cluster formation
- Classifies alerts into Low (0-33), Moderate (34-66), High (67-100) tiers
- Distributes alerts via SNS to SMS, push notifications, and voice calls
- Batches multiple diseases into single notification to prevent alert fatigue
- Implements rate limiting: Low (1/day), Moderate (3/day), High (unlimited)
- Stores alert history in DynamoDB Alert table

**Farmer_Dashboard_Service**
- Progressive Web App (PWA) built with React and service workers
- Implements offline-first architecture with IndexedDB for local storage
- Provides camera integration for image capture
- Displays current district risk score, personal detection history, active alerts
- Supports Telugu text and text-to-speech via Web Speech API
- Compresses and queues detections when offline
- Syncs with backend via REST API when connectivity restored

**Officer_Dashboard_Service**
- React-based web application with Mapbox GL for heatmap visualization
- Displays real-time detection heatmaps color-coded by risk intensity
- Provides trend analysis charts using Recharts library
- Enables filtering by disease, geography, date range, risk tier
- Generates PDF reports using jsPDF and CSV exports
- Implements role-based access control via Cognito user pools
- Queries aggregated data via GraphQL API (AppSync)


### AWS Infrastructure Components

**API Gateway**
- REST API for farmer mobile app endpoints
- GraphQL API (AWS AppSync) for officer dashboard queries
- Request throttling: 10,000 requests/second burst, 5,000 steady state
- Amazon Cognito Identity Pools for secure mobile client authentication using temporary AWS credentials
- JWT-based authentication validation using Cognito-issued tokens
- Cognito authorizer for officer dashboard role-based access
- Integrated with AWS WAF for protection against common web exploits

**Lambda Functions**
- ProcessImage: 1024MB memory, 10-second timeout
- DetectDisease: 3008MB memory, 30-second timeout (ML inference)
- LogDetection: 512MB memory, 5-second timeout
- AggregateDetections: 1024MB memory, 60-second timeout
- CalculateRiskScore: 512MB memory, 30-second timeout
- GenerateForecast: 1024MB memory, 60-second timeout
- DistributeAlerts: 512MB memory, 15-second timeout
- All functions use Python 3.11 runtime

**DynamoDB Tables**
- Detection: Partition key (district#mandal), Sort key (timestamp), GSI on farmer_id
- Aggregation: Partition key (district), Sort key (date#disease)
- RiskScore: Partition key (district), Sort key (timestamp)
- Forecast: Partition key (district#disease), Sort key (date)
- Alert: Partition key (farmer_id), Sort key (timestamp)
- All tables use on-demand billing mode

**S3 Buckets**
- krishirakshak-images: Stores crop images with 90-day lifecycle to Glacier
- krishirakshak-static: Hosts PWA assets distributed via CloudFront
- Versioning enabled, encryption at rest (AES-256)

**EventBridge**
- Detection event bus for real-time event routing
- Scheduled rules for Risk_Scoring_Engine (every 30 min) and Prediction_Engine (6 AM, 6 PM IST)

**SNS Topics**
- detection-events: Publishes new detection records
- alert-high: High-priority alerts with SMS and voice
- alert-moderate: Moderate alerts with SMS and push
- alert-low: Low alerts with push only

**CloudFront**
- Distributes Farmer_Dashboard PWA assets
- Caches API responses for static content (disease info, treatment guides)
- Edge locations in Mumbai and Chennai for low latency

**Cost Optimization Strategy**
- Target operational cost under ₹50,000/month for 10,000 active users
- DynamoDB single-table design to reduce read/write amplification
- Image lifecycle policy: S3 → Glacier after 90 days
- Lambda memory tuning based on CloudWatch profiling
- CloudFront edge caching for static disease/treatment data

**Cognito**
- User pools for officer authentication
- Identity pools for temporary AWS credentials
- MFA enabled for officer accounts

**Encryption Standards**
- All data in transit encrypted using TLS 1.3
- All data at rest encrypted using AES-256 (S3, DynamoDB with KMS-managed keys)
- KMS-managed encryption keys

## Components and Interfaces

### Detection Flow

```
Farmer Mobile → API Gateway → ProcessImage Lambda → S3 (image storage)
                                    ↓
                            DetectDisease Lambda (ML inference)
                                    ↓
                            LogDetection Lambda → DynamoDB Detection table
                                    ↓
                            EventBridge (detection event)
                                    ↓
                            AggregateDetections Lambda → DynamoDB Aggregation table
```

### Risk Scoring Flow

```
EventBridge (scheduled) → CalculateRiskScore Lambda
                                    ↓
                    Query: Detection, Aggregation, Forecast tables
                                    ↓
                    Calculate weighted score (0-100)
                                    ↓
                    Store: DynamoDB RiskScore table
                                    ↓
                    If threshold exceeded → DistributeAlerts Lambda
```

### Alert Distribution Flow

```
DistributeAlerts Lambda → Query: RiskScore, Farmer location
                                    ↓
                    Classify tier (Low/Moderate/High)
                                    ↓
                    SNS Topics → SMS (AWS SNS)
                                → Push (FCM via SNS)
                                → Voice (Amazon Connect for High tier)

- Alert deduplication logic prevents repeated alerts for unchanged risk levels within 24 hours

```

### REST API Endpoints

**Farmer Mobile API**

```
POST /api/v1/detections
Request: { 
  image: base64, 
  location: {lat, lon}, 
  farmer_id: string }

Response: {
  detection_id,
  disease_name_english: string,
  disease_name_telugu: string,
  confidence: float,
  treatment_advice: object
}

GET /api/v1/risk-score/{district}
Response: { district, 
score: int, 
tier: string, 
updated_at: timestamp }

GET /api/v1/alerts/{farmer_id}
Response: { 
  alerts: [{ 
    alert_id, 
    disease, 
    tier, 
    message, 
    timestamp }] 
  }

POST /api/v1/alerts/{alert_id}/acknowledge
Request: { farmer_id: string }
Response: { success: boolean }

GET /api/v1/treatment/{disease_id}
Response: { disease, 
treatments: [{ 
  name, 
  type,
  cost,
  instructions }],
videos: [] }

```

**Officer Dashboard GraphQL API**

```graphql
query GetDistrictHeatmap($district: String!, $dateRange: DateRange!) {
  heatmap(district: $district, dateRange: $dateRange) {
    mandal
    detectionCount
    riskScore
    coordinates { lat, lon }
  }
}

query GetTrendAnalysis($district: String!, $disease: String, $period: String!) {
  trends(district: $district, disease: $disease, period: $period) {
    date
    detectionCount
    riskScore
    forecastAccuracy
  }
}

query GetOutbreakClusters($district: String!) {
  clusters(district: $district) {
    cluster_id
    centroid { lat, lon }
    radius
    detectionCount
    primaryDisease
    affectedMandals
  }
}

mutation GenerateReport($params: ReportParams!) {
  generateReport(params: $params) {
    report_id
    downloadUrl
    expiresAt
  }
}
```


## Data Models

### Detection_Record

```python
{
  "detection_id": "uuid",
  "farmer_id": "hashed_string",  # Anonymized
  "district": "string",
  "mandal": "string", 
  "village": "string",
  "coordinates": {
    "lat": "float",
    "lon": "float",
    "accuracy": "float"  # meters
  },
  "timestamp": "iso8601",
  "image_url": "s3_url",
  "disease": {
    "primary": "string",
    "confidence": "float",
    "secondary": [{"name": "string", "confidence": "float"}]
  },
  "crop_stage": "string",  # seedling, vegetative, reproductive, maturity
  "severity": "string",  # low, moderate, high
  "offline_captured": "boolean",
  "sync_timestamp": "iso8601"
}
```

### Aggregation_Record

```python
{
  "aggregation_id": "district#mandal#date",
  "district": "string",
  "mandal": "string",
  "date": "date",
  "disease_counts": {
    "bacterial_blight": "int",
    "blast": "int",
    "brown_spot": "int",
    # ... other diseases
  },
  "total_detections": "int",
  "unique_farmers": "int",
  "detection_density": "float",  # detections per km²
  "baseline_density": "float",
  "std_deviation": "float",
  "is_outbreak_cluster": "boolean",
  "cluster_id": "string",
  "updated_at": "timestamp"
}
```

### Risk_Score_Record

```python
{
  "score_id": "district#timestamp",
  "district": "string",
  "timestamp": "iso8601",
  "overall_score": "int",  # 0-100
  "disease_scores": {
    "bacterial_blight": "int",
    "blast": "int",
    # ... other diseases
  },
  "components": {
    "current_density": "float",  # 40% weight
    "outbreak_clusters": "float",  # 30% weight
    "forecast": "float",  # 20% weight
    "historical_trend": "float"  # 10% weight
  },
  "tier": "string",  # Low, Moderate, High
  "change_24h": "int",
  "rapid_escalation": "boolean"
}
```

### Forecast_Record

```python
{
  "forecast_id": "district#disease#date",
  "district": "string",
  "mandal": "string | null",
  "disease": "string",
  "forecast_date": "date",
  "generated_at": "timestamp",
  "outbreak_probability": "float",  # 0-1
  "confidence": "float",  # 0-1
  "climate_factors": {
    "temperature_avg": "float",
    "humidity_avg": "float",
    "rainfall_mm": "float",
    "favorable_conditions": "boolean"
  },
  "historical_correlation": "float",
  "forecast_period": "int"  # days ahead (1-7)
}
```

### Alert_Record

```python
{
  "alert_id": "uuid",
  "farmer_id": "hashed_string",
  "district": "string",
  "mandal": "string",
  "timestamp": "iso8601",
  "tier": "string",  # Low, Moderate, High
  "diseases": ["string"],
  "risk_score": "int",
  "message_telugu": "string",
  "message_english": "string",
  "recommended_actions": ["string"],
  "channels": ["sms", "push", "voice"],
  "delivery_status": {
    "sms": "string",  # sent, delivered, failed
    "push": "string",
    "voice": "string"
  },
  "acknowledged": "boolean",
  "acknowledged_at": "timestamp"
}
```

### Offline_Queue_Item

```python
{
  "queue_id": "uuid",
  "farmer_id": "string",
  "detection_data": "Detection_Record",
  "local_timestamp": "iso8601",
  "sync_status": "string",  # pending, syncing, synced, failed
  "retry_count": "int",
  "created_at": "timestamp"
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Detection and Response Properties

**Property 1: Detection response completeness**
*For any* disease detection result, the response must contain disease name in English, disease name in Telugu, and a confidence score between 0 and 1.
**Validates: Requirements 1.3**

**Property 2: Multi-disease ranking**
*For any* detection result containing multiple diseases, the diseases must be ordered by confidence score in descending order.
**Validates: Requirements 1.5**

**Property 3: Invalid image rejection**
*For any* image that does not contain rice crops or is unclear (below quality threshold), the system must return an error message requesting recapture.
**Validates: Requirements 1.4**

**Property 4: Detection response time**
*For any* valid crop image submission, the system must return a detection result within 5 seconds.
**Validates: Requirements 1.1**

### Geo-Location and Data Integrity Properties

**Property 5: Detection record completeness**
*For any* detection record stored in the system, it must contain district, mandal, village, latitude, longitude, and accuracy fields.
**Validates: Requirements 2.2**

**Property 6: Timestamp duality**
*For any* detection record, it must contain both a UTC timestamp and a local timezone timestamp.
**Validates: Requirements 2.3**

**Property 7: Farmer ID anonymization**
*For any* detection record, the farmer_id field must be a one-way hash (irreversible) and not contain plaintext identifiable information.
**Validates: Requirements 2.5, 13.2**

**Property 8: Geographic boundary validation**
*For any* coordinate pair (latitude, longitude), if it falls outside Andhra Pradesh state boundaries, the system must reject the detection submission.
**Validates: Requirements 2.6**

### Aggregation and Clustering Properties

**Property 9: Aggregation accuracy**
*For any* set of detection records for a given district and mandal on a specific date, the aggregation count by disease type must equal the actual number of detections for that disease.
**Validates: Requirements 3.1**

**Property 10: Outbreak cluster identification**
*For any* geographic area where detection density exceeds baseline by more than 3 standard deviations within a 5km radius, the system must identify it as an outbreak cluster.
**Validates: Requirements 3.2**

**Property 11: Cluster metadata completeness**
*For any* identified outbreak cluster, it must contain centroid coordinates (lat, lon) and affected area radius in kilometers.
**Validates: Requirements 3.3**

**Property 12: Aggregation dimensionality**
*For any* aggregated data response, it must be queryable by disease type, time period, and geographic unit (district/mandal).
**Validates: Requirements 3.5**

### Prediction and Forecasting Properties

**Property 13: Forecast climate data completeness**
*For any* forecast record, it must contain temperature, humidity, and rainfall data for the forecast period.
**Validates: Requirements 4.1**

**Property 14: Favorable conditions probability**
*For any* forecast where climate conditions are favorable for disease proliferation, the outbreak probability must be higher than when conditions are unfavorable.
**Validates: Requirements 4.2**

**Property 15: Low confidence uncertainty indication**
*For any* forecast with confidence below 0.6, the system must include an uncertainty indicator in the output.
**Validates: Requirements 4.5**

**Property 16: Forecast geographic granularity**
*For any* disease and date, forecasts must exist at both district-level and mandal-level granularity.
**Validates: Requirements 4.6**

### Risk Scoring Properties

**Property 17: Risk score bounds**
*For any* calculated district risk score, the value must be within the range [0, 100] inclusive.
**Validates: Requirements 5.1**

**Property 18: Risk score weighting**
*For any* risk score calculation, the weighted sum of components (40% current density + 30% outbreak clusters + 20% forecast + 10% historical trend) must equal the overall score.
**Validates: Requirements 5.2**

**Property 19: Disease-specific scoring**
*For any* district risk score record, it must contain separate risk scores for each major rice disease category.
**Validates: Requirements 5.4**

**Property 20: Rapid escalation detection**
*For any* district where the risk score increases by more than 15 points within 24 hours, the system must flag it as a rapid escalation event.
**Validates: Requirements 5.5**

**Property 21: Score normalization**
*For any* set of district risk scores calculated at the same time, the scores must be normalized to enable fair comparison across districts with different baseline detection rates.
**Validates: Requirements 5.6**

### Alert System Properties

**Property 22: Alert tier classification**
*For any* risk score value, the alert tier must be: Low if score ∈ [0,33], Moderate if score ∈ [34,66], or High if score ∈ [67,100].
**Validates: Requirements 6.1**

**Property 23: Alert message completeness**
*For any* alert distributed to farmers, it must contain disease name, risk level, affected area, recommended actions, and agricultural office contact information.
**Validates: Requirements 6.3**

**Property 24: High tier channel inclusion**
*For any* High tier alert, the delivery channels must include SMS, push notification, and voice call.
**Validates: Requirements 6.4**

**Property 25: Alert rate limiting**
*For any* farmer, Low tier alerts must not exceed 1 per day, and multiple diseases must be batched into single notifications.
**Validates: Requirements 6.6**

### Mobile Dashboard Properties

**Property 26: Image compression size limit**
*For any* captured crop image, after compression the file size must not exceed 500KB.
**Validates: Requirements 8.4**

### Officer Dashboard Properties

**Property 27: Dashboard filtering correctness**
*For any* filter applied (disease type, geographic unit, date range, risk tier), the returned results must match all specified filter criteria.
**Validates: Requirements 9.3**

**Property 28: Report export data integrity**
*For any* generated report in PDF or CSV format, the exported data must match the source data from the database query.
**Validates: Requirements 9.4**

### Offline Operation Properties

**Property 29: Offline queue storage**
*For any* detection submission when network is unavailable, the system must store it in the Offline_Queue with a local timestamp.
**Validates: Requirements 10.1**

**Property 30: Sync chronological ordering**
*For any* set of queued offline detections, when connectivity is restored they must sync to the server in chronological order by local timestamp.
**Validates: Requirements 10.2**

**Property 31: Offline data accessibility**
*For any* cached data (risk scores, alerts, treatment guidance), it must remain accessible when the system is in offline mode.
**Validates: Requirements 10.4**

**Property 32: Offline conflict resolution**
*For any* offline detection with a timestamp older than the most recent server-side update, the system must apply conflict resolution rules to determine precedence.
**Validates: Requirements 10.5**

**Property 33: Queue size limit enforcement**
*For any* Offline_Queue, when it reaches 50 pending detections, the system must prevent additional submissions and prompt the farmer to establish connectivity.
**Validates: Requirements 10.6**

### Security and Privacy Properties

**Property 34: Data deletion with aggregate preservation**
*For any* farmer data deletion request, all associated detection records must be removed while anonymized aggregated statistics remain intact.
**Validates: Requirements 13.4**

**Property 35: Role-based access enforcement**
*For any* officer user, they must only be able to access detection data, aggregations, and reports for their assigned districts.
**Validates: Requirements 13.5**

**Property 36: Audit log creation**
*For any* data access or modification operation, an audit log entry must be created with timestamp, user ID, operation type, and affected resources.
**Validates: Requirements 13.6**

### Treatment Advisory Properties

**Property 37: Treatment recommendation completeness**
*For any* disease detection, the treatment recommendations must include pesticide names, application rates, and timing information.
**Validates: Requirements 14.1**

**Property 38: Treatment option diversity**
*For any* disease treatment response, it must include both organic and chemical treatment options with cost estimates in INR.
**Validates: Requirements 14.2**

**Property 39: Treatment prioritization**
*For any* set of treatment recommendations, they must be ordered by priority based on disease severity, crop growth stage, and local availability.
**Validates: Requirements 14.3**

**Property 40: Adjacent area preventive measures**
*For any* outbreak cluster detection, farmers in adjacent areas (within 10km radius) must receive preventive measures in addition to treatment recommendations.
**Validates: Requirements 14.4**

**Property 41: Safety information inclusion**
*For any* treatment recommendation, it must include safety precautions and protective equipment requirements in Telugu.
**Validates: Requirements 14.5**

**Property 42: Video link validity**
*For any* treatment recommendation, the video demonstration links must be valid URLs and optimized for low-bandwidth viewing (<5MB file size).
**Validates: Requirements 14.6**


## Error Handling

### Image Processing Errors

**Invalid Image Format**
- Detection: Check MIME type and file extension
- Response: HTTP 400 with error code `INVALID_IMAGE_FORMAT`
- Message: "Please upload a valid image file (JPEG, PNG)"
- Recovery: Prompt user to recapture image

**Image Too Large**
- Detection: Check file size before processing
- Response: HTTP 413 with error code `IMAGE_TOO_LARGE`
- Message: "Image size exceeds limit. Please try again."
- Recovery: Automatic compression on client side, retry

**Poor Image Quality**
- Detection: ML model confidence below threshold (< 0.3)
- Response: HTTP 200 with warning code `LOW_CONFIDENCE`
- Message: "Image unclear. Please recapture in better lighting."
- Recovery: Allow submission but flag for manual review

**ML Model Failure**
- Detection: Lambda timeout or model inference error
- Response: HTTP 503 with error code `SERVICE_UNAVAILABLE`
- Message: "Detection service temporarily unavailable. Please try again."
- Recovery: Retry with exponential backoff (3 attempts), queue for later processing

### Location Errors

**GPS Unavailable**
- Detection: Location permission denied or GPS disabled
- Response: Prompt for manual village selection
- Message: "Location unavailable. Please select your village."
- Recovery: Fallback to manual location entry

**Coordinates Out of Bounds**
- Detection: Lat/lon outside Andhra Pradesh boundaries
- Response: HTTP 400 with error code `INVALID_LOCATION`
- Message: "Location outside service area."
- Recovery: Prompt user to verify location settings

**Geocoding Failure**
- Detection: Reverse geocoding API error
- Response: Store coordinates only, retry geocoding asynchronously
- Recovery: Background job retries geocoding every 5 minutes

### Network and Connectivity Errors

**Network Timeout**
- Detection: Request exceeds 30-second timeout
- Response: Store in Offline_Queue
- Message: "Slow connection detected. Saved for later sync."
- Recovery: Automatic sync when connectivity improves

**API Rate Limit Exceeded**
- Detection: API Gateway throttling (>10,000 req/sec)
- Response: HTTP 429 with error code `RATE_LIMIT_EXCEEDED`
- Message: "Too many requests. Please wait and try again."
- Recovery: Client-side exponential backoff, queue request

**Server Error**
- Detection: Lambda function error or DynamoDB unavailability
- Response: HTTP 500 with error code `INTERNAL_SERVER_ERROR`
- Message: "Service error. Your request has been saved."
- Recovery: Dead letter queue for failed requests, manual investigation

### Data Validation Errors

**Missing Required Fields**
- Detection: Schema validation failure
- Response: HTTP 400 with error code `MISSING_REQUIRED_FIELD`
- Message: "Required information missing: {field_name}"
- Recovery: Prompt user to complete missing information

**Invalid Data Format**
- Detection: Type mismatch or format violation
- Response: HTTP 400 with error code `INVALID_DATA_FORMAT`
- Message: "Invalid data format for {field_name}"
- Recovery: Client-side validation before submission

**Duplicate Detection**
- Detection: Same farmer_id, location, timestamp within 5 minutes
- Response: HTTP 409 with error code `DUPLICATE_DETECTION`
- Message: "Recent detection already recorded."
- Recovery: Return existing detection result

### Authentication and Authorization Errors

**Invalid Authentication Token**
- Detection: JWT token validation failure or malformed token
- Response: HTTP 401 with error code `INVALID_AUTH_TOKEN`
- Message: "Authentication failed. Please log in again."
- Recovery: Redirect to login, refresh Cognito credentials

**Insufficient Permissions**
- Detection: Officer accessing data outside assigned district
- Response: HTTP 403 with error code `FORBIDDEN`
- Message: "Access denied to requested resource."
- Recovery: Log security event, display available districts

**Session Expired**
- Detection: Cognito token expiration
- Response: HTTP 401 with error code `SESSION_EXPIRED`
- Message: "Session expired. Please log in again."
- Recovery: Automatic token refresh, fallback to re-authentication

### Offline Mode Errors

**Queue Full**
- Detection: Offline_Queue reaches 50 items
- Response: Local error with code `QUEUE_FULL`
- Message: "Offline storage full. Please connect to sync."
- Recovery: Block new submissions until sync completes

**Sync Conflict**
- Detection: Offline detection timestamp conflicts with server data
- Response: Apply last-write-wins strategy
- Message: "Data synchronized with conflict resolution."
- Recovery: Log conflict for audit, prefer server timestamp

**Storage Quota Exceeded**
- Detection: IndexedDB storage limit reached
- Response: Local error with code `STORAGE_FULL`
- Message: "Device storage full. Please free up space."
- Recovery: Prompt to delete old cached data

### External Service Errors

**Weather API Failure**
- Detection: External weather provider API timeout or error
- Response: Use cached forecast data
- Message: "Using previous forecast data."
- Recovery: Retry every 30 minutes, alert if failure persists >6 hours

**SMS Delivery Failure**
- Detection: SNS delivery status = failed
- Response: Retry via alternate channel (push notification)
- Message: Logged in Alert table with delivery_status = failed
- Recovery: 3 retry attempts, escalate to voice call for High tier

**Voice Call Failure**
- Detection: Amazon Connect call status = failed
- Response: Fallback to SMS with urgent flag
- Message: Logged in Alert table
- Recovery: Manual follow-up by agricultural officer

## Testing Strategy

### Dual Testing Approach

The KrishiRakshak AI system requires comprehensive testing through both unit tests and property-based tests. These approaches are complementary:

- **Unit tests** validate specific examples, edge cases, error conditions, and integration points between components
- **Property-based tests** verify universal properties across all inputs through randomized testing

Together, they provide comprehensive coverage where unit tests catch concrete bugs and property-based tests verify general correctness.

### Property-Based Testing Configuration

**Framework**: Hypothesis (Python)

**Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `# Feature: krishirakshak-ai, Property {number}: {property_text}`

**Example Property Test Structure**:

```python
from hypothesis import given, strategies as st
import hypothesis.strategies as st

@given(
    disease_name_en=st.text(min_size=1),
    disease_name_te=st.text(min_size=1),
    confidence=st.floats(min_value=0.0, max_value=1.0)
)
@settings(max_examples=100)
def test_detection_response_completeness(disease_name_en, disease_name_te, confidence):
    """
    Feature: krishirakshak-ai, Property 1: Detection response completeness
    For any disease detection result, the response must contain disease name 
    in English, disease name in Telugu, and a confidence score between 0 and 1.
    """
    detection_result = create_detection_result(disease_name_en, disease_name_te, confidence)
    
    assert "disease_name_english" in detection_result
    assert "disease_name_telugu" in detection_result
    assert "confidence" in detection_result
    assert 0.0 <= detection_result["confidence"] <= 1.0
```

### Unit Testing Strategy

**Framework**: pytest

**Coverage Areas**:
1. **API Endpoint Tests**: Validate request/response formats, status codes, error handling
2. **Lambda Function Tests**: Test individual function logic with mocked AWS services
3. **Data Model Tests**: Verify serialization, deserialization, validation
4. **Integration Tests**: Test component interactions (e.g., Image_Processor → Disease_Detection_System)
5. **Edge Case Tests**: Boundary conditions, empty inputs, malformed data
6. **Error Condition Tests**: Network failures, timeouts, invalid credentials

**Example Unit Test**:

```python
def test_invalid_image_format_returns_400():
    """Test that uploading a non-image file returns 400 error"""
    response = api_client.post(
        "/api/v1/detections",
        files={"image": ("test.txt", b"not an image", "text/plain")}
    )
    assert response.status_code == 400
    assert response.json()["error_code"] == "INVALID_IMAGE_FORMAT"
```

### Integration Testing

**Scope**: End-to-end flows across multiple components

**Key Flows to Test**:
1. Complete detection flow: Image upload → ML inference → Geo logging → Aggregation
2. Alert distribution flow: Risk score calculation → Alert classification → Multi-channel delivery
3. Offline sync flow: Offline queue → Connectivity restoration → Chronological sync
4. Officer dashboard flow: Data query → Filtering → Report generation

**Tools**: pytest with moto for AWS service mocking

### Performance Testing

**Load Testing**:
- Tool: Locust
- Target: 10,000 concurrent users
- Scenarios: Image upload, risk score queries, alert distribution
- Success Criteria: <5s response time at p95, <1% error rate

**Stress Testing**:
- Gradual load increase to identify breaking point
- Monitor Lambda throttling, DynamoDB capacity, API Gateway limits

### Security Testing

**Penetration Testing**:
- SQL injection attempts (though using DynamoDB)
- Authentication bypass attempts
- Unauthorized data access attempts
- JWT token replay and credential misuse attempts

**Privacy Compliance Testing**:
- Verify farmer ID anonymization
- Test data deletion completeness
- Validate role-based access control
- Audit log completeness verification

### Offline Mode Testing

**Scenarios**:
1. Submit detection while offline → Verify queue storage
2. Restore connectivity → Verify chronological sync
3. Fill queue to 50 items → Verify blocking behavior
4. Create sync conflicts → Verify resolution logic

**Tools**: Chrome DevTools network throttling, manual airplane mode testing

### Mobile Device Testing

**Target Devices**:
- Low-end: Android 6.0, 1GB RAM, 2G network
- Mid-range: Android 10, 3GB RAM, 3G network
- High-end: Android 13, 6GB RAM, 4G network

**Test Cases**:
- PWA installation and offline functionality
- Image capture and compression
- Telugu text rendering and TTS
- Battery and data usage monitoring

## Observability Architecture

**CloudWatch Metrics**:
- Lambda invocation count, duration, errors
- API Gateway request count, latency, 4xx/5xx errors
- DynamoDB read/write capacity consumption
- S3 storage usage and request count

**CloudWatch Alarms**:
- Lambda error rate >1%
- API Gateway latency >5s at p95
- DynamoDB throttling events
- Daily cost exceeds budget by >20%

**X-Ray Tracing**:
- End-to-end request tracing
- Identify bottlenecks in detection flow
- Monitor external API call latency (Weather API)

**Custom Application Metrics**:
- Detection accuracy (confidence distribution)
- Alert delivery success rate by channel
- Offline queue sync success rate
- User engagement metrics (daily active farmers)

**SLA Monitoring**
- Detection API latency monitored to ensure <5 seconds at p95
- Alert dispatch latency monitored to ensure <5 minutes for High-tier alerts
- API error rate monitored to remain below 0.5%
