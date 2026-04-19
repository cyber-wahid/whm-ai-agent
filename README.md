# WHM AI Agent

> Manage WHM/cPanel servers via Telegram using natural language – with mandatory confirmation for all destructive actions.

This n8n workflow acts as an autonomous system administrator. You mention the bot in a Telegram group, describe what you want in plain English, and the bot will either respond conversationally or, if you include the word `CONFIRM`, execute the corresponding WHM/cPanel API call and return a human-readable summary.

---

## ✨ Features

- **Natural Language Understanding** – Powered by any OpenAI-compatible LLM (OpenRouter, OpenAI, local model, etc.).
- **Dual Mode** – Chat mode for greetings / small talk; Execution mode when `CONFIRM` is appended.
- **Secure Confirmation** – No action is performed without explicit `CONFIRM` in the message.
- **Full WHM/cPanel Coverage** – Automatically constructs correct API calls for:
  - WHM API 1 (server-level tasks: suspend, terminate, create account, etc.)
  - cPanel UAPI (user-level tasks: email, databases, domains, SSL, FTP) via WHM's `uapi_cpanel` wrapper.
- **Human-Readable Output** – After execution, a second LLM call converts raw API JSON into a friendly Telegram message.
- **Authorization** – Restricts usage to specified Telegram user IDs.

---

## 🔧 Prerequisites

- A running **n8n** instance (self-hosted or cloud).
- A **Telegram Bot Token** (from [@BotFather](https://t.me/BotFather)).
- An **OpenRouter API Key** (or any other OpenAI-compatible endpoint).
- A **WHM/cPanel Server** with API access enabled.
- A **WHM API Token** (or username/password) formatted for HTTP Header authentication.

---

## 📦 Installation

1. Download the workflow file: `WHM_AI_Agent.json`
2. In your n8n instance, go to **Workflows → Import from File** and select the downloaded JSON.
3. The workflow will appear with missing credentials – set those up next.

---

## ⚙️ Configuration

### 1. Telegram Credentials

1. In n8n, go to **Credentials → New → Telegram API**.
2. Enter your **Bot Token**.
3. Assign the credential to the **Telegram Trigger** and **Telegram** nodes in the workflow.

### 2. OpenRouter (LLM) Credentials

The workflow uses HTTP Request nodes with inline headers.  
Update the **AI Platform Settings** node (Set node) with your own values:

```javascript
LLM_API_ENDPOINT: "https://openrouter.ai/api/v1/chat/completions"
LLM_API_KEY:      "Bearer YOUR_OPENROUTER_API_KEY"
LLM_MODEL:        "nvidia/nemotron-3-nano-30b-a3b:free"  // or your preferred model
```

> Alternatively, replace the endpoint with any OpenAI-compatible URL (e.g., local Ollama).

### 3. WHM API Credentials

Create a **Header Auth** credential in n8n:

| Field        | Value                             |
|--------------|-----------------------------------|
| Name         | e.g., `WHM API`                 |
| Header Name  | `Authorization`                 |
| Header Value | `whm username:YOUR_API_TOKEN`   |

Assign this credential to the **Execute WHM API** node.

### 4. Authorized Telegram User IDs

In the **Is Authorized User?** node, edit the condition to include your Telegram user ID(s):

```javascript
{{ [YOUR_TELEGRAM_ID, ANOTHER_USER_ID].includes($json.message.from.id) }}

// Example:
// {{ [1234567890, 9876543210].includes($json.message.from.id) }}
```

> **How to find your Telegram user ID:**  
> Open Telegram and send any message to [@userinfobot](https://t.me/userinfobot). It will instantly reply with your numeric user ID.  
> Alternatively, you can use [@getidsbot](https://t.me/getidsbot) or [@RawDataBot](https://t.me/RawDataBot) — all work the same way.  
> Replace the placeholder values above with your actual numeric IDs.

### 5. WHM Hostname

In the **Execute WHM API** node, update the base URL:

```
https://your-server.com:2087/json-api/
```

> If using a self-signed certificate, ensure **Allow Unauthorized Certificates** is enabled (already set in the workflow).

---

## 🚀 Usage

> **⚠️ Important – Group Only:**  
> This bot does **not** work in direct/private messages. The workflow is specifically designed to listen for **group mentions** only. You must:
> 1. Create a new Telegram group.
> 2. Add the bot to that group.
> 3. Mention the bot by username (e.g., `@nlsadminbot`) followed by your command.
>
> Messaging the bot directly will result in no response, as the trigger is not configured for private chats.

1. Create a new Telegram group and add your bot to it.
2. Mention the bot by its username (e.g., `@nlsadminbot`).
3. Type a command in natural language.

### Examples

| User Message | Bot Response |
|---|---|
| `@mybot Hello` | Hello, I am ready! |
| `@mybot List all email accounts for user 'john'` | ⚠️ Please reply again with your command and add the word CONFIRM to execute it on the server. |
| `@mybot List all email accounts for user 'john' CONFIRM` | ✅ Action Executed – User john has 3 email accounts: john@example.com, ... |
| `@mybot Suspend account 'spammer' CONFIRM` | Suspends account and returns confirmation. |

> **Important:** The word `CONFIRM` must appear somewhere in the message for any server action to be executed. This prevents accidental changes.

---

## 🔄 How It Works

1. **Telegram Trigger** – Listens for messages where the bot is mentioned inside a group.
2. **Authorization Check** – Verifies the sender is allowed.
3. **LLM Engine** – Sends the user's message to the LLM with strict system instructions:
   - If no `CONFIRM` → reply `{ "action": "chat", "reply": "⚠️ Please add CONFIRM ..." }`
   - If `CONFIRM` present → output `{ "action": "whm_api", "url_path": "/uapi_cpanel?..." }`
4. **Action Router** – Routes to either a direct chat reply or the WHM API execution.
5. **Execute WHM API** – Sends the constructed API path to your server.
6. **LLM Summarizer** – Converts the raw API response into a friendly Telegram message.
7. **Send Notification** – Delivers the final result back to the group.

---

## 🔒 Security Considerations

- **Never expose your API keys** in public repositories. The included JSON contains placeholder values – replace them with your own.
- **Restrict the bot** to trusted Telegram user IDs using the authorization node.
- The **confirmation mechanism** prevents accidental or malicious execution.
- Consider using **environment variables** in n8n for sensitive data instead of hardcoding in nodes.

---

## 🛠️ Customization

- **Change the trigger phrase** – Modify the **Is Bot Mentioned?** node to look for a different pattern (e.g., `!whm`).
- **Add more LLM models** – Edit the `LLM_MODEL` value in the **AI Platform Settings** node.
- **Extend API coverage** – The LLM's system prompt contains WHM/UAPI guidelines; you can expand or refine it in the **LLM Engine** node.

---

## 🐛 Troubleshooting

| Issue | Likely Fix |
|---|---|
| Bot does not respond to direct messages | This is expected — the workflow only listens for group mentions. Add the bot to a Telegram group and mention it there. |
| Bot does not respond in group | Check that the webhook URL is set correctly in Telegram (use n8n's webhook URL for the trigger). |
| Error parsing AI response | The LLM may have returned malformed JSON. The workflow includes a fallback regex parser; check n8n execution logs. |
| WHM API returns Access Denied | Verify the `Authorization` header format and that the API token has the required privileges. |
| Self-signed certificate error | Ensure **Allow Unauthorized Certificates** is enabled in the HTTP Request node. |

---

## 📄 License

MIT – feel free to use, modify, and distribute.

---

## 🙏 Acknowledgments

- Built with [n8n](https://n8n.io/) – workflow automation for technical people.
- LLM services provided by [OpenRouter](https://openrouter.ai/).
- WHM/cPanel API documentation: [WHM API 1](https://api.docs.cpanel.net/openapi/whm/operation/listaccts/) | [UAPI](https://api.docs.cpanel.net/openapi/cpanel/operation/list_pops/)
