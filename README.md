# GigsWork — Freelancing Platform on AWS

A cloud-based freelancing marketplace connecting clients and freelancers, built with **API Platform (Symfony/PHP)** as the backend, **React.js** as the frontend, and deployed on **AWS** infrastructure.

> Built as an academic project to demonstrate cloud-native application architecture using real AWS services.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [AWS Services Used](#aws-services-used)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [Lambda Functions](#lambda-functions)
- [Project Structure](#project-structure)
- [Setup & Deployment](#setup--deployment)
- [Development Plan](#development-plan)

---

## Project Overview

GigsWork is a full-stack freelancing platform similar to Fiverr or Upwork. It allows:

- **Clients** to post gigs, review proposals, hire freelancers, and make payments
- **Freelancers** to browse gigs, submit proposals, communicate with clients, and receive payments

---

## Tech Stack

| Layer | Technology | Why we chose it |
|---|---|---|
| Frontend | React.js | Fast, component-based UI; industry standard |
| Backend | API Platform (Symfony/PHP) | Auto-generates REST & GraphQL APIs from PHP entity models |
| Database | AWS RDS (PostgreSQL) | Managed, reliable relational database |
| Auth | AWS Cognito | Handles JWT tokens, OAuth 2.0, and user pools |
| File Storage | AWS S3 | Stores portfolios, profile pictures, and uploaded files |
| CDN | AWS CloudFront | Delivers the React app globally over HTTPS |
| Serverless | AWS Lambda | Handles event-driven tasks like email triggers and payment hooks |
| Cache | AWS ElastiCache (Redis) | Session management and rate limiting |
| Monitoring | AWS CloudWatch | Logs, metrics, and alerts |

---

## System Architecture

```
User / Browser
      │
      ▼
 CloudFront (CDN + HTTPS)
      │
      ├──► S3 Bucket (React SPA static files)
      │
      ▼
 API Gateway (routing + rate limiting)
      │
      ▼
 ECS Fargate (API Platform / Symfony containers)
      │
      ├──► RDS Aurora PostgreSQL  (users, gigs, proposals, payments)
      ├──► ElastiCache Redis       (sessions, caching)
      ├──► S3 Media Bucket         (file uploads)
      └──► Lambda Functions        (email triggers, payment hooks)

Supporting Services:
  ├── AWS Cognito     → Authentication (JWT + OAuth 2.0)
  ├── AWS SES         → Transactional emails
  └── AWS CloudWatch  → Logs and monitoring
```

---

## AWS Services Used

### 1. Amazon S3 (Simple Storage Service)
Used for two purposes:
- **Static hosting** — the React build is uploaded here and served via CloudFront
- **Media storage** — user-uploaded files like profile photos and portfolio documents

### 2. Amazon CloudFront
A CDN that sits in front of the S3 bucket. It provides:
- HTTPS encryption for all traffic
- Global edge caching for fast load times
- Custom domain support

### 3. Amazon API Gateway
Acts as the single entry point for all API requests. It:
- Routes requests to the ECS backend
- Enforces rate limiting to prevent abuse
- Handles CORS headers

### 4. AWS ECS Fargate
Runs the API Platform (Symfony) Docker containers. Fargate is serverless — no EC2 instances to manage. It auto-scales based on traffic.

### 5. Amazon RDS (Aurora PostgreSQL)
The primary relational database. Stores all structured data — users, gigs, proposals, and payments. Aurora provides automated backups and failover.

### 6. AWS Cognito
Handles all authentication:
- User sign-up and sign-in
- JWT token generation and validation
- OAuth 2.0 (Google/GitHub login)

### 7. AWS Lambda
Serverless functions triggered by events (e.g. new proposal submitted → send email notification). See [Lambda Functions](#lambda-functions) section for full code.

### 8. Amazon SES (Simple Email Service)
Sends transactional emails such as:
- Welcome emails on registration
- Notification when a proposal is received
- Payment confirmation

### 9. ElastiCache (Redis)
Used for:
- Caching frequently accessed gig listings
- Storing user session data

### 10. CloudWatch
Captures logs from ECS and Lambda, sets alarms for errors, and provides metrics dashboards.

---

## Database Schema

Five core tables form the data model:

```sql
-- Users (freelancers and clients share this table, differentiated by role)
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(150) UNIQUE NOT NULL,
    role        VARCHAR(20) CHECK (role IN ('freelancer', 'client')) NOT NULL,
    bio         TEXT,
    skills      TEXT[],
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Gigs posted by clients
CREATE TABLE gigs (
    id          SERIAL PRIMARY KEY,
    client_id   INTEGER REFERENCES users(id) ON DELETE CASCADE,
    title       VARCHAR(200) NOT NULL,
    description TEXT NOT NULL,
    budget      NUMERIC(10, 2) NOT NULL,
    deadline    DATE,
    status      VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open', 'in_progress', 'closed')),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Proposals submitted by freelancers
CREATE TABLE proposals (
    id            SERIAL PRIMARY KEY,
    gig_id        INTEGER REFERENCES gigs(id) ON DELETE CASCADE,
    freelancer_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    cover_letter  TEXT NOT NULL,
    bid_amount    NUMERIC(10, 2) NOT NULL,
    status        VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'rejected')),
    submitted_at  TIMESTAMP DEFAULT NOW()
);

-- Payments once a proposal is accepted
CREATE TABLE payments (
    id            SERIAL PRIMARY KEY,
    gig_id        INTEGER REFERENCES gigs(id),
    freelancer_id INTEGER REFERENCES users(id),
    client_id     INTEGER REFERENCES users(id),
    amount        NUMERIC(10, 2) NOT NULL,
    status        VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'released', 'refunded')),
    paid_at       TIMESTAMP
);

-- Reviews left after work is delivered
CREATE TABLE reviews (
    id            SERIAL PRIMARY KEY,
    gig_id        INTEGER REFERENCES gigs(id),
    reviewer_id   INTEGER REFERENCES users(id),
    reviewee_id   INTEGER REFERENCES users(id),
    rating        INTEGER CHECK (rating BETWEEN 1 AND 5),
    comment       TEXT,
    created_at    TIMESTAMP DEFAULT NOW()
);
```

---

## API Endpoints

API Platform auto-generates these endpoints from the PHP entity classes:

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/users` | Register a new user |
| GET | `/api/users/{id}` | Get user profile |
| GET | `/api/gigs` | List all open gigs |
| POST | `/api/gigs` | Post a new gig (client only) |
| GET | `/api/gigs/{id}` | Get a single gig |
| POST | `/api/proposals` | Submit a proposal (freelancer only) |
| GET | `/api/proposals?gig={id}` | Get proposals for a gig |
| PATCH | `/api/proposals/{id}` | Accept or reject a proposal |
| POST | `/api/payments` | Create a payment record |
| POST | `/api/reviews` | Leave a review |

All endpoints are secured with JWT tokens issued by AWS Cognito.

---

## Lambda Functions

AWS Lambda functions handle background tasks triggered by events. All functions are written in Python 3.11.

---

### 1. Send Email on New Proposal

**Trigger:** API Gateway POST to `/api/proposals`

This function fires whenever a freelancer submits a proposal. It emails the client to notify them.

```python
import boto3
import json
import os

ses = boto3.client('ses', region_name=os.environ['AWS_REGION'])

def lambda_handler(event, context):
    """
    Triggered when a new proposal is submitted.
    Sends a notification email to the client who posted the gig.
    """
    try:
        body = json.loads(event['body'])

        client_email = body.get('client_email')
        freelancer_name = body.get('freelancer_name')
        gig_title = body.get('gig_title')

        if not all([client_email, freelancer_name, gig_title]):
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Missing required fields'})
            }

        response = ses.send_email(
            Source=os.environ['FROM_EMAIL'],
            Destination={'ToAddresses': [client_email]},
            Message={
                'Subject': {
                    'Data': f'New proposal received for: {gig_title}'
                },
                'Body': {
                    'Text': {
                        'Data': (
                            f'Hello,\n\n'
                            f'{freelancer_name} has submitted a proposal for your gig "{gig_title}".\n\n'
                            f'Log in to GigsWork to review and respond.\n\n'
                            f'— The GigsWork Team'
                        )
                    }
                }
            }
        )

        print(f"Email sent. MessageId: {response['MessageId']}")
        return {
            'statusCode': 200,
            'body': json.dumps({'message': 'Notification sent successfully'})
        }

    except Exception as e:
        print(f"Error sending email: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Failed to send notification'})
        }
```

---

### 2. Release Payment on Work Approval

**Trigger:** API Gateway PATCH to `/api/payments/{id}/release`

When a client approves delivered work, this function updates the payment status in the database.

```python
import boto3
import json
import os
import psycopg2
from datetime import datetime

def get_db_connection():
    """
    Returns a connection to the RDS PostgreSQL database.
    Credentials are stored in AWS Secrets Manager.
    """
    secrets_client = boto3.client('secretsmanager', region_name=os.environ['AWS_REGION'])
    secret = secrets_client.get_secret_value(SecretId=os.environ['DB_SECRET_ARN'])
    creds = json.loads(secret['SecretString'])

    return psycopg2.connect(
        host=creds['host'],
        database=creds['dbname'],
        user=creds['username'],
        password=creds['password'],
        port=creds.get('port', 5432)
    )

def lambda_handler(event, context):
    """
    Releases a payment to the freelancer after the client approves the work.
    Updates the payment record status to 'released'.
    """
    try:
        payment_id = event['pathParameters']['id']
        conn = get_db_connection()
        cursor = conn.cursor()

        # Check payment exists and is still pending
        cursor.execute(
            "SELECT id, status, freelancer_id, amount FROM payments WHERE id = %s",
            (payment_id,)
        )
        payment = cursor.fetchone()

        if not payment:
            return {'statusCode': 404, 'body': json.dumps({'error': 'Payment not found'})}

        if payment[1] != 'pending':
            return {'statusCode': 400, 'body': json.dumps({'error': f'Payment already {payment[1]}'})}

        # Release the payment
        cursor.execute(
            "UPDATE payments SET status = 'released', paid_at = %s WHERE id = %s",
            (datetime.utcnow(), payment_id)
        )
        conn.commit()
        cursor.close()
        conn.close()

        print(f"Payment {payment_id} released. Amount: {payment[3]} to freelancer {payment[2]}")
        return {
            'statusCode': 200,
            'body': json.dumps({'message': 'Payment released successfully', 'payment_id': payment_id})
        }

    except psycopg2.Error as db_err:
        print(f"Database error: {str(db_err)}")
        return {'statusCode': 500, 'body': json.dumps({'error': 'Database error'})}

    except Exception as e:
        print(f"Unexpected error: {str(e)}")
        return {'statusCode': 500, 'body': json.dumps({'error': 'Internal server error'})}
```

---

### 3. Send Welcome Email on Registration

**Trigger:** Amazon Cognito — Post Confirmation trigger

Fires automatically after a user confirms their email in Cognito.

```python
import boto3
import os

ses = boto3.client('ses', region_name=os.environ['AWS_REGION'])

def lambda_handler(event, context):
    """
    Cognito Post Confirmation trigger.
    Sends a welcome email after a user confirms their account.
    """
    user_email = event['request']['userAttributes']['email']
    user_name = event['request']['userAttributes'].get('name', 'there')

    try:
        ses.send_email(
            Source=os.environ['FROM_EMAIL'],
            Destination={'ToAddresses': [user_email]},
            Message={
                'Subject': {'Data': 'Welcome to GigsWork!'},
                'Body': {
                    'Text': {
                        'Data': (
                            f'Hi {user_name},\n\n'
                            f'Welcome to GigsWork! Your account is ready.\n\n'
                            f'As a freelancer, you can start browsing gigs and submitting proposals.\n'
                            f'As a client, you can post your first gig right away.\n\n'
                            f'Get started: https://gigswork.example.com\n\n'
                            f'— The GigsWork Team'
                        )
                    }
                }
            }
        )

        print(f"Welcome email sent to {user_email}")

    except Exception as e:
        # Do not block Cognito flow on email failure — just log
        print(f"Failed to send welcome email to {user_email}: {str(e)}")

    # Must return the event back to Cognito
    return event
```

---

### 4. Clean Up Expired Gigs (Scheduled)

**Trigger:** Amazon EventBridge — runs daily at midnight

Automatically closes gigs that have passed their deadline without being filled.

```python
import boto3
import json
import os
import psycopg2
from datetime import datetime

def get_db_connection():
    secrets_client = boto3.client('secretsmanager', region_name=os.environ['AWS_REGION'])
    secret = secrets_client.get_secret_value(SecretId=os.environ['DB_SECRET_ARN'])
    creds = json.loads(secret['SecretString'])
    return psycopg2.connect(
        host=creds['host'], database=creds['dbname'],
        user=creds['username'], password=creds['password']
    )

def lambda_handler(event, context):
    """
    Scheduled daily by EventBridge.
    Closes all gigs where deadline has passed and status is still 'open'.
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()

        today = datetime.utcnow().date()

        cursor.execute(
            """
            UPDATE gigs
            SET status = 'closed'
            WHERE deadline < %s AND status = 'open'
            RETURNING id, title
            """,
            (today,)
        )

        closed_gigs = cursor.fetchall()
        conn.commit()
        cursor.close()
        conn.close()

        print(f"Closed {len(closed_gigs)} expired gigs: {[g[0] for g in closed_gigs]}")
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': f'{len(closed_gigs)} gigs closed',
                'gig_ids': [g[0] for g in closed_gigs]
            })
        }

    except Exception as e:
        print(f"Error closing expired gigs: {str(e)}")
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

---

## Project Structure

```
gigswork/
│
├── backend/                        # API Platform (Symfony/PHP)
│   ├── src/
│   │   ├── Entity/
│   │   │   ├── User.php
│   │   │   ├── Gig.php
│   │   │   ├── Proposal.php
│   │   │   ├── Payment.php
│   │   │   └── Review.php
│   │   └── Security/
│   │       └── JwtAuthenticator.php
│   ├── config/
│   │   └── packages/
│   │       └── api_platform.yaml
│   ├── docker-compose.yml
│   └── Dockerfile
│
├── frontend/                       # React.js
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Home.jsx
│   │   │   ├── Login.jsx
│   │   │   ├── Register.jsx
│   │   │   ├── GigDetail.jsx
│   │   │   ├── PostGig.jsx
│   │   │   └── Dashboard.jsx
│   │   ├── components/
│   │   │   ├── GigCard.jsx
│   │   │   ├── ProposalForm.jsx
│   │   │   └── Navbar.jsx
│   │   └── services/
│   │       └── api.js              # Axios API calls
│   └── package.json
│
├── lambda/                         # AWS Lambda functions (Python)
│   ├── notify_new_proposal.py
│   ├── release_payment.py
│   ├── welcome_email.py
│   └── close_expired_gigs.py
│
├── infra/                          # AWS setup notes
│   └── aws-setup-checklist.md
│
└── README.md
```

---

## Setup & Deployment

### Prerequisites
- AWS account (free tier works for MVP)
- Node.js 18+ and npm
- PHP 8.2+ and Composer
- Docker Desktop

### Step 1 — Clone the repo

```bash
git clone https://github.com/yourusername/gigswork.git
cd gigswork
```

### Step 2 — Run backend locally with Docker

```bash
cd backend
docker-compose up --build
```

API will be available at `http://localhost:8080/api`

### Step 3 — Run frontend locally

```bash
cd frontend
npm install
npm start
```

Frontend runs at `http://localhost:3000`

### Step 4 — Set up AWS services

1. Create an **RDS Aurora PostgreSQL** instance (db.t3.micro for MVP)
2. Create two **S3 buckets** — one for the React build, one for media uploads
3. Create a **Cognito User Pool** — enable email sign-up and JWT token auth
4. Create a **CloudFront distribution** pointing to the React S3 bucket
5. Deploy Lambda functions from the `/lambda` folder via AWS Console or CLI
6. Set up **SES** — verify your sender email address

### Step 5 — Deploy backend to ECS Fargate

```bash
# Build and push Docker image to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin <your-ecr-uri>
docker build -t gigswork-api ./backend
docker tag gigswork-api:latest <your-ecr-uri>/gigswork-api:latest
docker push <your-ecr-uri>/gigswork-api:latest

# Create ECS service (via AWS Console or CLI)
```

### Step 6 — Deploy frontend to S3 + CloudFront

```bash
cd frontend
npm run build

# Upload to S3
aws s3 sync ./build s3://your-gigswork-frontend-bucket --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
```

---

## Development Plan

| Phase | Duration | What was done |
|---|---|---|
| Planning | Week 1–2 | Defined requirements, designed database schema, set up AWS account and services |
| Backend APIs | Week 3–4 | Built PHP entities, configured API Platform, set up JWT auth with Cognito, wrote Lambda functions |
| Frontend UI | Week 5–6 | Built React pages — homepage, auth, gig listings, proposal form, dashboard |
| Deployment & Testing | Week 7–8 | Dockerised the backend, deployed to ECS Fargate, pushed frontend to S3/CloudFront, end-to-end testing |

---

## Environment Variables

Create a `.env` file in `backend/` with:

```env
DATABASE_URL=postgresql://user:password@your-rds-endpoint:5432/gigswork
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
COGNITO_USER_POOL_ID=us-east-1_XXXXXXX
COGNITO_CLIENT_ID=your_client_id
AWS_REGION=us-east-1
```

Lambda environment variables (set in AWS Console):

```
FROM_EMAIL=no-reply@gigswork.example.com
DB_SECRET_ARN=arn:aws:secretsmanager:...
AWS_REGION=us-east-1
```

---

## License

This project was built for academic purposes. Free to use and modify.
