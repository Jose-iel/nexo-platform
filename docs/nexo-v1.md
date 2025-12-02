# NEXO v1 - Sistema de Atendimento Automatizado

## 1. Conceito do Projeto

O **NEXO** é um sistema completo para gestão de agentes de atendimento automatizados via WhatsApp. O projeto evolui do agente atual no N8N para uma plataforma SaaS multi-tenant completa, oferecendo:

- **Atendimento automatizado** via IA
- **Gestão de campanhas** de marketing
- **Interface unificada** para múltiplos clientes
- **Dashboard analítico** completo
- **Sistema multi-tenant** isolado

### Integração com APIs WhatsApp

O sistema suportará múltiplas APIs de WhatsApp com fallback automático:

- **Principal**: Quepasa API
- **Alternativas**: Evolution API e API Oficial do WhatsApp
- **Configuração por cliente**: Flexibilidade na escolha da API

## 2. Funcionalidades do Sistema

### 2.1 Dashboard Principal

Métricas e visibilidade para o cliente:

- Quantidade de mensagens recebidas
- Mensagens respondidas pelo agente
- Reuniões marcadas automaticamente
- Gráficos de conversas por período
- Filtros por data personalizáveis

### 2.2 Gestão de Contatos

Lista completa de contatos com controles avançados:

- Visualização de todos os contatos
- Controle individual de atendimento (pausar/retomar)
- Pausar disparo de campanhas por contato
- Segmentação via tags (JSON)
- Histórico de interações

### 2.3 Interface de Mensagens

Chat em tempo real integrado:

- Visualização das conversas como WhatsApp
- Envio de mensagens pela plataforma
- Refresh automático (polling)
- Takeover manual quando necessário
- Histórico completo de mensagens

### 2.4 Sistema de Campanhas

Configuração completa de campanhas de marketing:

- **Múltiplas variações**: Mínimo 5 versões da mensagem (anti-bloqueio)
- **Agendamento flexível**: Imediato, agendado ou recorrente
- **Upload CSV**: Importação em massa de contatos
- **Configuração de horários**: Timezone e periodicidade
- **Status tracking**: Controle completo dos envios

### 2.5 Painel SuperAdmin

Gestão centralizada para administradores:

- **Relatório de tokens**: Gasto por cliente e período
- **Lista de clientes**: Ativar/desativar contas
- **Logs do sistema**: Monitoramento de eventos críticos
- **Health check**: Status das APIs WhatsApp
- **Controle de acesso**: Múltiplos níveis de permissão

## 3. Arquitetura de Dados

### 3.1 Schema PostgreSQL Completo

```sql
-- AUTENTICAÇÃO E USUÁRIOS
users
  - id, email, password_hash
  - role (client/super_admin)
  - client_id (FK - NULL se super_admin)
  - last_login_at, created_at

-- CLIENTES (multi-tenant isolado)
clients
  - id, name, company_name
  - status (active/inactive/suspended)
  - api_provider (quepasa/evolution/oficial)
  - whatsapp_instance_id, whatsapp_connected (boolean)
  - plan_type, token_limit, token_usage
  - created_at, updated_at
  - token_usage_current_month
  - token_usage_last_reset_at

-- CONTATOS (isolados por client_id)
contacts
  - id, client_id (FK)
  - phone, name, email, company
  - bot_paused (boolean)
  - tags (JSON - para segmentação futura)
  - created_at, updated_at, last_interaction_at

-- CAMPANHAS
campaigns
  - id, client_id (FK)
  - name, status (draft/active/paused/completed)
  - message_variations (JSON - array com 5 versões)
  - schedule_type (immediate/scheduled/recurring)
  - schedule_config (JSON - {date, time, cron, timezone})
  - created_at, updated_at, last_run_at

-- CONTROLE DE ENVIOS
campaign_contacts
  - id, campaign_id (FK), contact_id (FK)
  - status (pending/sent/failed/blocked)
  - last_sent_at, next_scheduled_at
  - sent_count, error_message
  - created_at

-- CONVERSAS (agrupamento)
conversations
  - id, contact_id (FK), client_id (FK)
  - status (bot_active/human_takeover/closed)
  - last_message_at, unread_count
  - assigned_to_user_id (FK users - NULL se bot)

-- MENSAGENS (histórico completo)
messages
  - id, conversation_id (FK), client_id (FK)
  - contact_id (FK), content, media_url
  - direction (inbound/outbound)
  - sender_type (bot/human/campaign)
  - timestamp, read_status, delivered_status

-- REUNIÕES AGENDADAS (salvas pelo agente IA)
scheduled_meetings
  - id, client_id (FK), contact_id (FK)
  - conversation_id (FK)
  - meeting_date, meeting_time, meeting_type
  - notes (extraído pela IA)
  - status (scheduled/completed/cancelled)
  - created_at

-- LOGS DO SISTEMA (retenção 7 dias)
system_logs
  - id, client_id (FK - NULL se log global)
  - level (info/warning/error/critical)
  - event_type (api_fallback/message_sent/campaign_executed/auth_failed)
  - message, metadata (JSON)
  - created_at

-- HEALTH CHECK LOGS (retenção 7 dias)
api_health_logs
  - id, api_provider (quepasa/evolution/oficial)
  - status (healthy/degraded/down)
  - response_time_ms, error_message
  - checked_at
```

### 3.2 Controle de Acesso Multi-Tenant

**Isolamento de dados por cliente:**

- **Super Admin**: Acessa tudo (sem filtro de client_id)
- **Cliente**: Todas queries filtram automaticamente por `user.client_id`

Implementação via:

- Middleware no backend (automático)
- PostgreSQL Row-Level Security (alternativa)

### 3.3 Sistema de Logs para SuperAdmin

**Campos de filtro:**

- Cliente (dropdown com todos os clientes)
- Nível (info/warning/error/critical)
- Tipo de Evento (api_fallback, message_sent, campaign_executed...)
- Período (24h, 3 dias, 7 dias, range customizado)

**Visualização:**
| Timestamp | Cliente | Nível | Tipo | Mensagem | Detalhes (JSON) |

### 3.4 Captura Automática de Token Usage

Exemplo de resposta da API de IA:

```json
{
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 80,
    "total_tokens": 230
  }
}
```

### 3.5 Escolha do ORM: Prisma

**Vantagens do Prisma para o NEXO:**

✅ **Type-Safety Superior**

- Gera tipos TypeScript automaticamente do schema (10/10 vs 7/10 TypeORM)
- Zero `any` types - autocomplete perfeito na IDE
- Erros detectados em tempo de desenvolvimento

✅ **Developer Experience**

- Schema em arquivo único (`schema.prisma`) vs decorators espalhados
- Prisma Studio (GUI visual para inspecionar dados)
- Migrations automáticas e reversíveis

✅ **Multi-Tenant com Filtro Automático**

- Middleware nativo para filtros por `client_id`

## 4. Arquitetura de Implementação

### 4.1 Backend - Node.js + Express + TypeScript

**Estrutura Feature-Based** (recomendada para SaaS multi-tenant):

```
nexo-api/
├── src/
│   ├── features/                    # Módulos por domínio
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.validation.ts
│   │   │   └── auth.types.ts
│   │   ├── campaigns/
│   │   │   ├── campaigns.controller.ts
│   │   │   ├── campaigns.service.ts
│   │   │   ├── campaigns.routes.ts
│   │   │   ├── campaigns.validation.ts
│   │   │   └── campaigns.types.ts
│   │   ├── contacts/
│   │   │   ├── contacts.controller.ts
│   │   │   ├── contacts.service.ts
│   │   │   ├── contacts.routes.ts
│   │   │   ├── csv-parser.util.ts
│   │   │   └── contacts.types.ts
│   │   ├── messages/
│   │   │   ├── messages.controller.ts
│   │   │   ├── messages.service.ts
│   │   │   ├── messages.routes.ts
│   │   │   ├── polling.service.ts      # Refresh automático
│   │   │   └── messages.types.ts
│   │   ├── whatsapp/
│   │   │   ├── whatsapp.service.ts
│   │   │   ├── api-clients/
│   │   │   │   ├── quepasa.client.ts
│   │   │   │   ├── evolution.client.ts
│   │   │   │   └── oficial.client.ts
│   │   │   ├── fallback.service.ts     # Lógica de fallback
│   │   │   ├── health-check.service.ts
│   │   │   └── webhook.controller.ts
│   │   ├── nexo-agent/                  # Agente IA (futuro)
│   │   │   ├── nexo.service.ts
│   │   │   ├── prompt-builder.ts
│   │   │   ├── meeting-detector.ts
│   │   │   └── token-tracker.ts
│   │   ├── clients/                     # Gestão de clientes
│   │   │   ├── clients.controller.ts
│   │   │   ├── clients.service.ts
│   │   │   └── clients.routes.ts
│   │   └── admin/                       # SuperAdmin
│   │       ├── admin.controller.ts
│   │       ├── logs.service.ts
│   │       └── admin.routes.ts
│   │
│   ├── shared/                          # Código compartilhado
│   │   ├── database/
│   │   │   ├── prisma/
│   │   │   │   ├── schema.prisma
│   │   │   │   └── migrations/
│   │   │   ├── client.ts
│   │   │   └── seed.ts
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts       # JWT validation
│   │   │   ├── tenant.middleware.ts     # Multi-tenant filter
│   │   │   ├── error.middleware.ts
│   │   │   └── rate-limit.middleware.ts
│   │   ├── utils/
│   │   │   ├── logger.ts
│   │   │   ├── validator.ts
│   │   │   └── response.ts
│   │   └── types/
│   │       ├── express.d.ts             # Request type extension
│   │       └── global.types.ts
│   │
│   ├── config/
│   │   ├── database.config.ts
│   │   ├── env.config.ts
│   │   ├── redis.config.ts              # Cache/queue futuro
│   │   └── whatsapp.config.ts
│   │
│   ├── jobs/                            # Background jobs
│   │   ├── campaign-scheduler.job.ts
│   │   ├── log-cleanup.job.ts
│   │   └── health-check.job.ts
│   │
│   ├── app.ts                           # Express setup
│   └── server.ts                        # Entry point
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .env.example
├── tsconfig.json
├── package.json
└── README.md
```

### 4.2 Frontend - React + TypeScript + Vite

**Estrutura Feature-Based:**

```
nexo-web/
├── src/
│   ├── features/                        # Features isoladas
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   └── LoginForm.module.css
│   │   │   ├── hooks/
│   │   │   │   └── useAuth.ts
│   │   │   ├── services/
│   │   │   │   └── auth.service.ts
│   │   │   └── types/
│   │   │       └── auth.types.ts
│   │   ├── dashboard/
│   │   │   ├── components/
│   │   │   │   ├── MetricCard.tsx
│   │   │   │   ├── ConversationsChart.tsx
│   │   │   │   └── DateFilter.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useDashboardMetrics.ts
│   │   │   └── pages/
│   │   │       └── DashboardPage.tsx
│   │   ├── campaigns/
│   │   │   ├── components/
│   │   │   │   ├── CampaignForm.tsx
│   │   │   │   ├── CampaignList.tsx
│   │   │   │   ├── MessageVariations.tsx
│   │   │   │   └── ScheduleConfig.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useCampaigns.ts
│   │   │   │   └── useCSVUpload.ts
│   │   │   ├── services/
│   │   │   │   └── campaign.service.ts
│   │   │   └── pages/
│   │   │       ├── CampaignsPage.tsx
│   │   │       └── CreateCampaignPage.tsx
│   │   ├── contacts/
│   │   │   ├── components/
│   │   │   │   ├── ContactsList.tsx
│   │   │   │   ├── ContactCard.tsx
│   │   │   │   └── CSVUploadModal.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useContacts.ts
│   │   │   └── pages/
│   │   │       └── ContactsPage.tsx
│   │   ├── messages/
│   │   │   ├── components/
│   │   │   │   ├── ChatWindow.tsx
│   │   │   │   ├── MessageBubble.tsx
│   │   │   │   ├── ConversationList.tsx
│   │   │   │   └── MessageInput.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useMessages.ts
│   │   │   │   └── usePolling.ts       # Auto-refresh
│   │   │   └── pages/
│   │   │       └── MessagesPage.tsx
│   │   └── admin/
│   │       ├── components/
│   │       │   ├── ClientsList.tsx
│   │       │   ├── LogsViewer.tsx
│   │       │   ├── TokenUsageChart.tsx
│   │       │   └── LogFilters.tsx
│   │       ├── hooks/
│   │       │   ├── useAdminClients.ts
│   │       │   └── useSystemLogs.ts
│   │       └── pages/
│   │           ├── AdminDashboard.tsx
│   │           └── LogsPage.tsx
│   │
│   ├── shared/                          # Código compartilhado
│   │   ├── components/                  # UI Components
│   │   │   ├── Button/
│   │   │   │   ├── Button.tsx
│   │   │   │   └── Button.module.css
│   │   │   ├── Input/
│   │   │   ├── Table/
│   │   │   ├── Modal/
│   │   │   ├── Sidebar/
│   │   │   └── Layout/
│   │   │       ├── MainLayout.tsx
│   │   │       └── AdminLayout.tsx
│   │   ├── hooks/
│   │   │   ├── useDebounce.ts
│   │   │   ├── useLocalStorage.ts
│   │   │   └── useToast.ts
│   │   ├── services/
│   │   │   ├── api.ts                   # Axios instance
│   │   │   └── websocket.ts             # Futuro
│   │   ├── utils/
│   │   │   ├── formatters.ts
│   │   │   ├── validators.ts
│   │   │   └── constants.ts
│   │   └── types/
│   │       └── global.types.ts
│   │
│   ├── routes/
│   │   ├── AppRoutes.tsx
│   │   ├── ProtectedRoute.tsx
│   │   └── AdminRoute.tsx
│   │
│   ├── store/                           # State management (Zustand/Context)
│   │   ├── authStore.ts
│   │   ├── campaignStore.ts
│   │   └── uiStore.ts
│   │
│   ├── styles/
│   │   ├── globals.css
│   │   └── variables.css
│   │
│   ├── App.tsx
│   ├── main.tsx
│   └── vite-env.d.ts
│
├── public/
├── .env.example
├── vite.config.ts
├── tsconfig.json
├── package.json
└── README.md
```

## 5. Dependências e Configuração

### 5.1 Backend - Node.js + Express + TypeScript + Prisma

```json
{
  "name": "nexo-backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate",
    "db:studio": "prisma studio",
    "db:seed": "tsx prisma/seed.ts"
  },
  "dependencies": {
    "@prisma/client": "^6.0.0",
    "express": "^5.0.1",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "cors": "^2.8.5",
    "helmet": "^8.0.0",
    "express-rate-limit": "^7.4.1",
    "dotenv": "^16.4.5",
    "zod": "^3.23.8",
    "date-fns": "^4.1.0",
    "axios": "^1.7.9",
    "csv-parse": "^5.6.0",
    "bull": "^4.16.3",
    "ioredis": "^5.4.1",
    "winston": "^3.17.0"
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/node": "^22.10.1",
    "@types/bcryptjs": "^2.4.6",
    "@types/jsonwebtoken": "^9.0.7",
    "@types/cors": "^2.8.17",
    "prisma": "^6.0.0",
    "typescript": "^5.7.2",
    "tsx": "^4.19.2",
    "@types/bull": "^4.10.0"
  }
}
```

### 5.2 Frontend - Vite + React + Tailwind + shadcn/ui

```json
{
  "name": "nexo-frontend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.28.0",
    "axios": "^1.7.9",
    "date-fns": "^4.1.0",
    "react-hook-form": "^7.54.0",
    "zod": "^3.23.8",
    "@hookform/resolvers": "^3.9.1",
    "recharts": "^2.15.0",
    "lucide-react": "^0.468.0",
    "papaparse": "^5.4.1",
    "sonner": "^1.7.1",
    "@radix-ui/react-dialog": "^1.1.2",
    "@radix-ui/react-dropdown-menu": "^2.1.2",
    "@radix-ui/react-select": "^2.1.2",
    "@radix-ui/react-tooltip": "^1.1.4",
    "@radix-ui/react-switch": "^1.1.1",
    "@radix-ui/react-tabs": "^1.1.1",
    "@radix-ui/react-checkbox": "^1.1.2",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.6.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@types/papaparse": "^5.3.15",
    "@vitejs/plugin-react-swc": "^3.7.2",
    "vite": "^5.4.11",
    "typescript": "^5.7.2",
    "tailwindcss": "^3.4.16",
    "postcss": "^8.4.49",
    "autoprefixer": "^10.4.20",
    "eslint": "^9.16.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.14",
    "@typescript-eslint/eslint-plugin": "^8.15.0",
    "@typescript-eslint/parser": "^8.15.0"
  }
}
```

---

## Próximos Passos

1. **Configuração do ambiente de desenvolvimento**
2. **Implementação do schema Prisma**
3. **Desenvolvimento do middleware multi-tenant**
4. **Criação das APIs WhatsApp com fallback**
5. **Interface de usuário com shadcn/ui**
6. **Sistema de agendamento de campanhas**
7. **Dashboard analítico**
8. **Testes e documentação**
