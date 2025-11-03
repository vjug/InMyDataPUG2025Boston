# Step 9: Token Server - Optional

## What Changed in This Step

This optional step implements a secure token generation server that provides dynamic LiveKit access tokens for frontend clients. Instead of hardcoding tokens (which expire and pose security risks), this server generates fresh tokens on-demand for each connection.

### New Files

All files in the `TokenServer` directory:

- **`server.py`**: Flask-based HTTP server that:
  - Generates LiveKit access tokens
  - Creates unique room names
  - Manages room lifecycle
  - Handles CORS for browser requests

- **`requirements.txt`**: Python dependencies:
  - `flask`: Web framework for HTTP endpoints
  - `flask-cors`: Cross-Origin Resource Sharing support
  - `livekit-api`: LiveKit SDK for token generation
  - `python-dotenv`: Environment variable management

- **`.env`**: Configuration file containing:
  - `LIVEKIT_API_KEY`: Your LiveKit API key
  - `LIVEKIT_API_SECRET`: Your LiveKit API secret

### Code Overview

#### Token Server (server.py)

```python
from flask import Flask, request
from flask_cors import CORS
from livekit import api
import uuid

app = Flask(__name__)
CORS(app, resources={r"/*": {"origins": "*"}})  # Allow all origins

async def generate_room_name():
    """Generate unique room name like 'room-a3b7c2d1'"""
    name = "room-" + str(uuid.uuid4())[:8]
    rooms = await get_rooms()
    while name in rooms:  # Ensure uniqueness
        name = "room-" + str(uuid.uuid4())[:8]
    return name

async def get_rooms():
    """Get list of active rooms from LiveKit"""
    api_client = LiveKitAPI()
    rooms = await api_client.room.list_rooms(ListRoomsRequest())
    await api_client.aclose()
    return [room.name for room in rooms.rooms]

@app.route("/getToken")
async def get_token():
    # Get parameters from query string
    name = request.args.get("name", "my name")  # User's display name
    room = request.args.get("room", None)       # Optional room name
    
    # Generate room name if not provided
    if not room:
        room = await generate_room_name()
    
    # Create access token
    token = api.AccessToken(
        os.getenv("LIVEKIT_API_KEY"), 
        os.getenv("LIVEKIT_API_SECRET")
    ) \
        .with_identity(name) \
        .with_name(name) \
        .with_grants(api.VideoGrants(
            room_join=True,
            room=room
        ))
    
    return token.to_jwt()  # Return JWT token as string

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001, debug=True)
```

### API Endpoint

**GET `/getToken`**

**Query Parameters**:
- `name` (optional): User's display name (default: "my name")
- `room` (optional): Specific room to join (default: auto-generated)

**Response**: Plain text JWT token

**Example Request**:
```
GET http://localhost:5001/getToken?name=John%20Smith
```

**Example Response**:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MzA3MTU4OD...
```

## What This Implements

**Secure Token Generation**: This server provides several critical security features:

- **Dynamic Token Creation**: Each connection gets a fresh token with:
  - Limited lifetime (typically 1-6 hours)
  - Specific user identity
  - Specific room permissions
  
- **Room Management**:
  - Automatically generates unique room names
  - Prevents room name collisions
  - Can query active rooms from LiveKit

- **CORS Support**: Allows browser-based frontends to request tokens

- **Flexible Permissions**: The `VideoGrants` specify what the user can do:
  - `room_join=True`: Can join rooms
  - Can be extended with permissions for recording, admin access, etc.

## How This Helps Build Towards the Final Solution

This token server is essential for production deployments:

1. **Security Best Practice**:
   - **Problem**: Hardcoded tokens in frontend code are:
     - Visible to anyone who views page source
     - Can't be revoked without redeploying frontend
     - Have fixed expiration times
   - **Solution**: Server-side token generation:
     - Tokens never exposed in frontend code
     - Fresh token per session
     - Can implement additional authentication

2. **User Identity Management**:
   - Each user gets a unique identity in LiveKit
   - Agent can see who's connected
   - Enables features like:
     - User-specific greetings
     - Connection history
     - Analytics per user

3. **Room Isolation**:
   - Each session can have its own room
   - Prevents users from accidentally joining each other's calls
   - Supports multi-user scenarios (supervisor joining customer call)

4. **Production Architecture**:
   ```
   Browser → Token Server → LiveKit Cloud
                ↓
           Validates credentials
           Generates token
           Returns to browser
           
   Browser + Token → LiveKit Room ← Agent
   ```

5. **Scalability Foundation**:
   - Can add authentication (check username/password)
   - Can integrate with existing user databases
   - Can implement rate limiting
   - Can add logging/analytics
   - Can add queue management (if too many active calls)

6. **Integration Point**: The token server can:
   - Check if user is authenticated in your system
   - Look up user details from OpenEdge database
   - Apply business rules (e.g., "only customers can connect")
   - Log connection attempts

## Testing This Step

### Setup

1. **Create Directory**:
   ```powershell
   cd C:\Work
   mkdir TokenServer
   cd TokenServer
   ```

2. **Create Virtual Environment**:
   ```powershell
   py -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

3. **Copy Files**: Copy `server.py` and `requirements.txt` from examples

4. **Install Dependencies**:
   ```powershell
   pip install -r requirements.txt
   ```

5. **Configure Environment**: Create `.env`:
   ```
   LIVEKIT_API_KEY=your_api_key
   LIVEKIT_API_SECRET=your_api_secret
   ```

6. **Start Server**:
   ```powershell
   py server.py
   ```

   Expected output:
   ```
   * Running on http://0.0.0.0:5001
   * Debug mode: on
   ```

### Test Token Generation

**Test 1: Basic Token Request**
```powershell
# Using PowerShell
Invoke-WebRequest -Uri "http://localhost:5001/getToken?name=TestUser"
```

Expected: Long JWT string

**Test 2: With Browser**
- Open browser to: `http://localhost:5001/getToken?name=John`
- Should see JWT token displayed

**Test 3: Decode Token** (optional)
- Visit https://jwt.io
- Paste the token
- Verify it contains:
  - Your user name
  - Room name
  - Expiration time
  - Permissions

**Test 4: Integration with Frontend**
- Ensure token server is running (port 5001)
- Start frontend (from Step 8)
- Click "Connect to Support"
- Enter name
- Check browser DevTools → Network tab
- Should see request to `/api/getToken?name=YourName`
- Should receive token in response

## Key Concepts for OpenEdge Developers

- **Flask ≈ PASOE REST Service**: Both handle HTTP requests:
  ```abl
  /* ABL Web Handler */
  METHOD PUBLIC OVERRIDE INTEGER HandleGet():
      DEFINE VARIABLE cToken AS CHARACTER NO-UNDO.
      cToken = generateToken().
      THIS-OBJECT:Response:Entity = cToken.
      RETURN 200.
  END METHOD.
  ```
  vs.
  ```python
  @app.route("/getToken")
  def get_token():
      token = generate_token()
      return token
  ```

- **JWT Tokens ≈ Session IDs**: Like generating a unique session ID that expires:
  ```abl
  cSessionId = GUID + STRING(NOW) + cUser.
  /* Store in session table with expiration */
  ```

- **CORS**: Cross-Origin Resource Sharing allows browser apps to call your API. Similar to configuring PASOE to accept requests from web applications.

- **Async Functions**: The `async`/`await` pattern handles concurrent requests efficiently, similar to multi-session agents in PASOE

## Security Enhancements

For production, consider adding:

1. **Authentication**:
   ```python
   @app.route("/getToken")
   async def get_token():
       # Check authentication header
       auth_header = request.headers.get("Authorization")
       if not validate_auth(auth_header):
           return "Unauthorized", 401
       # ... generate token
   ```

2. **Rate Limiting**:
   ```python
   from flask_limiter import Limiter
   
   limiter = Limiter(app, default_limits=["10 per minute"])
   
   @app.route("/getToken")
   @limiter.limit("5 per minute")
   async def get_token():
       # ...
   ```

3. **User Validation**: Query OpenEdge database to verify user exists:
   ```python
   # Before generating token
   if not check_user_in_database(name):
       return "User not found", 404
   ```

4. **Logging**:
   ```python
   import logging
   
   logging.info(f"Token generated for user: {name}, room: {room}")
   ```

5. **HTTPS**: In production, use HTTPS:
   ```python
   app.run(ssl_context='adhoc')  # Development
   # Or configure with proper certificates
   ```

## What's Next

With the token server in place, you have a complete production-ready architecture:
- Frontend (Step 8) → Token Server (Step 9) → LiveKit → Agent (Steps 5-7) → OpenEdge

Step 10 (optional) introduces MCP for dynamic tool discovery, making the agent architecture even more flexible and maintainable.
