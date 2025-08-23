# Implementation Plan: MongoDB to Supabase Migration

## 1. Migration Overview

This document outlines the step-by-step process for migrating the Chat UI application from MongoDB to Supabase while updating the Docker configuration for containerized deployment.

### Migration Phases
1. **Preparation Phase**: Environment setup and dependency updates
2. **Database Migration**: Schema creation and data transfer
3. **Code Refactoring**: Replace MongoDB code with Supabase integration
4. **Docker Configuration**: Update containerization for Supabase
5. **Testing & Validation**: Ensure functionality and performance
6. **Deployment**: Production rollout strategy

## 2. Phase 1: Preparation

### 2.1 Supabase Project Setup

✅ **Already Completed**: Supabase project is connected
- Project URL: `https://ybibjftiefxmnznkrojm.supabase.co`
- Anon Key: Available for frontend use
- Service Role Key: Available for backend operations

### 2.2 Dependency Updates

**Add Supabase Dependencies**
```bash
npm install @supabase/supabase-js @supabase/auth-helpers-sveltekit
npm install --save-dev @supabase/auth-helpers-shared
```

**Update package.json**
```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.39.0",
    "@supabase/auth-helpers-sveltekit": "^0.10.7",
    "@supabase/auth-helpers-shared": "^0.5.3"
  }
}
```

### 2.3 Environment Configuration

**Create .env.local with Supabase credentials**
```bash
# Supabase Configuration
SUPABASE_URL=https://ybibjftiefxmnznkrojm.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Public Environment Variables
PUBLIC_SUPABASE_URL=https://ybibjftiefxmnznkrojm.supabase.co
PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Keep existing MongoDB for migration period
MONGODB_URL=mongodb://localhost:27017
MONGODB_DB_NAME=chat-ui
```

## 3. Phase 2: Database Migration

### 3.1 Create Supabase Schema

**Execute SQL in Supabase SQL Editor**

1. **Create profiles table and policies**
```sql
-- Profiles table (extends auth.users)
CREATE TABLE profiles (
    id UUID REFERENCES auth.users(id) PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    avatar_url TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile" ON profiles
    FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can update own profile" ON profiles
    FOR UPDATE USING (auth.uid() = id);
```

2. **Create conversations table**
```sql
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL DEFAULT 'New Conversation',
    metadata JSONB DEFAULT '{}',
    is_shared BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_conversations_user_id ON conversations(user_id);
CREATE INDEX idx_conversations_created_at ON conversations(created_at DESC);

ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own conversations" ON conversations
    FOR SELECT USING (auth.uid() = user_id OR is_shared = TRUE);

CREATE POLICY "Users can create conversations" ON conversations
    FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own conversations" ON conversations
    FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own conversations" ON conversations
    FOR DELETE USING (auth.uid() = user_id);
```

3. **Create remaining tables** (messages, assistants, user_settings, shared_conversations)
   - Follow the complete SQL schema from the Technical Architecture document

### 3.2 Data Migration Script

**Create migration script: `scripts/migrate-to-supabase.ts`**
```typescript
import { MongoClient } from 'mongodb';
import { createClient } from '@supabase/supabase-js';
import { config } from '../src/lib/server/config';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const mongoClient = new MongoClient(process.env.MONGODB_URL!);

async function migrateUsers() {
  const db = mongoClient.db(process.env.MONGODB_DB_NAME);
  const users = await db.collection('users').find({}).toArray();
  
  for (const user of users) {
    // Create user in Supabase Auth
    const { data: authUser, error } = await supabase.auth.admin.createUser({
      email: user.email,
      password: 'temp-password-' + Math.random(),
      email_confirm: true,
      user_metadata: {
        name: user.name,
        migrated_from_mongo: true
      }
    });
    
    if (authUser?.user) {
      // Create profile
      await supabase.from('profiles').insert({
        id: authUser.user.id,
        email: user.email,
        name: user.name,
        metadata: user.metadata || {},
        created_at: user.createdAt
      });
    }
  }
}

async function migrateConversations() {
  const db = mongoClient.db(process.env.MONGODB_DB_NAME);
  const conversations = await db.collection('conversations').find({}).toArray();
  
  for (const conv of conversations) {
    // Map MongoDB user ID to Supabase user ID
    const userMapping = await getUserMapping(conv.userId);
    
    if (userMapping) {
      await supabase.from('conversations').insert({
        id: conv._id,
        user_id: userMapping.supabase_id,
        title: conv.title,
        metadata: conv.metadata || {},
        is_shared: conv.shared || false,
        created_at: conv.createdAt
      });
    }
  }
}

async function migrateMessages() {
  const db = mongoClient.db(process.env.MONGODB_DB_NAME);
  const messages = await db.collection('messages').find({}).toArray();
  
  for (const msg of messages) {
    const userMapping = await getUserMapping(msg.from);
    
    if (userMapping) {
      await supabase.from('messages').insert({
        id: msg._id,
        conversation_id: msg.conversationId,
        user_id: userMapping.supabase_id,
        content: msg.content,
        role: msg.from === 'user' ? 'user' : 'assistant',
        metadata: msg.metadata || {},
        created_at: msg.createdAt
      });
    }
  }
}

async function main() {
  await mongoClient.connect();
  
  console.log('Starting migration...');
  await migrateUsers();
  console.log('Users migrated');
  
  await migrateConversations();
  console.log('Conversations migrated');
  
  await migrateMessages();
  console.log('Messages migrated');
  
  await mongoClient.close();
  console.log('Migration completed!');
}

main().catch(console.error);
```

## 4. Phase 3: Code Refactoring

### 4.1 Supabase Client Setup

**Create `src/lib/supabase.ts`**
```typescript
import { createClient } from '@supabase/supabase-js';
import { env } from '$env/dynamic/public';

export const supabase = createClient(
  env.PUBLIC_SUPABASE_URL,
  env.PUBLIC_SUPABASE_ANON_KEY
);

export type Database = {
  public: {
    Tables: {
      profiles: {
        Row: {
          id: string;
          email: string;
          name: string | null;
          avatar_url: string | null;
          metadata: any;
          created_at: string;
          updated_at: string;
        };
        Insert: {
          id: string;
          email: string;
          name?: string;
          avatar_url?: string;
          metadata?: any;
        };
        Update: {
          name?: string;
          avatar_url?: string;
          metadata?: any;
        };
      };
      conversations: {
        Row: {
          id: string;
          user_id: string;
          title: string;
          metadata: any;
          is_shared: boolean;
          created_at: string;
          updated_at: string;
        };
        Insert: {
          user_id: string;
          title?: string;
          metadata?: any;
          is_shared?: boolean;
        };
        Update: {
          title?: string;
          metadata?: any;
          is_shared?: boolean;
        };
      };
      // Add other table types...
    };
  };
};
```

### 4.2 Authentication Integration

**Update `src/hooks.server.ts`**
```typescript
import { createServerClient } from '@supabase/auth-helpers-sveltekit';
import { env } from '$env/dynamic/public';
import { env as privateEnv } from '$env/dynamic/private';
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  event.locals.supabase = createServerClient(
    env.PUBLIC_SUPABASE_URL,
    env.PUBLIC_SUPABASE_ANON_KEY,
    {
      event,
      serverApiKey: privateEnv.SUPABASE_SERVICE_ROLE_KEY
    }
  );

  event.locals.getSession = async () => {
    const {
      data: { session },
    } = await event.locals.supabase.auth.getSession();
    return session;
  };

  return resolve(event, {
    filterSerializedResponseHeaders(name) {
      return name === 'content-range';
    },
  });
};
```

### 4.3 Database Layer Refactoring

**Replace `src/lib/server/database.ts`**
```typescript
import { createClient } from '@supabase/supabase-js';
import { env } from '$env/dynamic/private';
import type { Database } from '../supabase';

const supabase = createClient<Database>(
  env.PUBLIC_SUPABASE_URL,
  env.SUPABASE_SERVICE_ROLE_KEY
);

export class SupabaseCollections {
  static get conversations() {
    return {
      async findOne(id: string, userId: string) {
        const { data, error } = await supabase
          .from('conversations')
          .select('*')
          .eq('id', id)
          .eq('user_id', userId)
          .single();
        
        if (error) throw error;
        return data;
      },
      
      async find(userId: string) {
        const { data, error } = await supabase
          .from('conversations')
          .select('*')
          .eq('user_id', userId)
          .order('created_at', { ascending: false });
        
        if (error) throw error;
        return data;
      },
      
      async insertOne(conversation: any) {
        const { data, error } = await supabase
          .from('conversations')
          .insert(conversation)
          .select()
          .single();
        
        if (error) throw error;
        return data;
      },
      
      async updateOne(id: string, update: any) {
        const { data, error } = await supabase
          .from('conversations')
          .update(update)
          .eq('id', id)
          .select()
          .single();
        
        if (error) throw error;
        return data;
      },
      
      async deleteOne(id: string, userId: string) {
        const { error } = await supabase
          .from('conversations')
          .delete()
          .eq('id', id)
          .eq('user_id', userId);
        
        if (error) throw error;
      }
    };
  }
  
  static get messages() {
    return {
      async find(conversationId: string) {
        const { data, error } = await supabase
          .from('messages')
          .select('*')
          .eq('conversation_id', conversationId)
          .order('created_at', { ascending: true });
        
        if (error) throw error;
        return data;
      },
      
      async insertOne(message: any) {
        const { data, error } = await supabase
          .from('messages')
          .insert(message)
          .select()
          .single();
        
        if (error) throw error;
        return data;
      }
    };
  }
  
  // Add other collections...
}

export const collections = SupabaseCollections;
```

### 4.4 Update Route Handlers

**Update conversation routes to use Supabase**
```typescript
// src/routes/conversation/[id]/+page.server.ts
import { collections } from '$lib/server/database';
import { redirect } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params, locals }) => {
  const session = await locals.getSession();
  
  if (!session) {
    throw redirect(302, '/login');
  }
  
  const conversation = await collections.conversations.findOne(
    params.id,
    session.user.id
  );
  
  const messages = await collections.messages.find(params.id);
  
  return {
    conversation,
    messages
  };
};
```

## 5. Phase 4: Docker Configuration Updates

### 5.1 Update Dockerfile

**Modified Dockerfile for Supabase integration**
```dockerfile
# syntax=docker/dockerfile:1
ARG INCLUDE_DB=false

FROM node:20-slim AS base
ENV PLAYWRIGHT_SKIP_BROWSER_GC=1

# Install dotenv-cli
RUN npm install -g dotenv-cli

# Switch to user that works for spaces
RUN userdel -r node
RUN useradd -m -u 1000 user
USER user

ENV HOME=/home/user \
    PATH=/home/user/.local/bin:$PATH

WORKDIR /app

# Add .env.local if user doesn't bind volume
RUN touch /app/.env.local

# Install Playwright
RUN npm i --no-package-lock --no-save playwright@1.52.0

USER root

# Create models directory
RUN mkdir -p /data/models
RUN chown -R 1000:1000 /data/models

# Install system dependencies
RUN apt-get update
RUN apt-get install gnupg curl git cmake clang libgomp1 -y

# Install Playwright browsers
RUN npx playwright install --with-deps chromium

RUN chown -R 1000:1000 /home/user/.npm

USER user

# Copy application files
COPY --chown=1000 .env /app/.env
COPY --chown=1000 entrypoint.sh /app/entrypoint.sh
COPY --chown=1000 package.json /app/package.json
COPY --chown=1000 package-lock.json /app/package-lock.json

RUN chmod +x /app/entrypoint.sh

# Builder stage
FROM node:20 AS builder

WORKDIR /app

COPY --link --chown=1000 package-lock.json package.json ./

ARG APP_BASE=
ARG PUBLIC_APP_COLOR=blue
ARG SKIP_LLAMA_CPP_BUILD
ENV BODY_SIZE_LIMIT=15728640
ENV SKIP_LLAMA_CPP_BUILD=$SKIP_LLAMA_CPP_BUILD

# Install dependencies including Supabase
RUN --mount=type=cache,target=/app/.npm \
    npm set cache /app/.npm && \
    npm ci

COPY --link --chown=1000 . .

RUN git config --global --add safe.directory /app && \
    npm run build

# Remove MongoDB stage since we're using Supabase
FROM base AS final

# Build args
ARG APP_BASE=
ARG PUBLIC_APP_COLOR=blue
ARG PUBLIC_COMMIT_SHA=
ENV PUBLIC_COMMIT_SHA=${PUBLIC_COMMIT_SHA}
ENV BODY_SIZE_LIMIT=15728640
ENV MODELS_STORAGE_PATH=/data/models

# Import build & dependencies
COPY --from=builder --chown=1000 /app/build /app/build
COPY --from=builder --chown=1000 /app/node_modules /app/node_modules

CMD ["/bin/bash", "-c", "/app/entrypoint.sh"]
```

### 5.2 Update entrypoint.sh

**Modified entrypoint script**
```bash
#!/bin/bash
ENV_LOCAL_PATH=/app/.env.local

if test -z "${DOTENV_LOCAL}" ; then
    if ! test -f "${ENV_LOCAL_PATH}" ; then
        echo "DOTENV_LOCAL was not found in the ENV variables and .env.local is not set using a bind volume. Make sure to set environment variables properly."
    fi;
else
    echo "DOTENV_LOCAL was found in the ENV variables. Creating .env.local file."
    cat <<< "$DOTENV_LOCAL" > ${ENV_LOCAL_PATH}
fi;

# Remove MongoDB startup since we're using Supabase
# MongoDB is no longer needed

export PUBLIC_VERSION=$(node -p "require('./package.json').version")

# Start the application
dotenv -e /app/.env -c -- node /app/build/index.js -- --host 0.0.0.0 --port 3000
```

### 5.3 Docker Compose for Development

**Create `docker-compose.yml`**
```yaml
version: '3.8'

services:
  chat-ui:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        INCLUDE_DB: "false"  # We're using Supabase, not local MongoDB
        PUBLIC_APP_COLOR: "blue"
    ports:
      - "3000:3000"
    environment:
      - SUPABASE_URL=https://ybibjftiefxmnznkrojm.supabase.co
      - SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      - SUPABASE_SERVICE_ROLE_KEY=${SUPABASE_SERVICE_ROLE_KEY}
      - PUBLIC_SUPABASE_URL=https://ybibjftiefxmnznkrojm.supabase.co
      - PUBLIC_SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      - HF_TOKEN=${HF_TOKEN}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - PUBLIC_APP_NAME=ChatUI
      - PUBLIC_APP_ASSETS=chatui
    volumes:
      - ./models:/data/models
      - ./.env.local:/app/.env.local:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 6. Phase 5: Testing & Validation

### 6.1 Unit Tests Update

**Update test files to use Supabase**
```typescript
// tests/supabase.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

describe('Supabase Integration', () => {
  it('should connect to Supabase', async () => {
    const { data, error } = await supabase.from('profiles').select('count');
    expect(error).toBeNull();
  });
  
  it('should create and retrieve conversation', async () => {
    // Test conversation CRUD operations
  });
  
  it('should handle authentication', async () => {
    // Test auth flow
  });
});
```

### 6.2 Integration Testing

**Test checklist:**
- [ ] User registration and login
- [ ] Conversation creation and retrieval
- [ ] Message sending and receiving
- [ ] Real-time updates
- [ ] Assistant functionality
- [ ] File uploads (if applicable)
- [ ] Settings management
- [ ] Admin functionality

## 7. Phase 6: Deployment Strategy

### 7.1 Staging Deployment

1. **Deploy to staging environment**
   ```bash
   docker build -t chat-ui-supabase .
   docker run -p 3000:3000 --env-file .env.production chat-ui-supabase
   ```

2. **Run migration script**
   ```bash
   npm run migrate-to-supabase
   ```

3. **Validate functionality**
   - Test all user flows
   - Verify data integrity
   - Check performance metrics

### 7.2 Production Rollout

1. **Backup existing MongoDB data**
2. **Deploy new version with feature flags**
3. **Gradually migrate users**
4. **Monitor system performance**
5. **Complete migration and remove MongoDB dependency**

### 7.3 Rollback Plan

- Keep MongoDB running during transition period
- Implement feature flag to switch between databases
- Maintain data synchronization during migration
- Quick rollback procedure if issues arise

## 8. Post-Migration Tasks

### 8.1 Performance Optimization

- Optimize Supabase queries
- Implement proper indexing
- Set up connection pooling
- Configure caching strategies

### 8.2 Monitoring & Observability

- Set up Supabase monitoring
- Implement application metrics
- Configure alerting
- Monitor database performance

### 8.3 Documentation Updates

- Update deployment documentation
- Create troubleshooting guides
- Document new environment variables
- Update development setup instructions

## 9. Timeline Estimate

| Phase | Duration | Dependencies |
|-------|----------|-------------|
| Preparation | 1-2 days | Supabase project setup |
| Database Migration | 2-3 days | Schema creation, data migration |
| Code Refactoring | 3-5 days | Database layer, auth integration |
| Docker Updates | 1-2 days | Container configuration |
| Testing | 2-3 days | Unit tests, integration tests |
| Deployment | 1-2 days | Staging, production rollout |

**Total Estimated Duration: 10-17 days**

## 10. Risk Mitigation

- **Data Loss**: Comprehensive backup and validation procedures
- **Downtime**: Blue-green deployment strategy
- **Performance Issues**: Load testing and optimization
- **Authentication Problems**: Thorough auth flow testing
- **Integration Failures**: Rollback procedures and monitoring

This implementation plan provides a structured approach to migrating from MongoDB to Supabase while ensuring minimal disruption to users and maintaining data integrity throughout the process.