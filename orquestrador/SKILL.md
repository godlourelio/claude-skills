---
name: "orquestrador"
description: "Orquestra desenvolvimento no Claude Code: delega cada tarefa a subagentes com o modelo e esforço certos, em paralelo, integra e reporta. Usar em features, UI, auth/pagamentos, testes, bugs, auditorias, code review e renames."
---

# Orquestrador

Esta é a sessão principal. O utilizador dá instruções aqui; tu divides o trabalho, delegas cada tarefa num subagente com o modelo e o esforço da tabela, integras o resultado no código principal e reportas. O trabalho pesado fica nos subagentes: cada tarefa corre no modelo adequado ao seu custo e risco, várias correm ao mesmo tempo, e o contexto desta sessão fica livre para coordenar.

Estas instruções valem para o resto da sessão, não só para o pedido que as carregou.

## Tabela de encaminhamento

A tabela é uma instrução explícita do utilizador: passa sempre `model` e `effort` em cada chamada à ferramenta Agent. Se o utilizador pedir outro modelo ou esforço para uma tarefa concreta, o pedido dele prevalece.

| Rota | `model` | `effort` | Sem Fable | Tipo |
|---|---|---|---|---|
| Planning features | `fable` | `high` | `opus` / `xhigh` | leitura |
| Build the feature | `sonnet` | `medium` | | escrita |
| Frontend UI | `opus` | `medium` | | escrita |
| Auth & Payments | `opus` | `xhigh` | | escrita |
| Searching the codebase | `haiku` | `low` | | leitura |
| Renames & boilerplate | `haiku` | `low` | | escrita |
| Write tests | `sonnet` | `low` | | escrita |
| Nasty bug | `opus` | `max` | | escrita |
| Security audit | `fable` | `xhigh` | `opus` / `max` | leitura |
| Code review | `opus` | `high` | | leitura |

"Extra" no vocabulário do utilizador é `xhigh`. Os aliases apontam sempre para o modelo mais recente de cada família.

### Escolher a rota

- **Auth & Payments** ganha a tudo: qualquer coisa que toque em autenticação, sessões, permissões, segredos, pagamentos ou faturação vai por aqui, mesmo que pareça UI ou uma feature normal. Um erro nesta área custa muito mais do que os tokens extra.
- **Nasty bug** é um bug cuja causa não se percebe à primeira: intermitente, atravessa módulos, envolve concorrência ou dados corrompidos, ou já resistiu a uma correção. Um bug com causa evidente vai por Build the feature, porque `max` é caro e tende a complicar problemas simples.
- **Planning** serve para features que mexem em vários ficheiros ou pedem decisões de desenho. Uma alteração pequena e clara vai direta para a rota de execução.
- **Security audit** e **Code review** só leem: devolvem achados, não corrigem. As correções entram como tarefas novas na rota adequada.
- Na dúvida entre duas rotas, escolhe a de maior risco.
- Se um subagente falhar duas vezes ou disser que está bloqueado, sobe uma rota (por exemplo Build para Nasty bug) e diz ao utilizador que escalaste.

Faz tu diretamente apenas o que não compensa delegar: responder a perguntas, comandos git de integração, verificações, e alterações de uma ou duas linhas (com commit, para a árvore ficar limpa).

### Fable disponível?

O Fable não existe em todos os planos, e pedir um modelo indisponível pode não dar erro: o subagente corre noutro modelo. Resolve isto uma vez por sessão, antes da primeira rota Fable. Se esta sessão já corre em Fable, está disponível. Caso contrário lança uma sonda mínima (`model: fable`, `effort: low`, prompt "Responde OK") e vê no resultado em que modelo correu de facto; se o resultado não o disser, pergunta ao utilizador se o plano dele inclui Fable. Se der erro ou tiver corrido noutro modelo, usa a coluna "Sem Fable" até ao fim da sessão e avisa o utilizador uma vez.

## Preparar o repositório

Uma vez por sessão, antes do primeiro trabalho de escrita:

1. **Git.** Sem repositório git não há worktrees: as tarefas de escrita correm uma de cada vez, ou propõe `git init`.
2. **Ramo de integração.** O ramo atual (`git branch --show-current`) é o "código principal" desta sessão. Todos os merges vão para lá, seja ou não o `main`.
3. **Árvore limpa.** Se `git status --porcelain` mostrar alterações, pede ao utilizador para fazer commit ou stash. Os subagentes só veem o que está em commit, e fazer merge por cima de alterações soltas arrisca perdê-las.
4. **Base dos worktrees.** Por omissão o Claude Code cria os worktrees a partir do ramo por omissão do remoto, não do HEAD local, por isso commits por enviar e ramos de feature ficam de fora. Se as settings do projeto não tiverem `"worktree": {"baseRef": "head"}`, propõe acrescentá-lo a `.claude/settings.json`. Em qualquer caso, o briefing leva o SHA de base (ver abaixo).
5. Sugere uma vez, sem bloquear: `.claude/worktrees/` no `.gitignore`, e um `.worktreeinclude` com os `.env` se os testes precisarem deles.

## Lançar trabalho

1. **Planear quando compensa.** Numa feature grande, lança primeiro a rota Planning e pede o plano dividido em pacotes: para cada um, a rota, os ficheiros que lhe pertencem, as dependências e o critério de conclusão. Mostra o plano e espera pelo OK do utilizador. Replanear é barato; construir a coisa errada não é.
2. **Paralelizar.** Lança na mesma mensagem todas as tarefas sem dependências entre si; correm em segundo plano e avisam quando terminam.
   - Tarefas de leitura correm em paralelo com tudo.
   - Tarefas de escrita correm em paralelo só se não tocarem nos mesmos ficheiros. Se dois pacotes precisam do mesmo ficheiro, corre-os em sequência ou junta-os num só: resolver conflitos custa mais do que o tempo que o paralelismo poupa.
   - Testes correm em paralelo com o build quando o plano fixa as interfaces; senão, depois do merge do build.
   - O limite é de 20 subagentes em simultâneo.
3. **Isolar.** Tarefas de escrita levam `isolation: "worktree"`; as de leitura não precisam. Usa os tipos `Explore` (pesquisa) e `Plan` (planeamento) se existirem, e `general-purpose` para o resto.
4. **Registar.** Cria uma entrada na lista de tarefas por cada subagente, com a rota, e mantém-na atualizada.
5. **Ficar disponível.** Não fiques à espera nem a sondar: as notificações chegam sozinhas. Entretanto o utilizador pode dar mais instruções.

### Briefing

O subagente começa do zero, sem esta conversa. O prompt tem de se bastar a si próprio:

- Objetivo e porquê, com a parte relevante do plano.
- Ficheiros ou áreas que pode alterar, e o que fica de fora.
- Critério de conclusão e comandos de verificação (testes, lint, build).
- Em escrita: "Confirma que `git rev-parse HEAD` é SHA_BASE (o HEAD do ramo de integração neste momento). Se não for, corre `git merge --ff-only SHA_BASE`; se isso falhar, pára e reporta. No fim, faz commit de tudo no ramo do worktree com mensagens claras. Não faças push nem merge."
- Em leitura: "Não alteres ficheiros." Achados com ficheiro, linha e gravidade.
- Relatório final: resumo, ficheiros alterados, verificações corridas e resultado, decisões tomadas, dúvidas ou bloqueios.

## Integrar e arquivar

Quando chega a notificação de uma tarefa de escrita:

1. **Ler o relatório.** O resultado traz o caminho e o ramo do worktree. Se ficaram alterações por fazer commit, retoma o subagente para as fechar.
2. **Rever.** Vê o diff contra o ramo de integração. Alterações de Auth & Payments, e qualquer diff pelo qual não consigas responder, passam por uma rota Code review antes do merge. Os achados voltam para o mesmo subagente (retoma-o com SendMessage: mantém contexto, modelo e esforço).
3. **Merge, um ramo de cada vez**, no checkout principal: `git merge --no-ff RAMO`. Um de cada vez, para que um conflito ou um teste partido aponte para um só ramo.
4. **Conflitos.** Os mecânicos (imports, linhas vizinhas) resolves aqui. Se as duas alterações discordam no conteúdo, `git merge --abort` e entrega a um subagente na rota de maior risco das duas, ou pergunta ao utilizador.
5. **Verificar.** Corre as verificações do projeto no ramo de integração. Se falharem, pára os merges seguintes e decide: tarefa de correção, ou desfazer o merge.
6. **Arquivar.** Só depois de um merge bem-sucedido: `git worktree remove CAMINHO`, `git branch -d RAMO`, tarefa marcada como concluída. Usa `-d` e não `-D`: recusa apagar trabalho por integrar, e é essa a rede de segurança. Se o worktree estiver bloqueado, `git worktree unlock CAMINHO` primeiro.
7. **Trabalho rejeitado ou adiado** fica onde está; diz ao utilizador o caminho e o ramo.

Tarefas de leitura não têm merge: reporta os achados e fecha a tarefa.

Nunca faças push, force ou reescrita de histórico sem o utilizador pedir.

## Reportar

- **Ao terminar cada tarefa:** o que era, a rota e o modelo/esforço que correram de facto, o que mudou, o resultado das verificações, e o estado (integrada, em revisão, em conflito, adiada).
- **Perguntas e bloqueios dos subagentes:** passa-os logo ao utilizador com o contexto necessário para decidir, e devolve a resposta ao mesmo subagente em vez de lançar outro.
- **Achados de auditoria e review:** ordenados por gravidade, com proposta de quais viram tarefas.
- **No fim de cada vaga:** tabela com tarefa, rota, estado e notas.
- Reporta resultados e decisões, não cada passo. Responde na língua do utilizador.

## Se algo não bater certo

- `effort` por chamada exige Claude Code v2.1.292 ou superior. Se a ferramenta Agent recusar o parâmetro, diz ao utilizador para atualizar e passa só `model` entretanto.
- Se o modelo ou o esforço parecerem ignorados, as causas habituais são `CLAUDE_CODE_EFFORT_LEVEL`, `CLAUDE_CODE_SUBAGENT_MODEL` ou um limite `maxEffortLevel`. `/tasks` mostra o modelo e o esforço de cada subagente.