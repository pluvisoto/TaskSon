---
applyTo: '**'
---

# Mapa de Dependencias de Features - Task Son

> **REGRA**: Consultar este mapa ANTES de qualquer alteracao de codigo.
> Identificar a feature afetada, verificar downstream e upstream.
> Atualizar apos qualquer mudanca que altere contrato de feature.

## Protocolo de consulta

1. Encontre a feature que sera alterada
2. Leia **Downstream** - sistemas que vao quebrar ou mudar comportamento
3. Leia **Upstream** - dependencias que sua alteracao assume que existem
4. Avalie **Risco** antes de prosseguir
5. Apos implementar, atualize este arquivo se o contrato mudou

## Tabela de impacto

| Feature alterada | Downstream imediato | Risco |
|---|---|---|
| Auth / RLS / Family / Device | TODO O SISTEMA | CRITICO |
| `Task` status workflow | Prova, Carteira, BullMQ | ALTO |
| Aprovacao de prova | Credito na carteira | ALTO |
| Constraint UNIQUE de Transaction | Creditos duplicados na carteira | CRITICO |
| BullMQ / Redis | Todos os jobs (credito, mesada, notificacoes) | CRITICO |
| `credit-wallet` job (idempotencia) | Saldo incorreto | CRITICO |
| Nivel de confianca - regras | Requerimento de prova em tarefas ativas | MEDIO |
| FamilyControls entitlement | Controle parental iOS bloqueado | ALTO |
| Accessibility Service (Android) | Bloqueio de apps deixa de funcionar | ALTO |
| PostGIS / Location schema | Rastreamento e queries de historico | MEDIO |

## Features e dependencias

### 1. Autenticacao e Gestao de Familia
- **Impl**: NestJS auth module, Supabase Auth (JWT), RLS em todas as tabelas, entidades User, Family, Child, Device
- **Upstream**: Supabase Auth (JWT), Firebase FCM (device token)
- **Downstream**: TODOS os modulos (Tarefas, Carteira, Nivel de Confianca, Controle Parental, Localizacao, Notificacoes)
- **Risco**: CRITICO - RLS incorreto expoe dados entre familias
- **Alto impacto**: alterar Family/Child (RLS quebra), PIN do filho (bloqueia acesso), schema Device (tokens FCM invalidados)

### 2. Gestao de Tarefas
- **Impl**: NestJS tasks module, entidade Task (titulo, descricao, prazo, recompensa R$, status)
- **Status workflow**: `pending` -> `in_progress` -> `awaiting_approval` -> `completed` / `rejected` / `expired`
- **Upstream**: Auth/Familia, Nivel de Confianca (prova obrigatoria)
- **Downstream**: Prova Fotografica, Carteira Virtual, Nivel de Confianca, Alarmes, BullMQ (auto-approve 48h), Notificacoes
- **Risco**: ALTO - campo `reward` consumido pelo credito; status controla aprovacao/credito
- **Alto impacto**: renomear campos Task (quebra workers), alterar maquina de estados sem migrar

### 3. Prova Fotografica
- **Impl**: NestJS proofs module, entidade TaskProof, Supabase Storage (90 dias, max 10MB), job `auto-approve-task` (48h)
- **Upstream**: Tarefas, Auth, Nivel de Confianca (nivel 4+ dispensa prova), Supabase Storage
- **Downstream**: Carteira Virtual (aprovacao dispara credito), Nivel de Confianca (taxa alimenta calculo), Notificacoes
- **Risco**: ALTO - aprovacao e gatilho do credito; job deve ser idempotente

### 4. Carteira Virtual
- **Impl**: NestJS wallet module, entidades Wallet (saldo por filho) e Transaction (log credito/debito)
- **Constraint**: UNIQUE em `(task_id, type)` previne creditos duplicados - NUNCA REMOVER
- **Upstream**: Auth, Prova Fotografica, BullMQ (`credit-wallet`), Mesada
- **Downstream**: Interface do filho, relatorios, Notificacoes
- **Risco**: CRITICO - saldo incorreto tem impacto financeiro real

### 5. Nivel de Confianca
- **Impl**: NestJS trust module, entidade TrustLevel (escala 1-5 por filho), job `trust-level-metrics` semanal
- **Regras**: nivel 1 = foto obrigatoria; nivel 5 = auto-aprovacao; pai decide elevacao
- **Upstream**: Tarefas (taxa conclusao), Prova (taxa aprovacao), Auth
- **Downstream**: Tarefas (prova obrigatoria), Prova (auto-aprovacao nivel 5), Notificacoes
- **Risco**: MEDIO - mudanca libera/bloqueia provas inesperadamente

### 6. Alarmes
- **Impl**: NestJS alarms module, entidade Alarm, job `schedule-alarm`, FCM IMPORTANCE_HIGH (Android), Critical Alerts (iOS)
- **Upstream**: Tarefas, Auth/Device, Notificacoes (FCM), BullMQ
- **Downstream**: UX do filho; iOS sem entitlement = sem som forcado
- **Risco**: MEDIO

### 7. Jobs Assincronos - BullMQ
- **Impl**: Workers BullMQ + Redis (Upstash)
- **Filas**: `credit-wallet`, `send-notification`, `schedule-alarm`, `allowance-cron`, `auto-approve-task`, `trust-level-metrics`
- **Upstream**: Redis, Auth, todos os modulos que disparam jobs
- **Downstream**: Carteira, Notificacoes, Prova, Mesada
- **Risco**: CRITICO - `credit-wallet` e `allowance-cron` tolerancia zero; jobs DEVEM ser idempotentes
- **Alto impacto**: alterar Redis sem continuidade (jobs perdidos), remover idempotencia, mudar payload sem versionar

### 8. Notificacoes
- **Impl**: NestJS notifications module, Firebase FCM, IMPORTANCE_HIGH (Android), Critical Alerts (iOS)
- **Upstream**: Auth/Device, Firebase FCM, BullMQ (`send-notification`)
- **Downstream**: UX (nao bloqueia fluxo principal)
- **Risco**: BAIXO

### 9. Mesada
- **Impl**: NestJS allowance module, config (valor, frequencia, dia/hora), job `allowance-cron`
- **Constraint**: UNIQUE em `(type, scheduled_at)` previne duplicidade
- **Upstream**: Auth, Carteira, BullMQ
- **Downstream**: Carteira (saldo), Notificacoes
- **Risco**: ALTO - valor financeiro acordado

### 10. Controle Parental - Android
- **Impl**: Kotlin nativo (AccessibilityService, DeviceAdminReceiver), Bridge Flutter via MethodChannel
- **Mecanismos**: Accessibility Service (bloqueio apps), UsageStatsManager, DeviceAdmin (lockNow)
- **Upstream**: Auth, Realtime Supabase (sync lista bloqueios), BullMQ (bloqueios agendados)
- **Downstream**: Autonomia do filho, Notificacoes
- **Risco**: ALTO - Google pode mudar politica Accessibility Service
- **Alto impacto**: alterar MethodChannel sem versionar (Flutter/Kotlin dessincronizam)

### 11. Controle Parental - iOS
- **Impl**: Apple FamilyControls (iOS 16+), ManagedSettingsStore, FamilyActivityPicker, DeviceActivityMonitor
- **Entitlements**: `com.apple.developer.family-controls`, `com.apple.developer.usernotifications.critical-alerts`
- **Upstream**: Auth, Apple FamilyControls (aprovacao obrigatoria)
- **Downstream**: Autonomia do filho Apple, Alarmes (Critical Alerts)
- **Risco**: ALTO - entitlement exige aprovacao Apple

### 12. Rastreamento de Localizacao
- **Impl**: Flutter geolocator + background_service, PostGIS, google_maps_flutter, Supabase Realtime
- **Config**: frequencia 30s / 5min / 15min, entidade Location (lat, lng, timestamp, childId)
- **Upstream**: Auth, PostGIS extension, permissoes Android/iOS, Google Maps API Key
- **Downstream**: Painel do pai (mapa), Geofencing (futuro), Controle Parental (contexto)
- **Risco**: MEDIO - localizacao background exige justificativa nas lojas

## Cadeia de dependencia por fase

```
Fase 1 - Fundacao
  Auth e Familia
    -> Gestao de Tarefas
         -> Prova Fotografica -> Carteira Virtual

Fase 2 - Engajamento
  Nivel de Confianca (controla Tarefas e Prova)
  Mesada -> Carteira Virtual
  Alarmes (vinculado a Tarefas)
  BullMQ (base de todos os jobs)

Fase 3 - Controle
  Localizacao (independente, exceto Auth)
  Controle Parental Android / iOS (independente, exceto Auth)
```
