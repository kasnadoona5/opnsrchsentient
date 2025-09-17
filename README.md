# OpenDeepSearch FastAPI Server

This repository provides a step-by-step guide to deploying the [OpenDeepSearch](https://github.com/sentient-agi/OpenDeepSearch) library as a persistent API endpoint using FastAPI. This setup is perfect for integrating OpenDeepSearch into applications like [n8n](https://n8n.io), [Zapier](https://zapier.com), or any custom workflows requiring a reliable, long-running search agent.

The key feature of this setup is its ability to dynamically reload OpenDeepSearch on every API request. This is ideal for securely passing API keys with each call, addressing OpenDeepSearch's reliance on initial environment variable configuration.

## Features
- **Persistent API Endpoint**: Run OpenDeepSearch as a 24/7 service.
- **Dynamic API Key Handling**: Securely pass API keys with each request, perfect for multi-tenant or dynamic applications.
- **Decoupled Architecture**: Host the AI agent on a separate server from your main application (e.g., n8n).
- **Easy Deployment**: A simple, step-by-step guide to getting started on a standard Linux VPS.
- **Health Check Endpoint**: Includes a root `/` endpoint to monitor service health.

## Prerequisites

Before you begin, ensure you have the following:
- A server running a modern Linux distribution (e.g., Ubuntu 22.04).
- SSH access to your server.
- An API key from a search provider (this guide uses [Serper.dev](https://serper.dev)).
- An API key from a language model provider (this guide uses [OpenRouter.ai](https://www.openrouter.ai)).

## Installation Guide

### Step 1: Install a Modern Python Version

OpenDeepSearch requires Python 3.9 or newer. We’ll install Python 3.11 to ensure full compatibility.

```bash
# Update package lists
sudo apt update
sudo apt install software-properties-common -y

# Add the deadsnakes PPA for newer Python versions
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# Install Python 3.11 and its virtual environment module
sudo apt install python3.11 python3.11-venv -y
Step 2: Create a Project Directory and Virtual Environment
To keep our project dependencies isolated, we'll use a virtual environment.

bash
Copy code
# Create a project directory
mkdir ~/opendeepsearch-api
cd ~/opendeepsearch-api

# Create a virtual environment using python3.11
python3.11 -m venv venv

# Activate the virtual environment
source venv/bin/activate
Your terminal prompt should now begin with (venv). Make sure to activate this environment whenever you work on the project.

Step 3: Install All Required Libraries
Next, install OpenDeepSearch and other necessary libraries.

bash
Copy code
# Ensure git is installed
sudo apt install git -y

# Upgrade pip to the latest version
pip install --upgrade pip

# Install required libraries
pip install "git+https://github.com/sentient-agi/OpenDeepSearch.git" fastapi uvicorn "torch --index-url https://download.pytorch.org/whl/cpu" loguru nest_asyncio litellm
Note: We're installing the CPU-only version of PyTorch to save disk space, as this build doesn't require a GPU.

Step 4: Create the FastAPI Application (main.py)
This is the core of our project. The main.py file creates a FastAPI server with a /search endpoint that dynamically loads and runs OpenDeepSearch.

Create the main.py file:

bash
Copy code
nano main.py
Copy and paste the code below into main.py. Be sure to replace YOUR_SUPER_SECRET_RANDOM_STRING_HERE with your own unique secret password.

python
Copy code
import os
import sys
import importlib
import traceback
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
import litellm

# --- Configuration & Setup ---
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
    os.environ["SERPER_API_KEY"] = serper_key
    os.environ["OPENROUTER_API_KEY"] = openrouter_key
    litellm.api_key = openrouter_key
    
    modules_to_reload = [
        'opendeepsearch.serp_search.serp_search',
        'opendeepsearch.ods_agent',
        'opendeepsearch.ods_tool',
        'opendeepsearch'
    ]
    
    for module_name in modules_to_reload:
        if module_name in sys.modules:
            importlib.reload(sys.modules[module_name])
    
    from opendeepsearch import OpenDeepSearchTool
    search_tool = OpenDeepSearchTool(model_name=model_name)
    
    if hasattr(search_tool, 'setup'):
        search_tool.setup()
    
    return search_tool

@app.post("/search")
async def run_search(req: Request, request_body: SearchRequest):
    headers = req.headers
    
    x_api_key = headers.get("x-api-key")
    serper_api_key = headers.get("serper-api-key")
    openrouter_api_key = headers.get("openrouter-api-key")
    
    if x_api_key != MY_SECRET_API_KEY:
        raise HTTPException(status_code=401, detail="Invalid API Key")
    
    if not serper_api_key or not openrouter_api_key:
        raise HTTPException(status_code=400, detail="Serper and/or OpenRouter API keys missing")
    
    try:
        search_tool = create_search_tool_with_env(
            serper_key=serper_api_key,
            openrouter_key=openrouter_api_key,
            model_name="openrouter/mistralai/mistral-7b-instruct:free"
        )
        
        result = search_tool.forward(request_body.query)
        return SearchResponse(result=str(result))
        
    except Exception as e:
        traceback.print_exc()
        raise HTTPException(status_code=500, detail=f"Search error: {str(e)}")

@app.get("/")
def read_root():
    return {"status": "OpenDeepSearch API is running"}
Save and exit the editor (Ctrl + X, then Y, then Enter).

Step 5: Run the Server Persistently
Use screen to keep the server running even after you disconnect from your SSH session.

bash
Copy code
# Install screen if it's not already installed
sudo apt install screen -y

# Start a named screen session
screen -S api

# Inside the new screen session, activate the environment and start the Uvicorn server
source venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8000
To detach from the session, press Ctrl + A, release, and then press D.
To reattach to the session, use screen -r api.

Step 6: Open the Firewall
Allow external traffic to reach your API on port 8000.

bash
Copy code
sudo ufw allow 8000/tcp
Using the API
Once the server is running, you can make a POST request to your OpenDeepSearch API at:

arduino
Copy code
http://<YOUR_SERVER_IP>:8000/search
Required Headers
x-api-key: The secret password you set in main.py.

serper-api-key: Your API key from Serper.dev.

openrouter-api-key: Your API key from OpenRouter.ai.

Required JSON Body
json
Copy code
{
  "query": "What are the latest advancements in AI agents?"
}
Health Check
You can check the status of your API by visiting the root / endpoint:

cpp
Copy code
http://<YOUR_SERVER_IP>:8000/
This should return a JSON response like:

json
Copy code
{
  "status": "OpenDeepSearch API is running"
}
