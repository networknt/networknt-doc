# MCP Gateway

## Features

* Unified Access Point: Aggregates multiple backend MCP servers into a single endpoint, allowing AI agents to connect to one URL instead of managing separate connections for every tool.
* Authentication & Authorization: Centralizes security by enforcing OAuth 2.1, SAML, or OIDC flows. It manages identity propagation, ensuring an agent only has the permissions of the specific user it is acting for.
* Granular Access Control (RBAC/ABAC): Restricts which teams, users, or agents can see and use specific tools. For example, a marketing agent might see social media tools but not database administration tools.
* Observability & Audit Logging: Records every tool call, parameter, and response. These logs are essential for security auditing, compliance (like SOC 2 or HIPAA), and debugging agent behavior.
* Privacy & Data Masking: Automatically detects and redacts PII (Personally Identifiable Information), secrets, or sensitive data before it reaches the AI model or the backend server.
* Protocol & Transport Translation: Converts between different MCP communication methods, such as bridging local stdio servers (running in containers) to remote HTTP/SSE clients.
* Intelligent Routing & Load Balancing: Directs requests to the appropriate server based on the tool name or semantic intent. It also handles retries, circuit breaking, and failovers to keep the system reliable.
* Session Management: Maintains "sticky" sessions so that multi-step agent workflows stay connected to the same server context, preventing state loss.
* Tool Filtering & Throttling: Limits the number of tools exposed to an agent to prevent "context bloat" and applies rate limits to prevent agents from overloading backend systems.