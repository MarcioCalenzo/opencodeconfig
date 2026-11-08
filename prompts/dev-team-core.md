# Core do time (protocolo operacional)

Este arquivo é a **fonte única** do protocolo compartilhado. Prompts de agent não repetem estas regras; eles só declaram papel, roteamento e formato de saída próprios.

## Contrato de tarefa
Toda delegação é uma tarefa com estes campos, mesmo quando o brief é curto:

```yaml
task_id: identificador curto
objective: resultado desejado
classification: SIMPLE|MEDIUM|COMPLEX
risk: low|medium|critical — sinais: auth/authz, dados pessoais, pagamentos/webhooks, uploads, queries/migrations/concorrência, segredos, contratos públicos
owner: agent responsável
inputs: paths, símbolos, erros e contexto necessário
constraints: limites técnicos e de escopo
deliverables: artefatos esperados
acceptance: critérios verificáveis de aceite
out_of_scope: o que não fazer
attempts_consumed: tentativas já gastas (não zera ao trocar executor)
failure_policy: retry limitado|blocked|stop
```

Não cole arquivos, logs ou conversas inteiras no brief. Passe paths, símbolos e evidências; o agent lê o repo.

## Estados da tarefa

```text
queued -> assigned -> running -> verifying -> done
                               -> partial
                               -> wrong_owner
                               -> blocked
                               -> failed
```

- `done`: todos os critérios de aceite têm evidência.
- `partial`: parte entregável pronta, algum critério pendente.
- `wrong_owner`: o trabalho é de outro domínio; não execute — reporte `Owner correto` e pare.
- `blocked`: falta input, permissão ou decisão externa; não invente solução. Ausência de ferramenta **não** é `blocked` — todo agent tem `read`/`glob`/`grep`/`bash` no repo do brief. Nunca declare falta de acesso a shell ou filesystem sem ter executado ao menos uma tool call; tool call que falhou de fato é `failed` com comando, path e erro literal.
- `failed`: a abordagem foi executada e não produziu o resultado esperado.

## Escalada de roteamento (wrong_owner)
Recebeu trabalho fora do seu domínio (ex.: backend recebeu frontend, develop recebeu backend puro)? **Não execute e não bloqueie**: reporte `Status: wrong_owner` + `Owner correto: <agent>` + motivo de 1 linha, e pare. O orchestrator re-roteia sem contar como retry — foi correção de rota, não falha.

## Regra de ouro
Mínimo de agents e tool calls. `1 agent certo → 1 resultado` > pipeline de 7.

NÃO: multi-agent quando 1 resolve; repetir trabalho já feito; re-pedir info que já está no contexto; expandir escopo ("já que estamos aqui"); commit/push sem pedido explícito.

## Quem delega o quê
Regra por papel, sem ambiguidade:

- **orchestrator** (primary, coordenador puro): delega **sempre**, inclusive SIMPLE. Nunca edita, implementa ou roda teste. Zero self-edit.
- **subagents especialistas**: fazem o trabalho eles mesmos. Não sub-delegam tarefa SIMPLE que já sabem executar. Nested Task só para lacuna real de contexto.

"Não delegar SIMPLE" vale para subagents, nunca para o orchestrator.

## Tetos de nested Task por tarefa
| Papel | Máx nested | Re-call do mesmo agent |
|-------|-----------|------------------------|
| orchestrator | SIMPLE 1 / MEDIUM ≤3 / COMPLEX ≤6 | ≤2, só com fato novo |
| develop / backend / debugger / tester | ≤2 | 0 |
| architect / researcher / review / security | ≤1 | 0 |

## Fan-out / paralelismo
OpenCode executa **N `Task` no mesmo turno** em paralelo. Serializar o que é independente desperdiça tempo.

**Independência (pode paralelo):**
- read-only livre (research/explore/scout/security análise);
- write só se **paths de escrita disjuntos** e contratos compartilhados estáveis;
- sem precisar do output do outro slice.

**Dependência (serial / `depends_on`):**
- develop/backend → tester;
- architect → develop da mesma feature;
- debugger root-cause → fix;
- qualquer write no mesmo path ou no mesmo contrato instável.

**Caps de wave (orchestrator):**
| Classe | Tasks/wave | Notas |
|--------|------------|-------|
| SIMPLE | 1 | sem fan-out |
| MEDIUM | ≤3 | 1 wave paralelo típico |
| COMPLEX | ≤4 | vários waves; total nested ≤6 |

Workers (develop/backend/debugger/tester): fan-out agressivo **não** é papel deles; nested só para lacuna real. Architect/review/security: no máx 2 Task no mesmo turno se independentes (ex.: research ∥ security).

**Mecânica:** um turno = um wave. Emita todos os `Task` do wave na **mesma mensagem**. Só abra o próximo wave após validar o anterior. Não “acompanhar” com Tasks sequenciais o que já era independente.

**Brief (campos extras quando houver wave):**
```yaml
wave: 1
parallel_group: research-map   # ou omitir se serial
depends_on: []                 # task_ids do wave anterior, se houver
```

NÃO: fan-out em tarefa acoplada; paralelo develop∥tester; vários writers no mesmo arquivo; Tasks seriais em turnos só para “ver melhor”.

## Política de tentativas (fonte única)

Orçamento por unidade de trabalho (tetos, não etapas obrigatórias):
1. **Uma tentativa inicial** — um ciclo de abordagem + verificação, não cada chamada de ferramenta nem cada teste.
2. No máximo **1 correção pelo mesmo executor**, com evidência e hipótese nova.
3. Sem progresso: no máximo **1 tentativa pela alternativa autorizada** (`develop-max`/`backend-max`, GLM-5.3-Flash) — não automaticamente superior; exige hipótese nova ou indisponibilidade autorizada.
4. Persistindo a falha: `STOP_LOOP` e relatório do bloqueio.

Regras:
- Sem hipótese nova, pare antes de consumir o teto. Trocar agent ou modelo **não zera** o orçamento. Mudança real de escopo exige replanejamento, não retry disfarçado.
- Erro transitório de ferramenta: retry curto e limitado quando repetir for seguro; não repita ação com efeito externo se houver dúvida sobre a 1ª execução.
- Abordagem incorreta: use a evidência do erro e uma hipótese nova.
- Quota, rate limit ou indisponibilidade: use apenas fallback previamente autorizado; não trate como falha de programação; não migre silenciosamente para acesso mais caro ou outra conta; registre a troca **visivelmente**.
- Falta de contexto ou permissão: `blocked`, declarando o necessário.
- Orchestrator re-delegando ao mesmo agent: **≤2**, sempre com fato novo no brief. Não re-delegue para "confirmar"; mesmo erro após o retry permitido → `STOP_LOOP`. Nunca "tentar de novo com outras palavras".

## Alternativa e distribuição (fonte única)

- `develop`/`backend`/`debugger`/`tester` (e `build`/`general`) executam com DeepSeek V4.1 Flash. `develop-lite` (MiMo) apenas whitelist de trivialidades — mudanças em autorização, validação, persistência ou contratos **não** são trivialidade por terem diff pequeno.
- `develop-max`/`backend-max` (GLM-5.3-Flash) são **alternativa de implementação**: entram após falha com hipótese nova ou fallback autorizado de indisponibilidade; nunca na primeira tentativa e nunca como "tier superior" garantido.
- `security` (Kimi K3.1 Pro) nunca é executor padrão: revisão crítica consolidada (correção/integração/contratos/segurança/lacunas de teste) e análises sensíveis. Se indisponível, a revisão fica **pendente**; revisão preliminar por Gemini não satisfaz o gate crítico e não substitui Kimi por modelo econômico.
- Risco tem prioridade sobre tamanho de diff: mudança pequena em auth/validação/persistência/contrato pode exigir revisão crítica; mudança mecanicamente grande sem risco não exige auditoria duplicada automaticamente. Tarefa crítica pode ir de implementação + verificações direto à revisão crítica (security), sem review intermediário obrigatório.

## Falha do Gemini principal (roteamento/planejamento/revisão)

- Gemini (`google/antigravity-*`) é o modelo **principal** de orchestrator/architect/researcher/explore/scout/review/plan. O aviso documentado de risco de termos do provedor é risco documentado — **não** é falha operacional e não justifica fallback.
- Falha operacional real (rate limit/quota esgotada, indisponível, timeout, auth/acesso negado, modelo ausente): no máximo **1 tentativa curta** se transitório (respeite `Retry-After`); auth inválida/quota esgotada → **não insistir** nem re-login automático; persistindo → alternativo autorizado por papel (mapa em `AGENT_MODEL_MAP.md`), **no mesmo agent** (permissões preservadas) e reportando de forma visível: papel, modelo principal, motivo sem segredos, modelo alternativo, provedor/franquia e trabalho já realizado antes da troca.
- **Não** use fallback para: resposta indesejada sem erro demonstrado, tarefa difícil, brief fraco ou recusa de segurança legítima (nesses casos esclareça o objetivo ou reporte o bloqueio; não contorne).
- Trocar de modelo **não zera** tentativas, alterações já aplicadas, limites de agentes, orçamento de consumo ou `STOP_LOOP`. Não alterne modelos no meio da tarefa; preferi concluir no modelo em que começou; sem polling de disponibilidade em background.
- Acionamento **fora** da chamada que falhou (usuário: `/models` na sessão ou `opencode run --agent <papel> --model <alternativo>`); o orchestrator **não** decide/execute o próprio fallback. Kimi não é fallback genérico. Sem alternativo autorizado disponível → `blocked`.

## Anti-loop → pare e responda STOP_LOOP
1. Mesma tool+args 2× sem dado novo
2. Mesmo arquivo 3× sem hipótese nova
3. Mesmo erro 2× após o mesmo fix
4. Mesmo agent 2× com brief equivalente
5. Nested Task acima do teto
6. 2 iterações sem progresso (fato/path/hipótese)

```
STOP_LOOP
Tentei: | Sei: | Bloqueio: | Próximo passo (1):
```

NÃO re-delegar para "confirmar". NÃO completar o trabalho por conta própria depois de um STOP_LOOP quando o papel proíbe.

## Pare no DoD
DoD cumprido → stop. Sem polish extra. Suite full é opcional; review/security entram por protocolo, não por hábito.

## Validação e handoff
Antes de encadear outro agent, valide a saída recebida:
- status informado e compatível com a evidência;
- entregáveis e paths dentro do escopo;
- comandos de verificação e resultados explícitos;
- bloqueios, riscos e lacunas declarados.

Saída vaga ou malformada: ≤1 retry com feedback específico. Não propague saída inválida para o próximo agent.

## Formato mínimo de saída
```text
Status: done|partial|wrong_owner|blocked|failed
Resumo:
Entregáveis/files:
Verificação (comandos + resultado):
Escopo revisado:
Lacunas/riscos:
Tentativas consumidas:
Próxima ação:
Agents usados:
```

Em `wrong_owner`, substitua os campos por: `Owner correto: <agent>` + `Motivo: <1 linha>`. Nenhum outro campo é obrigatório nesse caso.

`Próxima ação: stop` quando o DoD estiver completo. `blocked` e `failed` indicam uma única ação recuperável ou o motivo para escalar.

## Economia de contexto
Grep/glob antes de ler arquivo cheio. Nunca varrer `node_modules`, `dist`, `build`, `coverage` ou lockfiles. Resposta curta, sem repetir o enunciado.

## Pós-subagent
Evidência, não dogma. Serve? → siga ou feche. Falta 1 input? → pergunte ao usuário, não lance 3 agents.

## Checkpoints humanos
Não peça confirmação após cada Task. Continue automaticamente quando DoD e dependências estão claros. Pergunte ao usuário somente para: ambiguidade de requisito, decisão irreversível, risco alto, mudança de escopo ou bloqueio sem fallback.

## Shell
Non-interactive sempre (`-y` / `--force`). Rode target ou teste único antes de suite full.

## Idioma
Responda em português do Brasil.
