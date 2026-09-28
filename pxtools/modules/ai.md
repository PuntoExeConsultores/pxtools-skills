# @AI Module — AI Accounts and Connectors

Path: `@PXTools/@AI/`
Qualified name: `PXTools.AI`

Child module: `@PXTools/@AI/@Anthropic/` (`PXTools.AI.Anthropic`) — one provider's connector.
Subfolders: `APIs/`, `Personalized/`.

## 1. What it provides

One place to configure **how to call a language model**, and a contract any consumer can use without
knowing which provider answers. A row of `AIAccounts` is one configured way of calling a service: the
key, the model, the token cap and — crucially — which connector object does the talking.

It exists because that configuration used to live in system parameters, and a parameter is one value
per installation. There can be several ways of calling a model at once: a different model for a pilot,
a separate key per tenant, a transcription service alongside a chat service.

## 2. Core concept: the row is the configuration, the connector is the code

`AIAccounts` holds no logic. What makes a row usable is `AIAccountConnectorCode`, which points at an
entry of `DynamicCallReferences` — and that entry points at the object that actually speaks to a
provider's API. Changing provider is changing a value on a screen, not touching code.

**Two connector kinds, because they are two contracts**: `ReferenceType.AIChatConnector` takes
messages and returns text; `ReferenceType.AITranscriptionConnector` takes audio and returns text.
Separating them is what stops an account from pointing at an object that cannot answer what it will
be asked.

## 3. The `AIAccounts` transaction

| Attribute | Notes |
|---|---|
| `AIAccountId*` | `IdFirstLevel`, autonumbered |
| `AIAccountName!` | descriptor |
| `AIAccountDisabled` | |
| `AIAccountConnectorCode` | → `DynamicCallReferenceCode`; the combo is filtered to the two AI reference types |
| `AIAccountApiKey` | |
| `AIAccountModel` | |
| `AIAccountMaxTokens` | |

The key is stored as text for the same reason as elsewhere in PXTools: the `Password` accessor
truncates, and what protects the value is the access control on the screen.

## 4. System parameters

One: **`AIDefaultAccountId`** (a Combo over the enabled accounts). It is the account for consumers
that have nowhere to take the choice from — a mail reader, a batch classification, anything wanting a
model with no conversation behind it. A consumer that *can* choose should choose; this is the
fallback, not the answer.

## 5. The connector contract

```
Parm(in:&AIAccountId, in:&SystemPrompt, in:&Messages, in:&AccessToken, in:&UseTools,
     out:&ResponseText, out:&InputTokens, out:&OutputTokens, out:&Ok, out:&ErrorMessage);
```

Nothing in it belongs to one provider or to one consumer. Two parameters are worth explaining:

- **`&AIAccountId`** — the connector reads the key, the model and the cap from that row. It does not
  read system parameters.
- **`&AccessToken`** — the tool surface's credential **enters as a parameter**. The platform hands it
  to the engine; the engine does not own it and cannot obtain it.

`AICallAnthropic` (`@Anthropic/APIs/`) is the implementation for one provider.

## 6. What a project registers

The module is small, and all three of its registrations are of the obligatory kind — the sort whose
omission does not fail, it deletes or silently disables:

| DataProvider | Registered in | If you forget |
|---|---|---|
| `RetSystemParametersAI` | `AddSystemParameters` **and `RetSystemParameters`** | The parameter is not created; or it is created and its combo comes out empty — see [systemparameters.md](systemparameters.md) |
| `RetMenusAI` | `AddDefaultMenus` | The screen exists and nothing reaches it |
| `RetDynamicCallReferencesAI` | `AddDynamicCallReferences` | The connector is not offered and cannot be resolved |

The module itself must also be declared in `SaveSystemModules`, or the seeding of menus rolls back
with "Missing module" — and, worse, a module row created by hand is deleted the next time that
procedure runs, because it starts by clearing the table.

## 7. Adding a provider

1. Write the connector in a child module (`@PXTools/@AI/@<Provider>/APIs/`) honouring the contract in
   §5.
2. Add a value to `DynamicCallReferenceCode`.
3. Declare it in `RetDynamicCallReferencesAI` with the right `ReferenceType`.
4. Create the account row and pick the connector.

No consumer changes.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [dynamiccallreferences.md](dynamiccallreferences.md) — how a code resolves to an object.
- [systemparameters.md](systemparameters.md) — the two registrations a parameter needs.
- [messaging.md](messaging.md) — the module's first consumer.
