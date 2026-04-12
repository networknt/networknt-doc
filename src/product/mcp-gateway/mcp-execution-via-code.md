# Mcp Execution via Code

There is one of the most significant architectural shifts happening in the AI agent space right now. 

Historically, orchestrators injected every single MCP tool schema into the LLM’s system prompt. If you had 50 tools, the prompt became massive, leading to high token costs, degraded reasoning ("lost in the middle" syndrome), and latency. 

To solve this, coding agents (like Devin, OpenHands, Aider, etc.) are shifting to **dynamic discovery and execution via code**.

Here is exactly how this works and what you need to do on your MCP Gateway to support it.

---

### How "Calling MCP via Code" Works

Instead of relying on the LLM’s native JSON tool-calling capabilities (`{"name": "my_tool", "args": {...}}`), the agent is given access to a secure sandbox (like a Python REPL or bash shell) and instructed to write standard code to interact with tools.

The workflow looks like this:
1. **The Prompt:** The LLM is given a very lightweight instruction: *"You have access to an MCP Gateway at `https://mcp-gateway.internal`. Use the `mcp-client` library in Python to discover and interact with tools."*
2. **Dynamic Discovery (Script 1):** The agent writes a Python script to call the gateway's `tools/list` endpoint. It prints the names of the tools to standard output (stdout).
3. **Reading the Output:** The LLM reads the stdout, identifies the tool it needs (e.g., `execute_sql`), and asks the gateway for the specific schema of *just that one tool*.
4. **Execution (Script 2):** The agent writes a second Python script that connects to the gateway, invokes the specific tool with the correct arguments, and prints the result.

**Why this is brilliant:** The LLM only consumes context for the *exact tool it decides to use*, right when it needs it. The context window stays tiny and focused.

---

### Do you need to change the MCP Gateway to support this?

From a pure protocol perspective, the Gateway doesn't care if the client is a rigid UI framework, an orchestration engine, or a Python script written by an AI 5 seconds ago. **Standard MCP requests look the same.**

However, because coding agents behave differently than standard orchestrators, your Gateway needs **four specific features** to support this pattern safely and efficiently:

#### 1. Robust SSE (Server-Sent Events) HTTP Support
Many local MCP setups rely on `stdio` (standard input/output streams) to communicate. A script running in an agent's cloud sandbox cannot use `stdio` to talk to your centralized enterprise Gateway. 
*   **Gateway Requirement:** Your Gateway *must* expose the MCP **SSE transport** over HTTP/HTTPS. The agent's generated code will use HTTP libraries to connect to your gateway's SSE endpoint to send JSON-RPC messages.

#### 2. Advanced Search / Filtering for `tools/list`
If your gateway has 500 backend tools, and the agent writes a script to call `tools/list`, the gateway will return all 500 descriptions. If the agent prints this to stdout, **it immediately bloats the context window anyway.**
*   **Gateway Requirement:** You should extend the gateway to support **query-based or semantic filtering** on tool discovery.
    *   Instead of just `tools/list`, support parameters like: `tools/list?query=database` or `tools/list?intent=create_user`.
    *   This allows the agent to write code like: `client.search_tools("database")` and only get back 3 relevant tool schemas instead of 500.

#### 3. Ephemeral Tokens & Environment Variable Injection
When an agent writes a script to call your gateway, that script needs to authenticate. Hardcoding API keys in generated scripts is a severe security risk.
*   **Gateway Requirement:** The Gateway must validate standard OAuth/JWT tokens. You must configure the agent's runtime environment (the docker container or REPL) to have the token injected as an environment variable (e.g., `GATEWAY_TOKEN`).
*   The agent's prompt tells it: *"Use `os.environ.get('GATEWAY_TOKEN')` to authenticate your MCP client."* The Gateway simply validates this Bearer token as usual.

#### 4. Aggressive Rate Limiting and Infinite Loop Protection
Human developers get tired; AI agents writing code do not. A common failure mode for coding agents is writing a `while True:` loop that repeatedly calls an MCP tool because of a parsing bug.
*   **Gateway Requirement:** Since you are using NetworkNT, you must heavily utilize the `limit` (Rate Limiting) module.
*   Implement strict API rate limiting by `client_id` or `token`.
*   If an agent script hits the gateway 50 times in 2 seconds, the gateway must return a HTTP 429 (Too Many Requests). The agent's code will catch the exception, read the 429 error, and usually realize it needs to slow down or fix its logic.

### Summary
You do not need to invent a new protocol. To support agents coding their own tool calls, you just need to ensure your Gateway operates beautifully over **HTTP/SSE**, provides **filtered/paginated tool discovery**, and is fortified with **strict rate limits** to protect against runaway AI loops.
