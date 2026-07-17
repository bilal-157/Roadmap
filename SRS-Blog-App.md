# Software Requirements Specification — Administrative Blog Application

## 1. Introduction

### 1.1 Purpose
This document specifies the requirements for a full-featured administrative blog web application built as a **single Laravel monolith** (Laravel, Blade, Alpine.js, Tailwind CSS, Nginx, PostgreSQL, Redis, Cloudflare, Docker). The application enables content administrators to create, manage, and publish blog content with robust security, performance, and scalability, delivered through a fast, lightweight, server-rendered frontend enhanced with Alpine.js for smooth, app-like interactivity — with no separate JavaScript frontend framework or build-heavy SPA layer.

### 1.2 Scope
The application provides:
- Content management system (CMS) for blog posts
- User authentication and role-based access control
- Rich media management
- Comment moderation
- Analytics and reporting
- SEO optimization tools
- Multi-user administrative capabilities
- A responsive, snappy Blade + Alpine.js admin and public UI (no separate frontend service to build, deploy, or version)

### 1.3 Definitions and Acronyms
| Term | Definition |
|------|------------|
| SRS | Software Requirements Specification |
| CMS | Content Management System |
| RBAC | Role-Based Access Control |
| 2FA | Two-Factor Authentication |
| API | Application Programming Interface |
| WAF | Web Application Firewall |
| CDN | Content Delivery Network |
| SPA | Single Page Application (explicitly **not** used in this project) |

---

## 2. Overall Description

### 2.1 Product Perspective
The system follows a simplified, monolithic architecture — Laravel serves both the API and the rendered frontend, so there is no separate frontend service to deploy or scale:

```
┌─────────────────────────────────────────────────────────────┐
│                     Cloudflare (CDN + WAF)                  │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                        Nginx (Reverse Proxy)                │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│     Laravel Application (Blade Views + Alpine.js + Tailwind)│
│        Server-rendered pages, REST/JSON endpoints,          │
│        Alpine.js for reactive UI, Vite-built assets          │
└─────────────────────────────────────────────────────────────┘
                    │                      │
┌───────────────────┴──────────┐ ┌─────────┴──────────────────┐
│         PostgreSQL           │ │         Redis              │
│      (Primary Database)      │ │   (Cache & Sessions)       │
└──────────────────────────────┘ └────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                Cloudflare R2 (File Storage)                 │
└─────────────────────────────────────────────────────────────┘
```

**Frontend approach:** All pages (public blog and admin panel) are rendered server-side by Laravel using Blade templates. Alpine.js is loaded on top for lightweight, declarative interactivity (dropdowns, modals, live filters, inline validation, drag-and-drop reordering, auto-save indicators, toasts, etc.) without the overhead of a full JavaScript framework, virtual DOM, or separate build/deploy pipeline. Tailwind CSS handles styling. Vite compiles and bundles Blade-scoped JS/CSS assets. Optional partial-page updates (e.g., for comment moderation actions or live search) are handled via small `fetch()`/Axios calls that swap HTML fragments returned by Laravel — keeping the app feeling smooth without adopting Livewire, Inertia, or a client-side SPA framework.

### 2.2 User Characteristics

| User Type | Description | Technical Level |
|-----------|-------------|-----------------|
| Super Admin | Full system control, user management | High |
| Editor | Create, edit, publish content | Medium |
| Author | Create and edit own drafts | Medium |
| Comment Moderator | Manage comments | Low |
| Guest Reader | View published content | Low |

### 2.3 Operating Environment
- **Server:** Linux (Ubuntu 22.04 LTS recommended)
- **Database:** PostgreSQL 14+
- **Cache:** Redis 7+
- **Web Server:** Nginx 1.22+
- **Container:** Docker 24+ / Docker Compose
- **CDN/WAF:** Cloudflare
- **Storage:** Cloudflare R2 (S3-compatible)

### 2.4 Design and Implementation Constraints
- Must use Laravel 10+ for backend, API, **and frontend rendering** (single application, single codebase)
- Frontend must be built using **Blade templates + Alpine.js + Tailwind CSS** — no Next.js, React, Vue, or other separate JS frontend framework, and no Livewire
- Must use Vite for asset bundling (Laravel's default Vite integration)
- Must use PostgreSQL for primary database
- Must use Redis for caching and session management
- Must be containerized using Docker
- Must integrate with Cloudflare for CDN and security
- Must use Cloudflare R2 for file storage
- Must follow RESTful API principles for any external/API consumers
- Admin panel and public site use standard Laravel session-based authentication; JWT is offered only for external API consumers
- Must support dark/light theme (implemented via Tailwind `dark:` classes toggled by Alpine.js, persisted via cookie/localStorage)

### 2.5 Assumptions and Dependencies
**Assumptions:**
- Administrators have reliable internet connectivity
- Users have modern web browsers
- Content will be primarily in English (with localization support)

**Dependencies:**
- Cloudflare account and API access
- Cloudflare R2 bucket configured
- SMTP server for email notifications
- Domain name with DNS configured

---

## 3. Functional Requirements

### 3.1 User Authentication & Authorization

#### FR-01: User Registration
| Element | Description |
|---------|-------------|
| **ID** | FR-01 |
| **Description** | Admin users can register with email verification |
| **Priority** | High |
| **Inputs** | Name, email, password, company (optional) |
| **Outputs** | Verification email sent; user created with 'Author' role |
| **Preconditions** | Email not already registered |
| **Postconditions** | User account created in pending verification state |

#### FR-02: User Login
| Element | Description |
|---------|-------------|
| **ID** | FR-02 |
| **Description** | Secure authentication with rate limiting and optional 2FA, via Laravel session-based auth (Blade login form enhanced with Alpine.js for inline validation) |
| **Priority** | High |
| **Inputs** | Email/username, password |
| **Outputs** | Authenticated session (cookie-based); JWT issued only for API-only clients |
| **Preconditions** | Account exists and is verified |
| **Postconditions** | User authenticated with session/cookie set |
| **Security** | Rate limit: 5 attempts per 5 minutes; password hashing (bcrypt/Argon2) |

#### FR-03: Role-Based Access Control (RBAC)
| Element | Description |
|---------|-------------|
| **ID** | FR-03 |
| **Description** | Granular permissions based on user roles |
| **Priority** | High |
| **Roles** | Super Admin, Editor, Author, Comment Moderator, Guest |
| **Permissions** | create:posts, edit:posts, delete:posts, publish:posts, manage:users, manage:comments, view:analytics |

#### FR-04: Two-Factor Authentication (2FA)
| Element | Description |
|---------|-------------|
| **ID** | FR-04 |
| **Description** | Optional 2FA for enhanced security |
| **Priority** | Medium |
| **Method** | TOTP (Google Authenticator/Authy) |
| **Configuration** | Per user preference |
| **Fallback** | Backup codes (10 one-time use codes) |

#### FR-05: Password Reset
| Element | Description |
|---------|-------------|
| **ID** | FR-05 |
| **Description** | Secure password reset via email |
| **Priority** | High |
| **Inputs** | Email address |
| **Outputs** | Reset link sent to registered email |

---

### 3.2 Content Management

#### FR-06: Create Blog Post
| Element | Description |
|---------|-------------|
| **ID** | FR-06 |
| **Description** | Authors can create new blog posts via a Blade + Alpine.js editor screen (rich-text editor mounted with Alpine, auto-save via periodic `fetch()` calls) |
| **Priority** | High |
| **Inputs** | Title, content (rich text), excerpt, featured image, categories, tags, SEO meta data |
| **Outputs** | Draft post saved; auto-save indicator shown inline |
| **Data Validation** | Title: required, max 255 chars; Content: required, min 50 chars |
| **Statuses** | Draft, Published, Scheduled, Archived |

#### FR-07: Edit Blog Post
| Element | Description |
|---------|-------------|
| **ID** | FR-07 |
| **Description** | Users can edit posts they have permission for |
| **Priority** | High |
| **Inputs** | All post fields (editable) |
| **Outputs** | Updated post saved; version history updated |
| **Constraints** | Only Authors can edit own drafts; Editors can edit any post |

#### FR-08: Delete Blog Post
| Element | Description |
|---------|-------------|
| **ID** | FR-08 |
| **Description** | Move posts to trash (soft delete) or permanent deletion, confirmed via an Alpine.js modal dialog |
| **Priority** | Medium |
| **Inputs** | Post ID |
| **Outputs** | Post moved to trash; optional restore within 30 days |
| **Constraints** | Only Super Admin and Editors can delete |

#### FR-09: Publish Blog Post
| Element | Description |
|---------|-------------|
| **ID** | FR-09 |
| **Description** | Change post status from draft to published |
| **Priority** | High |
| **Inputs** | Post ID, publish date (scheduled) |
| **Outputs** | Post published; notification sent to subscribers |
| **Constraints** | Only Editors and Super Admin can publish |

#### FR-10: Content Versioning
| Element | Description |
|---------|-------------|
| **ID** | FR-10 |
| **Description** | Track changes to posts with version history; diff view rendered server-side as a Blade partial |
| **Priority** | Medium |
| **Inputs** | Auto-save or manual save |
| **Outputs** | Version created; diff view available |
| **Retention** | Last 50 versions stored |

#### FR-11: Content Search
| Element | Description |
|---------|-------------|
| **ID** | FR-11 |
| **Description** | Full-text search across posts; results filtered live in the browser via a small Alpine.js component calling a lightweight Laravel search endpoint (debounced `fetch()`, no page reload) |
| **Priority** | High |
| **Inputs** | Search query string |
| **Outputs** | Posts matching search criteria |
| **Fields** | Title, content, tags, categories |

---

### 3.3 Media Management

#### FR-12: Image Upload
| Element | Description |
|---------|-------------|
| **ID** | FR-12 |
| **Description** | Upload images for posts and media library, using an Alpine.js drag-and-drop / progress-bar component posting directly to a Laravel upload endpoint |
| **Priority** | High |
| **Inputs** | Image file |
| **Outputs** | Image stored in Cloudflare R2; thumbnail generated |
| **Validations** | Type: JPG, PNG, WebP, SVG, GIF; Size: Max 10MB; Dimensions: Min 800x600 |
| **Security** | File type verification; virus scanning optional |

#### FR-13: Image Optimization
| Element | Description |
|---------|-------------|
| **ID** | FR-13 |
| **Description** | Automatic image optimization |
| **Priority** | Medium |
| **Inputs** | Original image |
| **Outputs** | Multiple sizes (thumbnail, medium, large); WebP conversion |
| **CDN** | Cloudflare R2 with optimization rules |

#### FR-14: Media Library Management
| Element | Description |
|---------|-------------|
| **ID** | FR-14 |
| **Description** | Search, filter, and manage uploaded media in a Blade grid/list view; selection and bulk actions handled by Alpine.js state (no page reload needed for selecting/deselecting items) |
| **Priority** | Medium |
| **Inputs** | Search queries, filters |
| **Outputs** | Grid/list view of media assets |
| **Features** | Bulk delete, rename, folder organization |

---

### 3.4 Comment Management

#### FR-15: Comment Moderation
| Element | Description |
|---------|-------------|
| **ID** | FR-15 |
| **Description** | Approve, flag, or delete comments; moderation actions performed inline via Alpine.js-driven `fetch()` calls that update the row without a full page reload |
| **Priority** | High |
| **Inputs** | Comment ID, action |
| **Outputs** | Comment moderated; notification sent to user |
| **Statuses** | Pending, Approved, Spam, Trash |
| **Auto-moderation** | Spam detection (Akismet or custom) |

#### FR-16: Comment Notifications
| Element | Description |
|---------|-------------|
| **ID** | FR-16 |
| **Description** | Notify admins of new comments |
| **Priority** | Medium |
| **Outputs** | Email notification |
| **Settings** | Per-user preference |

---

### 3.5 User Management

#### FR-17: User Administration
| Element | Description |
|---------|-------------|
| **ID** | FR-17 |
| **Description** | Super Admin can manage all users |
| **Priority** | High |
| **Actions** | Create, suspend, delete, change roles, assign permissions |
| **Inputs** | User ID, action, data |
| **Outputs** | User account modified |
| **Audit** | All actions logged |

#### FR-18: User Profile Management
| Element | Description |
|---------|-------------|
| **ID** | FR-18 |
| **Description** | Users can manage their own profile |
| **Priority** | High |
| **Fields** | Name, email, avatar, bio, social links, preferences |
| **Inputs** | Updated profile data |
| **Outputs** | Profile updated |

---

### 3.6 Analytics & Dashboard

#### FR-19: Admin Dashboard
| Element | Description |
|---------|-------------|
| **ID** | FR-19 |
| **Description** | Central dashboard with key metrics, rendered server-side with chart data passed to a small Alpine.js/Chart.js component for interactive charts |
| **Priority** | High |
| **Metrics** | Total posts, page views, users, comments, conversion rates |
| **Charts** | Traffic trends, popular posts, engagement metrics |
| **Widgets** | Latest activity, quick actions, notifications |

#### FR-20: Content Analytics
| Element | Description |
|---------|-------------|
| **ID** | FR-20 |
| **Description** | Post performance analytics |
| **Priority** | Medium |
| **Metrics** | Views, time on page, bounce rate, social shares |
| **Outputs** | Reports, top-performing content |
| **Timeframe** | Daily, weekly, monthly, custom range |

#### FR-21: User Analytics
| Element | Description |
|---------|-------------|
| **ID** | FR-21 |
| **Description** | User behavior analytics |
| **Priority** | Low |
| **Metrics** | Active users, retention, engagement |
| **Outputs** | User segmentation reports |

---

### 3.7 SEO & Social

#### FR-22: SEO Optimization
| Element | Description |
|---------|-------------|
| **ID** | FR-22 |
| **Description** | Built-in SEO tools for content, rendered directly into Blade `<head>` partials (meta tags, Open Graph, JSON-LD) — server-rendered HTML gives strong SEO by default without needing SSR/SSG tooling |
| **Priority** | High |
| **Inputs** | Meta title, meta description, Open Graph tags, JSON-LD schema |
| **Outputs** | SEO metadata rendered in HTML |
| **Features** | SEO preview, readability score, keyword analysis |

#### FR-23: Sitemap Generation
| Element | Description |
|---------|-------------|
| **ID** | FR-23 |
| **Description** | Automatic XML sitemap generation |
| **Priority** | Medium |
| **Outputs** | sitemap.xml with all published posts |
| **Frequency** | Regenerated on publish/update |

#### FR-24: Social Sharing
| Element | Description |
|---------|-------------|
| **ID** | FR-24 |
| **Description** | Social media integration |
| **Priority** | Medium |
| **Platforms** | Twitter/X, LinkedIn, Facebook |
| **Features** | Share buttons, social card generation |

---

### 3.8 System Settings

#### FR-25: General Settings
| Element | Description |
|---------|-------------|
| **ID** | FR-25 |
| **Description** | System-wide configuration |
| **Priority** | High |
| **Settings** | Site name, tagline, timezone, language, maintenance mode |

#### FR-26: Email Configuration
| Element | Description |
|---------|-------------|
| **ID** | FR-26 |
| **Description** | SMTP and email template configuration |
| **Priority** | Medium |
| **Settings** | SMTP host, port, encryption, authentication |
| **Templates** | Email templates for notifications, password reset, etc. (Blade mailables) |

#### FR-27: Security Settings
| Element | Description |
|---------|-------------|
| **ID** | FR-27 |
| **Description** | Security configuration options |
| **Priority** | High |
| **Settings** | Password policy, session timeout, 2FA enforcement for admins |

---

### 3.9 API Requirements

#### FR-28: RESTful API
| Element | Description |
|---------|-------------|
| **ID** | FR-28 |
| **Description** | REST API exposed for external/mobile consumers; internal admin and public pages consume Laravel controllers/Blade directly and do **not** depend on this API |
| **Priority** | High |
| **Authentication** | JWT tokens (API consumers); session-based auth for the Blade frontend |
| **Rate Limiting** | 100 requests per minute |
| **Documentation** | Swagger/OpenAPI |
| **Versioning** | API v1 prefix |

#### FR-29: API Endpoints (Core)
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/auth/login` | POST | User authentication (external API clients) |
| `/api/v1/auth/register` | POST | User registration |
| `/api/v1/auth/logout` | POST | Logout |
| `/api/v1/posts` | GET | List posts |
| `/api/v1/posts` | POST | Create post |
| `/api/v1/posts/{id}` | GET | Get post |
| `/api/v1/posts/{id}` | PUT | Update post |
| `/api/v1/posts/{id}` | DELETE | Delete post |
| `/api/v1/posts/{id}/publish` | POST | Publish post |
| `/api/v1/posts/search` | GET | Search posts |
| `/api/v1/comments` | GET | List comments |
| `/api/v1/comments/{id}` | PUT | Update comment status |
| `/api/v1/media` | POST | Upload media |
| `/api/v1/media` | GET | List media |
| `/api/v1/media/{id}` | DELETE | Delete media |
| `/api/v1/users` | GET | List users |
| `/api/v1/users/{id}` | PUT | Update user |
| `/api/v1/analytics` | GET | Analytics data |

> Note: In addition to the above JSON API, Laravel also exposes lightweight internal-only fragment endpoints (e.g. `/admin/posts/search-fragment`, `/admin/comments/{id}/moderate-fragment`) that return small HTML partials for Alpine.js `fetch()` calls, keeping the Blade UI smooth without a full page reload.

---

## 4. Non-Functional Requirements

### 4.1 Performance Requirements

| Requirement | Metric |
|-------------|--------|
| Page Load Time | < 3 seconds (server-rendered Blade pages typically load faster than SPA bundles) |
| API Response Time | < 200ms (p95) |
| Concurrent Users | 10,000+ |
| Database Query Time | < 100ms |
| Cache Hit Ratio | > 80% (Redis + Blade view caching + Cloudflare edge caching) |
| Image Optimization | < 1s for processing |
| Search Response | < 1s |
| JS Payload | Alpine.js core is ~15KB gzipped; no framework bundle or hydration overhead |

### 4.2 Security Requirements

| Requirement | Description |
|-------------|-------------|
| **Encryption** | TLS 1.3 for all traffic |
| **Password Hashing** | Argon2id or bcrypt (cost 12+) |
| **Session Security** | HTTP-only, Secure, SameSite cookies |
| **Input Validation** | All user inputs validated and sanitized |
| **SQL Injection** | ORM/parameterized queries |
| **XSS Prevention** | Content Security Policy, Blade's automatic output escaping (`{{ }}`) |
| **CSRF Protection** | Laravel CSRF tokens on all Blade forms and Alpine.js `fetch()` calls |
| **Rate Limiting** | 100 requests per minute per IP |
| **Security Headers** | HSTS, X-Frame-Options, X-XSS-Protection |
| **File Upload** | MIME validation, file type restriction, size limits |
| **Audit Logging** | All admin actions logged |

### 4.3 Reliability Requirements

| Requirement | Metric |
|-------------|--------|
| Uptime | 99.9% (annual) |
| Data Backup | Daily automated backups |
| Backup Retention | 30 days (daily), 12 months (monthly) |
| Disaster Recovery | RTO < 4 hours, RPO < 24 hours |
| Error Rate | < 0.1% |
| Monitor | Real-time monitoring and alerting |

### 4.4 Scalability Requirements

| Metric | Target |
|--------|--------|
| Horizontal Scaling | Support multiple Laravel application instances behind a load balancer |
| Database Scaling | Read replicas for heavy loads |
| Cache Scaling | Redis cluster support |
| CDN Integration | Cloudflare caching of rendered HTML and static assets at the edge |
| Storage Scaling | Cloudflare R2 unlimited scalability |

### 4.5 Usability Requirements

| Requirement | Description |
|-------------|-------------|
| **UI** | Responsive, mobile-friendly Blade + Tailwind design |
| **Smoothness** | Alpine.js transitions (`x-transition`), debounced live search, inline partial updates, optimistic UI states, and skeleton loaders used throughout to make the server-rendered UI feel app-like |
| **Accessibility** | WCAG 2.1 AA compliant |
| **Internationalization** | Support for multiple languages via Laravel localization |
| **User Experience** | Intuitive navigation, dark/light theme |
| **Onboarding** | Tutorials, tooltips, help documentation |

### 4.6 Maintainability Requirements

| Requirement | Description |
|-------------|-------------|
| Code Quality | PSR-12 (PHP), ESLint (Alpine.js/JS assets) |
| Documentation | API docs, setup guide, code comments |
| Testing | PHPUnit unit/feature tests, Dusk for browser tests |
| Deployment | CI/CD pipeline, Docker deployment (single application image) |
| Monitoring | Application performance monitoring |

### 4.7 Compatibility Requirements

| Requirement | Description |
|-------------|-------------|
| **Browsers** | Chrome 90+, Firefox 88+, Safari 14+, Edge 90+ |
| **Devices** | Desktop, tablet, mobile |
| **Operating Systems** | Windows, macOS, Linux |
| **Screen Sizes** | Responsive down to 320px |

---

## 5. Data Requirements

### 5.1 Data Models

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    email_verified_at TIMESTAMP,
    password VARCHAR(255) NOT NULL,
    two_factor_secret TEXT,
    two_factor_recovery_codes TEXT,
    role VARCHAR(50) DEFAULT 'author',
    avatar VARCHAR(255),
    bio TEXT,
    preferences JSON,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Posts Table
```sql
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    content TEXT NOT NULL,
    excerpt TEXT,
    featured_image VARCHAR(255),
    status VARCHAR(20) DEFAULT 'draft',
    published_at TIMESTAMP,
    scheduled_for TIMESTAMP,
    views INTEGER DEFAULT 0,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);
```

#### Comments Table
```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    author_name VARCHAR(255),
    author_email VARCHAR(255),
    content TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Categories Table
```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    parent_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Tags Table
```sql
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Audit Logs Table
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id UUID,
    ip_address VARCHAR(45),
    user_agent TEXT,
    details JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 5.2 Data Relationships

```
Users (1) ──────── (N) Posts
Users (1) ──────── (N) Comments
Posts (1) ──────── (N) Comments
Posts (N) ──────── (N) Categories
Posts (N) ──────── (N) Tags
Posts (1) ──────── (N) PostVersions
```

### 5.3 Data Retention

| Data Type | Retention Period |
|-----------|------------------|
| Posts | Permanent (archival) |
| Draft Posts | 30 days after last edit |
| Comments | Permanent |
| Users | Permanent (anonymized if deleted) |
| Audit Logs | 1 year |
| Sessions | 30 days |
| Versions | Last 50 versions |

---

## 6. Infrastructure Requirements

### 6.1 Server Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 2 vCPU | 4+ vCPU |
| **RAM** | 4 GB | 8+ GB |
| **Storage** | 50 GB SSD | 100+ GB SSD |
| **Network** | 1 Gbps | 10+ Gbps |

> With no separate Next.js Node process to run, overall server footprint and RAM/CPU needs are lower than the original two-service (Laravel + Next.js) architecture.

### 6.2 Docker Services

```yaml
services:
  app:          # Laravel application (serves Blade views, Alpine.js/Tailwind assets via Vite, and REST API)
  nginx:        # Reverse proxy
  postgres:     # Database
  redis:        # Cache/Session
  worker:       # Queue worker (Laravel Horizon)
  scheduler:    # Cron scheduler
```

### 6.3 Database Configuration

| Setting | Value |
|---------|-------|
| max_connections | 100 |
| shared_buffers | 256MB |
| effective_cache_size | 768MB |
| work_mem | 8MB |
| maintenance_work_mem | 64MB |
| wal_buffers | 16MB |
| checkpoint_completion_target | 0.7 |

### 6.4 Redis Configuration

| Setting | Value |
|---------|-------|
| maxmemory | 256MB |
| maxmemory-policy | allkeys-lru |
| save | 900 1, 300 10, 60 10000 |

---

## 7. Security Architecture

### 7.1 Security Layers

```
┌─────────────────────────────────────────┐
│ 1. Cloudflare (DDoS Protection, WAF)    │
├─────────────────────────────────────────┤
│ 2. Nginx (Rate Limiting, SSL Termination)│
├─────────────────────────────────────────┤
│ 3. Laravel (Authentication, Validation)  │
├─────────────────────────────────────────┤
│ 4. Authorization (RBAC, Policies)        │
├─────────────────────────────────────────┤
│ 5. Data Security (Encryption, Hashing)   │
└─────────────────────────────────────────┘
```

### 7.2 Security Controls

| Control | Implementation |
|---------|----------------|
| Authentication | Laravel session-based auth for Blade frontend; JWT/OAuth2 for external API clients |
| Authorization | Laravel Policies and Gates |
| Input Validation | Laravel validation rules |
| Output Encoding | Blade's automatic escaping (`{{ }}`); manual escaping reviewed wherever `{!! !!}` is used |
| CSRF Protection | Laravel CSRF tokens (Blade `@csrf` directive + token header on Alpine.js `fetch()` calls) |
| XSS Protection | Content Security Policy header |
| Clickjacking | X-Frame-Options header |
| SQL Injection | Eloquent ORM, parameter binding |
| File Security | MIME validation, storage outside webroot |
| Secret Management | Environment variables, secrets vault |

### 7.3 Data Security

| Data Type | Protection |
|-----------|------------|
| Passwords | Argon2id hashing |
| Session Tokens | Encrypted, HttpOnly |
| User Data | HTTPS, database encryption |
| API Keys | Environment variables |
| Backups | Encrypted storage |

---

## 8. Monitoring & Logging

### 8.1 Application Monitoring

| Metric | Tool |
|--------|------|
| Server Health | Prometheus + Grafana |
| Application Performance | Laravel Telescope |
| Error Tracking | Sentry/Bugsnag |
| Uptime Monitoring | UptimeRobot/Pingdom |
| Log Management | ELK stack / Laravel Logs |

### 8.2 Logging Requirements

| Log Type | Retention | Level |
|----------|-----------|-------|
| Application | 30 days | INFO+ |
| Access | 30 days | INFO |
| Error | 90 days | WARNING+ |
| Security | 365 days | INFO+ |
| Audit | 365 days | INFO |

### 8.3 Alerting

| Alert | Threshold |
|-------|-----------|
| High CPU | >80% for 5 min |
| High Memory | >85% for 5 min |
| Low Disk | <20% free space |
| 5xx Errors | >1% of requests |
| Slow Response | >5 sec (p95) |
| Downtime | Any outage |

---

## 9. Deployment Strategy

### 9.1 Deployment Environment

| Environment | Purpose |
|-------------|---------|
| Development | Local development |
| Testing | Automated testing, QA |
| Staging | Pre-production validation |
| Production | Live application |

### 9.2 CI/CD Pipeline

```yaml
stages:
  - test:    # Run PHPUnit tests and build front-end assets (Vite)
  - build:   # Build single Docker image (Laravel app incl. compiled Blade/Tailwind/Alpine assets)
  - deploy:  # Deploy to environment
```

### 9.3 Rollback Strategy
- Automated rollback on deployment failure
- Database migrations with rollback scripts
- Zero-downtime deployments using blue-green or canary

---

## 10. Acceptance Criteria

### 10.1 Functional Acceptance

| Criterion | Description |
|-----------|-------------|
| User Authentication | Login/registration works with validation |
| Content Management | Create, edit, publish, delete posts |
| Media Upload | Upload and manage images |
| Comment Moderation | Approve/spam/delete comments |
| SEO | Meta tags, sitemap generation |
| Search | Full-text search returns relevant results, live-filtered via Alpine.js |
| Dashboard | Analytics and metrics display |
| Frontend Smoothness | Alpine.js interactions (modals, live search, inline moderation, transitions) feel instant with no full-page reload for common actions |

### 10.2 Non-Functional Acceptance

| Criterion | Target |
|-----------|--------|
| Performance | Page load < 3 sec |
| Security | No critical vulnerabilities |
| Usability | WCAG 2.1 AA compliance |
| Stability | 99.9% uptime |
| Compatibility | All target browsers/devices |

### 10.3 Security Acceptance

| Check | Status |
|-------|--------|
| SSL/TLS Implementation | ✅ |
| Authentication Bypass | ❌ Not possible |
| SQL Injection Prevention | ✅ |
| XSS Prevention | ✅ |
| CSRF Protection | ✅ |
| File Upload Security | ✅ |
| API Rate Limiting | ✅ |
| 2FA (admin) | ✅ |

---

## 11. Testing Requirements

### 11.1 Testing Types

| Type | Tool | Coverage |
|------|------|----------|
| Unit Tests | PHPUnit | >80% |
| Integration Tests | PHPUnit | Key workflows |
| End-to-End Tests | Laravel Dusk | Critical user paths (Blade + Alpine.js interactions) |
| Performance Tests | K6, Apache Bench | API endpoints and page routes |
| Security Tests | OWASP ZAP | All vulnerabilities |
| Visual Regression | Percy | UI components |

### 11.2 Testing Scenarios

| Scenario | Expected Outcome |
|----------|------------------|
| Admin creates post | Post saved, appears in list |
| Editor publishes post | Post visible on frontend |
| User comments on post | Comment pending moderation |
| Moderator approves comment | Comment visible on post, row updates in place without page reload |
| Media upload | Image optimized and stored |
| User registration | Verification email sent |
| Invalid login attempt | Rate limited after attempts |
| Live search | Results filter as the admin types, without a full page reload |

---

## 12. Documentation Requirements

### 12.1 Technical Documentation
- API Documentation (Swagger/OpenAPI)
- Database Schema Documentation
- Deployment Guide
- Docker Setup Guide
- Contributing Guidelines
- Code Style Guide
- Blade Component & Alpine.js Component Conventions Guide

### 12.2 User Documentation
- Admin User Guide
- Editor Guide
- End-User Support FAQ

### 12.3 Operational Documentation
- Backup and Restore Procedure
- Disaster Recovery Plan
- Incident Response Procedure
- Performance Tuning Guide
- Security Hardening Guide

---

## 13. Compliance Requirements

| Requirement | Compliance |
|-------------|------------|
| GDPR | Data protection measures, user data rights |
| Cookie Consent | GDPR-compliant cookie banner |
| Accessibility | WCAG 2.1 AA |
| Security Standards | OWASP Top 10 compliance |
| Data Privacy | Privacy policy, terms of service |

---

## 14. Project Timeline (Estimated)

| Phase | Duration | Activities |
|-------|----------|------------|
| Phase 1: Foundation | 3 weeks | Architecture, database, authentication, Blade/Tailwind/Alpine.js scaffolding |
| Phase 2: Core Features | 7 weeks | Content management, media upload |
| Phase 3: Security & Admin | 4 weeks | RBAC, logging, dashboard |
| Phase 4: Analytics & SEO | 3 weeks | Analytics, search, SEO |
| Phase 5: Testing & Polish | 4 weeks | Testing, deployment, documentation |

> Timeline is slightly shorter than the original two-stack estimate since there is no separate frontend application/service to build, integrate, and deploy.

---

## 15. Appendices

### A. Technology Stack Details

| Component | Technology | Version |
|-----------|------------|---------|
| Frontend Templates | Laravel Blade | Laravel 10+ |
| Frontend Interactivity | Alpine.js | 3+ |
| CSS Framework | Tailwind CSS | 3+ |
| Asset Bundling | Vite (Laravel integration) | Latest |
| Backend Framework | Laravel | 10+ |
| Backend Language | PHP | 8.2+ |
| Database | PostgreSQL | 14+ |
| Cache | Redis | 7+ |
| Web Server | Nginx | 1.22+ |
| CDN/WAF | Cloudflare | Latest |
| Object Storage | Cloudflare R2 | Latest |
| Containerization | Docker/Docker Compose | 24+ |

### B. References
- [Laravel Documentation](https://laravel.com/docs)
- [Blade Templates Documentation](https://laravel.com/docs/blade)
- [Alpine.js Documentation](https://alpinejs.dev/)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [Laravel Vite Plugin](https://laravel.com/docs/vite)
- [PostgreSQL Documentation](https://postgresql.org/docs)
- [Redis Documentation](https://redis.io/docs)
- [Docker Documentation](https://docs.docker.com)
- [Cloudflare Documentation](https://developers.cloudflare.com)
- [OWASP Top 10](https://owasp.org/Top10/)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)

---

**Document Version:** 2.0
**Last Updated:** July 2026
**Document Owner:** Project Architect
**Status:** Approved

---

## Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | July 2026 | System Architect | Initial version (Laravel + Next.js) |
| 2.0 | July 2026 | System Architect | Removed Next.js; consolidated to a single Laravel application using Blade + Alpine.js + Tailwind CSS for the frontend, per updated project decision to build one Laravel codebase with a smooth, lightweight UI |
