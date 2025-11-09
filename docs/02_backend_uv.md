# Part 2: Application Backend Setup (uv Method)

## Pre-requisites
- Complete [Part 0](../SETUP.md)
- Complete [Part 1: MCP Setup (uv)](01_mcp_uv.md)
- MCP server running on `http://localhost:8000/mcp`
- uv installed

## Summary
In this part, you will configure and run the backend service for the Microsoft AI Agentic Workshop using uv. The backend includes full support for Microsoft's Agent Framework with advanced multi-agent capabilities.

## Microsoft Agent Framework Options

**Single Agent (`agents.agent_framework.single_agent`):**
- Basic ChatAgent with MCP tools
- Token-by-token streaming via WebSocket
- Tool call visibility in React UI
- Session state persistence across requests

**Magentic Multi-Agent (`agents.agent_framework.multi_agent.magentic_group`):**
- Intelligent orchestrator coordinates specialist agents (CRM/Billing, Product/Promotions, Security)
- Real-time streaming of orchestrator planning and agent responses
- Custom progress ledger for human-in-the-loop support
- Checkpointing for resumable workflows
- React UI shows full internal process: task ledger, instructions, agent tool calls

**Handoff Multi-Agent (`agents.agent_framework.multi_agent.handoff_multi_domain_agent`):**
- Direct agent-to-user communication with intelligent domain routing
- Configurable context transfer between specialists (preserves customer info, history)
- Smart intent classification for seamless handoffs
- Cost-efficient (33% fewer LLM calls vs orchestrator patterns)

## Steps

[1. Configure Agent Framework](#1-configure-agent-framework)  
[2. Understanding Backend Code Structure](#2-understanding-backend-code-structure)  
[3. Run Backend Service](#3-run-backend-service)  
[4. Choose Frontend Experience](#4-choose-frontend-experience)

### 1. Configure Agent Framework

> **Action Items:**
> Configure the agent type in your `.env` file within the `agentic_ai/applications` folder. Uncomment one of the following lines:
> ```bash
> # In your .env file in agentic_ai/applications folder, uncomment one of following for agent framework:
> AGENT_MODULE="agents.agent_framework.single_agent"
> # OR
> AGENT_MODULE="agents.agent_framework.multi_agent.magentic_group"
> # OR
> AGENT_MODULE="agents.agent_framework.multi_agent.handoff_multi_domain_agent"
> ```
> 
> Add additional environment variables to the bottom of your existing `.env` file:
> 
> **For Magentic Orchestration Settings (magentic_group):**
> ```bash
> MAGENTIC_LOG_WORKFLOW_EVENTS=true
> MAGENTIC_ENABLE_PLAN_REVIEW=false  # Set to true for human-in-the-loop plan approval
> MAGENTIC_MAX_ROUNDS=10
> ```
> 
> **For Handoff Agent Context Transfer (handoff_multi_domain_agent):**
> ```bash
> HANDOFF_CONTEXT_TRANSFER_TURNS=-1  # -1=all history, 0=none, N=last N turns
> ```

📚 **[See detailed pattern guide and configuration →](../agentic_ai/agents/agent_framework/README.md)**

### 2. Understanding Backend Code Structure

Before running the backend, let's examine the key Microsoft Agent Framework concepts in `backend.py`. This section walks through the essential patterns for integrating Agent Framework into a FastAPI backend.

#### 2.1 Dynamic Agent Loading

The backend dynamically loads the agent module based on the `AGENT_MODULE` environment variable. This allows switching between different agent types without code changes.

```python
# insert the code below into backend.py to load your agent dynamically. Find "load agent" comment and replace with:
sys.path.insert(0, str(Path(__file__).resolve().parent.parent))  
agent_module_path = os.getenv("AGENT_MODULE")  
agent_module = __import__(agent_module_path, fromlist=["Agent"])  # type: ignore[arg-type]  
Agent = getattr(agent_module, "Agent")  
```

**Key Concept:** The `Agent` class is loaded at runtime, so you can configure `AGENT_MODULE="agents.agent_framework.multi_agent.handoff_multi_domain_agent"` or `AGENT_MODULE="agents.agent_framework.multi_agent.magentic_group"` in your `.env` file.

#### 2.2 State Store Integration

Agent Framework agents require a state store to persist session data (conversation history, agent state, checkpoints). Here we'll use dict to save costs on CosmosDB during the workshop. Find "state store setup" comment and replace with:

```python
from utils import get_state_store  
  
STATE_STORE = get_state_store()  # either dict or CosmosDBStateStore 
```

**Key Concept:** `STATE_STORE` is a dictionary-like object that persists across requests. For production, use CosmosDB or Redis instead of the in-memory dict.

#### 2.3 WebSocket Connection Manager

For this workshop, the backend uses a WebSocket manager to broadcast events to all clients connected to a session. WebSockets are chosen over traditional HTTP for real time streaming, AI agents (especially LLMs) can take seconds or minutes to generate responses. WebSockets allow token-by-token streaming, so users see text appearing progressively rather than waiting for the entire response. It continuously listens for JSON messages from the client. Depending on agent module selected, different attributes are injected.

Insert code below into `backend.py` to create a WebSocket manager. Find "WebSocket endpoint for streaming" comment and replace with:
```python
@app.websocket("/ws/chat")
async def ws_chat(ws: WebSocket):
    await ws.accept()
    connected_session: Optional[str] = None
    try:
        while True:
            data = await ws.receive_json()
            session_id = data.get("session_id")
            prompt = data.get("prompt")
            token = data.get("access_token")  # optional

            if not session_id:
                await ws.send_json({"type": "error", "message": "Missing session_id"})
                continue
            if connected_session is None:
                await MANAGER.connect(session_id, ws)
                connected_session = session_id
                await ws.send_json({"type": "info", "message": f"Registered session {session_id}"})

            # If only registering (no prompt) continue
            if not prompt:
                continue

            # Create agent for this session
            try:
                agent = Agent(STATE_STORE, session_id, access_token=token)
            except TypeError:
                agent = Agent(STATE_STORE, session_id)

            # Inject WebSocket manager for Magentic streaming
            if hasattr(agent, "set_websocket_manager"):
                agent.set_websocket_manager(MANAGER)

            # Set progress sink if supported (for some agent types)
            if hasattr(agent, "set_progress_sink"):
                async def progress_sink(ev: dict):
                    # Broadcast progress events
                    await MANAGER.broadcast(session_id, ev)
                agent.set_progress_sink(progress_sink)

            # Stream events from agent
            try:
                # Check if agent supports streaming (Autogen or Agent Framework)
                if hasattr(agent, "chat_stream"):
                    # Autogen streaming
                    async for event in agent.chat_stream(prompt):
                        evt = await serialize_autogen_event(event)
                        if evt and evt.get("type") in ("token", "message", "final"):
                            await MANAGER.broadcast(session_id, evt)
                elif hasattr(agent, "chat_async"):
                    # Agent Framework - may or may not use streaming callback
                    result = await agent.chat_async(prompt)
                    # If agent has _ws_manager attribute, it supports streaming and events sent via callback
                    # Otherwise, broadcast final result here
                    if not hasattr(agent, "_ws_manager"):
                        await MANAGER.broadcast(session_id, {"type": "final_result", "content": result})
                    # Else: events including final result are sent via streaming callback
                else:
                    await MANAGER.broadcast(session_id, {"type": "error", "message": "Agent does not support streaming"})

                await MANAGER.broadcast(session_id, {"type": "done"})
            except Exception as e:
                await MANAGER.broadcast(session_id, {"type": "error", "message": str(e)})
    except WebSocketDisconnect:
        pass
    finally:
        if connected_session:
            MANAGER.disconnect(connected_session, ws)
```

**Key Concept:** The manager tracks multiple WebSocket connections per session_id and broadcasts JSON events to all connected clients. `agent.chat_async(prompt)` returns the final response as a string.

#### 2.4 Agent Streaming Patterns

Agent Framework supports multiple streaming patterns depending on the agent type (illustrative, no need to insert code here):

```python
# Stream events from agent
try:
    # Check if agent supports streaming (Autogen or Agent Framework)
    if hasattr(agent, "chat_stream"):
        # Autogen streaming
        async for event in agent.chat_stream(prompt):
            evt = await serialize_autogen_event(event)
            if evt and evt.get("type") in ("token", "message", "final"):
                await MANAGER.broadcast(session_id, evt)
    elif hasattr(agent, "chat_async"):
        # Agent Framework - may or may not use streaming callback
        result = await agent.chat_async(prompt)
        # If agent has _ws_manager attribute, it supports streaming and events sent via callback
        # Otherwise, broadcast final result here
        if not hasattr(agent, "_ws_manager"):
            await MANAGER.broadcast(session_id, {"type": "final_result", "content": result})
        # Else: events including final result are sent via streaming callback
    else:
        await MANAGER.broadcast(session_id, {"type": "error", "message": "Agent does not support streaming"})

    await MANAGER.broadcast(session_id, {"type": "done"})
except Exception as e:
    await MANAGER.broadcast(session_id, {"type": "error", "message": str(e)})
```

**Key Concepts:**

- **Autogen agents**: Implement `chat_stream()` that yields events (tokens, tool calls, messages)
- **Agent Framework agents**: Use `chat_async()` with optional streaming via WebSocket manager injection
- **Streaming callback pattern**: Magentic agents with `_ws_manager` broadcast events internally, so we don't broadcast the final result again

#### 2.5 Review the magentic_group agent

The `magentic_group` agent uses an orchestrator to delegate tasks to specialist agents. Review the code here (you may need to return to root directory): `agentic_ai\agents\agent_framework\multi_agent\magentic_group.py` Key features include:
- Real-time plan review and execution tracking
- Custom progress ledger for human-in-the-loop scenarios
- Checkpointing for resumable workflows
- WebSocket manager injection for streaming events
- Progress sink for broadcasting custom events
- Multiple agent collaboration with context sharing

Insert code below into `magentic_group.py`. Find "import magentic agent" comment and replace with:

```python
from agent_framework import (
    ChatAgent,
    MagenticBuilder,
    MCPStreamableHTTPTool,
    WorkflowCheckpoint,
    WorkflowOutputEvent,
    CheckpointStorage,
    MagenticCallbackEvent,
    MagenticCallbackMode,
    MagenticOrchestratorMessageEvent,
    MagenticAgentDeltaEvent,
    MagenticAgentMessageEvent,
    MagenticFinalResultEvent,
)
```

#### 2.6 Review magentic agent setup
The `MagenticBuilder` class constructs the multi-agent orchestrator with specialist agents. Review the code for `class Agent(BaseAgent)` in the file. Declare magentic builder and build workflow. Find "magentic builder" comment and replace with:

```python
async def _build_workflow(
        self,
        participant_client: AzureOpenAIChatClient,
        manager_client: AzureOpenAIChatClient,
        tools: List[MCPStreamableHTTPTool] | None,
        checkpoint_storage: CheckpointStorage,
    ) -> Any:
        participants = await self._create_participants(participant_client, tools)

        builder = MagenticBuilder().participants(**participants)
        
        # Register streaming callback if WebSocket is available (MUST be before with_standard_manager)
        if self._ws_manager:
            logger.info(f"[STREAMING] Registering streaming callback for magentic events, session_id={self.session_id}")
            logger.info(f"[STREAMING] WebSocket manager type: {type(self._ws_manager)}")
            logger.info(f"[STREAMING] Callback function: {self._stream_magentic_event}")
            builder = builder.on_event(self._stream_magentic_event, mode=MagenticCallbackMode.STREAMING)
            logger.info("[STREAMING] Callback registered successfully")
        elif self._workflow_event_logging_enabled:
            logger.info("[STREAMING] Using workflow event logging instead of streaming")
            builder = builder.on_event(self._log_workflow_event)
        
        builder = (
            builder
            .with_standard_manager(
                chat_client=manager_client,
                instructions=self._manager_instructions,
                max_round_count=self._max_round_count,
                max_stall_count=self._max_stall_count,
                max_reset_count=self._max_reset_count,
                progress_ledger_prompt=self.CUSTOM_PROGRESS_LEDGER_PROMPT,
            )
            .with_checkpointing(checkpoint_storage)
        )

        # Optional: enable plan review if available
        if self._enable_plan_review:
            enable_plan_review = getattr(builder, "enable_plan_review", None)
            if callable(enable_plan_review):
                try:
                    builder = enable_plan_review()
                except Exception as exc:
                    logger.warning(
                        "[AgentFramework-Magentic] Failed to enable plan review: %s", exc
                    )
            else:
                logger.debug(
                    "[AgentFramework-Magentic] Plan review requested but not available in this framework version."
                )

        return builder.build()
```

#### 2.6 Review participants setup
Review the following lines. `async def _create_participants ()` class wraps each participant agent (e.g., your MCP-enabled assistants) so they can participate in the workflow graph.

#### 2.7 Review the handoff multi_domain_agent pattern
Review the `handoff_multi_domain_agent.py` file. Explore the key areas: Intent classifiers, Patterns, MCP tool creation and more.

#### 2.8 Summary: Agent Framework Integration Checklist

When building a backend for Agent Framework agents:

1. ✅ **Load agent dynamically** based on `AGENT_MODULE` env var
2. ✅ **Use a state store** (dict for dev, CosmosDB/Redis for production)
3. ✅ **Support both REST and WebSocket** endpoints (REST for simple, WebSocket for streaming)
4. ✅ **Inject WebSocket manager** for Magentic streaming (if supported)
5. ✅ **Declare agent code** using Agent Framework patterns (MagenticBuilder, Handoff patterns)

### 3. Run Backend Service

> **Action Items:**
> Open a new terminal window separate than the one running the MCP server.
> ![new terminal](media/01_mcp_new_terminal.png)
> Navigate to the applications directory and start the backend:
> ```bash
> cd agentic_ai/applications
> uv run python backend.py
> ```

### 3. Choose Frontend Experience

## 📊 Frontend Comparison

| Feature | React Frontend | Streamlit Frontend |
|---------|---------------|-------------------|
| **Real-time streaming** | ✅ Token-by-token | ❌ Full response only |
| **Internal process visibility** | ✅ Orchestrator, agents, tools | ❌ Final answer only |
| **Tool call tracking** | ✅ Per-turn history | ❌ Not shown |
| **Multi-agent visualization** | ✅ Agent timeline & planning | ❌ Not shown |
| **Best for Agent Framework** | ✅ **Recommended** | ⚠️ Basic support |
| **Setup complexity** | Medium (npm install) | Low (no additional setup) |
| **Best use case** | Development, demos, debugging | Quick testing, simple chat |

**Recommendation:**
- Use **React** for Agent Framework agents to see the full multi-agent orchestration
- Use **Streamlit** for quick testing of any agent type or simple demos

## Success criteria
- Backend service is running on `http://localhost:7000`
- Agent Framework is properly configured
- Backend can communicate with MCP server
  - A sample powershell commande to validate backend server:
    ```powershell
    # Define the URL
    $uri = "http://localhost:7000/chat"
  
    # Define headers
    $headers = @{
        Accept = "application/json"
        "Content-Type" = "application/json"
    }
    
    # Define the JSON body
    $body = @{
        session_id = "123"
        prompt     = "What can you help me with?"
    } | ConvertTo-Json
    
    # Send the POST request
    $response = Invoke-WebRequest -Uri $uri -Method POST -Headers $headers -Body $body
    
    # Output the response
    $response.Content
    ```
    <img src="media/02_backend_chat_response.png" />
 
  - A sample curl command to validate things are online:
    ```bash
    `curl -X 'POST' 'http://localhost:7000/chat'  -H 'accept: application/json'  -H 'Content-Type: application/json'  -d '{"session_id": "123", "prompt": "What can you help me with?"}'`
    ```

    **Note:** The local server does not return the AI response, if you view from the browser this is expected.

    <img src="media/02_backend_localhost_err.png" />

- Ready to connect to frontend

**Next Step**: Choose your frontend - [React Frontend (Recommended)](03_frontend_react.md) | [Streamlit Frontend](03_frontend_streamlit_uv.md)

**📌 Important:** Agent Framework works best with the **React frontend** to visualize the internal agent processes, orchestrator planning, and tool calls in real-time.


