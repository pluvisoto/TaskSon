# Task Son - Copilot Instructions

## Stack

Flutter (mobile), NestJS (backend), Supabase (Auth + RLS + Storage + Realtime), Firebase FCM, BullMQ + Redis.

## Mapa de dependencias

Antes de alterar qualquer feature, consulte `.github/instructions/mapa-dependencias.instructions.md`.
Este arquivo contem a tabela de impacto e as dependencias upstream/downstream de cada modulo.

**Protocolo**:
1. Identifique a feature que sera alterada
2. Verifique downstream (o que quebra) e upstream (o que voce assume)
3. Avalie o risco antes de prosseguir
4. Apos implementar, atualize o mapa se o contrato mudou (use o prompt `atualizar-mapa-dependencias`)

## Regras de codigo

- Jobs BullMQ devem ser idempotentes
- UNIQUE constraints em Transaction `(task_id, type)` e Mesada `(type, scheduled_at)` nunca devem ser removidos
- RLS policies devem isolar dados por familia - validar apos qualquer mudanca em schema
- MethodChannel Flutter/Kotlin deve ser versionado ao alterar
- Prova fotografica: aprovacao e o gatilho de credito na carteira - garantir consistencia
- Nivel de confianca: nivel 1 = foto obrigatoria, nivel 5 = auto-aprovacao

## Idioma

Documentacao e comentarios em portugues (pt-BR). Codigo (variaveis, funcoes, classes) em ingles.
