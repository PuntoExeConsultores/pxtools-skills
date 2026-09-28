# @MCPServer Module — Model Context Protocol Server

Path: `@PXTools/@MCPServer/`
Qualified name: `PXTools.MCPServer`

Subfolders: `APIs/Basic/` (engine, External Objects, SDTs), `Personalized/` (hooks and registrations),
`Personalized/Sample/` (a complete runnable server to copy), `#Domains/`.

## 1. What it provides

An MCP server engine: it speaks JSON-RPC 2.0 over HTTP POST, authenticates with an OAuth Bearer
token, rebuilds the caller's PXTools `Context` per request, and dispatches `tools/*`, `resources/*`
and `prompts/*` to ordinary GeneXus procedures listed in a registry.

**One application can publish several MCP servers.** Each is an endpoint procedure plus its own
registry DataProvider and its own required scope. The engine is shared.

Protocol parsing and response building are **not written in GeneXus** — they live in an external Java
library reached through two External Objects. That library is not part of the Knowledge Base.

## 2. No tables, no screens

The module owns no transactions and publishes no UI. Everything it persists belongs to other modules:
log rows and tokens. Its menu DataProvider ships empty and says so.

## 3. The entry point

An endpoint procedure is `IsMain='True'`, `CallProtocol='HTTP'` and does only four things: build its
registry, call `MCPDispatcher`, set the HTTP status, and write headers and body. Everything else is in
the dispatcher.

**POST only.** Stateless Streamable HTTP: one JSON response per POST, no SSE stream, no session id.
Anything else is rejected with `InvalidRequest` and a 405 carrying `Allow: POST`.

**Authentication** is `Authorization: Bearer <token>`, validated **in-process** against
@OAuthService — no HTTP self-call — and then checked against the server's required scope. A scope
failure is 403, anything else 401. On both, the response carries `WWW-Authenticate` pointing at the
server's RFC 9728 metadata document, which is how a client discovers where to get a token.

**The user context is rebuilt on every request** from the token's account reference, and never cached
in a web session: MCP is stateless, and a session per request would leak one nobody reuses. The
context also carries the authorization id, so a tool can tell *which* connection is calling and not
merely who.

## 4. Declaring and dispatching a tool

The chain, in order:

1. The endpoint procedure asks its **registry DataProvider** for an `SDTMCPServerDefinition`
   (config + tools + resources + prompts).
2. That registry composes the tool list from **one fragment DataProvider per tool**, each returning
   `SDTMCPTools` with the tool's name, description, parameters and `TargetProcedure`.
3. `MCPDispatcher` scans the list by name and calls the target with the fixed contract below, inside a
   generator-specific `try/catch`.

**Dispatch is static, not through @DynamicCallReferences.** The registry holds `<Proc>.Type`
references directly; the module's `RetDynamicCallReferences*` ships empty and exists only as an
extension point.

**To add a tool**, a project writes: one procedure with the tool signature, one fragment DataProvider,
and two lines in its server's registry. Nothing in the module is touched. To add a whole server: copy
the endpoint and metadata procedures, write a registry, and declare its scope.

### The tool contract

```
Parm(in:&Context, in:&ArgumentsJson,
     out:&ResultText, out:&ResultRowsJson, out:&IsError, out:&ErrorMessage);
```

`&IsError = True` is a **business** error: it comes back as a tool result flagged as an error, not as
a JSON-RPC error. Arguments are read through the protocol object's typed accessors.

`&ResultRowsJson` is the second output, and it is what makes generic charting possible: a chart
request re-runs the tool server-side and draws its rows, so **no tool knows anything about charts and
the figures never pass through the model**. A transcribed digit would produce a chart that lies and
nobody would notice.

There is **no hand-written JSON Schema**: the flat parameter list *is* the input schema, and the Java
library turns it into the MCP `inputSchema`. Descriptions come from the fragment DataProviders —
per tool and per parameter. Server-level guidance for the model is the registry's `Instructions`,
returned in `initialize`.

Resources and prompts have their own fixed contracts, same shape.

Methods implemented: `initialize`, `ping`, `tools/list`, `tools/call`, `resources/list`,
`resources/read`, `prompts/list`, `prompts/get`.

## 5. APIs vs Personalized

- **`APIs/Basic/`** — the engine: `MCPDispatcher` (log → POST gate → IP gate → parse → rate limit →
  auth → context → dispatch), `MCPAuthorize`, the protected-resource metadata builder, the two
  External Objects and the SDTs.
- **`Personalized/`** — the hooks:

  | Object | Ships as |
  |---|---|
  | `ChkMCPRateLimit` | **stub, allows everything** |
  | `MCPEstablishContext` | working, over @Security |
  | `RetMCPLogFilterData` | what identifies a caller in the log is a business decision |
  | `MCPDeliverChart` | how a chart reaches the person — per installation |
  | `MCPResolveClientResponse` | placeholder |

- **`Personalized/Sample/`** — a complete server with one composed tool, one inline tool, one
  resource, one prompt, its endpoint and its metadata endpoint. This is the copy-paste template.

## 6. Domains

`MCPJsonRpcErrorCode` — the standard JSON-RPC codes plus MCP-specific ones for auth, scope, context
and missing resource. `MCPParameterType` — the JSON types a parameter may declare.

## 7. Dependencies

@OAuthService (token validation, scope check, issuer), @Security (token subject → user context),
@WebServicesLog (one row per request), @WSLayer (IP accept lists), @SystemParameters, @APIs (HTTP
helpers, the `Context` SDT, the chart stack).

The dependency on a chat channel runs **only through `Personalized/MCPDeliverChart`**, deliberately:
the engine knows nothing about messaging, because the reverse would drag a chat channel into every
installation that only wants an MCP server.

## 8. Traps

- **A module-qualified domain is mandatory inside an expression.** Writing the domain unqualified
  makes GeneXus take it for a control name — it says so in three warnings — and generate invalid code.
- **Set the HTTP status before writing headers or body.** Once the response is committed the status
  can no longer be set.
- **The log row is stamped with the caller after authentication**, not when it is opened: at open time
  only the server name is known and the column filtered nothing.
- **A JSON-RPC *response* must be intercepted before the method dispatch** — it carries an id but no
  method, so it would fall through to "method not found" and the client would be told -32601 for
  answering what the server asked.
- **Notifications and responses get 202 and an empty body**, never a JSON body.
- **Every hook and every dispatched call is wrapped in `try/catch`**, because an escaping exception
  reaches the servlet, the client gets an HTML error page it cannot parse, and the log row stays open
  as "without response".
- **Two GeneXus modelling shapes do not work** for a composed registry: inserting an external
  DataProvider, and typing a fragment's output as a level item of the parent SDT. Both fail at
  specification. That is why the tool-fragment SDT is duplicated rather than reused.
- **The metadata document needs a URL rewrite at the host**: the spec looks for it at the domain root
  and GeneXus publishes under the application path.
- **The required scope must exist and match what the client was granted**, or every call is 403.

## 9. Known gaps

1. **A built-in tool is hard-coded in the engine in one natural language** — its name is a
   non-translatable literal and its parameter keys are in that language, so a client in another
   language still sees them. It is language coupling inside a module meant to be generic.
2. **Rate limiting cannot be completed as documented**: the hook computes a retry-after value that the
   dispatcher receives and never passes on, so the 429 cannot carry the header.
3. **The built-in chart tool is intercepted before the registry loop**, so a server that legitimately
   declares a tool by that name is silently overridden; and it is advertised only when the server has
   tools but is callable even when it has none.
4. **Six declared External Object methods are dead** across the whole KB. Consequences worth knowing:
   the protocol version header is never validated, elicitation can be received but never initiated,
   and structured tool results are never produced even though tools already return rows.
5. **`TargetProcedure` is a member of the SDTs whose raw JSON is handed to the protocol builders.**
   Whether the internal link is stripped before it reaches the client depends on the external library
   and **cannot be determined from the Knowledge Base**. It is worth testing: it would be an
   information leak.
6. The sample server — the thing a new project copies — reads a host-specific field from the context.
7. No pagination cursors on any list, no per-tool scope (only per server), no batch support.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [oauthservice.md](oauthservice.md) — where the Bearer token comes from and how it is validated.
- [security.md](security.md) — the user context a token is turned into.
- [webserviceslog.md](webserviceslog.md) — the row every request opens and closes.
- [messaging.md](messaging.md) — one channel that reaches this surface with a user's token.
