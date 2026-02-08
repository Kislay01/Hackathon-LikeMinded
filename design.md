# EDI (E-Commerce Data Insight Marketplace) - Design Document

## 1. System Overview

EDI is an AI-powered platform that helps Indian MSMEs and e-commerce sellers transform raw sales data into actionable insights. The platform combines data analytics, machine learning forecasting, and a marketplace for curated datasets.

**Core Value Proposition:**
- Upload sales data (CSV/Excel) → Get automated insights
- AI-generated recommendations in natural language
- Sales forecasting and demand prediction
- Marketplace to buy/sell curated datasets

**Target Users:** Sellers, Buyers, Analysts, Admins

---

## 2. Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client Layer                          │
│              Next.js (React + TypeScript)                │
└────────────────────┬────────────────────────────────────┘
                     │ REST API (HTTPS)
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   API Gateway                            │
│              Express.js + JWT Auth                       │
└────────┬──────────────────────────────┬─────────────────┘
         │                              │
         ▼                              ▼
┌──────────────────┐          ┌──────────────────┐
│  Business Logic  │          │   ML Service     │
│    (Node.js)     │◄────────►│   (Python)       │
└────────┬─────────┘          └──────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                    Data Layer                            │
│  PostgreSQL  │  Redis Cache  │  S3 File Storage         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Technology Stack

**Frontend:** Next.js 14, TypeScript, Tailwind CSS, Recharts, React Query
**Backend:** Node.js, Express.js, Prisma ORM, JWT, Multer, Bull Queue
**ML Service:** Python, FastAPI, pandas, scikit-learn, Prophet, OpenAI API
**Database:** PostgreSQL 15+, Redis
**Storage:** AWS S3 (production), Local (development)
**Deployment:** Vercel (frontend), AWS EC2/Render (backend), AWS Lambda (ML)

---

## 3. Module Breakdown

### 3.1 Authentication Module
- User registration with role selection (Seller/Buyer/Analyst/Admin)
- JWT-based authentication (access + refresh tokens)
- Role-based access control (RBAC)
- Password hashing with bcrypt

### 3.2 Dataset Upload Module
- File upload (CSV/Excel, max 50MB)
- File validation and parsing
- Column mapping to standard fields
- Async processing with job queue

### 3.3 Data Cleaning Module
- Remove duplicates
- Handle missing values (median/mode imputation)
- Type conversion (dates, numbers, categories)
- Data normalization
- Generate cleaning report

### 3.4 Dashboard Module
- Key metrics: Total Revenue, Orders, AOV, Growth Rate
- Time-series charts (daily/weekly/monthly)
- Product performance analysis
- Customer analytics
- Custom date range filtering

### 3.5 AI Insights Module
- Statistical analysis (trends, correlations)
- Anomaly detection (spikes, drops)
- LLM integration for natural language insights
- Categorized insights (positive/negative/neutral)
- Confidence scoring

### 3.6 Forecasting Module
- Time-series forecasting with Prophet
- 30/60/90 day predictions
- Confidence intervals
- Indian holiday integration
- Accuracy metrics (MAPE, RMSE)

### 3.7 Recommendation Engine
- Product recommendations (top performers, underperformers)
- Pricing optimization suggestions
- Inventory optimization (reorder points, safety stock)
- Customer targeting
- Cross-sell opportunities

### 3.8 Marketplace Module
- List datasets for sale
- Browse and search listings
- Purchase with payment integration (Razorpay)
- Dataset preview (first 100 rows)
- Reviews and ratings

### 3.9 Report Generation Module
- PDF report generation with Puppeteer
- Customizable sections
- Branded reports (logo, company name)
- Download and share functionality

### 3.10 Admin Panel Module
- User management
- Dataset moderation (approve/reject listings)
- Platform analytics dashboard
- Content moderation

---

## 4. Data Flow

### 4.1 Dataset Upload Flow
```
User uploads file → Validate → Save to S3 → Create DB record 
→ Queue cleaning job → Python service cleans data 
→ Store in dataset_data table → Update status → Notify user
```

### 4.2 Insight Generation Flow
```
Request insights → Check Redis cache → If miss: Fetch data 
→ Python statistical analysis → OpenAI LLM for text generation 
→ Store insights → Cache results → Return to client
```

### 4.3 Marketplace Purchase Flow
```
Initiate purchase → Validate listing → Process payment (Razorpay) 
→ Create purchase record → Update seller earnings 
→ Grant dataset access → Generate download link → Send notifications
```

---

## 5. Database Schema

### 5.1 Core Tables

**users**
```sql
id (UUID, PK), email (unique), password_hash, name, role, 
business_name, verified, earnings, created_at, updated_at
```

**datasets**
```sql
id (UUID, PK), user_id (FK), name, file_path, file_size, 
file_type, row_count, column_count, date_range_start, 
date_range_end, status, column_mapping (JSONB), 
cleaning_report (JSONB), created_at, updated_at
```

**dataset_data** (processed data)
```sql
id (BIGSERIAL, PK), dataset_id (FK), date, product_name, 
category, quantity, price, revenue, customer_id, region, 
discount, order_id, created_at
```

**insights**
```sql
id (UUID, PK), dataset_id (FK), type, category, metric, 
value, text, confidence, data (JSONB), created_at
```

**forecasts**
```sql
id (UUID, PK), dataset_id (FK), forecast_type, periods, 
forecast_data (JSONB), accuracy_metrics (JSONB), 
model_params (JSONB), created_at, valid_until
```

**listings**
```sql
id (UUID, PK), user_id (FK), dataset_id (FK), title, 
description, price, category, tags, sample_data (JSONB), 
status, views, created_at, updated_at, approved_at
```

**purchases**
```sql
id (UUID, PK), buyer_id (FK), seller_id (FK), listing_id (FK), 
amount, platform_fee, seller_earnings, payment_id, 
payment_method, status, created_at
```

**reviews**
```sql
id (UUID, PK), listing_id (FK), user_id (FK), rating, 
comment, created_at
```

**reports**
```sql
id (UUID, PK), dataset_id (FK), user_id (FK), url, 
options (JSONB), file_size, created_at
```

**sessions**
```sql
id (UUID, PK), user_id (FK), token_hash, expires_at, created_at
```

### 5.2 Indexes
- users: email, role
- datasets: user_id, status, created_at
- dataset_data: dataset_id, date, product_name
- insights: dataset_id, type
- listings: user_id, status, category, created_at
- purchases: buyer_id, seller_id, listing_id, created_at

---

## 6. API Endpoints Summary

### 6.1 Authentication
- `POST /auth/register` - User registration
- `POST /auth/login` - User login
- `POST /auth/refresh` - Refresh access token
- `POST /auth/logout` - Logout

### 6.2 Datasets
- `POST /datasets/upload` - Upload dataset
- `GET /datasets` - List user's datasets
- `GET /datasets/:id` - Get dataset details
- `DELETE /datasets/:id` - Delete dataset

### 6.3 Dashboard
- `GET /dashboard/metrics/:datasetId` - Get key metrics
- `GET /dashboard/charts/:datasetId` - Get chart data

### 6.4 Insights
- `GET /insights/:datasetId` - Get insights
- `POST /insights/:datasetId/regenerate` - Regenerate insights

### 6.5 Forecasting
- `GET /forecasts/:datasetId` - Get forecast
- `POST /forecasts/:datasetId/train` - Train forecast model

### 6.6 Recommendations
- `GET /recommendations/:datasetId` - Get recommendations

### 6.7 Marketplace
- `POST /marketplace/listings` - Create listing
- `GET /marketplace/listings` - Browse listings
- `GET /marketplace/listings/:id` - Get listing details
- `POST /marketplace/purchase` - Purchase dataset

### 6.8 Reports
- `POST /reports/generate` - Generate PDF report
- `GET /reports/:id` - Get report download URL

### 6.9 Admin
- `GET /admin/dashboard` - Platform analytics
- `GET /admin/users` - List users
- `PATCH /admin/users/:id` - Update user
- `GET /admin/listings/pending` - Pending listings
- `PATCH /admin/listings/:id/approve` - Approve listing
- `PATCH /admin/listings/:id/reject` - Reject listing

---

## 7. ML Pipeline Overview

### 7.1 Data Preparation
```python
# Feature engineering
- Time features: day_of_week, month, quarter, is_weekend
- Lag features: revenue_lag_7, revenue_lag_30
- Rolling statistics: revenue_rolling_7, revenue_rolling_30
```

### 7.2 Forecasting Model
```python
# Prophet model with Indian holidays
model = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=True,
    changepoint_prior_scale=0.05
)
model.add_country_holidays(country_name='IN')
model.fit(train_data)
forecast = model.predict(future_dataframe)
```

### 7.3 Insights Generation
```python
# Statistical analysis → LLM text generation
1. Calculate trends (linear regression)
2. Detect anomalies (Z-score method)
3. Identify patterns (seasonality detection)
4. Generate natural language with OpenAI GPT-4
```

### 7.4 Recommendation Engine
```python
# Product recommendations
- Score products by: revenue, growth, margin
- Weighted scoring: 0.4*revenue + 0.3*growth + 0.3*margin

# Pricing optimization
- Calculate demand elasticity
- Test price points for maximum revenue

# Inventory optimization
- Safety stock = z_score * std * sqrt(lead_time)
- Reorder point = avg_demand * lead_time + safety_stock
```

---

## 8. Security Considerations

### 8.1 Authentication & Authorization
- JWT tokens with 24-hour expiration
- Refresh tokens for extended sessions
- Role-based access control (RBAC)
- Password hashing with bcrypt (10 rounds)

### 8.2 Data Security
- HTTPS/TLS 1.3 for all communications
- Database encryption at rest
- S3 server-side encryption (SSE-S3)
- Sensitive fields encrypted with AES-256

### 8.3 Input Validation
- Zod schema validation for all inputs
- File type and size validation
- SQL injection prevention (Prisma ORM)
- XSS prevention (input sanitization)

### 8.4 Rate Limiting
- General API: 100 requests per 15 minutes
- ML endpoints: 10 requests per hour
- Upload endpoint: 5 uploads per hour

### 8.5 Data Access Control
- Users can only access their own datasets
- Purchased datasets grant read-only access
- Admin role for moderation

---

## 9. Deployment Plan

### 9.1 Development Environment
- Local PostgreSQL database
- Local file storage
- ML service on localhost:8000

### 9.2 Production Environment
```
Frontend: Vercel (automatic deployments from main branch)
Backend: AWS EC2 / Render (Docker container)
ML Service: AWS Lambda (serverless) or dedicated EC2
Database: AWS RDS PostgreSQL
Cache: AWS ElastiCache Redis
Storage: AWS S3 (ap-south-1 region)
CDN: CloudFlare
```

### 9.3 CI/CD Pipeline
```yaml
# GitHub Actions
1. Run tests (Jest, Pytest)
2. Build Docker images
3. Push to container registry
4. Deploy to production
5. Run smoke tests
```

### 9.4 Monitoring
- Sentry for error tracking
- CloudWatch for logs and metrics
- Uptime monitoring (UptimeRobot)

---

## 10. Future Scope

### Phase 2 
- API integrations (Shopify, WooCommerce, Amazon)
- Advanced ML models (customer segmentation, churn prediction)
- Email notifications and alerts
- Scheduled report generation

### Phase 3 
- Mobile app (React Native)
- Multi-language support (Hindi, Tamil, Telugu)
- Collaborative features (team accounts)
- White-label solution for enterprises

### Phase 4 
- AI chatbot for data queries
- Automated marketing campaign suggestions
- Integration with accounting software
- Industry-specific templates

---

## 11. MVP Scope (Hackathon - 48 Hours)

**Must Have:**
1. User authentication (register, login)
2. CSV upload and basic cleaning
3. Dashboard with key metrics
4. 3-5 AI-generated insights
5. Simple 30-day sales forecast
6. PDF report generation
7. Basic admin panel

**Should Have (if time permits):**
1. Excel file support
2. Marketplace listing (no payment)
3. Advanced visualizations
4. Custom date ranges


---

## Conclusion

This design provides a complete blueprint for building EDI within the hackathon timeline. The architecture is scalable, secure, and optimized for Indian MSMEs. The modular design allows for incremental development and easy extension post-hackathon.
