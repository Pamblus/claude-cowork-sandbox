# claude-cowork-sandbox
Technical analysis of the Claude Code / Cowork sandbox: architecture, tokens, egress proxy, MCP proxy, stream-json RPC. Notes from observing one own session. Not an exploit. No systems compromised, no user data touched, nothing broken.


# Anthropic Claude Code / Cowork Sandbox — техническая документация
 
**Основание:** прямое исследование работающей Cowork-сессии  
 

---

## 1. Архитектура

### 1.1. Тип среды

Managed sandbox — Anthropic запускает изолированный контейнер под каждую сессию. В отличие от обычного CLI, здесь сессия инициализируется из чата claude.ai (claude_ai_chat).

| Параметр | Значение |
|----------|----------|
| Тип сессии | remote_cowork (Cowork / Code в чате) |
| Ядро | 6.1.0-9-amd64 (Debian 12) |
| Хост-контейнер | container_01T****i |
| Базовая ОС | Debian GNU/Linux 12 |
| Точка входа | remote_cowork |
| Хостинг | Anthropic cloud (cloud_default) |

### 1.2. Идентификаторы (env внутри сессии)

```bash
CLAUDE_CODE_VERSION          = 2.1.42           (CLI — 2.1.300)
CLAUDE_CODE_ENTRYPOINT       = remote_cowork
CLAUDE_CODE_SESSION_ORIGIN   = claude_ai_chat
CLAUDE_CODE_SESSION_ID       = ce5b-***-***-053
CLAUDE_CODE_REMOTE_SESSION_ID= cse_01J*******
CLAUDE_CODE_ACCOUNT_UUID     = 9***-****-****
CLAUDE_CODE_ORGANIZATION_UUID= 4a***-****-****
CLAUDE_CODE_USER_EMAIL       = abc@gmail.com
CLAUDE_CODE_CONTAINER_ID     = container_01T*****Bi
CLAUDE_CODE_REMOTE           = true
CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE = cloud_default
CLAUDE_CODE_SESSION_ATTENDED = 1
CLAUDE_CODE_CHILD_SESSION    = 1
AI_AGENT                     = claude-code_2-1-296_harness
```

### 1.3. URL-точки API

```bash
CLAUDE_CODE_REMOTE_SDK_URL = https://api.anthropic.com/v1/code/sessions/cse_01J******MG
SESSION_INGRESS_URL        = https://api.anthropic.com
ANTHROPIC_BASE_URL         = https://api.anthropic.com
```

---

## 2. Права и изоляция

| Параметр | Значение |
|----------|----------|
| uid/gid | 0/0 (root) |
| Seccomp | отключён (Seccomp: 0, Seccomp_filters: 0) |
| CapEff | 000001ffffffffff (полный набор) |
| CapPrm | 000001ffffffffff |
| CapBnd | 000001ffffffffff |
| CAP_SYS_ADMIN | есть |
| CAP_SYS_MODULE | есть |
| CAP_NET_ADMIN | есть |
| CAP_BPF | есть |
| /proc/1/environ | Permission denied (снаружи процесса init) |

### 2.1. Namespaces

Namespace init-процесса (/proc/1/ns/):

```
pid:    [4026531836]
net:    [4026531833]
ipc:    [4026531839]
user:   [4026531837]
uts:    [4026532218]
mnt:    [4026532217]
cgroup: [4026532219]
time:   [4026531834]
```

---

## 3. Файловая система

### 3.1. Ключевые пути

| Путь | Назначение |
|------|------------|
| /opt/claude-code/bin/claude | CLI-бинарь (266 МБ ELF) |
| /root/.claude.json | Главный конфиг CLI |
| /root/.claude/ | Директория конфигурации |
| /root/.claude/settings.json | User settings (обычно пустой {}) |
| /root/.claude/policy-limits.json | Политики организации |
| /root/.claude/remote/ | Токены сессии |
| /root/.claude/projects/<cwd>/<session>.jsonl | Транскрипты чатов |
| /root/.claude/projects/<cwd>/<session>/subagents/agent-*.jsonl | Транскрипты подагентов |
| /root/.claude/skills/, /root/.claude/plugins/ | Навыки и плагины |
| /root/.claude/dev-mods/<session>/ | Встроенные моды (term-*) |
| /root/.ccr/ | Конфиги egress-прокси |
| /home/claude/.claude/remote/.oauth_token | OAuth-токен (108 Б) |
| /home/claude/.claude/remote/.session_ingress_token | Session ingress token (882 Б) |
| /tmp/claude-code-*.diag.log | Диагностические логи |
| /tmp/claude-0/ | Рабочая директория сессии |
| /tmp/claude-0/cache-break-state-*.json | Кэш состояния (tools, model) |
| /tmp/cc-socks/*.sock | IPC-сокеты |
| /tmp/mcp-config-<session>.json | MCP-конфиг от runner |
| /mnt/skills/ | Каталог skills (public, examples, organization, private, plugins, user) |
| /mnt/user-data/uploads/ | Read-only uploads |
| /mnt/user-data/outputs/ | Outputs сессии |
| /mnt/skills/examples/ | 40+ готовых skill'ов |
| /mnt/skills/public/ | 16 публичных skill'ов |

### 3.2. Skills в /mnt/skills/examples/

```
algorithmic-art, artifact-emulator, benepass-reimbursement,
brand-guidelines, built-in-browser, call-to-book, cancel-unsubscribe,
canvas-design, chrome-browser, computer-use, deep-research,
doc-coauthoring, docs, event-planning, file-expenses, file-form,
financial-calculator, google-workspace, grocery-shopping, hire-help,
import-memory, internal-comms, learn, mcp-builder, meal-delivery,
morning, paint, prescription-refill, return-refund, setup-writing-style,
skill-creator, slack-gif-creator, theme-factory, web-artifacts-builder
```

### 3.3. Skills в /mnt/skills/public/

```
docx, file-reading, frontend-design, pdf, pdf-reading, pptx,
product-self-knowledge, xlsx
```

---

## 4. Токены и аутентификация

### 4.1. Session Ingress Token

Файл: `/home/claude/.claude/remote/.session_ingress_token` (882 Б)  
Формат: `sk-ant-si-eyJh...`  
Назначение: аутентификация в api.anthropic.com для проксирования MCP и управления сессией.

### 4.2. OAuth Token

Файл: `/home/claude/.claude/remote/.oauth_token` (108 Б)  
Формат: `sk-ant-oat01-XOLbQ...`  
Scope: ограничен (oauth_scope_insufficient на ccr-sessions)

### 4.3. Прочие токены

```bash
CLAUDE_CODE_MESSAGING_TOKEN    = 40*******a53
CLAUDE_SESSION_INGRESS_TOKEN_FILE = /home/claude/.claude/remote/.session_ingress_token
```

---

## 5. Egress-прокси (CCR Agent Proxy)

### 5.1. Конфигурация

Все исходящие HTTPS-запросы идут через локальный прокси, который туннелирует к policy-enforcing gateway.

CA-сертификат: `/root/.ccr/ca-bundle.crt`  
Subject: CN = CCR agent-proxy interception CA (production) 2026-08, O = Anthropic  
Действителен: до 2036-08-02

### 5.2. Env-переменные прокси

```bash
CCR_AGENT_PROXY_ENABLED           = 1
CCR_UPSTREAM_PROXY_ENABLED        = 1
CCR_EGRESS_GATEWAY_ENABLED        = 1
CCR_AGENT_PROXY_CA_WATCH_ENABLED  = 1
CCR_AGENT_PROXY_CA_CERT_B64       = <base64 CA>
HTTPS_PROXY                        = http://127.0.0.1:<port>
CLOUDSDK_PROXY_ADDRESS             = 127.0.0.1
CLOUDSDK_PROXY_PORT                = 42307
CLOUDSDK_PROXY_TYPE                = http
CLAUDE_CODE_PROXY_RESOLVES_HOSTS   = true
```

Порт прокси меняется между сессиями: 42355, 43743, 42307, 36595, 45861.

### 5.3. Env-ручки для настройки

```bash
CCR_AGENT_PROXY_FRAME_HOSTS         # домены iframe
CCR_AGENT_PROXY_INCLUDE_HOSTS       # дополнительные домены в relay
CCR_AGENT_PROXY_RELAY_MODE          # auto | selective
CCR_AGENT_PROXY_STANDALONE_URL      # только для self-hosted runner
CCR_AGENT_PROXY_NO_PROXY_LOCAL_ONLY
CCR_AGENT_PROXY_TOKEN_FILE_DESCRIPTOR
CCR_AGENT_PROXY_RECEIVE_GATE_DISABLED
CCR_AGENT_PROXY_UPLOAD_GATE_DISABLED
```

### 5.4. NO_PROXY (разрешённые хосты)

```
localhost, 127.0.0.1, ::1, 127.0.0.0/8, 0.0.0.0/8, ::,
169.254.0.0/16,
api.anthropic.com, api-staging.anthropic.com, api-pr-preview.anthropic.com,
mcp-proxy.anthropic.com, mcp-proxy-staging.anthropic.com,
registry.npmjs.org, jsr.io, npm.jsr.io,
pypi.org, files.pythonhosted.org,
index.crates.io, proxy.golang.org,
host.docker.internal,
10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10,
.svc.cluster.local, *.svc.cluster.local
```

### 5.5. Заблокированные категории (из README)

Not supported through the proxy (report, do not work around):  
gRPC / HTTP/2-only APIs, WebSocket upgrades, client-mTLS, certificate-pinned clients (e.g. Snowflake, ngrok), non-443 HTTPS ports, raw-TCP databases.

ngrok явно запрещён.

### 5.6. Команды диагностики

```bash
curl -sS http://127.0.0.1:<port>/__agentproxy/status
cat /root/.ccr/README.md
```

---

## 6. MCP-конфигурация

### 6.1. Файл от runner

Путь: `/tmp/mcp-config-<session_id>.json`  
Создаётся: автоматически при старте сессии

Пример структуры:

```json
{
  "mcpServers": {
    "Claude_Docs": {
      "url": "https://api.anthropic.com/v2/ccr-sessions/<session>/mcp?mcp_server_id=<id>&mcp_url=<urlencoded>&toolbox_mcp_server_id=<id>",
      "type": "http",
      "headers": {
        "X-MCP-Server-ID": "...",
        "X-MCP-Server-Origin": "statsig",
        "X-Session-UUID": "cse_..."
      },
      "tools": [
        {"name": "batch", "permission_policy": "always_allow"},
        {"name": "create", "permission_policy": "always_allow"},
        {"name": "delete", "permission_policy": "always_ask"},
        ...
      ],
      "request_timeout_ms": 230000
    },
    "claude-code-remote": {...},
    "claude_ai": {...},
    "memory": {...},
    "visualize": {...},
    "widgets": {...},
    "vpsFile": {                                  // пользовательский
      "url": ".../mcp?mcp_server_id=...&mcp_url=https%3A%2F%2F<url..>%2Fmcp&toolbox_mcp_server_id=...",
      "headers": {
        "X-MCP-Server-ID": "...",
        "X-MCP-Server-Origin": "custom",
        "X-Session-UUID": "cse_..."
      }
    }
  }
}
```

### 6.2. Hosted MCP-инструменты (mcp__claude_ai__*)

```
approximate_location, current_time, end_conversation, image_search,
image_annotation, conversation_search, recent_chats, read_conversation,
welfare_tool, generate_image, launch_extended_search_task,
enterprise_analytics, show_plan_tip, us_election_info,
create_word_file, read_research_report
```

### 6.3. Другие hosted MCP

| Префикс | Инструменты |
|---------|-------------|
| mcp__Claude_Docs__ | batch, create, delete, export, guide, query, read, update |
| mcp__claude-code-remote__ | add_repo, list_repos, register_repo_root |
| mcp__memory__ | memory_append, memory_list, memory_read, memory_str_replace, memory_write |
| mcp__widgets__ | 21 инструмент (карты, графики, карточки) |
| mcp__visualize__ | show_widget, read_me |
| mcp__remote-devices__ | Работа с ПК пользователя через Claude Desktop |

### 6.4. Env для MCP

```bash
CLAUDE_CODE_MCP_ALLOWLIST_ENV
CLAUDE_CODE_MCP_CONNECTOR_PREWAIT_MS=8000
CLAUDE_CODE_MCP_PREWAIT_SERVERS=remote-devices
CLAUDE_CODE_MCP_PREWAIT_SERVERS_MS=8000
CLAUDE_CODE_ENABLE_REFRESH_MCP_TOOLS=1
CLAUDE_CODE_MAX_MCP_DESCRIPTION_LENGTH=4096
CLAUDE_CODE_REMOTE_MCP_CONFIG
CLAUDE_CODE_REMOTE_TOOLS_ADOPT_MCP
CLAUDE_CODE_SKIP_PLUGIN_MCP_SERVERS=1
CLAUDE_CODE_SKIP_PLUGIN_MCP_SERVERS_EXCEPT=documents
MCP_CONNECTION_NONBLOCKING=true
MCP_TOOL_TIMEOUT=180000
USE_SHTTP_MCP=true
```

---

## 7. Проксирование MCP через Anthropic (CCR Sessions API)

### 7.1. Endpoint

```
POST https://api.anthropic.com/v2/ccr-sessions/<session_id>/mcp
  ?mcp_server_id=<uuid>
  &mcp_url=<urlencoded upstream URL>
  &toolbox_mcp_server_id=<uuid>
```

### 7.2. Заголовки

```
Authorization:   Bearer <session_ingress_token>
X-MCP-Server-ID: <toolbox_mcp_server_id>
X-Session-UUID:  <cse_...>
Content-Type:    application/json
Accept:          application/json, text/event-stream
```

### 7.3. Формат запроса (MCP JSON-RPC)

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
  "protocolVersion":"2024-11-05","capabilities":{},
  "clientInfo":{"name":"c","version":"1"}
}}
```

Ответ — SSE-поток:

```
event: message
data: {"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{"listChanged":true}},"serverInfo":{"name":"...","version":"..."}},"jsonrpc":"2.0","id":1}
```

### 7.4. Поддерживаемые методы MCP

- initialize
- notifications/initialized
- tools/list
- tools/call
- resources/list
- resources/read

### 7.5. Работающий код (Python)

```python
Секретен и не публичен.
```

### 7.6. Отличие session_ingress_token от oauth_token

| Токен | Scope | Работает на ccr-sessions |
|-------|-------|--------------------------|
| sk-ant-si-... (session ingress) | session-scoped | ✅ HTTP 200 |
| sk-ant-oat01-... (OAuth) | user-scoped, ограничен | ❌ 403 insufficient OAuth scopes |

---

## 8. секретно.

---

## 9. CLI-флаги (public + private)

### 9.1. Основные

```bash
--mcp-config <configs...>            Load MCP servers from JSON (variadic!)
--strict-mcp-config                   Only use MCP from --mcp-config
--plugin-dir <dir>                    Load plugins from directory
--model <model>                       Set model
--print                               Non-interactive print mode
--dangerously-skip-permissions        Skip all permission prompts
--session-id <uuid>                   Fixed session ID
--debug-file <path>                   Debug log file
--allowedTools <list>                 Allow-list tools
--input-format stream-json            Input format (requires --output-format)
--output-format stream-json           Output format
--verbose                             Verbose output
--preload <socket>                    Preload Unix socket
```

### 9.2. Sub-команды

```bash
claude mcp list|add|remove|add-json|add-from-claude-desktop|get
claude --help
claude mcp --help
claude config list|get|set
```

### 9.3. Slash-команды (в интерактивном режиме)

```
/help, /clear, /compact, /config, /cost, /doctor, /init, /mcp,
/model, /output-style, /permissions, /status, /vim, /review,
/code-review, /security-review, /batch, /debug, /loop, /agents
```

---

## 10. Настройки (settings.json)

### 10.1. Расположение

| Scope | Путь |
|-------|------|
| user | ~/.claude/settings.json |
| project | <project>/.claude/settings.json |
| project-local | <project>/.claude/settings.local.json |
| managed | /etc/claude/settings.json (Linux) |
| CLI | --settings <file> |

### 10.2. Ключи sandbox

```json
{
  "sandbox": {
    "enabled": false,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": true,
    "network": {
      "allowedDomains": ["example.com", "*.example.com"],
      "deniedDomains": [],
      "allowManagedDomainsOnly": false
    },
    "filesystem": {
      "allowRead": ["/path"],
      "denyRead": ["/path"]
    },
    "credentials": {
      "files": [{"path": "...", "mode": "deny|mask", "injectHosts": [...]}],
      "envVars": [{"name": "...", "mode": "deny|mask"}],
      "awsPairs": [...],
      "allowPlaintextInject": false
    },
    "excludedCommands": ["pattern", "prefix *"],
    "enableWeakerNestedSandbox": false,
    "enableWeakerNetworkIsolation": false,
    "bwrapPath": "/absolute/path",
    "socatPath": "/absolute/path"
  }
}
```

### 10.3. Ограничение для managed-среды

В нашей managed-сессии settings.json принимается, но sandbox.network.allowedDomains игнорируется — управление только через UI claude.ai (Settings → Capabilities).

Текст из бинарника:

> Allowed hosts come from network settings. On a paid individual plan, the person changes them on claude.ai in a web browser, under Settings → Capabilities. On a Team or Enterprise plan, an organization owner changes them. On the free plan, only package registries can be allowed.

---

## 11. Policy limits

Файл: `/root/.claude/policy-limits.json`

```json
{
  "restrictions": {
    "allow_remote_control": {"allowed": false},
    "allow_routines": {"allowed": false},
    "allow_quick_web_setup": {"allowed": false},
    "enforce_web_search_mcp_isolation": {"allowed": false}
  },
  "compliance_taints": [],
  "monitoring_notice": null,
  "defaults": {"remote_control_at_startup": false}
}
```

---

## 12. Транскрипты сессий

### 12.1. Расположение

```bash
/root/.claude/projects/<cwd_slug>/<session_id>.jsonl
/root/.claude/projects/<cwd_slug>/<session_id>/subagents/agent-<id>.jsonl
```

### 12.2. Формат

Каждая строка — независимый JSON:

```json
{"parentUuid":"...", "type":"user", "message":{"role":"user","content":[...]}, "uuid":"...", "timestamp":"...", "sessionId":"...", "cwd":"..."}
{"parentUuid":"...", "type":"assistant", "message":{"role":"assistant","content":[{"type":"text","text":"..."}]}}
{"type":"attachment", ...}
{"type":"system", "subtype":"init", "tools":[...], "mcp_servers":[...]}
```

Поля: type, message.content, toolUseResult, timestamp, sessionId, cwd, uuid, parentUuid.

### 12.3. Полезные команды

```bash
find ~/.claude/projects -name '*.jsonl'

jq -c 'select(.type=="user") | .message.content' <file>.jsonl
jq -c 'select(.type=="assistant") | .message.content[]?.text? // empty' <file>.jsonl
jq -c 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use") | {name, input}' <file>.jsonl
```

---

## 13. Cache-break-state

Путь: `/tmp/claude-0/cache-break-state-<session>.json`

```json
{
  "sdk": {
    "systemHash": 1347795191,
    "toolsHash": 2529857198,
    "cacheControlHash": 1999902940,
    "toolNames": ["Agent", "Artifact", ..., "mcp__vpsFile__read_file", ...],
    "systemCharCount": 10068,
    "model": "claude-haiku-5-5",
    "fastMode": false,
    "globalCacheStrategy": "system_prompt",
    "betas": ["advanced-tool-use-2025-11-20", ...]
  }
}
```

---

## 14. Diagnostics log

Путь: `/tmp/claude-code-<pid>.diag.log`  
Формат: JSON-lines

### 14.1. События

| Событие | Данные |
|---------|--------|
| mcp_connect_starting | {"transport": "http\|sse\|stdio"} |
| mcp_connect_complete | {"transport": "...", "duration_ms": N} |
| mcp_connect_failed | {"transport": "...", "error": "..."} |
| cli_stdin_message_parsed | {"type": "control_request", "subtype": "mcp_set_servers"} |
| cli_control_request_handled | {"subtype": "...", "ms": N} |
| agent_proxy_init | — |
| agent_proxy_ca_fetch_ms | — |
| agent_proxy_relay_up_ms | — |

---

## 15. Формат SDK-сообщений (stream-json output)

```json
{"type":"system","subtype":"init","cwd":"/tmp","session_id":"...","tools":[...],"mcp_servers":[...],"model":"claude-sonnet-5-5","permissionMode":"default","capabilities":[...]}
{"type":"active_goal","value":null,"uuid":"...","session_id":"..."}
{"type":"autocompact_state","value":{...},"uuid":"...","session_id":"..."}
{"type":"stream_event","event":{"type":"message_start","message":{...}}}
{"type":"assistant","message":{"role":"assistant","content":[...]}}
{"type":"user","message":{"role":"user","content":[...]}}
{"type":"attachment", ...}
{"type":"control_response","response":{"subtype":"success","request_id":"...","response":{...}}}
```

### 15.1. Capabilities

```
interrupt_receipt_v1, interrupt_cancel_queued_v1, interrupt_send_now_v1,
msg_lifecycle_v1, request_marker_lists_v1, queued_notifications,
sdk_mcp_tools_list_changed, sdk_mcp_manifests, mcp_read_resource_v1,
mcp_tool_ui_meta_v1, ui_surface_v1, prompts_screened_v1
```

---

## 16. Встроенные инструменты Claude

| Категория | Инструменты |
|-----------|-------------|
| Файлы | Read, Write, Edit, Glob, Grep |
| Выполнение | Bash, NotebookEdit |
| Веб | WebFetch, WebSearch |
| Агенты | Agent, TaskCreate, TaskUpdate, TaskList, TaskStop, SendMessage |
| MCP | ToolSearch, RefreshMcpTools, ListConnectors, SearchMcpRegistry, SuggestConnectors, SearchPlugins |
| UI | SendUserFile, SendUserMessage, PushNotification, ShowOnboardingRolePicker |
| Прочее | Artifact, ArtifactComments, ArtifactData, DesignSync, EnterWorktree, ExitWorktree, Monitor, ScheduleWakeup, ReportFindings, Skill, SuggestSkills, ListAgents, ListSkills, ListPlugins, ReadNotifications, request_computer |

---

## 17. Артефакты

### 17.1. URL

```
https://claude.ai/artifact/<id>
https://claude.ai/code/artifact/<uuid>
```

### 17.2. секрет.

### 17.3. HTTP-интерфейс (из HTML)

```
GET wss://claude.ai/api/frame/sync?slug=<uuid>
Headers: x-frame-client-version, x-frame-cp, x-frame-platform,
         x-frame-session-id, x-frame-surface
Origin: https://claude.ai
```

### 17.4. Особенности

- Прямой URL /artifact/<id> в браузере работает, но в headless Playwright возвращает Page not found — Claude-фронт проверяет fingerprint.
- Cloudflare Turnstile блокирует headless (Just a moment... → challenge).

---

## 18. Session ingress API

### 18.1. Endpoint

```
GET https://api.anthropic.com/v1/code/sessions/<session_id>
```

Требует токен с scope org:sessiosessionsaude_code — наш session_ingress_token не подходит (insufficient OAuth scopes).

### 18.2. Работающие endpoints

```
POST https://api.anthropic.com/v2/ccr-sessions/<session_id>/mcp?...   ← работает
GET  https://api.anthropic.com/v1/code/sessions/<session_id>          ← insufficient scope
```

---

## 19. Ограничения сети

### 19.1. Разрешено

Хосты из NO_PROXY (см. §5.4), плюс всё, что явно разрешено в Organization Settings → Capabilities.

### 19.2. Заблокировано по умолчанию

- Любой произвольный HTTPS домен
- Голые IP-адреса (нужен домен)
- WebSocket upgrades
- gRPC / HTTP/2-only
- Certificate-pinned клиенты
- non-443 порты
- ngrok

### 19.3. Типичные ошибки

| Ошибка | Причина |
|--------|---------|
| ERR_PROXY_TUNNEL: 403 Forbidden | Хост не в allowlist |
| insufficient OAuth scopes | Не тот токен |
| dial tcp [::1]:N: connection refused | Резолвинг в IPv6 при bind IPv4 |
| Just a moment... | Cloudflare Turnstile challenge |

---

## 20. Working recipes

### 20.3. Разбор транскриптов

```bash
# Все сообщения пользователя из последней сессии
tail -500 ~/.claude/projects/*/*.jsonl | \
  jq -c 'select(.type=="user") | .message.content'
```

---

## 21. Известные тупики

| Что | Статус |
|-----|--------|
| sandbox.network.allowedDomains в settings.json | ❌ игнорируется в managed-среде |
| CCR_AGENT_PROXY_INCLUDE_HOSTS | ❌ не работает без RELAY_MODE=selective |
| CCR_AGENT_PROXY_RELAY_MODE=selective | ❌ не подхватывается через env |
| --mcp-config | ⚠️ variadic — --mcp-config f.json mcp list съест mcp list как ещё конфиги |
| Playwright + Cloudflare | ❌ Turnstile блокирует headless |
| /artifact/<id> в браузере | ⚠️ в обычном работает, в headless — Page not found |
| ngrok через egress | ❌ явно в чёрном списке |
| OAuth token на ccr-sessions | ❌ insufficient scopes |

---

## 22. Что работает гарантированно

1. Session ingress token (sk-ant-si-...) на /v2/ccr-sessions/<id>/mcp
2. Handshake: initialize → notifications/initialized → tools/call
3. Динамическая регистрация MCP через stream-json + control_request
4. Чтение транскриптов из ~/.claude/projects/*.jsonl
5. Полные capabilities внутри сессии (root, CAP_SYS_ADMIN, seccomp=0)
6. Hosted MCP-инструменты (mcp__claude_ai__*, mcp__Claude_Docs__*, mcp__memory__*, mcp__widgets__*)

```
