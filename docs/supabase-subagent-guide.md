# Guida: Sub-Agent OpenClaw con Supabase Database

## Overview

Questa guida spiega come implementare un sub-agent OpenClaw che utilizza Supabase come database backend, sostituendo SQLite con PostgreSQL cloud.

## Architettura

```
┌─────────────────┐     HTTP/REST      ┌──────────────┐     SQL      ┌─────────────┐
│  Sub-Agent      │ ──────────────────→ │   Supabase   │ ────────────→│  PostgreSQL │
│  (OpenClaw)     │    (fetch/exec)     │   REST API   │              │   Database  │
└─────────────────┘                     └──────────────┘              └─────────────┘
        │
        └── ENV: SUPABASE_URL, SUPABASE_KEY
```

**Vantaggi:**
- ✅ Persistenza cloud
- ✅ Accesso real-time
- ✅ Scalabilità automatica
- ✅ Row Level Security (RLS)
- ✅ API REST auto-generata

## Step 1: Setup Supabase

### 1.1 Crea Progetto

1. Vai su [supabase.com](https://supabase.com)
2. Crea nuovo progetto
3. Ottieni:
   - `Project URL`: `https://xxxx.supabase.co`
   - `Anon Key`: `eyJhbG...`

### 1.2 Crea Tabella

```sql
-- SQL Editor in Supabase Dashboard
CREATE TABLE tasks (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  status TEXT DEFAULT 'todo',
  priority TEXT DEFAULT 'medium',
  due_date DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  user_id UUID DEFAULT auth.uid()
);

-- Enable RLS
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY "Users can only access their own tasks"
ON tasks FOR ALL
USING (user_id = auth.uid());
```

## Step 2: Struttura Skill Sub-Agent

```
workspace/
└── skills/
    └── taskmanager-supabase/
        ├── SKILL.md
        ├── supabase-client.js
        └── .env.example
```

## Step 3: Codice Implementazione

### 3.1 supabase-client.js

```javascript
// Simple Supabase REST client (no SDK needed)
class SupabaseClient {
  constructor(url, key) {
    this.url = url;
    this.key = key;
    this.headers = {
      'apikey': key,
      'Authorization': `Bearer ${key}`,
      'Content-Type': 'application/json'
    };
  }

  async select(table, options = {}) {
    let query = `${this.url}/rest/v1/${table}`;
    
    if (options.eq) {
      const [col, val] = Object.entries(options.eq)[0];
      query += `?${col}=eq.${encodeURIComponent(val)}`;
    }
    
    const response = await fetch(query, {
      method: 'GET',
      headers: this.headers
    });
    
    if (!response.ok) throw new Error(`Select failed: ${response.status}`);
    return await response.json();
  }

  async insert(table, data) {
    const response = await fetch(`${this.url}/rest/v1/${table}`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify(data)
    });
    
    if (!response.ok) throw new Error(`Insert failed: ${response.status}`);
    return await response.json();
  }

  async update(table, id, data) {
    const response = await fetch(`${this.url}/rest/v1/${table}?id=eq.${id}`, {
      method: 'PATCH',
      headers: this.headers,
      body: JSON.stringify(data)
    });
    
    if (!response.ok) throw new Error(`Update failed: ${response.status}`);
    return await response.json();
  }

  async delete(table, id) {
    const response = await fetch(`${this.url}/rest/v1/${table}?id=eq.${id}`, {
      method: 'DELETE',
      headers: this.headers
    });
    
    if (!response.ok) throw new Error(`Delete failed: ${response.status}`);
    return true;
  }
}

module.exports = { SupabaseClient };
```

### 3.2 SKILL.md

```markdown
# TaskManager Supabase

## Description
Task management with Supabase PostgreSQL backend.

## Environment Variables
- SUPABASE_URL: https://your-project.supabase.co
- SUPABASE_KEY: your-anon-key

## Tools
- fetch: for HTTP requests to Supabase API
- read/write: for local operations

## Usage
```javascript
const { SupabaseClient } = require('./supabase-client.js');
const db = new SupabaseClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

// List tasks
const tasks = await db.select('tasks');

// Create task
await db.insert('tasks', { title: 'New task', priority: 'high' });
```
```

## Step 4: Configurazione OpenClaw

### 4.1 Environment Variables

Aggiungi a `.env` o config OpenClaw:

```bash
# Supabase Configuration
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_KEY=eyJhbGc...
```

### 4.2 Configurazione Sub-Agent

```json
{
  "id": "taskmanager-supabase",
  "name": "TaskManager Supabase",
  "model": {
    "primary": "moonshot/kimi-k2.5"
  },
  "env": [
    "SUPABASE_URL",
    "SUPABASE_KEY"
  ],
  "tools": {
    "allow": ["fetch", "read", "write", "exec"]
  }
}
```

## Step 5: Esempio Pratico - Spawn Sub-Agent

### 5.1 Da Main Session

```javascript
// Spawn sub-agent con task
await sessions_spawn({
  agentId: "taskmanager-supabase",
  task: `
    Aggiungi questa task al database Supabase:
    - Titolo: "Review Q1 metrics"
    - Priorità: high
    - Due: 2026-03-01
    
    Usa le variabili d'ambiente SUPABASE_URL e SUPABASE_KEY.
    Conferma quando fatto.
  `,
  timeoutSeconds: 60
});
```

### 5.2 Skill Completa

```javascript
// /skills/taskmanager-supabase/index.js
const { SupabaseClient } = require('./supabase-client');

class TaskManager {
  constructor() {
    this.db = new SupabaseClient(
      process.env.SUPABASE_URL,
      process.env.SUPABASE_KEY
    );
  }

  async listTasks(filters = {}) {
    return await this.db.select('tasks', filters);
  }

  async createTask(task) {
    return await this.db.insert('tasks', {
      title: task.title,
      priority: task.priority || 'medium',
      due_date: task.due_date,
      status: 'todo'
    });
  }

  async completeTask(id) {
    return await this.db.update('tasks', id, { status: 'done' });
  }

  async deleteTask(id) {
    return await this.db.delete('tasks', id);
  }
}

module.exports = { TaskManager };
```

## Step 6: Testing

```bash
# Test connection
curl -X GET \
  'https://your-project.supabase.co/rest/v1/tasks' \
  -H "apikey: your-anon-key" \
  -H "Authorization: Bearer your-anon-key"
```

## Sicurezza - Best Practices

### ❌ NON fare:
- Usare `service_role` key nel sub-agent
- Disabilitare RLS
- Loggare API keys
- Dare permessi DELETE irrestritti

### ✅ FARE:
- Usare solo `anon` key con RLS
- Validare input prima di INSERT/UPDATE
- Implementare rate limiting
- Audit log delle operazioni

## Troubleshooting

| Errore | Causa | Soluzione |
|--------|-------|-----------|
| 401 Unauthorized | Key errata | Verifica SUPABASE_KEY |
| 403 Forbidden | RLS bloccante | Configura policy corretta |
| 409 Conflict | Duplicate key | Usa UPSERT o gestisci conflitto |
| Network error | Timeout | Aumenta timeout fetch |

## Conclusione

Con questo setup hai:
- ✅ Database cloud scalabile
- ✅ Sub-agent specializzato
- ✅ Sicurezza RLS integrata
- ✅ API REST auto-generata

Per domande o problemi, controlla i log del sub-agent e verifica RLS policies in Supabase Dashboard.
