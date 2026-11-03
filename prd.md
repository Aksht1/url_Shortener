URLForge — Scalable URL Shortening Platform
Project objective: Build a production-ready URL shortening platform similar to Bitly and TinyURL, capable of handling millions of shortened URLs, high read traffic, analytics, abuse prevention, and horizontal scaling.

This should not be just a basic CRUD application. It should demonstrate real-world software engineering skills, including distributed systems, database design, caching, API development, security, observability, and cloud deployment.
1. Product Overview
URLForge allows users to convert long URLs into short, shareable links.

Example:
- Original URL: https://example.com/products/electronics/smartphone/model-xyz
- Short URL: https://urlforge.com/aB92xK
- Redirect: Visiting the short URL redirects the visitor to the original destination.
The platform should support both anonymous and registered users, with different usage limits and permissions.
2. Goals and Success Criteria
Goal	Requirement
High availability	Target 99.9% monthly uptime
Scalability	Support millions of stored URLs
Performance	P95 redirect latency below 100 ms under defined benchmark conditions
Reliability	No lost confirmed URL creations
Security	Protect against malicious URLs and abuse
Analytics	Track clicks, devices, referrers, and geography where permitted
Maintainability	Modular architecture with automated testing

Important: These are engineering targets, not claims of achieved performance. They must be validated through load testing and production monitoring.
3. Target Users
1. Guest users: Create short URLs without registration, subject to rate limits.
2. Registered users: Manage URLs, customize aliases, view analytics.
3. Business users: Manage branded domains, campaigns, and team links.
4. Administrators: Monitor abuse, manage users, and maintain system health.
4. Core Functional Requirements
FR-01: URL Shortening
- Accept a valid HTTP or HTTPS URL.
- Generate a unique short code.
- Return the shortened URL.
- Support custom aliases.
- Detect duplicate URLs optionally.
- Support URL expiration.
- Allow users to deactivate or delete their links.
FR-02: URL Redirection
When a visitor opens a short URL:
1. Extract the short code.
2. Check Redis cache.
3. If found, retrieve the destination URL.
4. Otherwise, query the database.
5. Cache the result.
6. Record the click event asynchronously.
7. Redirect the visitor.
Use HTTP 302 as the default redirect initially. Allow 301 only when explicitly requested because permanent redirects can be cached by browsers and intermediaries.
FR-03: User Authentication
- Email registration and login.
- Google OAuth (optional).
- JWT-based authentication or secure sessions.
- Password hashing using Argon2id or bcrypt.
- Password reset.
- Role-based access control.
FR-04: Custom Aliases
Example:

urlforge.com/my-portfolio

Requirements:
- Alias must be unique.
- Reserved words must be blocked.
- Maximum alias length: 50 characters.
- Users cannot claim system-reserved routes.
FR-05: Link Management Dashboard
Users should be able to:
- View all shortened URLs.
- Search and filter URLs.
- Edit link metadata.
- Activate/deactivate links.
- Delete links.
- Copy shortened URLs.
- View creation and expiration dates.
FR-06: Analytics
Track:
- Total clicks.
- Unique visitors (privacy-preserving approximation).
- Clicks by day.
- Device type.
- Browser.
- Operating system.
- Referrer.
- Country-level location where legally permitted.
Analytics must be asynchronous so tracking does not slow down redirection.
5. System Architecture
The architecture should use a modular backend with independently scalable components. Start with a modular monolith if building the first version, but design the services so they can be separated later.
High-level architecture
Client Applications

Web Dashboard · Mobile Browser · API Clients


CDN + WAF + Load Balancer

TLS · DDoS protection · Rate limiting · Routing


Backend API / Application Layer
URL Service
Short code generation

Redirect Service
Fast URL resolution

Auth Service
Users and permissions

Analytics Service
Click tracking




PostgreSQL
Persistent URL and user data

Redis
Cache and rate limits

Message Queue
Async click events

Analytics Store
Aggregated click metrics



Architecture component responsibilities
Component	Responsibility
CDN	Edge caching and global delivery
WAF	Filter malicious requests
Load balancer	Distribute incoming traffic
API service	Manage users and URL operations
Redirect service	Resolve short codes
PostgreSQL	Store permanent records
Redis	Cache frequently accessed links
Message queue	Process click events asynchronously
Analytics worker	Aggregate click data
Object storage	Store exports and reports

5.1 URL Code Generation Strategy
Implement a unique short-code generation system.

Recommended approach: Base62 encoding.

Character set:
0123456789
abcdefghijklmnopqrstuvwxyz
ABCDEFGHIJKLMNOPQRSTUVWXYZ


A Base62 code uses 62 possible characters per position.
Code length	Possible combinations
4	14,776,336
5	916,132,832
6	56,800,235,584
7	3,521,614,606,208
8	218,340,105,584,896

Recommended initial code length: 7 characters.

Implementation options:
- Database-generated numeric IDs encoded in Base62.
- Distributed ID generation using a Snowflake-style algorithm.
- Random secure codes with database uniqueness constraints and collision retries.
Recommendation: Start with database-generated IDs and Base62 encoding. Use unique constraints and a collision-safe allocation strategy. Do not expose sequential internal IDs directly.
5.2 Caching Strategy
Use Redis to minimize database queries.

Cache structure:
Key: url:aB92xK
Value: https://example.com/products
TTL: 3600 seconds


Requirements:
- Cache frequently accessed links.
- Cache invalidation when URLs are updated or disabled.
- Use negative caching carefully for nonexistent codes.
- Prevent cache stampedes.
- Apply bounded cache memory and eviction policies.
5.3 Asynchronous Analytics
Redirects must not wait for analytics processing.

Recommended flow:
Visitor clicks short URL
        |
        v
Redirect Service
        |
        +------> Redis / PostgreSQL
        |              |
        |              v
        |         Destination URL
        |
        +------> Event Queue
                       |
                       v
                 Analytics Worker
                       |
                       v
                 Analytics Database


If the analytics queue is unavailable, the redirect service should still prioritize successful redirection. Define event durability and acceptable tracking loss separately.
6. Database Design
Use PostgreSQL as the primary relational database.
6.1 Users Table
Column	Type	Description
id	UUID	Primary key
name	VARCHAR(100)	User name
email	VARCHAR(255)	Unique email
password_hash	TEXT	Hashed password
role	VARCHAR(30)	User role
created_at	TIMESTAMP	Registration time
updated_at	TIMESTAMP	Last update

6.2 URLs Table
Column	Type	Description
id	BIGINT	Primary key
user_id	UUID	Owner
short_code	VARCHAR(20)	Unique short code
original_url	TEXT	Destination URL
custom_alias	VARCHAR(50)	Optional alias
is_active	BOOLEAN	Link status
expires_at	TIMESTAMP	Expiration time
created_at	TIMESTAMP	Creation time
updated_at	TIMESTAMP	Last modification

6.3 Click Events Table
Column	Type	Description
id	BIGINT	Event ID
short_code	VARCHAR(20)	Visited link
clicked_at	TIMESTAMP	Click time
country_code	VARCHAR(2)	Country
device_type	VARCHAR(30)	Device
browser	VARCHAR(100)	Browser
referrer_domain	VARCHAR(255)	Referrer
visitor_hash	VARCHAR(128)	Privacy-preserving identifier

Avoid storing raw IP addresses indefinitely. Use appropriate consent, retention, anonymization, and deletion policies.
6.4 Additional Tables
- api_keys: API access credentials.
- domains: Custom branded domains.
- teams: Business workspaces.
- team_members: Team permissions.
- url_tags: Link categorization.
- daily_analytics: Pre-aggregated analytics.
- audit_logs: Security and administrative actions.
Important database indexes:
CREATE UNIQUE INDEX idx_urls_short_code
ON urls(short_code);

CREATE INDEX idx_urls_user_id
ON urls(user_id);

CREATE INDEX idx_urls_created_at
ON urls(created_at);

CREATE INDEX idx_clicks_short_code_clicked_at
ON click_events(short_code, clicked_at);


For large analytics tables, use time-based partitioning and retention policies.
7. API Requirements
Use REST APIs with JSON responses. Version all public APIs under /api/v1.
7.1 URL APIs
Method	Endpoint	Purpose
POST	/api/v1/urls	Create short URL
GET	/api/v1/urls	List user's URLs
GET	/api/v1/urls/:id	Get URL details
PATCH	/api/v1/urls/:id	Update URL
DELETE	/api/v1/urls/:id	Delete URL
GET	/api/v1/urls/:id/analytics	Get analytics

7.2 Redirect API
GET /:shortCode


Example:
GET /aB92xK


Response:
HTTP/1.1 302 Found
Location: https://example.com/products


7.3 Authentication APIs
Method	Endpoint	Purpose
POST	/api/v1/auth/register	Register
POST	/api/v1/auth/login	Login
POST	/api/v1/auth/logout	Logout
POST	/api/v1/auth/refresh	Refresh session
POST	/api/v1/auth/forgot-password	Reset password

7.4 API Example: Create URL
Request:
{
  "originalUrl": "https://example.com/products",
  "customAlias": "my-product",
  "expiresAt": null
}


Response:
{
  "success": true,
  "data": {
    "shortCode": "my-product",
    "shortUrl": "https://urlforge.com/my-product",
    "originalUrl": "https://example.com/products",
    "createdAt": "2026-11-03T10:00:00Z"
  }
}


7.5 API Error Handling
Use consistent HTTP status codes.
Code	Meaning
200	Successful request
201	Resource created
301	Permanent redirect
302	Temporary redirect
400	Invalid request
401	Unauthorized
403	Forbidden
404	URL not found
409	Alias conflict
422	Validation error
429	Rate limit exceeded
500	Internal server error
503	Service unavailable

8. Frontend Requirements
Build a modern, responsive dashboard with a clean interface.
Suggested dashboard design







yorl.io


Dashboard features
- Total shortened URLs.
- Total clicks.
- Unique visitor estimates.
- Top-performing links.
- Click activity graph.
- Recent links.
- Create URL shortcut.
  
  Short URL creation page
  
  
  
  
  
  
  
  codester.com
  
  
  Creation features
- Long URL input.
- Custom alias.
- Expiration date.
- Copy short URL.
- QR code generation.
  
  Analytics page
  
  
  
  
  
  
  
  buffer.com
  
  
  Analytics features
- Click trends.
- Country breakdown.
- Browser distribution.
- Device distribution.
- Referrer information.

Frontend pages
1. Landing page.
2. Login and registration.
3. URL creation page.
4. User dashboard.
5. Link management.
6. Link analytics.
7. Account settings.
8. Admin dashboard.
9. API documentation.
10. 404 and expired-link pages.
9. Technology Stack
The following stack is recommended for a production-oriented implementation that is also practical for a student developer to build and deploy.









Frontend: Next.js + TypeScript
React-based dashboard, routing, SSR and responsive UI











Backend: Node.js + Express.js
REST APIs, URL creation, authentication and redirects











Database: PostgreSQL
Persistent data and relational queries











Cache: Redis
Fast URL lookup, distributed rate limiting











Event processing: Kafka or RabbitMQ
Asynchronous click tracking and analytics











Containerization: Docker
Consistent development and deployment











Cloud: AWS
EC2, RDS, ElastiCache, S3 and CloudWatch











CI/CD: GitHub Actions
Automated testing, building and deployment


Additional tools:
- Prisma ORM for database access.
- Zod for request validation.
- Jest or Vitest for unit testing.
- Playwright for end-to-end testing.
- Prometheus and Grafana for monitoring.
- OpenTelemetry for distributed tracing.
10. Security Requirements
Security is a critical part of a public URL shortener because users can create links pointing to external destinations.
10.1 URL Validation
- Accept only HTTP and HTTPS schemes.
- Reject javascript:, file:, data: and unsupported schemes.
- Reject malformed URLs.
- Normalize URLs before storing.
- Detect suspicious domains using threat-intelligence services.
- Recheck destination safety periodically where appropriate.
10.2 Abuse Prevention
- IP-based and account-based rate limiting.
- CAPTCHA for suspicious anonymous activity.
- Block phishing and malware destinations.
- Maintain domain blocklists.
- Provide abuse reporting.
- Provide administrator link suspension.
- Limit URL creation per account and per IP.
10.3 Authentication Security
- Secure password hashing.
- HTTP-only, Secure, SameSite cookies if using sessions.
- CSRF protection where applicable.
- JWT expiration and refresh-token rotation if using JWT.
- Role-based authorization.
- Account lockout or progressive delays for repeated failed logins.
10.4 Infrastructure Security
- HTTPS everywhere.
- Secrets stored in a secret manager.
- Database access restricted to application services.
- Encryption at rest.
- Regular backups.
- Audit logging.
- Dependency vulnerability scanning.
10.5 Redirect Security
Do not blindly redirect to arbitrary schemes or internal infrastructure addresses.

Implement protection against:
- SSRF.
- DNS rebinding.
- Phishing.
- Malicious redirects.
- Open redirect abuse in management flows.
A URL shortener intentionally redirects to external URLs, so the security objective is to screen dangerous destinations, not to block all external redirects.
11. Non-Functional Requirements
Category	Requirement
Availability	99.9% monthly target
Latency	P95 redirect under 100 ms in the defined benchmark environment
Scalability	Horizontal application scaling
Reliability	Durable URL creation
Consistency	Unique short codes enforced by database
Security	Authentication, validation and rate limiting
Maintainability	Modular codebase
Observability	Logs, metrics and traces
Backup	Automated database backups
Recovery	Documented disaster recovery process

Performance and scalability targets
Use these as initial capacity-planning scenarios, not guaranteed system capabilities.
Metric	Initial benchmark target
Stored URLs	10 million
Daily redirects	10 million
Peak redirects	5,000 requests/second
URL creation	100 requests/second
Cache hit ratio	90% or higher
Redirect latency	P95 below 100 ms

The system should be load-tested with realistic traffic distributions, cache misses, database failures, and traffic spikes.
12. Deployment Architecture
Recommended AWS deployment:
Users / Internet


Route 53 + CloudFront + AWS WAF
DNS, CDN and application protection

Application Load Balancer


EC2 / ECS
API instances

Worker instances
Analytics consumers

Amazon RDS
PostgreSQL

ElastiCache
Redis

SQS / MSK
Event queue

CloudWatch
Logs and monitoring



Deployment should include:
- Separate development, staging and production environments.
- Docker images.
- Infrastructure as Code using Terraform or AWS CDK.
- CI/CD pipeline.
- Rolling or blue-green deployments.
- Health checks and automated rollback.
- Database migration strategy.
13. Testing Requirements
The platform must pass the following test categories before production release.
Test	What to verify
Unit testing	Code generation, validation, expiration
API testing	Authentication, CRUD, error handling
Integration testing	PostgreSQL, Redis, queue
Load testing	High redirect traffic
Stress testing	Behavior beyond normal capacity
Security testing	SSRF, XSS, CSRF, abuse
End-to-end testing	Complete user workflows
Recovery testing	Database and cache failures

Critical test scenarios
1. Generate 100,000 short URLs without duplicate codes.
2. Redirect a user to the correct destination.
3. Ensure expired links return an appropriate error page.
4. Verify deleted and disabled links cannot redirect.
5. Confirm unauthorized users cannot edit another user's URLs.
6. Verify analytics events are not duplicated during retries.
7. Confirm redirect service continues operating during analytics-worker failure.
8. Test traffic spikes and cache outages.
9. Test malicious URLs and internal-address destinations.
10. Verify backups can be restored.
14. Development Roadmap
Recommended development timeline: 8 weeks.
1. Week 1 — Project foundation
   Foundation
   - Set up GitHub repository.
   - Configure frontend and backend.
   - Connect PostgreSQL.
   - Design database schema.
   - Configure Docker.
2. Week 2 — URL shortening engine
   - Implement Base62 code generation.
   - Implement URL validation.
   - Build create and redirect APIs.
   - Add expiration and custom aliases.
3. Week 3 — Authentication and dashboard
   - Registration and login.
   - Authentication middleware.
   - User dashboard.
   - URL management and search.
4. Week 4 — Redis and performance
   - Integrate Redis.
   - Cache URL lookups.
   - Add rate limiting.
   - Implement cache invalidation.
5. Week 5 — Analytics
   - Integrate message queue.
   - Implement asynchronous click tracking.
   - Create analytics workers.
   - Build charts and reports.
6. Week 6 — Security
   - Implement abuse detection.
   - Add domain blocklists.
   - Add audit logging.
   - Test SSRF and redirect protections.
7. Week 7 — Testing and optimization
   - Unit and integration testing.
   - Load testing.
   - Query optimization.
   - Failure and recovery testing.
8. Week 8 — Cloud deployment
   - Deploy on AWS.
   - Configure domain and HTTPS.
   - Set up CI/CD.
   - Configure monitoring and backups.
   - Prepare production documentation.

15. Production Release Acceptance Criteria
The project is ready for production only when:
- [ ] Users can create and manage short URLs.
- [ ] Short codes are unique under concurrent requests.
- [ ] Redirects work correctly.
- [ ] Redis caching is operational.
- [ ] URL expiration and deletion work correctly.
- [ ] Authentication and authorization are enforced.
- [ ] Analytics are processed asynchronously.
- [ ] Rate limits and abuse protection are implemented.
- [ ] Database backups and recovery are tested.
- [ ] CI/CD pipeline is operational.
- [ ] HTTPS is configured.
- [ ] Monitoring and alerting are configured.
- [ ] Load and security tests pass.
- [ ] Privacy and data-retention policies are documented.
16. GitHub Repository Structure
Keep the code organized and scalable from the beginning.
urlforge/
│
├── frontend/
│   ├── app/
│   ├── components/
│   ├── hooks/
│   ├── services/
│   ├── styles/
│   └── public/
│
├── backend/
│   ├── src/
│   │   ├── config/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── urls/
│   │   │   ├── redirects/
│   │   │   ├── analytics/
│   │   │   └── users/
│   │   ├── middleware/
│   │   ├── queues/
│   │   ├── workers/
│   │   ├── utils/
│   │   └── app.ts
│   │
│   └── tests/
│
├── infrastructure/
│   ├── docker/
│   ├── terraform/
│   └── monitoring/
│
├── docs/
│   ├── architecture.md
│   ├── api-documentation.md
│   └── database-design.md
│
├── docker-compose.yml
├── .github/
│   └── workflows/
├── .env.example
├── README.md
└── LICENSE


17. Future Enhancements
These features can be added after the core platform is stable.
Feature	Description
QR Code Generation	Generate downloadable QR codes
Branded Domains	Use custom domains
Bulk URL Shortening	Create hundreds of links at once
A/B Testing	Distribute visitors across destinations
Smart Redirects	Device- or location-based destinations
API Subscription Plans	Usage-based billing
Team Collaboration	Shared workspaces and permissions
Webhooks	Notify external applications
AI Link Classification	Categorize suspicious or legitimate URLs
Link Preview	Display destination information before redirect

18. Expected Final Deliverables
1. Fully functional URL shortening website.
2. Responsive frontend dashboard.
3. Production-oriented backend REST APIs.
4. PostgreSQL database.
5. Redis caching implementation.
6. Asynchronous analytics pipeline.
7. Authentication and authorization.
8. Admin dashboard.
9. Docker configuration.
10. Automated testing suite.
11. AWS deployment.
12. CI/CD pipeline.
13. API documentation.
14. Architecture documentation.
15. Load-testing report.

19. Instructions for AI Development (Antigravity)
Since you want to build a proper working project using an AI coding IDE, don't ask the AI to generate the entire application in one shot. Give it the PRD and make it implement the project in controlled phases.

Here is the master development prompt:
MASTER DEVELOPMENT PROMPT — URLForge
Act as a Senior Software Architect, Backend Engineer, Frontend Engineer, DevOps Engineer, and Security Engineer.

Build a production-oriented URL shortening platform called URLForge based on the attached Product Requirements Document.
Technology Stack
- Frontend: Next.js, React, TypeScript, Tailwind CSS.
- Backend: Node.js, Express.js, TypeScript.
- Database: PostgreSQL.
- ORM: Prisma.
- Cache: Redis.
- Message Queue: RabbitMQ initially, with an architecture that can migrate to Kafka.
- Authentication: Secure sessions or JWT with refresh-token rotation.
- Testing: Jest/Vitest, Supertest, Playwright.
- Containerization: Docker and Docker Compose.
- Cloud: AWS.
- CI/CD: GitHub Actions.
Mandatory Development Rules
1. Build a real working application, not a static UI prototype.
2. Implement actual database operations.
3. Implement working authentication and authorization.
4. Use unique, collision-safe short-code generation.
5. Implement Redis caching with cache invalidation.
6. Implement asynchronous analytics processing.
7. Implement proper error handling and request validation.
8. Add security protections against SSRF, phishing, malicious URLs, and abuse.
9. Use environment variables for all secrets and credentials.
10. Never hardcode production secrets.
11. Write unit, integration, and end-to-end tests.
12. Document all APIs.
13. Use clean, modular, maintainable code.
14. Do not leave placeholder functions or fake analytics.
15. Do not claim a feature is complete unless it has been implemented and tested.
Implementation Phases
Phase 1: Foundation
- Create frontend and backend.
- Configure TypeScript.
- Configure PostgreSQL and Prisma.
- Create database schema.
- Configure Docker Compose.
- Create health-check endpoints.
Phase 2: URL Engine
- Implement URL validation.
- Implement Base62 code generation.
- Implement unique alias handling.
- Implement URL creation.
- Implement redirects.
- Implement URL expiration.
- Add automated tests.
Phase 3: Authentication
- Registration.
- Login and logout.
- Password hashing.
- Session management.
- Protected routes.
- Role-based permissions.
Phase 4: Dashboard
- Responsive dashboard.
- URL creation form.
- URL management table.
- Search and filters.
- Copy URL functionality.
- Link status management.
Phase 5: Redis
- Cache frequently accessed URLs.
- Implement cache invalidation.
- Add rate limiting.
- Handle cache failures.
Phase 6: Analytics
- Integrate RabbitMQ.
- Create asynchronous click events.
- Implement analytics workers.
- Create daily analytics aggregation.
- Display analytics charts.
Phase 7: Security
- URL scheme validation.
- SSRF protection.
- Domain reputation checks.
- Abuse reporting.
- API rate limits.
- Secure authentication.
- Audit logs.
Phase 8: Testing and Optimization
- Unit testing.
- Integration testing.
- End-to-end testing.
- Load testing.
- Performance optimization.
- Failure recovery.
Phase 9: Deployment
- Production Docker images.
- AWS infrastructure.
- HTTPS and domain configuration.
- CI/CD.
- Logging and monitoring.
- Backup and recovery.
Execution Instructions
Before writing code:
1. Analyze the complete PRD.
2. Prepare a detailed technical architecture.
3. Create a development roadmap.
4. Define the database schema.
5. Define API contracts.
6. Create a checklist of all requirements.
Then implement one phase at a time.

After completing each phase:
- Run the application.
- Execute relevant tests.
- Fix all errors.
- Update documentation.
- Report completed features.
- Report remaining tasks.
Do not move to the next phase if critical tests are failing.
Final Objective
Deliver a fully functional, secure, scalable URL shortening platform with real database persistence, caching, asynchronous analytics, monitoring, automated tests, and cloud deployment.

The final product should demonstrate professional software engineering and be suitable for a serious portfolio and production-readiness evaluation.





20. My recommendation for your implementation
I recommend building URLForge in three levels rather than attempting the full distributed architecture immediately.
Level	Implementation	Objective
Level 1	Next.js + Node.js + PostgreSQL	Fully functional application
Level 2	Redis + RabbitMQ + analytics	Performance and asynchronous processing
Level 3	AWS + load balancing + monitoring + CI/CD	Production deployment and scaling

One important distinction: A project that runs successfully on your laptop is not automatically production-scale. The production claim should be supported by measured load-test results, reliability tests, security checks, and operational monitoring.