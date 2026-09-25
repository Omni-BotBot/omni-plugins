# Omni Plugins

Show your app inside the Omni conversation sidebar. Your page receives the selected contact and can act on the conversation.

[Português (Brasil)](README.md)

## Try it in 1 minute

You need to be a workspace owner.

1. Open Omni → **Integrations** → **Plugins** → **Add Plugin**.
2. Fill in:
   - **Plugin Name**: `Playground`
   - **Endpoint URL (HTTPS)**: `https://omni-botbot.github.io/omni-plugins/examples/playground/`
   - **Height (pixels)**: `600`
3. Click **Create Plugin**.
4. Open any conversation. The Playground card shows the live conversation data and buttons that call every action.

## Build your plugin in 3 steps

### 1. Add the SDK and listen for the conversation

```html
<div id="contact">Waiting for a conversation…</div>
<button id="hello">Say hello</button>

<script src="https://omni.botbot.chat/omnijs.js"></script>
<script>
  OmniBotBot.on("conversation", function (conversation) {
    const c = conversation.contact || {};
    document.getElementById("contact").textContent = `${c.name || "Unknown"} · ${c.phone || "no phone"} · ${c.email || "no email"}`;
  });
  OmniBotBot.on("no_conversation", function () {
    document.getElementById("contact").textContent = "No conversation selected.";
  });
  document.getElementById("hello").addEventListener("click", function () {
    OmniBotBot.toast("success", "Hello from my plugin!");
  });
  OmniBotBot.ready();
</script>
```

Full file: [`examples/hello-world/index.html`](examples/hello-world/index.html).

Always load the SDK from `https://omni.botbot.chat/omnijs.js`. Do not copy it into your project.

### 2. Allow Omni to display your page

Omni shows your page in an iframe. Your server must allow it:

- Send the header `Content-Security-Policy: frame-ancestors https://omni.botbot.chat`.
- Do not send `X-Frame-Options: DENY` or `X-Frame-Options: SAMEORIGIN`.
- Serve the page over HTTPS.

nginx:

```nginx
add_header Content-Security-Policy "frame-ancestors https://omni.botbot.chat" always;
```

Express:

```js
res.setHeader("Content-Security-Policy", "frame-ancestors https://omni.botbot.chat");
```

Laravel:

```php
return response($html)->header('Content-Security-Policy', 'frame-ancestors https://omni.botbot.chat');
```

### 3. Register it in Omni

Open Omni → **Integrations** → **Plugins** → **Add Plugin**:

| Field | What to enter |
| --- | --- |
| **Plugin Name** | The card title agents see, e.g. `CRM`. |
| **Endpoint URL (HTTPS)** | Your page, e.g. `https://yourapp.com/omni`. |
| **Height (pixels)** | Card height, from `100` to `2000`. Default `600`. |
| **Visibility** | No role selected = every agent sees it. Select roles to limit it. **Just for me** = only you. |
| **Auth Secret** | Optional. With a secret, Omni signs every conversation it sends to the plugin (the [`context_token`](#signed-context-token) event), so your server can trust the contact it receives. Without one, nothing is signed. |

Click **Create Plugin**. Done.

## Migrating an existing integration?

You do not need to change your plugin page. Your existing SDK can keep listening for `conversation` and using the same actions. Omni sends `conversation` and `no_conversation` and accepts `toast`, `loading`, `dialog`, `addTag`, `removeTag`, `assignTo`, `moveToGroup`, `setProperty`, and `getWorkspaceInfo`.

Make these two updates:

1. Add `https://omni.botbot.chat` to your `frame-ancestors` header ([step 2](#2-allow-omni-to-display-your-page)).
2. Register the plugin in Omni ([step 3](#3-register-it-in-omni)).

Optional: load `https://omni.botbot.chat/omnijs.js` and use the `OmniBotBot` global. Legacy method names continue to work as aliases.

## Reference

### Events

Subscribe with `OmniBotBot.on(event, handler)`.

| Event | When | Data |
| --- | --- | --- |
| `conversation` | An agent opens a conversation. Supported for existing integrations. | [Conversation payload](#conversation-payload) |
| `thread` | Same moment and same payload as `conversation`, with Omni's name. Subscribe to one, not both. | [Conversation payload](#conversation-payload) |
| `no_conversation` | No conversation is selected. | none |
| `context_token` | Same moment as `thread`, only when the plugin has an **Auth Secret**. | `{ conversationId, token, expiresAt }`. See [Signed context token](#signed-context-token). |

Omni sends them when your page loads, whenever the agent switches conversation, and after you call `ready()`.

### Conversation payload

```json
{
  "id": "k5Qx8LmN",
  "contact": {
    "id": "Rz3pW9aB",
    "name": "Ada Lovelace",
    "phone": "+55 11 99999-9999",
    "email": "ada@example.com",
    "tags": ["VIP"],
    "notes": "Prefers WhatsApp",
    "location": "São Paulo, BR",
    "createdAt": "2026-09-01T12:00:00Z",
    "avatar": "https://omni.botbot.chat/avatars/ada.png"
  },
  "channel": "whatsapp",
  "inboxName": "Sales",
  "status": "open",
  "priority": 2,
  "labels": ["VIP", "Billing"],
  "assignedAgent": {
    "id": "Jm2Yt7cD",
    "name": "Grace Hopper",
    "avatar": "",
    "status": "online",
    "role": "agent"
  },
  "lastMessage": "Can you check my invoice?",
  "lastMessageTime": "2026-09-23T09:41:00Z",
  "messages": []
}
```

- `channel`: `whatsapp`, `telegram`, `instagram`, `facebook`, `email` or `webchat`.
- `status`: `open`, `pending`, `resolved`, `deleted` or `spam`.
- `priority`: `0` (none) to `4` (highest).
- `labels`: label names.

`id`, `contact.*`, `channel`, `status`, `priority`, `labels` and `assignedAgent` are stable. Other fields may change. `phone` is as stored by Omni: strip non-digits before matching it with your data.

### Signed context token

The `conversation` and `thread` events come from the agent's browser and are not signed: do not use their phone or email to unlock sensitive data. For that, set an **Auth Secret**. Omni's server reads the contact from its own database and signs the conversation with that secret.

`token` is a compact JWT (RFC 7515) with header `{"alg":"HS256","typ":"JWT"}`, signed with HMAC-SHA256 using the secret. `token: null` means Omni could not mint a token. `expiresAt` is the `exp` in Unix seconds. Send the token to your backend and verify there:

1. The signature, with a constant-time comparison. Reject any `alg` other than `HS256`.
2. `iss` is `https://omni.botbot.chat` and `aud` is the origin of the plugin endpoint (`scheme://host[:port]`, e.g. `https://yourapp.com`).
3. `exp` has not passed. Each token lasts 5 minutes. When you call `ready()`, Omni re-sends the current token and mints a new one only when it has 1 minute or less left.

```json
{
  "iss": "https://omni.botbot.chat",
  "aud": "https://yourapp.com",
  "iat": 1790000000,
  "exp": 1790000300,
  "jti": "0b6f7f3e-6c2a-4d0e-9d8e-0a3f1c2b4d5e",
  "plugin_id": "Xy7Qa2Lm",
  "workspace_id": "Pq4Rs8Tu",
  "conversation_id": "k5Qx8LmN",
  "contact": { "id": "Rz3pW9aB", "name": "Ada Lovelace", "phone": "+55 11 99999-9999", "email": "ada@example.com" },
  "agent": { "id": "Jm2Yt7cD", "name": "Grace Hopper", "is_owner": false }
}
```

`contact` is `null` when the conversation has no customer contact.

### Actions

| Method | What it does | Argument |
| --- | --- | --- |
| `toast(type, message)` | Shows an Omni notification. | `type`: `success`, `error`, `warning` or `info`. |
| `loading(isLoading)` | Shows or hides a loading indicator on the card. | `true` or `false`. |
| `showConfirm(options, cb)` | Opens a confirm dialog. `cb(confirmed)` gets `true` or `false`. | `{ title, description, okText, cancelText }` |
| `showAlert(options, cb)` | Opens an alert dialog. `cb` runs when it closes. | `{ title, description, okText }` |
| `addLabel(label)` | Adds a label to the conversation. Alias `addTag`. | Label name or id. |
| `removeLabel(label)` | Removes a label from the conversation. Alias `removeTag`. | Label name or id. |
| `assignTo(agentId)` | Assigns the conversation to an agent. | Agent `id` from `getWorkspaceInfo().agents`. |
| `setGroup(groupId)` | Moves the contact to a contact group. Alias `moveToGroup`. | Group `id` from `getWorkspaceInfo().contacts_groups`. |
| `setProperty(fields)` | Updates the contact. | Any of `display_name`, `email`, `phone`, `notes`, `tag`, `language`, `company_name`, `city_name`, `country_name`, `attributes`. |
| `getWorkspaceInfo(cb)` | Reads workspace data. | `cb({ agents: [{ id, display_name, … }], groups: [{ id, name }], contacts_groups: [{ id, name }], labels: [{ id, name, color }] })` |
| `getAgentInfo(cb)` | Reads the signed-in agent. | `cb(agent)` |
| `getConversationInfo(cb)` | Reads the current conversation. | `cb(conversation)`: the [payload](#conversation-payload). |
| `ready()` | Tells Omni your page is listening. Omni re-sends the conversation and, when a secret is set, the `context_token`. | none |

Only one callback can be pending at a time: call the next getter inside the previous callback.

```js
OmniBotBot.getWorkspaceInfo(function (workspace) {
  OmniBotBot.getAgentInfo(function (agent) {
    console.log(workspace.labels, agent);
  });
});
```

## Troubleshooting

- **The card is blank or says "refused to connect".** Your server blocks framing. See [step 2](#2-allow-omni-to-display-your-page).
- **The card shows but no data arrives.** Listen for `conversation`; older SDK versions may not know `thread`. Load the SDK `<script>` before your listeners.
- **`alert()` and `confirm()` do nothing.** The iframe blocks browser dialogs. Use `showAlert` and `showConfirm`.
- **Links do not open.** Use `target="_blank"`. The plugin cannot navigate the Omni window.
- **The page does not load at all.** Omni only loads HTTPS pages.
- **Agents do not see the plugin.** Check **Visibility**: remove role limits or select their role. **Just for me** hides it from everyone else.

---

SDK: `https://omni.botbot.chat/omnijs.js` · Examples: [hello-world](examples/hello-world/index.html), [playground](examples/playground/index.html) · License: [MIT](LICENSE)
