# Plugins do Omni

Mostre seu app dentro da barra lateral da conversa no Omni. Sua página recebe o contato selecionado e pode agir sobre a conversa.

[English](README.en.md)

## Teste em 1 minuto

Você precisa ser dono (owner) do workspace.

1. Abra o Omni → **Integrações** → **Plugins** → **Adicionar Plugin**.
2. Preencha:
   - **Nome do plugin**: `Playground`
   - **URL do endpoint (HTTPS)**: `https://omni-botbot.github.io/omni-plugins/examples/playground/`
   - **Altura (pixels)**: `600`
3. Clique em **Criar Plugin**.
4. Abra qualquer conversa. O card Playground mostra os dados da conversa ao vivo e botões que chamam cada ação.

## Crie seu plugin em 3 passos

### 1. Adicione o SDK e escute a conversa

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

Arquivo completo: [`examples/hello-world/index.html`](examples/hello-world/index.html).

Sempre carregue o SDK de `https://omni.botbot.chat/omnijs.js`. Não copie o arquivo para o seu projeto.

### 2. Permita que o Omni exiba sua página

O Omni mostra sua página dentro de um iframe. Seu servidor precisa permitir isso:

- Envie o header `Content-Security-Policy: frame-ancestors https://omni.botbot.chat`.
- Não envie `X-Frame-Options: DENY` nem `X-Frame-Options: SAMEORIGIN`.
- Sirva a página em HTTPS.

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

### 3. Cadastre no Omni

Abra o Omni → **Integrações** → **Plugins** → **Adicionar Plugin**:

| Campo | O que preencher |
| --- | --- |
| **Nome do plugin** | O título do card que os agentes veem, ex.: `CRM`. |
| **URL do endpoint (HTTPS)** | Sua página, ex.: `https://seuapp.com/omni`. |
| **Altura (pixels)** | Altura do card, de `100` a `2000`. Padrão `600`. |
| **Visibilidade** | Nenhuma função marcada = todos os agentes veem. Marque funções para limitar. **Só para mim** = só você. |
| **Segredo de autenticação** | Opcional. Com um segredo, o Omni assina cada conversa enviada ao plugin (evento [`context_token`](#token-de-contexto-assinado)), e seu servidor pode confiar no contato recebido. Sem segredo, nada é assinado. |

Clique em **Criar Plugin**. Pronto.

## Migrando uma integração existente?

Não é preciso alterar a página do plugin. O SDK existente pode continuar escutando `conversation` e usando as mesmas ações. O Omni envia os eventos `conversation` e `no_conversation` e aceita `toast`, `loading`, `dialog`, `addTag`, `removeTag`, `assignTo`, `moveToGroup`, `setProperty` e `getWorkspaceInfo`.

Só há duas mudanças:

1. Adicione `https://omni.botbot.chat` ao seu header `frame-ancestors` ([passo 2](#2-permita-que-o-omni-exiba-sua-página)).
2. Cadastre o plugin no Omni ([passo 3](#3-cadastre-no-omni)).

Opcional: carregue `https://omni.botbot.chat/omnijs.js` e use o objeto global `OmniBotBot`. Os nomes dos métodos legados continuam funcionando como apelidos.

## Referência

### Eventos

Assine com `OmniBotBot.on(evento, handler)`.

| Evento | Quando | Dados |
| --- | --- | --- |
| `conversation` | Um agente abre uma conversa. Compatível com integrações existentes. | [Payload da conversa](#payload-da-conversa) |
| `thread` | Mesmo momento e mesmo payload de `conversation`, com o nome do Omni. Assine um dos dois, não ambos. | [Payload da conversa](#payload-da-conversa) |
| `no_conversation` | Nenhuma conversa selecionada. | nenhum |
| `context_token` | Mesmo momento de `thread`, só quando o plugin tem **Segredo de autenticação**. | `{ conversationId, token, expiresAt }`. Veja [Token de contexto assinado](#token-de-contexto-assinado). |

O Omni envia esses eventos quando sua página carrega, sempre que o agente troca de conversa e depois que você chama `ready()`.

### Payload da conversa

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

- `channel`: `whatsapp`, `telegram`, `instagram`, `facebook`, `email` ou `webchat`.
- `status`: `open`, `pending`, `resolved`, `deleted` ou `spam`.
- `priority`: `0` (nenhuma) a `4` (máxima).
- `labels`: nomes das etiquetas.

`id`, `contact.*`, `channel`, `status`, `priority`, `labels` e `assignedAgent` são estáveis. Os demais campos podem mudar. `phone` vem como está salvo no Omni: remova tudo que não for dígito antes de comparar com seus dados.

### Token de contexto assinado

Os eventos `conversation` e `thread` vêm do navegador do agente e não são assinados: não use o telefone ou o e-mail deles para liberar dados sensíveis. Para isso, cadastre um **Segredo de autenticação**. O servidor do Omni lê o contato do próprio banco e assina a conversa com esse segredo.

`token` é um JWT compacto (RFC 7515) com header `{"alg":"HS256","typ":"JWT"}`, assinado com HMAC-SHA256 usando o segredo. `token: null` significa que o Omni não conseguiu gerar o token. `expiresAt` é o `exp` em segundos Unix. Envie o token ao seu backend e valide lá:

1. A assinatura, com comparação em tempo constante. Recuse qualquer `alg` diferente de `HS256`.
2. `iss` é `https://omni.botbot.chat` e `aud` é a origem do endpoint do plugin (`scheme://host[:porta]`, ex.: `https://seuapp.com`).
3. `exp` não passou. Cada token vale 5 minutos. Ao chamar `ready()`, o Omni reenvia o token atual e só gera um novo quando faltar 1 minuto ou menos para ele expirar.

```json
{
  "iss": "https://omni.botbot.chat",
  "aud": "https://seuapp.com",
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

`contact` é `null` quando a conversa não tem contato de cliente.

### Ações

| Método | O que faz | Argumento |
| --- | --- | --- |
| `toast(type, message)` | Mostra uma notificação do Omni. | `type`: `success`, `error`, `warning` ou `info`. |
| `loading(isLoading)` | Mostra ou esconde um indicador de carregamento no card. | `true` ou `false`. |
| `showConfirm(options, cb)` | Abre um diálogo de confirmação. `cb(confirmed)` recebe `true` ou `false`. | `{ title, description, okText, cancelText }` |
| `showAlert(options, cb)` | Abre um diálogo de aviso. `cb` roda quando ele fecha. | `{ title, description, okText }` |
| `addLabel(label)` | Adiciona uma etiqueta à conversa. Apelido `addTag`. | Nome ou id da etiqueta. |
| `removeLabel(label)` | Remove uma etiqueta da conversa. Apelido `removeTag`. | Nome ou id da etiqueta. |
| `assignTo(agentId)` | Atribui a conversa a um agente. | `id` do agente em `getWorkspaceInfo().agents`. |
| `setGroup(groupId)` | Move o contato para um grupo de contatos. Apelido `moveToGroup`. | `id` do grupo em `getWorkspaceInfo().contacts_groups`. |
| `setProperty(fields)` | Atualiza o contato. | Qualquer um de `display_name`, `email`, `phone`, `notes`, `tag`, `language`, `company_name`, `city_name`, `country_name`, `attributes`. |
| `getWorkspaceInfo(cb)` | Lê dados do workspace. | `cb({ agents: [{ id, display_name, … }], groups: [{ id, name }], contacts_groups: [{ id, name }], labels: [{ id, name, color }] })` |
| `getAgentInfo(cb)` | Lê o agente logado. | `cb(agent)` |
| `getConversationInfo(cb)` | Lê a conversa atual. | `cb(conversation)`: o [payload](#payload-da-conversa). |
| `ready()` | Avisa o Omni que sua página está escutando. O Omni reenvia a conversa e, se houver segredo, o `context_token`. | nenhum |

Só um callback pode ficar pendente por vez: chame o próximo getter dentro do callback anterior.

```js
OmniBotBot.getWorkspaceInfo(function (workspace) {
  OmniBotBot.getAgentInfo(function (agent) {
    console.log(workspace.labels, agent);
  });
});
```

## Solução de problemas

- **O card fica em branco ou diz "recusou a conexão".** Seu servidor bloqueia o iframe. Veja o [passo 2](#2-permita-que-o-omni-exiba-sua-página).
- **O card aparece, mas não chegam dados.** Escute `conversation`; versões antigas do SDK podem não conhecer `thread`. Carregue o `<script>` do SDK antes dos seus listeners.
- **`alert()` e `confirm()` não fazem nada.** O iframe bloqueia diálogos do navegador. Use `showAlert` e `showConfirm`.
- **Links não abrem.** Use `target="_blank"`. O plugin não pode navegar a janela do Omni.
- **A página nem carrega.** O Omni só carrega páginas HTTPS.
- **Os agentes não veem o plugin.** Confira a **Visibilidade**: remova o limite de funções ou marque a função deles. **Só para mim** esconde o plugin de todos os outros.

---

SDK: `https://omni.botbot.chat/omnijs.js` · Exemplos: [hello-world](examples/hello-world/index.html), [playground](examples/playground/index.html) · Licença: [MIT](LICENSE)
