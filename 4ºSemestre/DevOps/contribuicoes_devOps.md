# Contribuições Individuais

Este documento lista, commit a commit, as minhas contribuições ao projeto **devops-agent** (agente que analisa falhas de workflows do GitHub Actions com um LLM local) e os padrões de projeto (GoF) aplicados em cada uma. A visão geral de todos os padrões do sistema está no `README.md` do projeto.

| Commit | Data | O que foi feito | Padrões |
| :--- | :--- | :--- | :--- |
| `48a854a` | 18/08/2026 | Proteção do token do GitHub (`.env` fora do Git) e dependência da API do GitHub. | — |
| `b522fd8` | 01/09/2026 | Integração com o LLM local (Jan) via Spring AI e reorganização dos pacotes em `client` e `dto`. | Builder, Singleton/DI |
| `ce6d5a4` | 03/09/2026 | Correção do build no CI e modelo de configuração local sem segredos. | — |
| `b11b019` | 06/09/2026 | Núcleo do agente: ciclo de vida, estratégias de planejamento, contrato com os comandos e testes. | State, Strategy, Singleton/DI |

As demais partes do sistema foram feitas pelos colegas: estrutura inicial e integração com os workflow runs (Leandro), consulta de logs de job (Gabriel Nunes), consulta de run por ID, padrões Command e Observer e README (Gabriel Rocha) e interface Swing com o padrão Composite (Rodolfo).

---

## Commit `48a854a` — 18/08/2026

Preparei o projeto para usar o token do GitHub com segurança: adicionei o `.env` ao `.gitignore`, para que as credenciais nunca sejam enviadas ao repositório, e incluí no `pom.xml` a biblioteca `github-api`. Também criei o esqueleto da classe que depois virou o `LogTruncator`.

---

## Commit `b522fd8` — 01/09/2026

Conectei o agente ao **Jan**, um LLM rodando localmente que expõe uma API compatível com a da OpenAI, usando o Spring AI (`spring-ai-starter-model-openai`).

### Cliente do LLM

**Classes:** `JanClient`, `DepuradorController`

O `JanClient` concentra toda a conversa com o modelo em um único método, `analisar(texto)`. O resto do sistema não precisa saber que existe um `ChatClient` do Spring AI por trás:

```java
public JanClient(ChatClient.Builder builder) {
    this.chatClient = builder.build();
}

public String analisar(String texto) {
    return chatClient.prompt()
            .user(texto)
            .call()
            .content();
}
```

- **Builder:** o Spring AI injeta um `ChatClient.Builder` já configurado (URL, modelo, temperatura) e o cliente é montado com `build()`.
- **Singleton / Injeção de Dependência:** `JanClient` e `GitHubClient` são beans únicos do Spring (`@Component`), recebidos pelo construtor de quem precisa deles.

Criei também o endpoint `GET /jan/teste` (`DepuradorController`) e um `CommandLineRunner` para verificar, ao subir a aplicação, se a comunicação com o modelo estava funcionando.

### Organização dos pacotes

- `GitHubService` passou a se chamar `GitHubClient` e foi para o pacote `client`, junto do `JanClient`: as duas classes são clientes de serviços externos, não regras de negócio.
- `WorkflowRun` e `WorkflowRunsResponse` saíram de `entity` e foram para `dto`, porque são objetos de transferência da API do GitHub, não entidades de banco.
- Corrigi o formato do `jobId` na URL de logs (`%d` → `%s`, já que o id chega como `String`).
- Reescrevi o `pom.xml` para o Spring Boot 3.5 com o BOM do Spring AI.

---

## Commit `ce6d5a4` — 03/09/2026

O build quebrava fora da rede local, porque o Maven Wrapper apontava para um Nexus interno (`10.0.0.13`). Troquei pelo repositório oficial do Maven (3.9.9), o que fez o CI voltar a funcionar.

Também:

- coloquei o `application.yml` no `.gitignore` e criei o `application.yml.example`, com a configuração do Jan e um valor de exemplo no lugar da chave; cada pessoa copia o arquivo e preenche os próprios dados;
- ajustei os records `WorkflowRun` e `WorkflowRunsResponse` aos campos que a API do GitHub realmente devolve (`name` em vez de `display_title`).

---

## Commit `b11b019` — 06/09/2026

**Pacotes:** `agent`, `agent.state`, `agent.strategy`

Implementei o núcleo do agente: o ciclo de vida, as duas formas de planejar, o modelo de dados que circula entre as etapas e o contrato que os comandos (feitos depois pelo Gabriel) deveriam seguir.

### Modelo do agente

| Classe | Papel |
| :--- | :--- |
| `Action` | Enum com as ações possíveis: `FETCH_RUN`, `FETCH_JOBS`, `FETCH_LOGS`, `ANALYZE_WITH_JAN`, `ANSWER`. |
| `Step` | Um passo do plano: ação + argumento. |
| `Plan` | Lista de passos devolvida pela estratégia. |
| `Observation` | Resultado de um passo: ação, sucesso e conteúdo. |
| `Phase` | Enum com as fases do ciclo de vida. |
| `AgentContext` | Guarda o `runId`, a estratégia escolhida, o plano, as observações, a resposta e o erro. |

### State — ciclo de vida do agente

**Pacote:** `agent.state`

Cada fase virou uma classe que implementa `AgentState` (`phase()`, `handle(context)` e `isTerminal()`): `InitState`, `PlanningState`, `ExecutingState`, `ObservingState`, `DoneState` e `FailedState`.

O `AgentRunner` roda o laço: chama `handle(...)` no estado atual, que faz o seu trabalho e **devolve o próximo estado**, até chegar a um estado final:

```java
AgentState state = new InitState();
while (!state.isTerminal()) {
    state = state.handle(context);
}
```

O fluxo é INIT → PLANNING → EXECUTING → OBSERVING, voltando para PLANNING até terminar em DONE ou FAILED. Cada estado decide sozinho o próximo, sem um `if/switch` central. O `ObservingState`, por exemplo, olha a última observação:

```java
if (last == null)            return new FailedState("nenhuma observacao produzida");
if (!last.success())         return new FailedState(last.content());
if (last.action() == Action.ANSWER) {
    context.setAnswer(last.content());
    return new DoneState();
}
return new PlanningState();   // volta pro topo do loop
```

Proteções que ficaram dentro dos próprios estados:

- `InitState` falha logo se não houver `runId`;
- `PlanningState` conta as iterações e vai para `FailedState` ao passar do limite (`agent.max-iterations`), evitando loop infinito;
- `ExecutingState` interrompe o plano no primeiro passo que falhar;
- o motivo do `FailedState` é copiado para o contexto, e o chamador sabe por que a análise falhou.

### Strategy — como planejar

**Pacote:** `agent.strategy`

A interface `PlanningStrategy` (`name()` e `plan(context)`) tem duas implementações intercambiáveis:

- `PlanThenExecuteStrategy` (`plan-then-execute`): monta o plano inteiro de uma vez. É rápida e previsível, mas "cega": planeja `FETCH_LOGS` antes de saber qual job falhou, então o argumento vai vazio.
- `ReActStrategy` (`react`): decide **um passo por vez**, olhando o que já foi observado. Dá mais voltas no laço, mas cada passo já nasce com o argumento certo; o `jobId` do `FETCH_LOGS` vem da observação do `FETCH_JOBS`:

```java
if (!context.hasExecuted(Action.FETCH_LOGS)) {
    return Optional.of(new Step(Action.FETCH_LOGS, jobIdFromObservations(context)));
}
```

O `AgentRunner` recebe do Spring todas as estratégias, indexa pelo nome e escolhe uma durante a execução: a pedida pelo chamador ou a padrão da configuração (`agent.strategy`). Um nome desconhecido gera um erro listando as opções disponíveis. O `PlanningState` é o único lugar que chama a estratégia, então um novo modo de raciocínio é só uma nova classe, sem mexer nos estados.

### Contrato com o Command

Deixei pronta a interface `CommandExecutor` (`execute(step, context)`), que é o único ponto de contato entre o `ExecutingState` e as ações reais. Enquanto o padrão Command não estava pronto, um `StubCommandExecutor` permitia subir a aplicação e testar o fluxo. Depois o Gabriel criou o `CommandDispatcher` implementando essa mesma interface, e ele substituiu o stub sem que nenhum estado precisasse mudar.

### Singleton / Injeção de Dependência

`AgentRunner`, as estratégias e o executor de comandos são beans únicos do Spring. As estratégias são descobertas automaticamente (`List<PlanningStrategy>`), sem registro manual, e o executor é injetado pela interface, o que permite trocar a implementação real por um falso nos testes.

### Testes

**Classe:** `AgentRunnerTest`, usando um executor falso (`RecordingExecutor`) que registra cada passo pedido e pode simular falhas.

| Teste | O que verifica |
| :--- | :--- |
| `planThenExecute_executaTudoMasNaoSabeOJobId` | **Strategy:** executa os 5 passos, mas o `FETCH_LOGS` vai sem `jobId`. |
| `reAct_executaOsMesmosPassosMasComOJobIdPreenchido` | **Strategy:** mesmos 5 passos, com o `jobId` vindo da observação anterior. |
| `falhaNoMeioInterrompeEregistraOMotivo` | **State:** uma falha no 3º passo leva ao `FailedState`, para a execução e guarda o motivo. |
| `runIdNuloFalhaLogoNoInit` | **State:** sem `runId`, o agente falha no `InitState` sem executar nada. |

---

## Padrões aplicados por mim

| Padrão | Tipo | Onde | Commit |
| :--- | :--- | :--- | :--- |
| State | Comportamental | `agent.state.*`, `AgentRunner`, `AgentContext` | `b11b019` |
| Strategy | Comportamental | `agent.strategy.*`, seleção por nome no `AgentRunner` | `b11b019` |
| Builder | Criacional | `ChatClient.Builder` no `JanClient` | `b522fd8` |
| Singleton / DI | Criacional | Beans Spring (`JanClient`, `GitHubClient`, `AgentRunner`, estratégias) | `b522fd8`, `b11b019` |
