---
description: Atualizar o mapa de dependencias apos alteracao de feature
---

# Atualizar Mapa de Dependencias

Voce acabou de alterar uma feature do Task Son. Siga este protocolo:

## 1. Identificar a feature alterada

Qual feature foi modificada? (Auth, Tarefas, Prova, Carteira, Nivel de Confianca, Alarmes, BullMQ, Notificacoes, Mesada, Controle Parental Android/iOS, Localizacao)

## 2. Verificar se houve mudanca de contrato

Uma mudanca de contrato inclui:
- Schema de entidade alterado (campos adicionados, removidos, renomeados)
- Integracao adicionada ou removida
- Regra de negocio de nivel de confianca ou carteira alterada
- Status workflow alterado
- Payload de job BullMQ alterado
- Constraint de banco adicionada ou removida

## 3. Atualizar os arquivos

Se houve mudanca de contrato, atualize:

1. `.github/instructions/mapa-dependencias.instructions.md` neste repositorio
2. `Referencias/mapa-dependencias.md` no vault Task Son (Second Brain)

Ambos devem refletir a mesma informacao. O arquivo do repo e a versao condensada.

## 4. Validar downstream

Liste as features downstream afetadas e confirme que:
- Nenhuma feature downstream quebrou
- Testes das features downstream ainda passam
- Jobs BullMQ afetados continuam idempotentes
