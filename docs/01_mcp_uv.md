# Part 1: MCP Set Up

## Pre-requisites
- Complete [Part 0](../SETUP.md)

## Summary
In this part, you will set up the MCP (Model Control Protocol) for the Microsoft AI Agentic Workshop. This involves installing the necessary Python dependencies and running the MCP server.

## Steps
1. Install uv

- ⚡Use `uv` for fastest setup

- [**uv**](https://github.com/astral-sh/uv) is a blazing-fast Python package installer and resolver written in Rust. It's 10-100x faster than `pip` and automatically manages virtual environments.

    > **Action Items:**
    > Install `uv` by running the following command in your terminal:
    > Windows (PowerShell):
    ```bash
    powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
    ```
    > macOS/Linux:
    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

2. Install dependencies with uv:

    > Note: uv automatically creates .venv and installs dependencies from pyproject.toml
    > **Action Items**:
    > Navigate to the `agentic_ai/applications` folder and install the dependencies by running the following command:
    ```bash
    # Navigate to the applications directory
    cd agentic_ai/applications

    # Create virtual environment and install all dependencies in one command
    uv sync
    ```

3. Create MCP Service

> **Action Items:**
> Follow the steps below to create the backend service by coding the `mcp_service.py` file. This will help you understand how to set up and run the MCP server.

Note: If you are currently in the `agentic_ai/applications` folder, go back to the root folder first:

```bash
cd ../..
```

3a: Import FastMCP

Add the following import statement at the top of your `mcp_service.py` file:
```python
from fastmcp import FastMCP
```

3b: Declare MCP

Declare the MCP instance in your `mcp_service.py` file as shown below, find the keyword "insert FastMCP initialization code here" and replace it with the following code:

```python
mcp = FastMCP(
    name="Contoso Customer API as Tools",
    instructions=(
        "All customer, billing and knowledge data is accessible ONLY via the declared "
        "tools below.  Return values follow the pydanticschemas.  Always call the most "
        "specific tool that answers the user’s question."
    ),
    auth=auth,  
)
```

3c: Write MCP Tools

Add the following MCP tools to your `mcp_service.py` file. These tools define the endpoints for interacting with customer data, go to line 421:

**Tool 1: List All Customers**
```python
@mcp.tool(description="List all customers with basic info")
async def get_all_customers() -> List[CustomerSummary]:  
    data = await get_all_customers_async()
    return [CustomerSummary(**r) for r in data]
```

**Tool 2: Get Subscription Detail**

```python
@mcp.tool(  
    description=(  
        "Detailed subscription view → invoices (with payments) + service incidents."  
    )  
)  
async def get_subscription_detail(  
    subscription_id: Annotated[int, "Subscription identifier value"],  
) -> SubscriptionDetail:  
    data = await get_subscription_detail_async(subscription_id)

    # Convert nested data to Pydantic models
    invoices = []
    for inv_data in data['invoices']:
        payments = [Payment(**p) for p in inv_data['payments']]
        invoices.append(Invoice(**{**inv_data, 'payments': payments}))
    
    service_incidents = [ServiceIncident(**si) for si in data['service_incidents']]
    
    return SubscriptionDetail(**{**data, 'invoices': invoices, 'service_incidents': service_incidents})
```

3d: Run MCP Server

Finally, add the following code to the bottom of your `mcp_service.py` file to start the MCP server:

```python
if __name__ == "__main__":  
    asyncio.run(mcp.run_http_async(host="0.0.0.0", port=8000))  
```

4. Start MCP Server

    > **Action Items:**
    > Note: If you are currently in the `agentic_ai/applications` folder, go back to the root folder first:
    ```bash
    cd ../..
    ```
    > Navigate to the `mcp` folder and start the MCP server:
    ```bash
    cd mcp
    uv run python mcp_service.py
    ```
    > Note: Let the MCP server run in this terminal window. Open a new terminal window to proceed to the next step. 
    
    <img src="media/01_mcp_fastmcp.jpg" />

## Success criteria

- MCP server is running and ready to accept requests.
- A sample powershell command for checking the MCP server status:  
    ```powershell
    Invoke-WebRequest -Uri "http://localhost:8000/mcp" -Method POST -Headers @{Accept="application/json, text/event-stream";"Content-Type"="application/json"} -Body               '{"jsonrpc":"2.0","id":"init-1","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"curl","version":"8"}}}'
    ```

    <img src="media/01_mcp_local_ok.jpg" />
    
    **Note:** This is an MCP server endpoint and cannot be accessed directly via a browser or unsupported transports like SSE. Please use a streamable HTTP transport. This error is expected on your browser.
  
    <img src="media/01_mcp_localhost_err.png" />

 - A sample curl command for ensuring the MCP server is online:
    ```bash
    curl -sS -i -X POST "http://localhost:8000/mcp" -H "Accept: application/json, text/event-stream" -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","id":"init-            1","method":"initialize", "params":{"protocolVersion":"2024-11-05","capabilities":{}, "clientInfo":{"name":"curl","version":"8"}}}'`
    ```

**Alternative**: Use `pip` and `venv` (slower): [Run MCP with pip](01_mcp_pip.md)

**Next Step**: [Run the Backend Application](02_backend_uv.md)
