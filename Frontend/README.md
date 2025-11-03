# Step 8: Frontend (React UI) - Optional

## What Changed in This Step

This optional step adds a web-based user interface for the voice agent, moving from the LiveKit Agents Playground to a custom React application. This demonstrates how to build a production-ready frontend for your voice agent.

### New Components

This step involves creating a new React application in the `Frontend` directory with two progressive stages:

#### Frontend Step 1: Static Token
A basic React UI with a hardcoded LiveKit token for quick testing.

#### Frontend Step 2: Dynamic Token  
An enhanced UI that fetches tokens from a token server (Step 9).

### Frontend Step 1 Files

- **`package.json`**: React app dependencies:
  - `react`, `react-dom`: Core React framework
  - `@livekit/components-react`: Pre-built LiveKit UI components
  - `@livekit/components-styles`: Styling for LiveKit components
  - `livekit-client`: LiveKit JavaScript SDK
  - `vite`: Fast development server and build tool

- **`src/App.jsx`**: Main application component:
  - Welcome screen with "Connect to Support" button
  - Shows/hides the LiveKit modal

- **`src/components/LiveKitModal.jsx`**: Modal containing the voice agent:
  - `LiveKitRoom` component for connection
  - Uses hardcoded token (temporary)
  - Audio-only configuration

- **`src/components/SimpleVoiceAssistant.jsx`**: Voice assistant UI:
  - Displays agent connection status
  - Shows audio level visualization
  - Provides disconnect button

- **`vite.config.js`**: Build configuration
- **`index.html`**: Entry HTML file
- **`.env`**: Contains `VITE_LIVEKIT_URL`

### Frontend Step 2 Changes

- **`src/components/LiveKitModal.jsx`**: Enhanced version:
  - **Name input form**: User enters their name
  - **Dynamic token fetching**: Calls `/api/getToken` endpoint
  - **Loading states**: Shows connection progress

- **`vite.config.js`**: Updated with proxy configuration:
  ```javascript
  proxy: {
    '/api': {
      target: 'http://localhost:5001',
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api/, '')
    }
  }
  ```

### Code Overview

#### App Component (App.jsx)

```jsx
function App() {
  const [showSupport, setShowSupport] = useState(false);

  return (
    <div className="App">
      <h1>Welcome to OpenEdge Autos</h1>
      <button onClick={() => setShowSupport(true)}>
        Connect to Support
      </button>
      
      {showSupport && (
        <LiveKitModal setShowSupport={setShowSupport} />
      )}
    </div>
  );
}
```

#### LiveKit Modal - Step 1 (Hardcoded Token)

```jsx
<LiveKitRoom
  serverUrl={import.meta.env.VITE_LIVEKIT_URL}
  token="[TOKEN_HERE]"  // Hardcoded token from LiveKit dashboard
  connect={true}
  video={false}
  audio={true}
>
  <RoomAudioRenderer />
  <SimpleVoiceAssistant />
</LiveKitRoom>
```

#### LiveKit Modal - Step 2 (Dynamic Token)

```jsx
const [name, setName] = useState("");
const [token, setToken] = useState(null);

const getToken = async (userName) => {
  const response = await fetch(
    `/api/getToken?name=${encodeURIComponent(userName)}`
  );
  const token = await response.text();
  setToken(token);
};

// Show name form first
{isSubmittingName ? (
  <form onSubmit={handleNameSubmit}>
    <input 
      type="text" 
      value={name}
      onChange={(e) => setName(e.target.value)}
      placeholder="Your name"
    />
    <button type="submit">Connect</button>
  </form>
) : (
  <LiveKitRoom token={token} ...>
    {/* Voice assistant */}
  </LiveKitRoom>
)}
```

## What This Implements

**Production-Ready Web Interface**: This step creates a professional frontend:

- **React Framework**: Modern, component-based UI architecture
- **LiveKit Components**: Pre-built, tested components for voice/video
- **Vite Build System**: Fast development server with hot module replacement
- **Responsive Design**: Works on desktop and mobile browsers
- **User Experience**:
  - Clear call-to-action ("Connect to Support")
  - Visual feedback (connection status, audio levels)
  - Clean modal interface
  - Easy disconnect

**Two-Stage Approach**:
1. **Step 1**: Quick setup with static token for testing
2. **Step 2**: Production approach with dynamic token generation

## How This Helps Build Towards the Final Solution

This step is crucial for real-world deployment:

1. **User Accessibility**: 
   - No need for LiveKit Playground
   - Users access via standard web browser
   - Can be embedded in existing websites

2. **Branding and Customization**:
   - Custom styling to match company branding
   - Tailored user experience
   - Can add company logos, colors, messaging

3. **Security Foundation**:
   - Step 1: Quick testing with temporary tokens
   - Step 2: Secure token generation per session
   - Prepares for authentication integration

4. **Integration Path**:
   - Can be embedded in existing web applications
   - Can integrate with existing user authentication
   - API proxy pattern allows backend integration

5. **Production Architecture**:
   ```
   User Browser (React)
        ↓
   Token Server (Step 9)
        ↓
   LiveKit Cloud
        ↓
   Agent (Python)
        ↓
   OpenEdge Database
   ```

6. **Modern Development Workflow**:
   - Component-based development
   - Hot reload during development
   - Optimized production builds
   - Can add features like chat history, transcripts, etc.

## Testing This Step

### Step 1: Static Token Setup

1. **Create React App**:
   ```powershell
   cd C:\Work
   npm create vite@latest frontend -- --template react
   cd frontend
   ```

2. **Install Dependencies**:
   ```powershell
   npm install
   npm install @livekit/components-react @livekit/components-styles livekit-client --save
   ```

3. **Copy Example Files**: Copy contents from `Frontend/Step 1/`

4. **Get Static Token**:
   - Log into LiveKit Dashboard
   - Go to Settings → API Keys → Generate Token
   - Copy token into `LiveKitModal.jsx` where it says `[TOKEN_HERE]`

5. **Configure Environment**:
   Create `.env`:
   ```
   VITE_LIVEKIT_URL=https://your-project.livekit.cloud
   ```

6. **Run Frontend**:
   ```powershell
   npm run dev
   ```

7. **Test**:
   - Open browser to `http://localhost:5173`
   - Click "Connect to Support"
   - Should connect to your agent
   - Try voice interaction

### Step 2: Dynamic Token Setup

1. **Ensure Token Server is Running** (Step 9):
   ```powershell
   # In C:\Work\TokenServer
   py server.py
   ```

2. **Copy Step 2 Files**: Copy updated files from `Frontend/Step 2/`

3. **Update Vite Config**: Ensure proxy is configured (see Frontend Step 2)

4. **Restart Frontend**:
   ```powershell
   npm run dev
   ```

5. **Test**:
   - Open browser to `http://localhost:5173`
   - Click "Connect to Support"
   - Enter your name
   - Click "Connect"
   - Token should be fetched from server
   - Should connect to agent

## Key Concepts for OpenEdge Developers

- **React Components ≈ WebSpeed Objects**: Like building web UIs with embedded ABL, but component-based:
  ```abl
  /* ABL WebSpeed */
  RUN outputContentType("text/html").
  {&OUT} "<html>...".
  ```
  vs.
  ```jsx
  // React
  function App() {
    return <div>...</div>;
  }
  ```

- **State Management**: React's `useState` is like managing session variables in WebSpeed
  
- **NPM ≈ Procedure Library**: NPM packages are like .p files you can reuse

- **Vite ≈ ProxyGen**: Both are development tools that:
  - Serve your application
  - Auto-reload on changes
  - Build for production

- **API Proxy**: The Vite proxy rewrites URLs, similar to URL rewriting in PASOE

## Production Considerations

Before deploying to production:

1. **Build Optimized Version**:
   ```powershell
   npm run build
   ```
   Creates optimized files in `dist/` directory

2. **Serve with Web Server**: Use nginx, Apache, or IIS to serve the built files

3. **HTTPS Required**: LiveKit requires secure connections in production

4. **CORS Configuration**: Ensure token server allows requests from your frontend domain

5. **Environment Variables**: Use proper environment variables for production URLs

## What's Next

Step 9 implements the Token Server that provides secure, dynamic token generation for the frontend.
