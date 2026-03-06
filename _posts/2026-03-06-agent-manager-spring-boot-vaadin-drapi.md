---
layout: post
title: "Como Criar a Funcionalidade do HCL Domino Agents com Spring Boot, Vaadin e Domino REST API"
date: 2026-03-06
categories: [domino, drapi, java, spring, vaadin, agents, scheduling]
---

Um dos recursos mais poderosos do HCL Domino e o **Agent Manager** — um sistema integrado que permite criar rotinas que podem ser agendadas ou acionadas sob demanda. Ao migrar um ERP de Lotus Notes/Domino para uma stack moderna, essa funcionalidade precisa ser preservada. Este post mostra como recriei o Agent Manager usando Spring Boot 4, Vaadin 25 e o Domino REST API como camada de persistencia.

## O problema

No HCL Domino, agents sao cidadaos de primeira classe: voce os cria no Designer, define o tipo (agendado, manual, trigger), configura a periodicidade e o Agent Manager do servidor cuida do resto — execucao, log, controle de concorrencia. Na migracao para Java/Spring Boot, perdemos esse runtime e precisamos reconstrui-lo.

## A solucao: arquitetura hibrida

A abordagem combina:

1. **Logica do agent** = Spring beans implementando uma interface `AgentTask`
2. **Metadados de runtime** (enabled, cron, historico) = documentos no Domino via DRAPI
3. **Scheduling dinamico** = `TaskScheduler` do Spring com cron expressions alteraveis em runtime
4. **UI de gerenciamento** = Vaadin views para monitorar e controlar agents

### Por que persistir no Domino via DRAPI?

Consistencia com o resto da aplicacao (tudo via DRAPI), sem necessidade de banco adicional, e administradores podem ver os documentos de agent diretamente no Notes.

## Interface AgentTask

Cada agent implementa uma interface simples:

```java
public interface AgentTask {
    String getName();
    String getDescription();
    AgentResult execute();

    default String getDefaultCron() { return null; }
    default AgentType getType() {
        return getDefaultCron() != null ? AgentType.BOTH : AgentType.MANUAL;
    }
}
```

O resultado e encapsulado num record:

```java
public record AgentResult(boolean success, String message,
                          long durationMs, LocalDateTime timestamp) {
    public static AgentResult success(String msg, long ms) { ... }
    public static AgentResult failure(String msg, long ms) { ... }
}
```

## Modelo de metadados (DRAPI)

O `AgentMetadata` extends `AbstractModelDoc` — a classe base para todos os documentos Domino na aplicacao:

```java
public class AgentMetadata extends AbstractModelDoc {
    private String cronExpression;
    private String enabled;        // "Sim" ou "Nao"
    private ZonedDateTime lastRunTime;
    private String lastResult;     // "SUCCESS" ou "FAILURE"
    private String lastMessage;
    private Long lastDurationMs;
    private Long runCount;
    private Long errorCount;
    // Historico via campos multivalue (padrao Notes)
    private List<String> historyDates;
    private List<String> historyResults;
    private List<String> historyMessages;
    private List<String> historyDurations;
}
```

No Domino, isso corresponde ao Form `Agent` na base `sistema.nsf`.

## Service com suporte headless

O detalhe critico: agents agendados rodam em threads do servidor, **sem sessao Vaadin**. O `AbstractService` normalmente obtem o token da sessao do usuario. Para agents, usamos um fallback:

```java
@Override
protected String getUserToken() {
    try {
        return super.getUserToken(); // tenta sessao do usuario
    } catch (IllegalStateException e) {
        return adminTokenService.getAdminBearerToken(); // fallback admin
    }
}
```

## Auto-descoberta de agents

O `AgentRegistry` usa a injecao de dependencia do Spring para descobrir automaticamente todos os beans `AgentTask`. Adicionar um novo agent e tao simples quanto criar um `@Component`:

```java
@Component
public class DrapiHealthCheckAgent implements AgentTask {
    @Override public String getName() { return "drapi-health-check"; }
    @Override public String getDefaultCron() { return "0 */5 * * * *"; }

    @Override
    public AgentResult execute() {
        var health = statusService.checkDrapiHealth();
        return health.online()
            ? AgentResult.success("DRAPI online. Latencia: " + health.latencyMs() + "ms", ...)
            : AgentResult.failure("DRAPI offline: " + health.error(), ...);
    }
}
```

Na inicializacao, o Registry sincroniza com o Domino — se o documento ainda nao existe, cria automaticamente.

## Scheduler dinamico

O `AgentSchedulerService` usa `TaskScheduler` com `CronTrigger` para agendamento dinamico. Cron expressions podem ser alteradas pela UI sem redeploy:

```java
public void scheduleAgent(String name) {
    CronTrigger trigger = new CronTrigger(meta.getCronExpression());
    ScheduledFuture<?> future = taskScheduler.schedule(() -> runAgent(name), trigger);
    scheduledTasks.put(name, future);
}
```

A execucao usa `AtomicBoolean` como guard contra concorrencia — mesmo padrao que o Domino usa para impedir que um agent rode duas vezes simultaneamente.

## UI com Vaadin

A `AgentsView` exibe um grid com todos os agents:

- **Nome** (link para detalhe)
- **Status** (badge colorido: IDLE/RUNNING/ERROR/DISABLED)
- **Cron expression**
- **Ultima execucao** e resultado
- **Botoes**: Executar Agora, Ativar/Desativar

A `AgentDetailView` permite editar a cron expression e ver o historico completo de execucoes.

## Habilitacao no Spring Boot

```java
@SpringBootApplication
@EnableScheduling
public class Application {
    @Bean
    public TaskScheduler agentTaskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(3);
        scheduler.setThreadNamePrefix("agent-");
        return scheduler;
    }
}
```

## Mapeamento Domino vs Spring Boot

| Conceito Domino | Equivalente Spring Boot |
|----------------|------------------------|
| Agent | @Component implementando AgentTask |
| Agent Manager | AgentSchedulerService + TaskScheduler |
| Schedule | Cron expression dinamica |
| Run on schedule | @PostConstruct + CronTrigger |
| Run manually | AgentSchedulerService.runAgent() |
| Agent log | Campos multivalue no documento Agent |
| Enable/Disable | Flag no documento + unschedule/schedule |
| Concurrent guard | AtomicBoolean por agent |
| Agent properties | AgentMetadata no Domino via DRAPI |

## Conclusao

Com essa arquitetura, reproduzimos toda a funcionalidade do Agent Manager do HCL Domino dentro do Spring Boot, mantendo persistencia via DRAPI. O resultado e um framework extensivel onde adicionar um novo agent e criar um `@Component` que implementa `AgentTask` — o sistema descobre, sincroniza com o Domino e agenda automaticamente.
