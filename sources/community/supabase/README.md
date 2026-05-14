# Supabase

Query Supabase platform metadata through the [Management API](https://supabase.com/docs/reference/api/introduction). Covers organizations, projects, database branches, edge functions, secrets, storage buckets, service health, database backups, and PostgREST configuration.

## Authentication

Create a personal access token (PAT) at <https://supabase.com/dashboard/account/tokens>.

PATs carry the same privileges as your user account. Keep them secret and do not commit them to version control. For third-party integrations, Supabase also supports OAuth2 with scoped tokens. See [Build a Supabase Integration](https://supabase.com/docs/guides/integrations/build-a-supabase-integration) for details.

## Inputs

| Name                    | Kind   | Required | Description                                              |
|-------------------------|--------|----------|----------------------------------------------------------|
| `SUPABASE_ACCESS_TOKEN` | secret | yes      | Personal access token from the Supabase dashboard        |

## Tables

| Table                  | Endpoint                                             | Filter Required      | Notes                                       |
|------------------------|------------------------------------------------------|----------------------|---------------------------------------------|
| `organizations`        | `GET /v1/organizations`                              | none                 | All orgs for the authenticated user         |
| `projects`             | `GET /v1/projects`                                   | none                 | All projects across all orgs                |
| `organization_members` | `GET /v1/organizations/{slug}/members`               | `slug` (required)    | Members and roles within an org             |
| `edge_functions`       | `GET /v1/projects/{ref}/functions`                   | `project_ref` (req.) | Deployed Edge Functions                     |
| `secrets`              | `GET /v1/projects/{ref}/secrets`                     | `project_ref` (req.) | Secret names only, values are redacted      |
| `storage_buckets`      | `GET /v1/projects/{ref}/storage/buckets`             | `project_ref` (req.) | Public and private storage buckets          |
| `service_health`       | `GET /v1/projects/{ref}/health`                      | `project_ref`, `services` (req.) | Per-service health and version info |
| `backups`              | `GET /v1/projects/{ref}/database/backups`            | `project_ref` (req.) | Logical and physical backup snapshots       |
| `branches`             | `GET /v1/projects/{ref}/branches`                    | `project_ref` (req.) | Database branches (requires branching plan) |
| `postgrest_config`     | `GET /v1/projects/{ref}/postgrest`                   | `project_ref` (req.) | Data API (PostgREST) settings               |

## Discovery flow

Most tables require a project ref or organization slug as a filter. Start with the top-level tables to discover those identifiers:

```
organizations
  -> slug
    -> organization_members (slug)

projects
  -> ref (project_ref)
    -> edge_functions     (project_ref)
    -> secrets            (project_ref)
    -> storage_buckets    (project_ref)
    -> service_health     (project_ref + services)
    -> backups            (project_ref)
    -> branches           (project_ref)
    -> postgrest_config   (project_ref)
```

## Example queries

```sql
-- List all organizations
SELECT id, name, slug FROM supabase.organizations;

-- List all projects with status and region
SELECT id, ref, name, region, status, created_at
FROM supabase.projects;

-- Check which projects are inactive
SELECT ref, name, status, created_at
FROM supabase.projects
WHERE status = 'INACTIVE';

-- List members of an organization
SELECT user_name, email, role_name, mfa_enabled
FROM supabase.organization_members
WHERE slug = 'my-org-slug';

-- List edge functions for a project
SELECT id, name, slug, status, version, verify_jwt
FROM supabase.edge_functions
WHERE project_ref = 'abcdefghijklmnopqrst';

-- List secrets (names only, values are not exposed)
SELECT name, updated_at
FROM supabase.secrets
WHERE project_ref = 'abcdefghijklmnopqrst';

-- List storage buckets and their visibility
SELECT id, name, public, owner, created_at
FROM supabase.storage_buckets
WHERE project_ref = 'abcdefghijklmnopqrst';

-- Check service health
SELECT name, healthy, status, info__version, error
FROM supabase.service_health
WHERE project_ref = 'abcdefghijklmnopqrst'
  AND services = 'auth,realtime,storage,postgrest';

-- List database backups
SELECT id, status, is_physical_backup, inserted_at
FROM supabase.backups
WHERE project_ref = 'abcdefghijklmnopqrst';

-- List database branches (requires branching to be enabled)
SELECT name, git_branch, is_default, persistent, status, created_at
FROM supabase.branches
WHERE project_ref = 'abcdefghijklmnopqrst';

-- Check PostgREST configuration
SELECT db_schema, max_rows, db_pool
FROM supabase.postgrest_config
WHERE project_ref = 'abcdefghijklmnopqrst';
```

## Rate limits

The Supabase Management API enforces 120 requests per minute per user per scope (project or organization). Rate limits are tracked independently per scope, so requests to different projects do not interfere with each other. Every response includes `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers.

## Limitations

- **Read-only**: no create, update, or delete operations
- **No pagination**: most Management API list endpoints return all items in a single response; projects or orgs with very large counts may hit API-side limits
- **Secret values**: the API returns empty strings for secret values; only key names and timestamps are visible
- **Branching**: the branches table requires a paid plan with branching enabled
- **Service health**: requires specifying which services to check via the `services` filter
