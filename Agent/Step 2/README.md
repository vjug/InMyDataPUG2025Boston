# Step 2: Python Driver – Environment & Connectivity

## What Changed in This Step

This step introduces the foundational Python environment setup and validates connectivity to the OpenEdge backend service.

### New Files

- **`requirements.txt`**: Lists the Python dependencies needed for this step
  - `python-dotenv`: Loads environment variables from a `.env` file
  
- **`OEDatabaseDriver.py`**: A minimal Python script that:
  - Imports and uses `python-dotenv` to load environment variables
  - Reads the `OE_SERVICE_URL` from the `.env` file
  - Prints the service URL to verify configuration

- **`.env`** (created by user): Contains the OpenEdge service URL:
  ```text
  OE_SERVICE_URL=http://localhost:8080/AgentTools/web/
  ```

### Code Overview

The `OEDatabaseDriver.py` file at this stage is extremely simple:

```python
from dotenv import load_dotenv
import os

load_dotenv(".env", override=True)

print(os.getenv("OE_SERVICE_URL"))
```

This script:
1. Loads environment variables from the `.env` file
2. Retrieves the `OE_SERVICE_URL` value
3. Prints it to confirm the configuration is working

## What This Implements

**Environment Configuration Foundation**: This step establishes the pattern for managing configuration and secrets that will be used throughout the workshop:

- **Separation of Configuration from Code**: By using a `.env` file, we keep sensitive URLs, API keys, and other configuration separate from the codebase
- **Python Virtual Environment**: The use of a virtual environment (`.venv`) isolates project dependencies
- **Configuration Validation**: The simple print statement confirms that the environment is set up correctly before proceeding

## How This Helps Build Towards the Final Solution

This step is crucial for several reasons:

1. **Configuration Management Pattern**: Establishes the `.env` file pattern that will later hold:
   - OpenEdge service URLs
   - LiveKit credentials
   - OpenAI API keys
   
2. **Development Environment Setup**: Creates a clean, isolated Python environment that will host:
   - The database driver
   - The LiveKit agent
   - All dependencies

3. **Connectivity Validation**: Before writing complex code, we verify that:
   - Python is installed correctly
   - Virtual environments work
   - Environment variable loading works
   - The OpenEdge service URL is configured

4. **Foundation for OEDatabaseDriver**: This simple script evolves in subsequent steps into a full-featured database driver class that the AI agent will use to interact with the OpenEdge database.

## Testing This Step

Run the script to verify your environment:

```powershell
py OEDatabaseDriver.py
```

**Expected Output**: The script should print your OpenEdge service URL, confirming that:
- Python is working
- The virtual environment is active
- `python-dotenv` is installed correctly
- The `.env` file is being read successfully

If you see the URL printed, you're ready to move on to Step 3!

## Key Concepts for OpenEdge Developers

- **Python Virtual Environments**: Similar to having separate PROPATHs for different projects, a Python `venv` keeps dependencies isolated
- **Environment Variables**: Like using `OS-GETENV` in ABL, Python uses environment variables for configuration
- **Dependencies Management**: `requirements.txt` is like a dependency manifest, listing all libraries needed for the project
