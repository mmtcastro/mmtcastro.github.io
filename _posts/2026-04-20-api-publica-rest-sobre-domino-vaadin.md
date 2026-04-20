---
layout: post
title: "Construindo uma API REST Pública sobre Domino + Vaadin em um Dia"
date: 2026-04-20
categories: [domino, drapi, java, spring, vaadin, api, rest, security, api-keys]
---

Nosso portal interno é Vaadin sobre Domino REST API (DRAPI). Funciona bem pra operadores humanos com login LDAP. Mas integradores externos — parceiros, sistemas de terceiros, automações — precisam de outra coisa: REST padrão, Swagger, autenticação stateless. Este post descreve o dia em que separamos a API pública do portal interno, do zero até o primeiro `curl` respondendo 200. Seis horas, 3.058 linhas, quatro pegadinhas catalogadas.

## Por que separar

O portal interno tem sessão Vaadin, CSRF, LDAP, cookies stateful. Encaixar integração externa ali significa explicar `v-r=init`, forçar simulação de browser, expor topologia interna no HTTP. O custo operacional e cognitivo não vale.

Separamos em um módulo novo — Spring Boot isolado, porta própria, **reusa a camada DRAPI existente** como biblioteca interna. O módulo antigo não sabe que o novo existe.

```
public-api (:8083)          portal-interno (:8081)
     │                             │
     └──────┬──────────────────────┘
            │
        DrapiClient (lib compartilhada)
            │
        DRAPI / Domino
```

## Autenticação: X-API-Key sobre documento NSF

Decisão de storage: chaves ficam na mesma NSF dos outros dados transversais (`sistema.nsf`), como docs do tipo `ApiKey`. Sem banco relacional extra. Schema do doc:

| Campo | Conteúdo |
|-------|----------|
| `Id` | `sistema_ApiKey_<uuid>` (convenção da plataforma) |
| `Codigo` | **SHA-256 hex da chave crua** (lookup O(1) via `_intraCodigos`) |
| `Descricao` | Rótulo humano ("Portal Parceiro X — Produção") |
| `Scopes` | `empresas:READ,WRITE;noc:READ` |
| `Ativo` | `true` / `false` (soft revoke) |

O ponto interessante é `Codigo = hash`. Viola a convenção "Código é humano" mas elimina a necessidade de view custom — a `_intraCodigos` que já existe em toda base vira nosso índice de chaves gratuitamente. Ninguém vai olhar esses docs pelo Notes UI, então a feiúra do hash como código não cobra caro.

O filtro de auth é um `OncePerRequestFilter`:

```java
String raw = req.getHeader("X-API-Key");
String hash = sha256(raw);
apiKeyDao.findByHash(hash).ifPresent(apiKey -> {
    ApiKeyPrincipal principal = new ApiKeyPrincipal(apiKey);
    SecurityContextHolder.getContext().setAuthentication(
        new UsernamePasswordAuthenticationToken(principal, null, List.of()));
});
```

A chave crua nunca é persistida — só o hash. Na emissão pelo portal admin, mostramos a chave uma única vez num dialog não-dispensável ("copie agora, não será mostrada de novo"). Depois só o hash fica no DRAPI.

## Enforcement: @RequireScope + Interceptor

Para autorização granular, uma anotação + um `HandlerInterceptor`:

```java
@GetMapping("/sobreavisos")
@RequireScope(base = "noc", permission = Permission.READ)
public List<SobreavisoResponse> listar(...) { ... }
```

```java
public class ScopeCheckInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        RequireScope req = ((HandlerMethod) handler).getMethodAnnotation(RequireScope.class);
        if (req == null) return true;
        ApiKeyPrincipal p = (ApiKeyPrincipal) SecurityContextHolder.getContext()
                .getAuthentication().getPrincipal();
        if (!p.allows(req.base(), req.permission()))
            throw new AccessDeniedException("sem permissão " + req.base() + ":" + req.permission());
        return true;
    }
}
```

Inicialmente tentamos um `@Aspect` pra isso — mais elegante. Mas `spring-boot-starter-aop:4.0.5` não existe no repo público no momento em que escrevo. O interceptor é equivalente e não exige nenhum starter extra.

## As pegadinhas do dia

Plataformas legadas têm zonas cinzentas que só aparecem quando você pressiona. Aqui estão as quatro que o dia descobriu — cada uma virou memória pra não cair de novo.

### 1. DRAPI descarta silenciosamente campos fora do schema

Tentamos gravar um campo `Id` novo em docs legados via PATCH. Status **200 OK**. Response até omite o campo. GET posterior: vazio.

Causa: DRAPI valida o payload contra o **schema do Form declarado no scope**. Campos ausentes ali são descartados — sem erro, sem warning, sem nada. Notes em si é schemaless, o campo pode até ir pra NSF, mas DRAPI filtra na leitura.

Fix: adicionar o campo ao schema antes de gravar. Helper seguro (read-modify-write com backup):

```bash
python drapi_add_id_field.py --scope noc \
       --forms Sobreaviso,DiaSobreaviso,AnotacaoTrabalho,TagAnotacao --apply
```

### 2. `documents=true` vs entries — decisão de performance

DRAPI lê uma view de dois jeitos:
- `documents=true` — carrega o **documento completo** de cada entry (incluindo RichText/MIME)
- `documents=false` — só as **colunas publicadas pela view**

Pra views com as colunas certas, o doc completo é overhead puro. Em views de centenas+ de docs, vira gargalo.

Refatoramos o `DrapiClient` em dois métodos explícitos:

```java
drapi.queryViewEntries(scope, view, key, count)     // rápido
drapi.queryViewDocuments(scope, view, key, count)   // carrega doc
```

E atualizamos as `@Tool` descriptions (o código usa agentes internos) pra orientar a decisão:

```java
@Tool(description = """
    **PREFIRA ESTA TOOL** — consulta view retornando APENAS colunas (entries).
    Use quando os campos que você precisa estão expostos como colunas.
    Em ~80% dos casos é suficiente.
    """)
public String consultarViewEntries(...) { ... }

@Tool(description = """
    **USE COM PARCIMÔNIA** — carrega documento completo, mais lento.
    Use APENAS quando precisar de campo NÃO exposto na view.
    """)
public String consultarViewDocumentos(...) { ... }
```

### 3. Menu não é `@Menu`

Demoramos pra descobrir por que um novo `@Route("sistema/apikeys")` com `@Menu(order=110)` não aparecia no drawer. Cold restart, limpeza de dev-bundle, mudança de slug, testes via Playwright — nada.

Tracing no Playwright: o item **nem estava no DOM**. A causa: o `MainLayout` customizado popula o drawer iterando uma classe hardcoded `ModuleNavigation.MODULES`, não o registry do Vaadin `MenuConfiguration`. `@Menu` é ignorado.

Fix trivial: adicionar uma `NavEntry` na lista certa. Mas descobrir isso consumiu 40 minutos de debug. Salvamos como memória de sessão pra não tropeçar de novo.

### 4. UNID é identificador proprietário

Domino usa UNID (32 chars hex) como identificador físico de doc. Dia que migrar pra SQL, todo UNID vira abóbora. A convenção do projeto é persistir `Id = scope_Form_uuid` como identidade lógica. Mas views legadas ainda passam UNID na URL (`/sobreaviso/<UNID>`) e nos services.

Tratamos em camadas:

**(a) Família "ById" no `AbstractService`** — `findById` já existia; adicionamos `deleteById`, anexos por Id, com cache 5s pro resolve Id→UNID (DRAPI só deleta por UNID internamente).

**(b) Rota smart** no `AbstractViewDoc.beforeEnter`:

```java
String param = event.getRouteParameters().get("unid").orElse("novo");
// Id lógico contém '_' (scope_Form_uuid), UNID é só hex 32 chars
Response<T> response = param.contains("_")
    ? getService().findById(param)
    : getService().findByUnid(param);
```

Backward-compat total. Bookmark antigo `/sobreaviso/<UNID hex>` continua funcionando.

**(c) Piloto na view de lista**: troca `s.getUnid()` por `s.getId()` com fallback pra UNID. Smoke test no Playwright confirmou — 5 de 5 links gerados com `noc_Sobreaviso_<uuid>`, zero UNID.

**(d) Backfill dos legados**: muitos docs antigos nunca tiveram `Id`. Script Python idempotente itera `_intraForms` e PATCHa `scope_Form_uuid` em quem faltava. Dry-run default:

```bash
python drapi_backfill_ids.py --scope noc              # dry-run
python drapi_backfill_ids.py --scope noc --apply      # grava
```

Resultado do dia: 32 docs NOC populados (Sobreaviso, DiaSobreaviso). Os outros 1.025 já tinham.

## Endpoint emitindo 200 no final do dia

```bash
# health — sem auth
curl http://localhost:8083/v1/public/health
# {"status":"UP"}

# com chave noc:READ
curl -H "X-API-Key: sk_..." \
     "http://localhost:8083/v1/noc/sobreavisos?ano=2024&limit=2"
# [{"id":"noc_Sobreaviso_750eb284-...","codigo":"2024/02",
#   "ano":"2024","mesExtenso":"Fevereiro",...}]

# sem chave
curl http://localhost:8083/v1/noc/sobreavisos
# 403
```

Swagger UI em `/docs`, OpenAPI JSON em `/v3/api-docs`.

## O que vai na próxima

- Rate limit Bucket4j (dependência no pom, falta wiring)
- Persistir `apiKey.id + UltimoUso + latência` na chamada (cobrança/auditoria)
- Continuar UNID→Id área por área (NOC → faturas → sales → …)
- Permission granular separando CREATE/UPDATE/DELETE em vez do `WRITE` atual

## Quatro mental models que eu levo

1. **API pública e UI interna têm ciclos de vida diferentes.** Separar desde cedo é menos custoso que refatorar depois.
2. **Schemaless esconde atrito.** Um dia o schema vai te cobrar — via DRAPI strict, via cliente que espera tipo consistente, via migração SQL. Declarar o schema desde cedo paga juros negativos.
3. **Convenções do stack aparecem tarde.** Documente-as onde a próxima sessão de trabalho (sua ou de IA) vai esbarrar. Memória compartilhada entre sessões é tão valiosa quanto o código.
4. **Migrações de identificador pedem heurística transparente, não feature flag.** Rota aceita ambos → backfill → remove código antigo quando cobertura é alta. Nunca big-bang.

Três commits limpos, push pra `master`, 43 arquivos novos, 3.058 linhas adicionadas, 32 docs legados recuperados. E memórias gravadas pra quatro pegadinhas que a próxima sessão não vai reaprender do zero.
