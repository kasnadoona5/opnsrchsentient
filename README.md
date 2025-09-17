OpenDeepSearch FastAPI Server
This repository provides a step-by-step guide to deploying the OpenDeepSearch library as a robust, persistent API endpoint using FastAPI. This setup is ideal for integrating OpenDeepSearch into other applications like n8n, Zapier, or any custom workflow that needs to call a reliable, long-running search agent.
The primary challenge this setup solves is the library's reliance on initial environment variable configuration. Our solution uses a "factory function" to dynamically reload the library on each API call, allowing you to pass API keys securely and dynamically with each request.
Features
Persistent API Endpoint: Run OpenDeepSearch as a 24/7 service.
Dynamic API Key Handling: Securely pass API keys with each request, perfect for multi-tenant or dynamic applications.
Decoupled Architecture: Host your AI agent on a separate server from your main application (e.g., n8n).
Easy Deployment: A clear, step-by-step guide to get you up and running on a standard Linux VPS.
Health Check: Includes a root / endpoint to easily check if the service is online.
Prerequisites
Before you begin, you will need:
A server running a modern Linux distribution (e.g., Ubuntu 22.04).
SSH access to your server.
An API key from a search provider (this guide uses Serper.dev).
An API key from a language model provider (this guide uses OpenRouter.ai).
Step-by-Step Installation Guide
Step 1: Install a Modern Python Version
OpenDeepSearch requires Python 3.9 or newer. We will install Python 3.11 to ensure full compatibility.
code
Bash
# Update package lists
sudo apt update
sudo apt install software-properties-common -y

# Add the deadsnakes PPA for newer Python versions
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# Install Python 3.11 and its virtual environment module
sudo apt install python3.11 python3.11-venv -y
Step 2: Create a Project Directory and Virtual Environment
It's crucial to isolate our project's dependencies in a virtual environment.
code
Bash
# Create a project directory
mkdir ~/opendeepsearch-api
cd ~/opendeepsearch-api

# Create a virtual environment using python3.11
python3.11 -m venv venv

# Activate the virtual environment
source venv/bin/activate
Note: Your terminal prompt should now start with (venv). You must activate this environment every time you work on the project.
Step 3: Install All Required Libraries
We will install opendeepsearch directly from its GitHub repository to get the latest version, along with all other necessary libraries.
code
Bash
# First, ensure git is installed
sudo apt install git -y

# Upgrade pip to the latest version
pip install --upgrade pip

# Install all required libraries in a single command
pip install "git+https://github.com/sentient-agi/OpenDeepSearch.git" fastapi uvicorn "torch --index-url https://download.pytorch.org/whl/cpu" loguru nest_asyncio litellm
Info: We are installing the CPU-only version of PyTorch to save significant disk space, as this build does not require a GPU.
Step 4: Create the FastAPI Application (main.py)
This is the core of our project. This script creates a web server with a /search endpoint that dynamically rigs and runs the OpenDeepSearchTool.
Create a file named main.py:
code
Bash
nano main.py
Copy and paste the entire code block below into the file. The code is also available in the main.py file in this repository.
code
Python
import os
import sys
import importlib
import traceback
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
import litellm

# --- Configuration & Setup ---
# !!! IMPORTANT: Change this to a long, secret password !!!
MY_SECRET_API_KEY = "YOUR_SUPER_SECRET_RANDOM_STRING_HERE"

app = FastAPI()

# --- Pydantic Models for Request/Response ---
class SearchRequest(BaseModel):
    query: str

class SearchResponse(BaseModel):
    result: str
    status: str = "success"

def create_search_tool_with_env(serper_key, openrouter_key, model_name):
    """
    Creates a search tool with proper environment variables set.
    This function forces a fresh import of the library on each call,
    making it suitable for a long-running server environment.
    """
    # Set environment variables for the library to read
    os.environ["SERPER_API_KEY"] = serper_key
    os.environ["OPENROUTER_API_KEY"] = openrouter_key
    litellm.api_key = openrouter_key
    
    # Force reload of the modules to pick up new environment variables
    modules_to_reload = [
        'opendeepsearch.serp_search.serp_search',
        'opendeepsearch.ods_agent',
        'opendeepsearch.ods_tool',
        'opendeepsearch'
    ]
    
    for module_name in modules_to_reload:
        if module_name in sys.modules:
            importlib.reload(sys.modules[module_name])
    
    # Now that modules are reloaded, we can import and create the tool
    from opendeepsearch import OpenDeepSearchTool
    
    # Create the tool
    search_tool = OpenDeepSearchTool(model_name=model_name)
    
    if hasattr(search_tool, 'setup'):
        search_tool.setup()
    
    return search_tool

# --- API Endpoint ---
@app.post("/search")
async def run_search(req: Request, request_body: SearchRequest):
    headers = req.headers
    
    # Extract header values sent from the client (e.g., n8n)
    x_api_key = headers.get("x-api-key")
    serper_api_key = headers.get("serper-api-key")
    openrouter_api_key = headers.get("openrouter-api-key")
    
    # 1. Security Check: Protect your endpoint with a static API key
    if x_api_key != MY_SECRET_API_KEY:
        raise HTTPException(status_code=401, detail="Invalid API Key")
    
    # 2. Check for required provider keys
    if not serper_api_key or not openrouter_api_key:
        raise HTTPException(status_code=400, detail="Serper and/or OpenRouter API keys missing")
    
    try:
        # 3. Create a fresh instance of the tool on every request
        search_tool = create_search_tool_with_env(
            serper_key=serper_api_key,
            openrouter_key=openrouter_api_key,
            model_name="openrouter/mistralai/mistral-7b-instruct:free" # Or any other model
        )
        
        # 4. Execute the search using the tool's .forward() method
        result = search_tool.forward(request_body.query)
        
        return SearchResponse(result=str(result))
        
    except Exception as e:
        traceback.print_exc()
        raise HTTPException(status_code=500, detail=f"Search error: {str(e)}")

# --- Health Check Endpoint ---
@app.get("/")
def read_root():
    return {"status": "OpenDeepSearch API is running"}
IMPORTANT: Before saving, remember to change YOUR_SUPER_SECRET_RANDOM_STRING_HERE to your own unique, secret password. This will be used to protect your API.
Save and exit the editor (Ctrl + X, then Y, then Enter).
Step 5: Run the Server Persistently
To ensure your API stays online even after you disconnect your SSH session, we'll use screen.
code
Bash
# Install screen if it's not already
sudo apt install screen -y

# Start a named screen session
screen -S api

# Inside the new screen session, activate the environment and start the Uvicorn server
source venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8000
Your server is now running. To leave it running in the background, detach from the session by pressing Ctrl + A, releasing, and then pressing D.
To re-attach to the session later (to view logs or restart), use screen -r api.
Step 6: Open the Firewall
Allow external traffic to reach your API on port 8000.
code
Bash
sudo ufw allow 8000/tcp
Using Your API
Your OpenDeepSearch API is now live. You can call it from any application (like n8n, Postman, or a custom script) by sending a POST request to:
http://<YOUR_SERVER_IP>:8000/search
Required Headers
x-api-key: The secret password you set in main.py.
serper-api-key: Your API key from Serper.dev.
openrouter-api-key: Your API key from OpenRouter.ai.
Required JSON Body
code
JSON
{
  "query": "What are the latest advancements in AI agents?"
}
