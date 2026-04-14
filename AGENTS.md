# Task Son - Instrucoes para Agentes de IA

## Stack

Flutter (mobile), NestJS (backend), Supabase (Auth + RLS + Storage + Realtime), Firebase FCM, BullMQ + Redis.

## Arquivos obrigatorios

| Arquivo | Finalidade |
|---|---|
| `.github/instructions/mapa-dependencias.instructions.md` | Mapa de dependencias entre features - consultar ANTES de qualquer alteracao |
| `.github/prompts/atualizar-mapa-dependencias.prompt.md` | Prompt para atualizar o mapa apos mudancas |
| `.github/copilot-instructions.md` | Instrucoes globais do Copilot |

## Vault de documentacao

O vault de planejamento e documentacao esta em `Second Brain/Task Son/`. Contem:
- `Areas/` - Dominios continuos (engenharia, produto, controle parental)
- `Projetos/` - Subprojetos com escopo e entregaveis
- `Pesquisa/` - Investigacao tecnica e benchmarks
- `SOPs/` - Procedimentos operacionais
- `Processos/` - Fluxos de negocio e regras do produto
- `Referencias/mapa-dependencias.md` - Versao completa do mapa de dependencias

## Regras criticas

1. **Consultar mapa de dependencias** ANTES de alterar qualquer feature
2. **Atualizar mapa** APOS qualquer mudanca que altere contrato de feature
3. **Jobs BullMQ devem ser idempotentes** - podem ser reprocessados sem efeito duplicado
4. **UNIQUE constraints em Transaction e Mesada** - NUNCA remover
5. **RLS policies** - validar isolamento de dados entre familias apos qualquer mudanca em schema
6. **MethodChannel Flutter/Kotlin** - versionar ao alterar (controle parental Android)
