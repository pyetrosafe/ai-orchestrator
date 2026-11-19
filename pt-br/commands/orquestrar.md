---
description: Inicia o ciclo de orquestração (Cloud planeja e audita, Local executa em microtarefas)
agent: orchestrator
---
Analise a requisição abaixo e execute-a usando o fluxo de orquestração Cloud + Local:

$ARGUMENTS

Lembre-se:
1. Apresente brevemente o plano dividido em tarefas, subtarefas e microtarefas (`T1.1.1`, `T1.1.2`, `T2.1.1`...) antes ou ao iniciar a primeira delegação.
2. Uma tarefa representa um grupo maior de ações, uma subtarefa é uma ação específica dentro de uma tarefa, e uma microtarefa é a menor unidade de trabalho que pode ser delegada ao `local-worker`, por exemplo, "escrever um parágrafo", "gerar uma lista de itens", "criar um diagrama simples", "corrigir um erro de sintaxe", etc.
3. Delegue **apenas uma microtarefa por vez** ao `local-worker`, exigindo o contrato de resposta padronizada e registro em `.opencode/state/last-result.md`.
4. Audite cada retorno: se falhar, envie correção cirúrgica; se passar, avance para a próxima microtarefa até concluir tudo.
5. Instruções:
   5.1. Protocolo de Chamada de Subagentes - A comunicação com o `local-worker` deve sempre aderir ao formato de chamada explícito da ferramenta. Nunca chame `subagent` omitindo o argumento `description`. O formato técnico correto é: `subagent(agent: <nome>, description: <resumo>, prompt: <instruções>)`. O campo `description` é um metadado obrigatório do agente, mesmo que o Orquestrador precise injetar esse valor a partir do arquivo de configuração.