# Zen Bali Deployment Guide

## Local Development

### Prerequisites
- Go 1.22+
- Docker & Docker Compose
- PostgreSQL client (optional, for direct DB access)

### Setup Steps

1. **Clone Repository**
```bash
git clone https://github.com/net1io/zenbali.git
cd zenbali
```

2. **Start Database**
```bash
docker-compose up -d
```

3. **Configure Environment**
```bash
cp .env.example .env
# Edit .env with your settings
```

4. **Run Application**
```bash
cd backend
go mod download
go run ./cmd/server
```

5. **Access**
- Frontend: http://localhost:8080
- API: http://localhost:8080/api

### Default Admin
- Email: admin@zenbali.org
- Password: admin123

---

## Production Deployment (GCP)

### Prerequisites
- GCP Account with billing enabled
- gcloud CLI installed
- Domain pointing to GCP

### GCP Services Used
- Cloud Run (application)
- Cloud SQL (PostgreSQL)
- Cloud Storage (images)
- Cloud Build (CI/CD)

### Step 1: Create Cloud SQL Instance

```bash
# Create PostgreSQL instance
gcloud sql instances create zenbali-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=asia-southeast1

# Create database
gcloud sql databases create zenbali --instance=zenbali-db

# Set user password
gcloud sql users set-password postgres \
  --instance=zenbali-db \
  --password=YOUR_SECURE_PASSWORD
```

### Step 2: Create Cloud Storage Bucket

```bash
gsutil mb -l asia-southeast1 gs://zenbali-images
gsutil iam ch allUsers:objectViewer gs://zenbali-images
```

### Step 3: Build & Deploy to Cloud Run

```bash
# Build container
gcloud builds submit --tag gcr.io/YOUR_PROJECT/zenbali

# Deploy
gcloud run deploy zenbali \
  --image gcr.io/YOUR_PROJECT/zenbali \
  --platform managed \
  --region asia-southeast1 \
  --allow-unauthenticated \
  --add-cloudsql-instances YOUR_PROJECT:asia-southeast1:zenbali-db \
  --set-env-vars "DB_HOST=/cloudsql/YOUR_PROJECT:asia-southeast1:zenbali-db" \
  --set-env-vars "DB_USER=postgres" \
  --set-env-vars "DB_PASSWORD=YOUR_SECURE_PASSWORD" \
  --set-env-vars "DB_NAME=zenbali" \
  --set-env-vars "ENV=production"
```

### Step 4: Configure Domain (Cloudflare)

1. Add domain to Cloudflare
2. Update DNS:
   - A record: @ → Cloud Run IP
   - CNAME: www → your-cloud-run-url.run.app
3. Enable SSL (Full Strict)

### Step 5: Setup Stripe

1. Create Stripe account
2. Get API keys from Stripe Dashboard
3. Create webhook endpoint: `https://zenbali.org/api/webhooks/stripe`
4. Subscribe to events:
   - checkout.session.completed
   - checkout.session.expired

---

## Environment Variables (Production)

```
PORT=8080
ENV=production
BASE_URL=https://zenbali.org

DB_HOST=/cloudsql/PROJECT:REGION:INSTANCE
DB_USER=postgres
DB_PASSWORD=secure_password
DB_NAME=zenbali
DB_SSL_MODE=disable

JWT_SECRET=long_random_string_min_32_chars

STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_PUBLISHABLE_KEY=pk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRICE_CENTS=1000

GCS_BUCKET=zenbali-images
GCS_PROJECT_ID=your-project-id

ADMIN_EMAIL=admin@zenbali.org
ADMIN_PASSWORD=secure_admin_password
```

---

## Monitoring

- Cloud Run metrics in GCP Console
- Set up Cloud Monitoring alerts
- Enable Cloud Logging for error tracking

---

## Estimated Costs

| Service | Estimate/Month |
|---------|---------------|
| Cloud Run | $5-15 |
| Cloud SQL (db-f1-micro) | $10-15 |
| Cloud Storage | $1-5 |
| **Total** | **$20-40** |
