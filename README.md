# verisigndoc

verisigndoc is a multi-tenant document platform for applying legally valid digital marks to documents, including:

- e-meterai
- e-signature
- e-initial
- text annotation
- image annotation

## Target architecture

The system is designed as a dockerized microservice platform with the following components:

| Component | Technology | Responsibility |
| --- | --- | --- |
| Web application | Laravel | Tenant management, user management, document management, subscription billing, API gateway |
| Signing service | Java | Certificate validation and document signing workflow |
| Email service | Go | Transactional emails, trial reminders, invitations, notifications |
| Payment service | Go | Subscription lifecycle, plan changes, invoicing, payment callbacks |
| Main database | PostgreSQL | Core relational data |
| Cache | Redis | Session, cache, rate-limit, temporary state |
| Message broker | RabbitMQ | Async events between services |
| Logging | MongoDB | Centralized application and audit logs |
| Object storage | MinIO | Document and asset storage |

## Multi-tenancy model

The application uses workspace-based multi-tenancy.

### Roles

- **Landlord/Admin**: owns the tenant account, creates the primary organization and workspace, manages billing and members.
- **Tenant user**: end user or client who works inside a workspace based on assigned role.

### Tenancy rules

- Every newly registered account starts with a **14-day free trial**.
- After the trial expires, the account is downgraded to the **Free** plan.
- Every tenant user belongs to one primary **organization** and one primary **workspace**.
- A user may also be invited to collaborate on a specific document **without joining** the target organization or workspace as a full member.
- A user may participate in multiple collaborations and additional workspaces over time.
- Document visibility must always follow the user's role and permissions in the current workspace.

## Subscription model

Supported plans:

- **Free** (automatic after trial expiration)
- **Professional**
- **Team**
- **Corporate**

Plan differentiation is based on:

- storage quota
- maximum users per workspace

Recommended subscription fields:

- `plan_code`
- `plan_name`
- `trial_days`
- `storage_quota_bytes`
- `workspace_user_limit`
- `is_default_free_plan`
- `is_active`

## Core domain model

### Tenant and access entities

- **users**
- **organizations**
- **workspaces**
- **workspace_members**
- **document_collaborators**
- **roles**
- **permissions**
- **subscriptions**
- **subscription_plans**

### Document entities

- **folders**: supports nested folders using `parent_id`
- **documents**
- **document_versions**
- **document_annotations**
- **document_sign_requests**
- **document_audit_logs**

### Suggested relationship rules

- One organization has many workspaces.
- One workspace has many members.
- One user has one primary organization and one primary workspace at registration time.
- One document belongs to one workspace and optionally one folder.
- One folder can have many child folders and many documents.
- One document can have many collaborators with document-specific permissions.

## Access control

Use role-based access control inside each workspace:

- **Owner/Admin**: full workspace access
- **Manager**: manage documents and members based on policy
- **Member**: create and manage own or assigned documents
- **Viewer/Collaborator**: restricted access to shared documents only

Minimum document permission checks:

- view
- upload
- edit metadata
- annotate
- sign
- invite collaborator
- delete

## Trial and billing flow

1. User registers.
2. System creates the primary organization and workspace.
3. System assigns a 14-day trial subscription.
4. Trial reminder emails are sent before expiration.
5. If the user does not upgrade, the subscription is converted to the Free plan.
6. Workspace limits are enforced based on the active plan.

## Document management requirements

- Users manage documents similarly to Google Drive.
- Documents can be grouped in **nested folders**.
- Access is scoped by workspace membership or explicit document collaboration.
- Files are stored in MinIO, while metadata and permissions remain in PostgreSQL.
- Signing and annotation requests are processed asynchronously through RabbitMQ.

## Implementation notes

- Keep tenant ownership and billing in the Laravel web application.
- Isolate signing logic in the Java signing service.
- Use Redis for caching quotas, invitations, and temporary signing state.
- Store audit and operational logs in MongoDB.
- Use MinIO object keys that include workspace and document identifiers for isolation.
- Enforce authorization in both API and background workers.