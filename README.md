# Sistema Agêntico Governado e Avaliado — Banco Aurora

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/steingui/aurora-governado/blob/main/sistema_agentico_governado_aurora.ipynb)

Evolução do sistema multiagente da Entrega II (Coordenador, Especialista de Domínio,
Analista SQL e Auditor) para uma solução **governada, observável e avaliada**.

O notebook é a entrega principal e foi construído para o **Google Colab**. O único
requisito externo é a chave da API do LLM. Sem chave, tudo roda em modo offline com
`MockLLM`: 25 testes, demonstração de HITL e avaliação completa.

## O que a entrega contém

| Requisito | Implementação |
|---|---|
| Guardrails e controles de execução | validação de entrada, SQL somente leitura, allowlist de tabelas, timeout, limite de linhas, orçamento de propostas/revisões e sanitização de saída |
| Human-in-the-Loop | `interrupt` do LangGraph com cartão de aprovação (finalidade, base legal, risco, SQL, PII, SLA); decisões **aprovar**, **rejeitar** e **ajustar**; função opcional `demo_aprovacao_manual()` para decisão humana real |
| Audit log | trilha append-only com `correlation_id`, ator, ação, resultado, severidade, política, `prev_hash` e `hash`; consulta, exportação JSONL e verificação de integridade |
| LGPD | catálogo de classificação por coluna, finalidades e bases legais, titular único, minimização, retenção, mascaramento, **redação de PII na entrada antes do LLM** e relatório de conformidade baseado em evidências |
| Métricas | qualidade, sucesso da tarefa, conformidade, auditabilidade, confiabilidade, erros, bloqueios, escalonamento, HITL, custo, interações e latência (média/p95) |
| Avaliação | ablação **G0** (sem governança) × **G1** (guardrails + audit) × **G2** (completo com HITL), 10 cenários × 3 repetições = 90 execuções |

## Resultados da avaliação offline

| Perfil | Sucesso da tarefa | Conformidade | Auditável | Qualidade | Confiabilidade | HITL | Custo (US$ ref.) | Latência média |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| **G0** — sem governança | 0,60 | 0,60 | 0,00 | 0,665 | 1,00 | 0,00 | 0,0168 | 100,6 ms |
| **G1** — guardrails + audit | 0,80 | **1,00** | **1,00** | 0,885 | 1,00 | 0,00 | **0,0140** | **86,6 ms** |
| **G2** — completo + HITL | **1,00** | **1,00** | **1,00** | **1,000** | 1,00 | 0,30 | 0,0155 | 94,9 ms |

Leitura: G1 é o mais barato e totalmente conforme, mas perde tarefas legítimas que
exigiriam aprovação humana (fail-closed). G2 recupera essas tarefas com HITL e atinge
100% de sucesso/conformidade/auditabilidade, com custo e latência ligeiramente maiores.
G0 responde a várias perguntas, mas vaza PII, não recusa acesso massivo e não tem
trilha auditável.

> A avaliação offline é um **harness controlado**, não um benchmark estatístico de
> qualidade de LLM. O DW é sintético, o `MockLLM` é determinístico e a latência é
> simulada (10 ms/chamada) para comparar o overhead de interações. A demo real
> complementa com o provedor configurado.

## Privacidade e exposição

- **Nenhuma chave real** está no repositório ou no notebook. A chave é lida em runtime de Secrets/env e nunca é impressa, logada ou gravada.
- O DW é **sintético**; não há dado de cliente real.
- Os valores usados nos testes (`000.000.000-00`, `teste@exemplo.invalid`) são fictícios/inválidos e existem apenas para verificar a redação.
- **PII na entrada** (CPF, CNPJ, e-mail, telefone) é mascarada antes de qualquer chamada ao LLM; o texto original não é enviado ao provedor nem gravado no audit.
- O audit log remove campos sensíveis e padrões de PII; as saídas também passam por sanitização.
- O repositório é público apenas com esse conteúdo sintético. Se a preferência for não expor nem o código, ele pode ser tornado privado.

## Como executar no Colab

1. Abra o notebook no Colab (ou faça upload do `.ipynb`).
2. Configure um dos Secrets no painel **Secrets**:
   - `OPENROUTER_API_KEY` (chave `sk-or-...`), ou
   - `OPENCODE_API_KEY` (chave `oc_sk_...`), ou
   - `LLM_API_KEY`.
3. Execute **Runtime → Run all**.

Sem chave, o notebook pergunta no terminal (quando interativo) ou segue direto para o
modo offline. Nenhuma chave é impressa ou gravada no arquivo.

### Provedores aceitos

| Prefixo | Provedor | Modelos |
|---|---|---|
| `sk-or-...` | OpenRouter | modelos `:free` com descoberta dinâmica e fallback |
| `oc_sk_...` | OpenCode Zen | `deepseek-v4.1-flash`, `claude-haiku-5-5`, `glm-5.3-flash` |
| `sk-...` (exceto `sk-or-...`) | OpenAI direto | `gpt-4.1-mini`, `gpt-4o-mini` |

## Estrutura do notebook

| Seção | Conteúdo |
|---|---|
| 1 | contexto, nível de autonomia, registro de riscos, princípios e arquitetura |
| 2 | setup, DW, catálogo LGPD, contratos, audit log, métricas, camada LLM, PDP/PEP e perfis |
| 3 | grafo governado, `interrupt`, orçamento e instrumentação |
| 4 | demonstração offline: aprovar, rejeitar e comparar G0/G1/G2 |
| 5 | 25 testes offline |
| 6 | avaliação, tabelas e gráficos |
| 7 | relatório de conformidade LGPD (G2 aprovado; contraste G0 reprovado) |
| 8 | demonstração opcional com LLM real |
| 9 | discussão, limitações, roadmap, checklist da rubrica e referências |

## Testes cobertos

- integridade do audit log (hash chain) e ausência de PII no log;
- classificação LGPD completa do DW;
- **PII na entrada:** CPF/CNPJ/e-mail/telefone são redigidos antes do LLM; o teste verifica que o valor não aparece nas mensagens enviadas nem no audit;
- política: agregado, finalidade ausente, PII massiva, titular único, `SELECT *`;
- HITL: aprovar executa mascarado; rejeitar não executa; G1 fail-closed;
- guardrails: destrutivo, prompt injection, SQL inválido, sanitização;
- orçamento, checkpoint/replay, métricas, auditoria completa e resiliência a falha do LLM;
- detecção de provedores OpenRouter, OpenCode Zen e OpenAI direto;
- relatório LGPD;
- contraste intencional: o baseline G0 vaza PII (teste documenta o risco).

## LGPD

O sistema implementa finalidade e base legal por tratamento, minimização de colunas,
titular único para atendimento, mascaramento antes do auditor, retenção declarada,
auditabilidade e relatório de conformidade. O relatório G2 passa em todos os checks;
incluindo G0, falha em minimização, auditabilidade e PII na resposta.

**Limitações:** não há autenticação de titular, gestão de consentimento, DLP de produção,
criptografia em repouso nem DPIA. O audit e o checkpointer são em memória. O relatório é
um artefato técnico de apoio, não um parecer jurídico.

## Estrutura do repositório

```text
aurora-governado/
├── sistema_agentico_governado_aurora.ipynb   # entrega principal, Colab-ready
├── README.md
└── requirements.txt
```

## Referências

- LGPD — Lei nº 13.709/2018.
- ANPD — guias e ciclo de vida do tratamento.
- LangGraph — `StateGraph`, checkpointing, `interrupt`, `Command`.
- ReAct (Yao et al., 2022).
- NIST AI Risk Management Framework.
- OWASP Top 10 for LLM Applications.
- OpenRouter e OpenCode Zen.
