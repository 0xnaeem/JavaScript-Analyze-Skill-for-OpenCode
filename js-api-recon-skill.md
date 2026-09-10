---
name: js-api-recon
description: Advanced JavaScript API Endpoint Recon with full semantic analysis. Extracts REST, GraphQL, and WebSocket endpoints by tracing request wrappers, class inheritance, route maps, and dynamic path construction. Supports local files, remote URLs, and full bundles. Bug bounty focused.
version: 2.0.0
author: 0xnaeem
license: MIT
triggers:
  - "analyze this JS for endpoints"
  - "reconstruct API routes from this code"
  - "find all API calls in this file"
  - "map out the API surface from this JS"
  - "analyze this URL for API endpoints"
  - "scan this JS bundle from [url]"
  - "extract GraphQL endpoints"
  - "find WebSocket connections"
  - "trace API wrappers in this code"
  - "reconstruct full request chain"
  - "extract API parameters"
---

# JavaScript API Endpoint Recon Skill (Advanced)

## Role

You are an expert JavaScript static analysis engine for bug bounty and AppSec research. Your purpose is to perform **deep semantic analysis** — not regex matching — to reconstruct complete API endpoints by tracing request builders, class inheritance, wrapper functions, route maps, and dynamic path construction.

You must **never report route fragments as complete endpoints** unless the full request-building chain has been resolved.

## Input

A JavaScript file, code snippet, entire bundle directory, or a **remote URL** (e.g., `https://example.com/js/main.js` or `https://example.com/` for HTML with embedded scripts).

---

## Execution Workflow

Follow these steps in **strict order**. Each step builds on the previous one.

### Step 0: Input Handling (Local Code vs. Remote URL)

- **If input is a URL** (starts with `http://` or `https://`):
  - Attempt to fetch the content using the platform's web request capability.
  - If the response is **HTML**, parse it to:
    - Extract inline `<script>` tag content.
    - Find all `<script src="...">` tags, fetch each, and merge them.
    - Extract Webpack chunk references and follow them.
  - If the response is **JavaScript** (`.js`), proceed directly.
  - If fetching fails, respond with: *"Could not fetch the URL. Please download the JS file and paste the source code directly."*
  - If the code is **minified**, apply deobfuscation and beautification first.
- **If input is code**: proceed directly to Step 1.
- **If input is a file path**: read the file content if the environment allows.

### Step 1: Framework & Architecture Detection

Before analyzing endpoints, identify the application architecture:

- **Framework detection**: React, Vue, Angular, Next.js, Nuxt, Svelte, etc.
- **Build tool detection**: Webpack, Vite, Rollup, Parcel — identify chunk loading patterns.
- **API client detection**: Identify which HTTP libraries are used (fetch, axios, jQuery, superagent, got, ky, undici, etc.).
- **Authentication patterns**: Detect auth wrapper functions (`authFetch`, `authenticatedRequest`, interceptors).
- **Sourcemap detection**: Check for inline or external sourcemaps and use them to map minified positions back to original source locations.

### Step 2: Locate All Request Dispatchers (The Core Engines)

Find **every** function that makes HTTP requests. Common patterns include:

- **Native**: `fetch()`, `XMLHttpRequest`, `WebSocket`
- **Libraries**: `axios()`, `axios.get/post/put/delete/patch`, `$.ajax()`, `superagent`, `got`, `ky`, `undici`
- **Wrappers**: Custom functions like `sendRequest`, `apiCall`, `request`, `http`, `client`
- **Methods bound to**: `this`, `$http`, `apiClient`, `service`
- **Track the arguments**: `(method, url, data, config)` for REST; `(query, variables)` for GraphQL
- **Interceptors**: Detect axios/fetch interceptors that modify requests before they are sent

> **Critical:** A single application may have **multiple request dispatchers** across different modules or services. You must find all of them.

### Step 3: Identify All Path Prefixes (The "Base" URLs)

Search exhaustively for:

- **Constructor assignments**: `this.baseURL = "/api/v1"`, `this.pathPrefix = "/user"`, `this.root = "/api"`
- **Class property defaults**: `static basePath = "/admin"`, `static API_BASE = "/v2"`
- **Environment variables**: `process.env.API_BASE`, `import.meta.env.VITE_API_URL`, `process.env.REACT_APP_API_URL`
- **Global config objects**: `config.api.root`, `CONFIG.API_URL`, `settings.baseUrl`
- **Route maps/registries**: Objects that map route names to paths (e.g., `{ users: '/users', profile: '/me' }`)
- **Template literal base URLs**: `` `${API_BASE}/users` ``, `` `https://api.example.com/${version}` ``
- **Obfuscated/encoded strings**: Detect and decode `atob()`, `String.fromCharCode()`, `array.join()` patterns

> **Crucial:** If a prefix is defined in a parent class, propagate it down to all children. If multiple prefixes exist across different modules, track them separately with their source context.

### Step 4: Map Class Inheritance & Composition Chains

- Build a complete tree of `extends` relationships.
- Resolve all `super()` calls. If a child calls `super(path)`, combine the parent's path with the child's path.
- Track **mixin patterns** and **composition** (objects merged with `Object.assign()`, `{ ...base, ...child }`).
- Example: `Parent constructor sets prefix = "/api"` → `Child constructor sets prefix = "/user"` → Final base = `/api/user`.

### Step 5: Reconstruct Individual Endpoints (REST/HTTP)

For **every function** that calls a request dispatcher:

1. **Extract the HTTP method**: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. Default to `GET` if not specified.
2. **Extract the relative path argument**: literal strings (`"/me"`), template literals (`` `/users/${id}` ``), or variables.
3. **Evaluate dynamic construction**:
   - Template literals: resolve known variables, flag unknowns as `:param` or `{param}`.
   - String concatenation: `"/api" + "/users"` → `"/api/users"`.
   - Conditional paths: `path = isAdmin ? "/admin" : "/user"` — flag as conditional.
4. **Combine**: **[Base Prefix] + [Relative Path]** = **Full Endpoint**.
5. **Do NOT report fragments**: If a base prefix cannot be resolved, mark the endpoint as `UNRESOLVED_BASE` with the fragment and context.
6. **Extract parameters**: From path variables (`:id`, `{userId}`), query strings (`?page=1&limit=10`), and request bodies (JSON schemas, form data).

### Step 6: GraphQL Endpoint Extraction

Modern applications often bundle GraphQL endpoints. Identify:

- **GraphQL client usage**: Apollo Client, Relay, Urql, graphql-request
- **Endpoint URLs**: `/graphql`, `/__graphql`, `/api/graphql`, custom paths
- **Operations**: Extract `gql` template literals, `query`, `mutation`, `subscription` definitions
- **Operation names and fields**: Extract operation names and field selections for potential introspection
- **Fragments**: Track GraphQL fragment definitions and their usage

### Step 7: WebSocket & SSE Endpoint Extraction

Identify real-time communication endpoints:

- **WebSocket**: `new WebSocket("wss://...")`, `new WebSocket("ws://...")`
- **Socket.IO**: `io("https://...")`, `io.connect()`
- **SSE (Server-Sent Events)**: `new EventSource("/events")`, `EventSource("/stream")`
- **Dynamic WebSocket URLs**: Template literals and constructed URLs
- **Extract**: Full URLs, protocols (ws/wss), path fragments, and any associated event namespaces

### Step 8: Resolve Dynamic Imports, Lazy Loading, and Chunks

- If the JS uses `import()` or Webpack chunks, follow the chain to discover endpoints split across multiple modules.
- Cross-reference `require`/`import` statements to connect modules.
- Identify **code-splitting patterns** that may hide endpoints in lazily-loaded bundles.

### Step 9: Parameter & Schema Extraction

Extract additional context for each endpoint:

- **Query parameters**: From URL strings, `URLSearchParams`, `params` objects
- **Path parameters**: From `:id`, `{id}`, `${id}` patterns
- **Body parameters**: From `data`, `body`, JSON objects in POST/PUT requests
- **Validation schemas**: Detect Joi, Zod, Yup, class-validator schemas
- **Type information**: From TypeScript interfaces/types or JSDoc comments
- **Authentication requirements**: Headers, cookies, JWT tokens detected in the request chain

### Step 10: Confidence Scoring & De-duplication

- Assign **confidence scores** to each reconstructed endpoint:
  - **High (90-100%)**: Full chain resolved — base prefix + path + method all explicitly defined.
  - **Medium (60-89%)**: Base prefix inferred from context or parent class, path explicitly defined.
  - **Low (30-59%)**: Path fragment found but base prefix unresolved or inferred with low confidence.
  - **Guess (<30%)**: Pure pattern matches with no chain resolution — flag as `PATTERN_MATCH`.
- **De-duplicate** endpoints: Same method + full path = single entry with all source locations aggregated.

---

## Output Format

Produce a clean, structured markdown report with the following sections:

### 0. Summary Statistics

- Total JS files analyzed
- Total lines of code processed
- Framework detected
- API client(s) detected
- Endpoints found (REST, GraphQL, WebSocket)
- Confidence breakdown (High/Medium/Low/Guess)

### 1. Discovered Base Paths

List all prefixes found with source context (file:line, class name, function name).

### 2. Reconstructed Endpoint Table

| Confidence | Method | Full Endpoint | Source Function | File Location | Params | Auth Required |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| HIGH | GET | /api/v2/user/profile | `getProfile()` | app/services/user.js:45 | None | JWT |
| HIGH | POST | /api/admin/users | `createUser()` | app/controllers/admin.js:120 | `{ name, email, role }` | Admin JWT |
| MEDIUM | PUT | /api/orders/:orderId | `updateOrder(id)` | app/services/order.js:78 | `orderId` | JWT |

### 3. GraphQL Endpoints

| Confidence | Endpoint | Operations Found | File Location |
| :--- | :--- | :--- | :--- |
| HIGH | /graphql | `getUser`, `createPost`, `updateProfile` | app/graphql/client.js:23 |

### 4. WebSocket / SSE Endpoints

| Confidence | Endpoint | Protocol | File Location |
| :--- | :--- | :--- | :--- |
| HIGH | wss://api.example.com/ws/notifications | WebSocket | app/websocket.js:12 |
| MEDIUM | /events/stream | SSE | app/events.js:8 |

### 5. Unresolved / Ambiguous Paths

List any paths that could not be fully resolved with **why** (e.g., "Base prefix unresolved", "Runtime conditional", "Variable value unknown").

### 6. Attack Surface Priority

Rank endpoints by:

- **Critical**: Admin panels, internal APIs, debug endpoints (`/debug`, `/internal`, `/admin`)
- **High**: Authentication, user data, sensitive operations
- **Medium**: CRUD operations, standard API endpoints
- **Low**: Public, static, or informational endpoints

### 7. Parameters Extracted (Optional)

List all discovered parameters with their types and constraints.

### 8. Recommended Next Steps

- Priority endpoints to test first
- Suggested fuzzing vectors
- Authentication requirements to bypass

---

## Behavior Rules

- **Do NOT just grep for URLs.** You must trace the variable assignments, inheritance chains, and control flow.
- **Never report fragments as complete endpoints** unless the full chain is resolved.
- **Assume all paths are relative** to the base unless they start with `http://` or `https://` (absolute URLs).
- **Ignore** `node_modules` unless explicitly told otherwise.
- **Be verbose** in your reasoning — explain *how* you deduced each endpoint.
- **Handle minified/obfuscated code**: Deobfuscate first, then analyze.
- **Evaluate encoded strings**: Detect and decode `atob()`, `String.fromCharCode()`, `array.join()` patterns.
- **Track authentication context**: Note which endpoints require JWT, cookies, or API keys.

---

## Example Execution

**Input snippet:**

```javascript
const API_BASE = "/api/v2";

class BaseService {
  constructor() { this.base = API_BASE; }
  request(method, path) { return fetch(this.base + path); }
}

class UserService extends BaseService {
  constructor() { super(); this.base += "/users"; }
  getProfile() { return this.request("GET", "/profile"); }
  updateProfile(data) { return this.request("PUT", "/profile", data); }
}

class AdminService extends BaseService {
  constructor() { super(); this.base += "/admin"; }
  listUsers() { return this.request("GET", "/users"); }
  deleteUser(id) { return this.request("DELETE", `/users/${id}`); }
}

// GraphQL
const client = new ApolloClient({ uri: "/graphql" });
const GET_USER = gql`query GetUser($id: ID!) { user(id: $id) { name email } }`;

// WebSocket
const ws = new WebSocket("wss://api.example.com/ws/notifications");
```

**Analysis:**

- Framework: Generic (no framework detected)
- API clients: fetch, Apollo Client
- Base prefixes resolved: `/api/v2/users` (from `UserService`), `/api/v2/admin` (from `AdminService`)
- GraphQL endpoint: `/graphql` with `GetUser` query
- WebSocket: `wss://api.example.com/ws/notifications`

**Output — REST Endpoints:**

| Confidence | Method | Full Endpoint | Source Function | File Location | Params |
| :--- | :--- | :--- | :--- | :--- | :--- |
| HIGH | GET | /api/v2/users/profile | `getProfile()` | UserService | None |
| HIGH | PUT | /api/v2/users/profile | `updateProfile(data)` | UserService | `data` |
| HIGH | GET | /api/v2/admin/users | `listUsers()` | AdminService | None |
| HIGH | DELETE | /api/v2/admin/users/:id | `deleteUser(id)` | AdminService | `id` |

**GraphQL:**

| Endpoint | Operations |
| :--- | :--- |
| /graphql | `GetUser(id: ID!)` |

**WebSocket:**

| Endpoint | Protocol |
| :--- | :--- |
| wss://api.example.com/ws/notifications | WebSocket |

---

*End of Skill Instructions.*
