# Case Study - HR System

## Problem Statement

A certain company needs a new HR system for managing its employees, salaries, vacations and payments.

---

## Functional Requirements

**Web-based system** with the following capabilities:

### Employee Management
- Perform CRUD operations on employees
- Support file attachments

### Salary Management
- Allow managers to request employee salary changes
- Allow HR manager to approve or reject salary change requests
- Support file attachments

### Vacation Management
- Manage vacation days
- Support file attachments

### Payment Integration
- Use external payment system

---

## Non-Functional Requirements

- Classic information system
- Not a lot of users
- Not a lot of data
- Interface to external system

---

## External Payment System

- **Technology**: Legacy system written in C
- **Communication**: File-based (input only)
- **Frequency**: Files received once a month

---

<img width="2609" height="1600" alt="HR System" src="https://github.com/user-attachments/assets/ab6d1a13-3629-4d3a-8be2-98b43de2058c" />

---

# Architecture Reasoning

## DNS, TLS, and Content Delivery

### Amazon Route 53
- Manages DNS records for the HR system domain.
- Resolves user requests to CloudFront.

### AWS Certificate Manager (ACM)
- Provides SSL/TLS certificates for HTTPS.
- Secures communication between users and the system.

### Amazon CloudFront
- Serves the frontend PWA from Amazon S3.
- Caches static assets such as:
  - JavaScript
  - CSS
  - Images
  - Documents
- Reduces latency and improves load times by serving content closer to users.

---

## API Layer

### Amazon API Gateway
- Single entry point for all backend APIs.
- Routes requests to the correct Lambda service.
- Can handle:
  - Authentication
  - Authorization
  - Throttling
  - Request validation
  - Monitoring

---

## Backend Services

### AWS Lambda

Separate Lambda functions are used for:
- Employee Management
- Salary Management
- Vacation Management

### Why Lambda?
- Cost efficient for low-traffic systems
- No need to manage servers
- Automatically scales
- Pay only when functions are invoked

### Tradeoff: Cold Starts
- Lambda may have cold start delays after inactivity.
- Acceptable for this HR system because:
  - HR operations are administrative tasks
  - Not customer-facing real-time operations
  - Small delays are acceptable

---

## Networking and Security

### VPC + Private Subnet
- Backend Lambdas run inside a VPC.
- The database is placed in a private subnet.
- The database is not publicly accessible from the internet.
- Only backend services inside the VPC can access the database.

---

## Database

### Amazon RDS
- Relational database fits the HR system data model well.
- Suitable for:
  - Employees
  - Salaries
  - Vacation records
  - Approval workflows
  - Payment records

---

## File Upload Strategy

### Amazon S3
- File attachments are stored in S3 instead of the database.
- Backend generates pre-signed URLs.
- Frontend uploads files directly to S3.

### Benefits
- Reduces backend workload
- Avoids sending large files through Lambda
- More scalable and efficient

---

## Payment File Export

### Scheduled AWS Lambda
- Runs automatically once per month.
- Retrieves payment data from the database.
- Generates payment files required by the legacy external system.
- Uploads or sends the generated file to the external system.

---

# Mermaid Sequence Diagram

# Mermaid Sequence Diagram - Main User Flow

```mermaid
sequenceDiagram
    autonumber

    actor User

    participant R53 as Route 53
    participant CF as CloudFront
    participant S3Frontend as S3 Frontend PWA
    participant API as API Gateway

    participant Emp as Employee Lambda
    participant Salary as Salary Lambda
    participant Vacation as Vacation Lambda

    participant DB as Amazon RDS
    participant FileS3 as S3 File Storage

    User->>R53: Access HR System
    R53->>CF: Resolve domain
    CF->>S3Frontend: Request frontend assets
    S3Frontend-->>CF: Return HTML/CSS/JS
    CF-->>User: Load PWA

    User->>API: Send API request

    alt Employee Management
        API->>Emp: Forward employee request
        Emp->>DB: CRUD employee data
        DB-->>Emp: Return result
        Emp-->>API: Response
    end

    alt Salary Management
        API->>Salary: Forward salary request
        Salary->>DB: Process salary workflow
        DB-->>Salary: Return result
        Salary-->>API: Response
    end

    alt Vacation Management
        API->>Vacation: Forward vacation request
        Vacation->>DB: Manage vacation records
        DB-->>Vacation: Return result
        Vacation-->>API: Response
    end

    User->>API: Request pre-signed upload URL
    API->>Emp: Generate S3 pre-signed URL
    Emp-->>API: Return upload URL
    API-->>User: Return upload URL

    User->>FileS3: Upload attachment directly
```

---

# Mermaid Sequence Diagram - Monthly Payment Export Flow

```mermaid
sequenceDiagram
    autonumber

    participant Scheduler as EventBridge Scheduler
    participant PaymentLambda as Scheduled Payment Lambda
    participant DB as Amazon RDS
    participant S3 as S3 Payment File Bucket
    participant External as External Payment System

    Scheduler->>PaymentLambda: Trigger monthly payment export

    PaymentLambda->>DB: Retrieve approved payment data
    DB-->>PaymentLambda: Return payment data

    PaymentLambda->>PaymentLambda: Generate payment file

    PaymentLambda->>S3: Upload payment file

    External->>S3: Read / pull payment file
    S3-->>External: Return payment file
```
