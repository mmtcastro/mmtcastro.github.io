---
layout: post
title: "How to Recreate HCL Domino Agents with Spring Boot, Vaadin and Domino REST API"
date: 2026-03-06
categories: [domino, drapi, java, spring, vaadin, agents, scheduling]
---

One of HCL Domino's most powerful features is the **Agent Manager** — a built-in system that lets you create routines that can be scheduled or triggered on demand. When migrating a custom ERP from Lotus Notes/Domino to a modern stack, this functionality must be preserved. This post shows how I recreated the Agent Manager using Spring Boot 4, Vaadin 25 and the Domino REST API as the persistence layer.

## The problem

In HCL Domino, agents are first-class citizens: you create them in Designer, define the type (scheduled, manual, trigger), configure the schedule, and the server's Agent Manager handles the rest — execution, logging, concurrency control. When migrating to Java/Spring Boot, we lose that runtime and need to rebuild it.

## The solution: hybrid architecture

The approach combines:

1. **Agent logic** = Spring beans implementing an `AgentTask` interface
2. **Runtime metadata** (enabled, cron, history) = documents in Domino via DRAPI
3. **Dynamic scheduling** = Spring's `TaskScheduler` with runtime-changeable cron expressions
4. **Management UI** = Vaadin views to monitor and control agents

### Why persist in Domino via DRAPI?

Consistency with the rest of the application (everything via DRAPI), no need for an additional database, and administrators can view agent documents directly in Notes.

## AgentTask interface

Each agent implements a simple interface:

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

The result is encapsulated in a record:

```java
public record AgentResult(boolean success, String message,
                          long durationMs, LocalDateTime timestamp) {
    public static AgentResult success(String msg, long ms) { ... }
    public static AgentResult failure(String msg, long ms) { ... }
}
```

## Metadata model (DRAPI)

`AgentMetadata` extends `AbstractModelDoc` — the base class for all Domino documents in the application:

```java
public class AgentMetadata extends AbstractModelDoc {
    private String cronExpression;
    private String enabled;        // "Sim" or "Nao"
    private ZonedDateTime lastRunTime;
    private String lastResult;     // "SUCCESS" or "FAILURE"
    private String lastMessage;
    private Long lastDurationMs;
    private Long runCount;
    private Long errorCount;
    // History via multivalue fields (Notes pattern)
    private List<String> historyDates;
    private List<String> historyResults;
    private List<String> historyMessages;
    private List<String> historyDurations;
}
```

In Domino, this maps to the `Agent` Form in the `sistema.nsf` database.

## Service with headless support

The critical detail: scheduled agents run on server threads, **without a Vaadin session**. The `AbstractService` normally gets the token from the user's session. For agents, we use a fallback:

```java
@Override
protected String getUserToken() {
    try {
        return super.getUserToken(); // try user session
    } catch (IllegalStateException e) {
        return adminTokenService.getAdminBearerToken(); // fallback to admin
    }
}
```

## Auto-discovery of agents

`AgentRegistry` uses Spring's dependency injection to automatically discover all `AgentTask` beans. Adding a new agent is as simple as creating a `@Component`:

```java
@Component
public class DrapiHealthCheckAgent implements AgentTask {
    @Override public String getName() { return "drapi-health-check"; }
    @Override public String getDefaultCron() { return "0 */5 * * * *"; }

    @Override
    public AgentResult execute() {
        var health = statusService.checkDrapiHealth();
        return health.online()
            ? AgentResult.success("DRAPI online. Latency: " + health.latencyMs() + "ms", ...)
            : AgentResult.failure("DRAPI offline: " + health.error(), ...);
    }
}
```

On startup, the Registry syncs with Domino — if the document doesn't exist yet, it creates one automatically.

## Dynamic scheduler

`AgentSchedulerService` uses `TaskScheduler` with `CronTrigger` for dynamic scheduling. Cron expressions can be changed via the UI without redeployment:

```java
public void scheduleAgent(String name) {
    CronTrigger trigger = new CronTrigger(meta.getCronExpression());
    ScheduledFuture<?> future = taskScheduler.schedule(() -> runAgent(name), trigger);
    scheduledTasks.put(name, future);
}
```

Execution uses `AtomicBoolean` as a concurrency guard — the same pattern Domino uses to prevent an agent from running twice simultaneously.

## UI with Vaadin

`AgentsView` displays a grid with all agents:

- **Name** (link to detail view)
- **Status** (color-coded badge: IDLE/RUNNING/ERROR/DISABLED)
- **Cron expression**
- **Last execution** and result
- **Buttons**: Run Now, Enable/Disable

`AgentDetailView` allows editing the cron expression and viewing the complete execution history.

## Spring Boot setup

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

## Mapping: Domino vs Spring Boot

| Domino Concept | Spring Boot Equivalent |
|----------------|----------------------|
| Agent | @Component implementing AgentTask |
| Agent Manager | AgentSchedulerService + TaskScheduler |
| Schedule | Dynamic cron expression |
| Run on schedule | @PostConstruct + CronTrigger |
| Run manually | AgentSchedulerService.runAgent() |
| Agent log | Multivalue fields on Agent document |
| Enable/Disable | Flag on document + unschedule/schedule |
| Concurrent guard | AtomicBoolean per agent |
| Agent properties | AgentMetadata in Domino via DRAPI |

## Conclusion

With this architecture, we successfully reproduced all the functionality of HCL Domino's Agent Manager within Spring Boot, maintaining persistence via DRAPI. The result is an extensible framework where adding a new agent means creating a `@Component` that implements `AgentTask` — the system discovers it, syncs with Domino and schedules it automatically.
