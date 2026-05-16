# SendGrid

Query email templates, suppression lists (bounces, blocks, spam
reports, invalid emails), marketing lists, single sends,
suppression groups, sender identities, verified senders,
API keys, and teammates from Twilio SendGrid.

## Setup

### Get Your API Key

1. Log in to [SendGrid](https://app.sendgrid.com)
2. Navigate to **Settings > API Keys** or visit
   [API Keys settings](https://app.sendgrid.com/settings/api_keys)
3. Click **Create API Key** with at least **Read Access** to the
   resources you want to query
4. Copy the generated key (it starts with `SG.`)

### Add the Source

```bash
export SENDGRID_API_KEY="SG.your_api_key_here"
coral source add --file sources/community/sendgrid/manifest.yaml
```

## Tables

### `api_keys`

Lists all API keys on the account (2 columns).

**Example:**

```sql
SELECT api_key_id, name
FROM sendgrid.api_keys;
```

### `teammates`

Lists all teammates on the account with username, email,
admin status, user type, and permission scopes (7 columns).

**Useful for:**

- Auditing team access and admin privileges
- Reviewing permission scopes per teammate

**Example:**

```sql
SELECT username, email, is_admin, user_type
FROM sendgrid.teammates;
```

### `verified_senders`

Lists all verified sender identities with email, name,
reply-to, company address, and verification status (11 columns).

**Example:**

```sql
SELECT id, nickname, from_email, from_name, verified
FROM sendgrid.verified_senders;
```

### `templates`

Lists all transactional email templates with name, generation
(legacy/dynamic), last update time, and version details
(5 columns). Both legacy and dynamic templates are returned.

**Example:**

```sql
SELECT id, name, generation, updated_at
FROM sendgrid.templates;
```

### `suppression_groups`

Lists all suppression (unsubscribe) groups with name,
description, default status, and unsubscribe count (5 columns).

**Example:**

```sql
SELECT id, name, description, is_default, unsubscribes
FROM sendgrid.suppression_groups;
```

### `bounces`

Lists all bounced email addresses with reason, enhanced
SMTP status code, and creation timestamp (4 columns).

**Useful for:**

- Auditing delivery issues and bounce patterns
- Identifying problematic recipient addresses

**Example:**

```sql
SELECT email, reason, status, created
FROM sendgrid.bounces
LIMIT 50;
```

### `blocks`

Lists all blocked email addresses with reason, status
code, and creation timestamp (4 columns).

**Example:**

```sql
SELECT email, reason, status, created
FROM sendgrid.blocks
LIMIT 50;
```

### `spam_reports`

Lists all spam report email addresses with the IP that
sent the reported email and creation timestamp (3 columns).

**Example:**

```sql
SELECT email, ip, created
FROM sendgrid.spam_reports
LIMIT 50;
```

### `invalid_emails`

Lists all invalid email addresses with reason for invalidity
and creation timestamp (3 columns).

**Example:**

```sql
SELECT email, reason, created
FROM sendgrid.invalid_emails
LIMIT 50;
```

### `marketing_lists`

Lists all marketing contact lists with name and contact
count (3 columns).

**Example:**

```sql
SELECT id, name, contact_count
FROM sendgrid.marketing_lists;
```

### `marketing_senders`

Lists all marketing sender identities with nickname,
from/reply-to addresses, verification status, and physical
address (13 columns).

**Example:**

```sql
SELECT id, nickname, from_email, from_name, reply_to_email
FROM sendgrid.marketing_senders;
```

### `singlesends`

Lists all marketing single sends with name, status, scheduled
send time, categories, and A/B test flag (8 columns).

**Example:**

```sql
SELECT id, name, status, send_at, categories
FROM sendgrid.singlesends;
```

## Authentication

The source uses Bearer token authentication with your
SendGrid API key.

- API keys start with `SG.`
- Create keys at **Settings > API Keys** in the SendGrid dashboard
- Use the minimum required permissions (Read Access)

## Inputs

| Input | Kind | Description |
|---|---|---|
| `SENDGRID_API_KEY` | secret | SendGrid API key |

## Pagination

Tables use different pagination strategies based on the
SendGrid API endpoint:

- **Offset pagination** (`limit` + `offset`): `teammates`,
  `bounces`, `blocks`, `spam_reports`, `invalid_emails`
- **Fixed-size single request**: `api_keys`
- **Single-page fetch (may be incomplete)**: `templates`,
  `marketing_lists`, `singlesends`
- **No pagination**: `verified_senders`, `suppression_groups`,
  `marketing_senders`

## Example Queries

### Audit API key inventory

```sql
SELECT api_key_id, name
FROM sendgrid.api_keys;
```

### Review team access

```sql
SELECT username, email, is_admin, user_type
FROM sendgrid.teammates;
```

### Audit verified sender identities

```sql
SELECT nickname, from_email, reply_to, verified
FROM sendgrid.verified_senders;
```

### Find bounced emails by status code

```sql
SELECT email, reason, status
FROM sendgrid.bounces
WHERE status LIKE '5.%'
LIMIT 20;
```

### Review suppression group unsubscribe counts

```sql
SELECT name, description, unsubscribes, is_default
FROM sendgrid.suppression_groups;
```

### List marketing campaigns and their status

```sql
SELECT name, status, send_at, is_abtest
FROM sendgrid.singlesends;
```

### Review marketing list sizes

```sql
SELECT name, contact_count
FROM sendgrid.marketing_lists
ORDER BY contact_count DESC;
```

### Audit sender verification status

```sql
SELECT nickname, from_email, from_name, verified
FROM sendgrid.marketing_senders;
```

## Notes

- The source is read-only — no create, update, or delete operations
- API keys start with `SG.` — do not confuse with other token formats
- Suppression timestamps (`created` in bounces, blocks, spam_reports,
  invalid_emails) are Unix epoch integers, not ISO 8601 strings
- Marketing sender `verified` is a boolean
- Marketing sender `created_at`/`updated_at` are Unix epoch integers
- The `templates` table returns both legacy and dynamic templates
  by default (filtered via `generations=legacy,dynamic`)
- The `templates`, `marketing_lists`, and `singlesends` APIs expose
  page-token pagination that Coral cannot fully follow yet, so this
  source requests the largest documented first page where supported
- The `versions` column in templates is a JSON array containing
  version details (id, name, subject, active status)
- EU-hosted SendGrid accounts should update the base URL to
  `https://api.eu.sendgrid.com/v3` in the manifest
