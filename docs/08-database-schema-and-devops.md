# Volume 8: Database Topology & DevOps Schema

## 1. PostgreSQL Database Schema (Supabase)

Northveil utilizes PostgreSQL with Row-Level Security (RLS) policies for persistent state management. In strict accordance with the non-custodial security invariants, no private keys, seed phrases, or master encryption keys are stored.

### Entity Relationship Diagram
```
users (id, email, google_sub)
  ├── passkeys (credential_id, credential_public_key, counter)
  ├── wallets (address, chain_family, mpc_provider, mpc_wallet_id)
  └── agent_clients (client_key_hash, status, expires_at)
        ├── grants (mode, wallet_ids[], spend limits, allowed recipients)
        │     └── spend_counters (daily spent volume)
        └── pending_approvals (payload_hash, canonical_tx, used, expires_at)
```

## 2. Table DDL Definitions

```sql
-- 1. Users from Google OAuth
create table if not exists public.users (
  id uuid primary key default gen_random_uuid(),
  email text not null,
  email_verified boolean not null default false,
  google_sub text not null unique,
  name text,
  avatar_url text,
  created_at timestamptz not null default now(),
  last_login_at timestamptz
);

-- 2. WebAuthn Passkeys enrolled for user verification
create table if not exists public.passkeys (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.users(id) on delete cascade,
  credential_id text not null unique,
  credential_public_key bytea not null,
  counter bigint not null default 0,
  transports text[],
  created_at timestamptz not null default now(),
  last_used_at timestamptz
);

-- 3. Wallets (Address and MPC vendor handle ONLY - NO private keys)
create table if not exists public.wallets (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.users(id) on delete cascade,
  address text not null,
  chain_family text not null default 'evm',
  mpc_provider text not null default 'turnkey',
  mpc_wallet_id text not null,
  status text not null default 'active',
  created_at timestamptz not null default now(),
  unique (address)
);

-- 4. Agent Clients (Scoped assistant credentials Claude/Cursor use)
create table if not exists public.agent_clients (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.users(id) on delete cascade,
  name text not null default 'Claude',
  client_key_hash text not null,
  status text not null default 'active', -- active | paused | revoked
  expires_at timestamptz,
  created_at timestamptz not null default now()
);

-- 5. Grants (Strict permission policies attached 1:1 to agent clients)
create table if not exists public.grants (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null unique references public.agent_clients(id) on delete cascade,
  user_id uuid not null references public.users(id) on delete cascade,
  wallet_ids uuid[] not null,
  mode text not null default 'always_ask', -- always_ask | autonomous
  chains text[] not null default array['eip155:8453'],
  allowed_assets text[] not null default array['ETH','USDC'],
  allowed_recipients text[] not null default array[]::text[],
  allow_any_recipient boolean not null default false,
  max_wei_per_tx numeric not null default 0,
  max_wei_per_day numeric not null default 0,
  authorized_by_credential_id text,
  authorized_at timestamptz,
  updated_at timestamptz not null default now()
);

-- 6. Pending Approvals (Single-use, hash-bound tickets with 10m expiry)
create table if not exists public.pending_approvals (
  id text primary key,
  user_id uuid not null references public.users(id) on delete cascade,
  client_id uuid not null references public.agent_clients(id),
  wallet_id uuid not null references public.wallets(id),
  payload_hash text not null,
  canonical_tx jsonb not null,
  used boolean not null default false,
  expires_at timestamptz not null,
  created_at timestamptz not null default now()
);

-- 7. Spend Counters (Daily volume tracking per grant)
create table if not exists public.spend_counters (
  grant_id uuid not null references public.grants(id) on delete cascade,
  day_utc date not null default current_date,
  spent_wei numeric not null default 0,
  primary key (grant_id, day_utc)
);

-- 8. Audit Logs (Sanitized, immutable audit events)
create table if not exists public.audit_logs (
  id bigint generated always as identity primary key,
  user_id uuid,
  wallet_address text,
  client_id uuid,
  action text not null,
  details jsonb not null default '{}',
  created_at timestamptz not null default now()
);
```
