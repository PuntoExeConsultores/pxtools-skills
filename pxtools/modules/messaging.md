# @Messaging Module — Conversational Channels

Path: `@PXTools/@Messaging/`
Qualified name: `PXTools.Messaging`

Subfolders: `APIs/Basic/` (the engine), `APIs/WS/` (the endpoints the provider calls), `APIs/Web/`
(the screens a user sees), `Personalized/` (what a project customizes), `#Domains/`.

## 1. What it provides

A chat channel between the application and a person: the application receives what someone writes to
a bot, answers it — optionally through a language model with tools — and can push documents and
messages out to a chat on its own initiative. Telegram is the channel implemented; the model is
written for more than one (`MessagingChannel` also declares `WhatsApp`).

It is not only an assistant. Half the module is the **outbound** side: a queue that delivers a
document to a chat with retries and error classification, and a mechanism for actions that need the
person to confirm before they happen.

## 2. Core concept: the account is the bot, the link is the credential

Two ideas carry the whole module.

**A row of `MessagingAccounts` is one bot.** It holds the bot token, the webhook secret and the
username. Everything the module does is scoped by an account: a chat belongs to an account, a message
was received by an account, a document goes out through an account. An installation can run several
bots at once — one per tenant, one for a pilot — without them seeing each other.

**A row of `MessagingLink` is not a directory entry, it is the credential.** A chat arrives as a
number with no identity and no permission. When someone redeems a link code, `MessagingLinkChat`
identifies the user, **issues an internal OAuth authorization in their name**, and stores user + chat
+ authorization as the link. From then on every incoming message resolves, through
`RetMessagingAccessToken`, to a live access token — the token the tool layer receives on every call.
The link does not say who the person is; it says **what they may do**.

Two behaviours follow from that and make no sense without it:

- If the authorization died but the link is still `Active`, a new one is issued silently and the
  person is not asked to link again.
- Unlinking does not flag the row and stop there: `MessagingUnlink` **revokes the authorization**,
  because a live token would otherwise keep working until it expired.

**The chat id does not identify the bot.** On the platforms implemented, a private chat's id is the
person's own user id, and it is the same number across bots. That is why every table that holds a
chat also holds its account: without it, the same person talking to two bots is one row, and the
credential issued for one bot answers the other.

## 3. Module transactions (7)

| Transaction | What a row is |
|---|---|
| `MessagingAccounts` | One bot: token, webhook secret, username, channel, disabled flag |
| `MessagingLink` | A chat turned into an authorized identity — user, chat, authorization |
| `MessagingLinkCode` | A short-lived, single-use code a logged-in user generates to link their chat |
| `MessagingMessage` | The inbox, the log **and the conversation memory** (see §5) |
| `MessagingOutbox` | One outbound delivery: text, optional document, optional reply markup |
| `MessagingOutboxTargets` | Level of the above: one chat, with status, retries and next attempt |
| `MessagingPendingAction` | Something waiting on the person: a confirmation, a document, a choice |
| `MessagingMiniAppInvitation` | A single-use token backing an embedded screen (see §7) |

Secrets are stored as text, not as `Password`. That is deliberate: the `Password` accessor returns
through a `Character(64)` that truncates, and what protects these values is the access control on the
screen that shows them.

## 4. Module domains

`MessagingChannel`, `MessagingDirection`, `MessagingLinkStatus`, `MessagingLinkCodeStatus`,
`MessagingMessageStatus` (`Pending`, `Processing`, `Answered`, `Error`, `Ignored`),
`MessagingMessageType` (`Text`, `Voice`, `Photo`, `Document`, `Callback`, `Command`),
`MessagingOutboxStatus`, `MessagingOutboxTargetStatus` (`Pending`, `Retry`, `Sent`, `Failed`),
`MessagingPendingActionStatus`, `MessagingPendingActionType` (`ConfirmWrite`, `SendDocument`,
`Choice`), `MessagingMiniAppInvitationStatus`.

## 5. Inbound: from a webhook to an answer

1. **`TelegramWebhook`** (`APIs/WS/`) receives the update. It authenticates it by looking the header
   secret up in `MessagingAccounts` — **the same query that proves the origin also says which bot it
   is**, so there is no URL or endpoint per account. No matching account → 403.
2. It writes a `MessagingMessage` row with the raw payload, stamped with the account, deduplicates by
   the provider's message id, answers 200 immediately and hands the id to `MessagingProcessMessage`.
3. **`MessagingProcessMessage`** runs in background. One turn per chat: if another turn is running,
   the message waits and the running turn picks it up when it finishes — without that, two messages
   in a row would start two conversations with inconsistent histories.
4. It dispatches by type: a command, a callback from a button, a linking code, or a conversation turn.
5. A conversation turn loads the recent history **from `MessagingMessage` itself** — there is no other
   conversation memory in the system — builds the prompt, and calls the configured AI connector.

**The history filter is why the account matters most.** It reads the last turns of *this chat*; with
two bots and one chat id, leaving the account out of that filter feeds one conversation into the
other — and feeds it to the model.

## 6. Outbound: the outbox

A delivery is a header (`MessagingOutbox`: account, text, document, reply markup) and one row per
chat (`MessagingOutboxTargets`). Two levels, not three: grouping targets would only help a transport
that pays per connection, and it would let one blocked chat hold up the rest.

`TskMessagingOutbox`, a cyclic TaskManager task, drains it. Two passes — one entering by `Pending`,
one by `Retry` — because a single pass with an `or` over the status does not bound its entry point
and ends up reading everything.

**Errors are classified by code, not by text**: the terminal ones stop and, when the code means the
person blocked the bot, the subscription is dropped through an extension point; a rate-limit code
reschedules respecting what the provider asked for; the rest retry with a growing wait. The next
attempt is a **mutable column**, not part of a key — that is what makes a backoff possible at all.

`TskMessagingRetryStuck` picks up whatever was left mid-flight by a process that died.

## 7. Pending actions and the Mini App

A `MessagingPendingAction` is something the conversation owes the person: a write waiting for a
confirmation, a document to hand over, a choice to make. The turn drains them after answering, so the
model's reply and the action it triggered arrive in order.

A **Mini App invitation** backs an embedded screen. The page that receives the person is public and
what proves their identity is a payload signed by the provider — which is verifiable but **replayable
forever**, because the provider never invalidates it. The invitation is the other half: it expires and
it is spent once. Neither half is enough alone. The signature is verified **with the bot token**, so
the account has to travel with the invitation or it would be checked against the wrong key.

## 8. APIs vs Personalized

- **`APIs/Basic/`** — the engine and the transport: the transactions, the turn processor, the outbox
  and its tasks, and the per-channel primitives (send message, send document, set commands, download a
  file, build a deep link…). Not meant to be edited by a project.
- **`APIs/WS/`** — the endpoints the provider calls: the webhook and the Mini App redemption.
- **`APIs/Web/`** — the screens: the linking panel and the Mini App entry/close pages.
- **`Personalized/`** — the hook points:

  | Object | What a project customizes |
  |---|---|
  | `MessagingHandleStartPayload` | What to do with a deep-link payload this module does not recognise — an invitation of the project's own. Ships answering "not mine" |
  | `MessagingHandleChatCallback` | What a button's callback means for a chat with no identity |
  | `RetMessagingSystemPrompt` | What the model is told it is |
  | `RetMessagingExtraCommands` | Situational commands, offered only while they apply |
  | `RetMessagingMiniAppTarget` | Which screen an invitation opens |
  | `RetMessagingLogFilterData` | What to stamp the log row with, once the chat has an identity |
  | `MessagingTargetRejected` | What to do when the provider says a chat rejected the delivery |
  | `RetSystemParametersMessaging` / `RetMenusMessaging` / `RetDynamicCallReferencesMessaging` | The module's registrations |

## 9. Pattern instances

`PXWorkWithMessagingAccounts`, `PXWorkWithMessagingMessage`, `PXWorkWithMessagingOutbox`,
`PXWorkWithMessagingPendingAction`, `PXWorkWithMessagingLink` — CRUD and consultation screens.
`PXParameterRequestMessagingTelegramLink` is the panel where a user links their own chat.

## 10. Traps

- **A deep link's payload is not shown to the person.** The client displays only the command; the bot
  receives the argument. An empty-looking `/start` in a transcript is not evidence that the payload
  was lost.
- **The extension point for an unknown payload is only asked after the link code failed**, so a
  payload can never be claimed by two owners.
- **`&Handled = False` from a hook means "not mine", not "it failed".** The caller keeps the message
  the link attempt already produced; answering `True` with empty text leaves the person staring at
  nothing.
- **A command dispatched before the not-linked rejection** is what lets a `/start` from an unknown
  chat work at all. The order of the `Do Case` is load-bearing.
- **A reply keyboard's button text arrives as an ordinary message; an inline keyboard's callback does
  not.** They are not two styles of the same thing. With a reply keyboard, pressing the button is
  indistinguishable from typing its text, so the model sees the answer and the conversation continues.
  With an inline keyboard the callback goes to `MessagingHandleChatCallback` and the model never
  learns it happened. Choosing between them is choosing whether the answer is part of the
  conversation — an inline button is right for an action on a chat that has no identity, a reply
  keyboard for a choice the assistant asked for.

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [oauthservice.md](oauthservice.md) — the authorizations a link issues and revokes.
- [mcpserver.md](mcpserver.md) — the tool surface a turn's access token reaches.
- [ai.md](ai.md) — where the model, the key and the token cap are configured.
- [taskmanager.md](taskmanager.md) — how the outbox dispatcher is scheduled.
