# Mapa de Agents e Modelos do OpenCode

Atualizado: 2026-11-03 · OpenCode v2.2.4 · pós-migração (devwservices removido e bloqueado)
Políticas de uso, tentativas e risco: ver `prompts/dev-team-core.md` (fonte única).
Este documento descreve a distribuição; não define política.

## Providers em uso

| Provider | Acesso | Franquia | Observações |
|---|---|---|---|
| `google` (antigravity-*) | OAuth Google via plugins `opencode-antigravity-auth` + `opencode-gemini-auth` | **separada do OpenCode Go** | Mecanismo NÃO-oficial; fontes/publicações sobre os termos do Google indicam risco de ban. Custo 0 no cadastro **não** comprova franquia ilimitada |
| `opencode` | Console OpenCode (pessoal) | console pessoal | Modelos `*-free` são promocionais/temporários; política de dados **por modelo** (ver pendências) |
| `opencode-go` | OpenCode Go (assinatura) | quota da assinatura | Kimi K3.1 Pro só existe aqui |

## Bloqueio devwservices

- Provider removido do `opencode.jsonc` (catálogo apagado).
- Bloqueio nativo v2: `policies: [{ "action": "provider.use", "resource": "devwservices", "effect": "deny" }]` — o provider desaparece do catálogo mesmo com credenciais válidas; projetos não podem reabilitar (precedência de policies).
- Variáveis de ambiente `DEVWORLD_API_KEY` / `DEVWORLD_API_BASE_URL` continuam no shell (fora do escopo desta mudança); não há credencial devwservices no `auth.json`.
- Referências históricas podem existir em backups `*.bak-*` e no histórico de sessões (intencionalmente preservadas).

## Agentes → modelo (efetivo)

| Agent | Onde | Modelo (ID exato) | Franquia | Papel/restrição |
|---|---|---|---|---|
| **orchestrator** (default_agent, primary) | `agents/orchestrator.md` | `google/antigravity-gemini-3.8-flash-medium` | Antigravity | Coordenação pura; sem edit/bash; delega via Task |
| **architect** | `agents/architect.md` | `google/antigravity-gemini-3.8-flash-high` | Antigravity | Plano/contratos; sem edição de produção |
| **review** | `agents/review.md` | `google/antigravity-gemini-3.8-flash-high` | Antigravity | Revisão intermediária read-only (quando útil) |
| **researcher** | `agents/researcher.md` | `google/antigravity-gemini-3.8-flash-medium` | Antigravity | Síntese de contexto; read-only (bash só inspeção) |
| **explore / scout** | `opencode.jsonc` (agent) | `google/antigravity-gemini-3.8-flash-low` | Antigravity | Buscas delimitadas read-only |
| **develop / backend / debugger / tester** | `agents/*.md` | `opencode/deepseek-v4.1-flash-free` | OpenCode (zero-retention oficial) | Executores padrão |
| **build / general** | `opencode.jsonc` (agent) | `opencode/deepseek-v4.1-flash-free` | OpenCode | Fallback manual / multi-propósito |
| **develop-lite** | `agents/develop-lite.md` | `opencode-go/mimo-v2.6-flash` | OpenCode Go | Só whitelist de trivialidades (free do MiMo usa dados p/ melhoria → não usar p/ código corporativo) |
| **develop-max / backend-max** | `agents/*-max.md` | `opencode/glm-5.3-flash-free` | OpenCode (zero-retention oficial) | **Alternativa** de implementação; não é tier superior |
| **security** (revisão crítica consolidada) | `agents/security.md` | `opencode-go/kimi-k3.1-pro` | OpenCode Go | Read-only; correção+integração+contratos+segurança+lacunas de teste |
| small_model (auxiliar: títulos etc.) | `opencode.jsonc` | `google/antigravity-gemini-3.5-flash-lite` | Antigravity | Mantido |

Low/Medium/High de Gemini 3.8 Flash são **IDs distintos** (não variants): `google/antigravity-gemini-3.8-flash-{low,medium,high}` — sem campo `variant`.

## Como trocar o modelo de um agent

- Agent file (`agents/*.md`): linha `model:` no frontmatter (remover `variant:` quando o ID novo não tem variants).
- Built-ins (build/plan/explore/general/scout): bloco `agent` do `opencode.jsonc`.
- Bloqueio de provider: `policies` no `opencode.jsonc` (sintaxe v2.2.4).

## Fallbacks (sem automação nativa — acionamento explícito; v2.2.4)

Sem fallback automático nativo para chat; não inventar campos. O acionamento troca **só o modelo da sessão** — o agent (papel + permissões) permanece:

- Sessão em execução (TUI): `/models` → escolher o alternativo; vale só para a sessão, não grava config. Retorno: mesma via, escolher o modelo principal.
- Nova tarefa (CLI): `opencode run --agent <papel> --model <alternativo> "..."`. Retorno: mesmo comando com o modelo principal.
- Persistência de fallback (comportamento real, verificado): a troca vale **para a sessão** e continuações dela (`-s`/`-c`); a config nada sofre. **Nova sessão/tarefa abre automaticamente com o principal (Gemini)** — não é preciso ação. Retorno explícito **na mesma sessão**: `/models` → escolher o modelo principal.
- **Orchestrator não alterna a si mesmo**: acionar fora da chamada que falhou (comando/nova sessão).
- Depois de iniciar uma tarefa no fallback, prefira concluí-la nele; sem polling de disponibilidade.

### Gatilhos autorizados (falha operacional real)
rate limit/quota esgotada · indisponibilidade · timeout · auth/acesso negado · modelo ausente.
**Não** são gatilhos: aviso documental já conhecido, resposta indesejada sem erro demonstrado, tarefa difícil, brief fraco, recusa legítima de segurança (não contorne).

| Papel | Principal | Alternativo |
|---|---|---|
| orchestrator | `google/antigravity-gemini-3.8-flash-medium` | `opencode/deepseek-v4.1-flash-free` |
| architect | `google/antigravity-gemini-3.8-flash-high` | `opencode/deepseek-v4.1-flash-free` |
| researcher | `google/antigravity-gemini-3.8-flash-medium` | `opencode/deepseek-v4.1-flash-free` |
| explore / scout | `google/antigravity-gemini-3.8-flash-low` | `opencode/deepseek-v4.1-flash-free` |
| review / plan | `google/antigravity-gemini-3.8-flash-high` | `opencode/glm-5.3-flash-free` |
| develop/backend/debugger/tester; build/general | `opencode/deepseek-v4.1-flash-free` | `opencode-go/deepseek-v4.1-flash` |
| develop-max / backend-max | `opencode/glm-5.3-flash-free` | `opencode-go/glm-5.3-flash` |
| develop-lite | `opencode-go/mimo-v2.6-flash` | sem alternativa autorizada → `blocked` |
| security (revisão crítica) | `opencode-go/kimi-k3.1-pro` | **sem fallback autorizado** — pendência; revisão fica pendente |

Regras: permissões do agent preservadas; Kimi nunca fallback genérico; troca sempre visível no resultado (papel, principal, motivo, alternativo, franquia, trabalho já feito); Go só dentro da assinatura, sem extra usage/recarga; retorno ao Gemini na próxima tarefa; orçamento de tentativas não zera com a troca.

### Free dos executores → Go

Quando um `*-free` faltar/limitar: trocar `model:`/sessão para o equivalente `opencode-go/<modelo>` (dentro da assinatura), registrar visivelmente e voltar ao free quando disponível.

## Proteção de segredos — estado (Opção B reforçada, aplicada 03/11/2026)

- Ferramenta `read` (todos os agents): `.env`/`.env.local` em qualquer subdiretório → **deny duro** (policy `tool.use read:*.env` + regras globais); `.env.example` sanitizado → **permitido**.
- `researcher`, `explore`, `scout`: **sem shell** (`bash: deny` efetivo), **sem busca de conteúdo** (`grep: deny`) e sem Code Mode (`execute: deny`); MCP atlassian bloqueado (`atlassian_*: deny`). Sem interpre­ters/scripts como alternativa (sem shell de forma alguma).
- Exploração continua por **descoberta → leitura**: `glob`/listagem por nome e caminho → `read` nos arquivos permitidos. Proibido contornar por MCP/shell/outra ferramenta.
- **LIMITE desta correção: protege somente os três exploradores.** Executores com shell (develop/backend/debugger/tester, build/general e sessões primárias) leem o filesystem do usuário — **denied de read ≠ isolamento de filesystem**; o ambiente não impede acesso a segredos de modo geral.
- review e security mantêm `bash["rg *"]: allow` (política atual); sem vazamento prático **hoje** porque `rg` não está no PATH (`command not found`). Se `rg` for instalado, ambos conseguiriam varrer `.env` por shell — **proposta pendente** (não aplicada): trocar essas linhas por `ask`/deny específica antes de qualquer instalação de `rg`.
- Tool `grep` nos exploradores = `deny` (não `ask`): a permissão do grep na v2.2.4 valida apenas a query — não é possível revisar o escopo de arquivos varridos antes de aprovar; busca de conteúdo fica para executores sob brief específico.

## Pendências

- Franquias efetivas (Antigravity, Go) não confirmáveis localmente — confirmar no Console.
- Checkpoint servido do DeepSeek V4.1 Flash pelo gateway não verificado sem inferência.
- Mecanismo antigravity: risco de ToS/aplicação do Google documentado; alternativa oficial (sem ativar): autenticação oficial do Gemini CLI (plugin `opencode-gemini-auth` já instalado expõe provider do Google Gemini CLI) ou API key oficial do AI Studio — decisão pendente do usuário.
- MiMo free: dados usados para melhoria do modelo durante o período promocional → uso apenas via `opencode-go`.
