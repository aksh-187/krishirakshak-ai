KrishiRakshak AI

AI-powered Early Warning System for Rice Crop Disease Detection & Prediction

KrishiRakshak AI is a scalable, serverless AI solution designed to reduce crop losses among rice farmers in India by enabling early disease detection, outbreak prediction, and multilingual farmer alerts.
Built for the AI for Bharat Hackathon – AI for Communities, Access & Public Impact Track.

Problem Statement: 
Rice farmers in India lose up to 30% of yield annually due to late detection of diseases such as
Rice Blast
Brown Spot
Bacterial Blight

Key challenges:
Limited access to agricultural experts
Poor internet connectivity in rural areas
Language barriers
Lack of early outbreak warning systems

Solution Overview : 
KrishiRakshak AI provides
1. Image-Based Disease Detection : Farmers upload a crop image → AI model detects disease → Instant diagnosis with treatment suggestions.
2. District-Level Data Aggregation : Aggregates disease reports to identify emerging outbreak clusters.
3. Predictive Outbreak Risk Engine : Uses time-series models + weather data to forecast disease risk.
4. Multilingual Alerts (Telugu Support) : Voice + text alerts tailored for rural farmers.
5. Farmer & Officer Dashboards : Simple mobile-optimized interface for Farmers (detection + alerts), Agricultural officers (district analytics)

Architecture :
Designed using a serverless AWS architecture:
AWS Lambda – AI processing
Amazon S3 – Image storage
DynamoDB – Farmer & disease records
Amazon Bedrock – AI model integration
Amazon SNS – Alert notifications
API Gateway – Secure endpoints

The system is optimized for:
Low bandwidth environments
2G/3G compatibility
Cost-effective scaling

Project Documentation'

This repository contains:
requirements.md – Detailed system requirements (EARS-based specification)
design.md – Technical system design and architecture
tasks.md – Implementation roadmap (for prototype phase)

Target Impact:
Reduce preventable crop loss
Improve early intervention by officers
Increase farmer income stability
Strengthen rural agricultural resilience

Future Roadmap :
Integration with real-time IMD weather APIs
Mobile app deployment (Android-first)
Expansion to additional crops
Government integration (Agri portals)

Author

Akula Akshaya
