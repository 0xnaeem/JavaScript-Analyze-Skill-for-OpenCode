```markdown
# 🕵️ JS-API-Recon

**An OpenCode AI Skill for Semantic JavaScript API Endpoint Reconstruction**

[![OpenCode](https://img.shields.io/badge/OpenCode-Skill-blueviolet)](https://opencode.ai)
[![Bug Bounty](https://img.shields.io/badge/Bug%20Bounty-AppSec-red)](https://hackerone.com)
[![JavaScript](https://img.shields.io/badge/JavaScript-Analysis-yellow)](https://developer.mozilla.org/en-US/docs/Web/JavaScript)

---

## 📖 Overview

**JS-API-Recon** is a specialized AI skill designed for **OpenCode** that truly *understands* JavaScript code—not just regex matching, but semantic tracing of control flow, class inheritance, and string composition. It rebuilds complete API endpoint URLs from fragmented code scattered across request builders, constructors, and path prefixes.

**Inspired by 11 months of real-world bug bounty hunting**, this skill solves the #1 pain point: manually tracing JavaScript to find hidden API endpoints.

---

## 🎯 The Problem It Solves

Modern JavaScript applications (SPAs, React, Vue, Angular) split API routes across multiple files and classes:

```javascript
// File A: Base class
class BaseApi {
  constructor() { this.base = "/api"; }
  request(method, path) { return fetch(this.base + path); }
}

// File B: Child class
class UserApi extends BaseApi {
  constructor() { super(); this.base += "/user"; }
  getProfile() { return this.request("GET", "/profile"); }
}

// File C: Child's child
class AdminApi extends UserApi {
  constructor() { super(); this.base += "/admin"; }
  listUsers() { return this.request("GET", "/list"); }
}
```

A human or regex tool sees only "/profile" and "/list". But the real endpoints are:

· GET /api/user/profile
· GET /api/user/admin/list

JS-API-Recon traces these chains automatically.

---

⚡ Key Features

· ✅ Tracks Request Dispatchers – Finds the core fetch/axios call.
· ✅ Resolves Path Prefixes – Follows this.baseURL, static paths, and config objects.
· ✅ Maps Inheritance Chains – Propagates prefixes from parent to child classes via super().
· ✅ Evaluates Template Literals – Flags dynamic parameters (${userId} → :userId).
· ✅ Cross-References Imports – Follows lazy-loaded chunks and module imports.
· ✅ Structured Output – Generates a clean markdown table with methods, full paths, sources, and parameters.

---

🚀 Installation

1. Open OpenCode in your workspace.
2. Navigate to Skills → Create New.
3. Copy the content from js-api-recon.md (or the skill definition).
4. Paste it, name it JS API Recon, and save.
5. Activate the skill.

---

🛠️ Usage

Trigger the skill with any of these natural language prompts:

· "Analyze this JS for endpoints"
· "Reconstruct API routes from this code"
· "Find all API calls in this file"
· "Map out the API surface from this JS bundle"

Example

Input snippet (React/Vue):

```javascript
class ApiClient {
  constructor(version) {
    this.version = version || "v1";
    this.root = `/api/${this.version}`;
  }
  get(path) { return axios.get(this.root + path); }
}

class AuthService extends ApiClient {
  constructor() { super("v2"); }
  login() { return this.get("/auth/login"); }
  logout() { return this.get("/auth/logout"); }
}
```

Output:

HTTP Method Full Endpoint Source Function Dynamic Params
GET /api/v2/auth/login login() None
GET /api/v2/auth/logout logout() None

---

📊 Output Report Structure

The skill generates a comprehensive markdown report with:

1. Discovered Base Paths – All prefixes found with file references.
2. Endpoint Table – Method, Full URL, Source, Params.
3. Unresolved Paths – Runtime-dependent routes with best guesses.
4. Attack Surface Priority – Ranks Admin/Internal endpoints as High, user CRUD as Medium, public assets as Low.

---

💡 Why This Matters for Bug Bounty

· Reduce manual analysis time by ~90% – Focus on exploiting vulnerabilities, not tracing code.
· Find hidden admin panels – Paths like /internal/health or /dev/debug often hide in JS.
· Uncover prototype pollution sinks – By understanding how paths are built.
· Automate recon – Pair with Burp/ZAP for rapid API mapping.

---

🤝 Contributing

Found a bug or have an improvement?
Feel free to open an issue or submit a PR. We welcome enhancements for:

· Better evaluation of complex template literals.
· Support for Webpack/Rollup chunk parsing.
· Integration with Swagger/OpenAPI generation.

---

📄 License

MIT – Use it freely for hacking, research, or commercial work.

---

🙏 Credits

· Inspired by Abdalkreem R. A. (@abdalkreem-dagga) and his 11-month bug bounty journey.
· Built for the OpenCode AI ecosystem.

---

📬 Contact

For questions or suggestions, reach out via GitHub Issues.

---

Happy Hacking! 🚀

```
