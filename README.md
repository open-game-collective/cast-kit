# 📺 Cast Kit

> Add TV casting to your web games in minutes

Cast Kit is a TypeScript library for adding TV casting capabilities to web games. It transforms phones into controllers and TVs into displays, enabling immersive gameplay experiences on larger screens.

[![npm version](https://img.shields.io/npm/v/@open-game-collective/cast-kit.svg?style=flat)](https://www.npmjs.com/package/@open-game-collective/cast-kit)
[![TypeScript](https://img.shields.io/badge/%3C%2F%3E-TypeScript-%230074c1.svg)](https://www.typescriptlang.org/)
[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## Features

- 🎮 **Phone as Controller** - Transform the player's phone into a game controller
- 📺 **TV as Display** - Stream your game to any Chromecast-enabled device
- 🌐 **Web-Based** - No native app required, works in the browser
- 🚀 **Easy Integration** - Add casting with just a few lines of code
- 🪄 **React Support** - Ready-to-use React components and hooks
- 🔎 **Inspector Tools** - Debug your cast sessions with built-in tools

## Architecture

Cast Kit consists of three main components:

1. **Client SDK**: Runs in the player's browser and provides the API for initiating and managing cast sessions
2. **Server SDK**: Runs on Cloudflare Workers and handles session management, communication, and coordination
3. **Renderer Service**: Runs in Google Cloud Run and manages browser instances for streaming game content to TVs

```mermaid
graph TD
    subgraph "Game Client"
        ClientSDK[Cast Kit Client SDK]
        GameUI[Game Interface]
    end
    
    subgraph "Cloudflare Workers"
        ServerSDK[Cast Kit Server SDK]
        Workflow[Durable Workflow]
    end
    
    subgraph "Google Cloud Run"
        Renderer[Renderer Service]
        Browser[Headless Browser]
    end
    
    subgraph "User Devices"
        TV[TV / Chromecast]
        Phone[Phone]
    end
    
    GameUI -->|Uses| ClientSDK
    ClientSDK -->|API Calls| ServerSDK
    ServerSDK -->|Creates| Workflow
    Workflow -->|HTTP Calls| Renderer
    Renderer -->|Launches| Browser
    Browser -->|WebRTC Stream| TV
    Phone -->|Control Input| ClientSDK
```

## Prerequisites

- **Cloudflare Workers** account with Workflows enabled
- **Google Cloud Run** access for the Renderer Service (or use our hosted service)
- **Node.js** 18+ and npm/yarn
- **TypeScript** 4.5+ (recommended)

## Quick Start

### 1. Install the package

```bash
npm install @open-game-collective/cast-kit
```

### 2. Client-Side Integration

```jsx
import { createCastClient } from '@open-game-collective/cast-kit/client';
import { CastButton } from '@open-game-collective/cast-kit/react';

// Create a client
const castClient = createCastClient({
  host: 'triviajam.tv', // Domain where your Cast Router service is running
});

// In your React component
function GameUI() {
  return (
    <div>
      <h1>Your Game</h1>
      <CastButton client={castClient} 
                 gameUrl="https://triviajam.tv/cast?gameId=123&roomCode=ABCD" />
    </div>
  );
}
```

### 3. Server-Side Integration (Cloudflare Workers)

```typescript
// worker.ts
import { createCastRouter } from '@open-game-collective/cast-kit/server';
import { createKVStorageHooks } from '@open-game-collective/cast-kit/server/storage';

export interface Env {
  CAST_SESSIONS: KVNamespace;
  RENDER_HOST: string;
  CAST_SESSION_WORKFLOW: any;
}

// Use the class pattern for better performance with singleton router
export default class Worker {
  // Create the cast router once when the worker is instantiated
  private castRouter: ReturnType<typeof createCastRouter>;
  
  constructor(env: Env) {
    // Create storage hooks using Cloudflare KV
    const storageHooks = createKVStorageHooks(env.CAST_SESSIONS);
    
    // Create the cast router once 
    this.castRouter = createCastRouter({
      renderHost: env.RENDER_HOST,
      storageHooks,
      workflowBinding: env.CAST_SESSION_WORKFLOW
    });
  }
  
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    
    // Handle cast routes
    if (url.pathname.startsWith('/api/cast/')) {
      return this.castRouter(request);
    }
    
    return new Response('Not Found', { status: 404 });
  }
}
```

## Detailed Setup Guide

### Client-Side Setup

The client-side SDK manages cast sessions and provides a UI for the player to initiate and control casting.

#### Basic Client Setup

```typescript
import { createCastClient } from '@open-game-collective/cast-kit/client';

// Create a cast client
const castClient = createCastClient({
  host: 'triviajam.tv',  // Your game's domain
});

// Create a broadcast session
async function startCasting() {
  try {
    // Generate a broadcast URL with game state
    const broadcastUrl = new URL('https://triviajam.tv/cast');
    broadcastUrl.searchParams.append('gameId', '123');
    broadcastUrl.searchParams.append('roomCode', 'ABCD');
    
    // Start broadcasting
    await castClient.createBroadcastSession({
      gameUrl: broadcastUrl.toString()
    });
    
    console.log('Started casting!');
  } catch (error) {
    console.error('Casting error:', error);
  }
}

// End the session
async function stopCasting() {
  await castClient.endSession();
  console.log('Stopped casting');
}
```

#### React Integration

Cast Kit provides React components and hooks for easy integration:

```jsx
import { 
  createCastKitContext, 
  CastButton, 
  CastStatus 
} from '@open-game-collective/cast-kit/react';
import { createCastClient } from '@open-game-collective/cast-kit/client';

// Create the context
export const CastKitContext = createCastKitContext();

// Create the client
const castClient = createCastClient({
  host: 'triviajam.tv'
});

// In your App component
function App() {
  return (
    <CastKitContext.Provider client={castClient}>
      <GameUI />
    </CastKitContext.Provider>
  );
}

// In your game UI component
function GameUI() {
  // Access client methods
  const client = CastKitContext.useClient();
  
  // Subscribe to specific state
  const isCasting = CastKitContext.useSelector(state => 
    state.session?.status === 'active'
  );
  
  // Create a URL with game state
  const getBroadcastUrl = () => {
    const url = new URL('https://triviajam.tv/cast');
    url.searchParams.append('gameId', '123');
    url.searchParams.append('roomCode', 'ABCD');
    return url.toString();
  };
  
  return (
    <div>
      <h1>Your Game</h1>
      
      {!isCasting ? (
        <CastButton onCast={() => {
          client.createBroadcastSession({
            gameUrl: getBroadcastUrl()
          });
        }} />
      ) : (
        <div>
          <CastStatus />
          <button onClick={() => client.endSession()}>
            Stop Casting
          </button>
        </div>
      )}
    </div>
  );
}
```

#### Integration with Auth Kit

You can integrate Cast Kit with Auth Kit to ensure that authenticated sessions are maintained when broadcasting:

```jsx
import { 
  createCastKitContext, 
  CastButton 
} from '@open-game-collective/cast-kit/react';
import { createCastClient } from '@open-game-collective/cast-kit/client';
import { AuthContext } from '@open-game-collective/auth-kit/react';

// Create the cast context
export const CastKitContext = createCastKitContext();

// In your game component
function GameUI() {
  // Get auth client
  const authClient = AuthContext.useClient();
  
  // Get cast client
  const castClient = CastKitContext.useClient();
  
  // Get auth state
  const { isAuthenticated, userId } = AuthContext.useSelector(state => ({
    isAuthenticated: !!state.userId,
    userId: state.userId
  }));
  
  // Function to handle casting with auth
  const handleCast = async () => {
    try {
      // If authenticated, get a web auth code
      if (isAuthenticated) {
        // Get web auth code from Auth Kit
        const { code } = await authClient.getWebAuthCode();
        
        // Create URL with auth code for the cast page
        const castUrl = new URL('https://triviajam.tv/cast');
        castUrl.searchParams.append('gameId', '123');
        castUrl.searchParams.append('roomCode', 'ABCD');
        castUrl.searchParams.append('code', code); // Add auth code to URL
        
        // Start broadcasting with authenticated session
        await castClient.createBroadcastSession({
          gameUrl: castUrl.toString()
        });
      } else {
        // Handle unauthenticated case
        const castUrl = new URL('https://triviajam.tv/cast');
        castUrl.searchParams.append('gameId', '123');
        castUrl.searchParams.append('roomCode', 'ABCD');
        
        await castClient.createBroadcastSession({
          gameUrl: castUrl.toString()
        });
      }
    } catch (error) {
      console.error('Error starting cast:', error);
    }
  };
  
  return (
    <div>
      <h1>Your Game</h1>
      <CastButton onCast={handleCast} />
      {isAuthenticated && <p>Casting as user: {userId}</p>}
    </div>
  );
}

// In your App component
function App() {
  // Initialize clients
  const castClient = createCastClient({
    host: 'triviajam.tv'
  });
  
  return (
    <AuthContext.Provider client={authClient}>
      <CastKitContext.Provider client={castClient}>
        <GameUI />
      </CastKitContext.Provider>
    </AuthContext.Provider>
  );
}
```

On your cast page, you'll need to handle the auth code:

```jsx
// pages/cast.tsx
import { useEffect, useState } from 'react';
import { useGameState } from '../hooks/useGameState';

export default function CastPage() {
  const [params, setParams] = useState(null);
  
  useEffect(() => {
    const url = new URL(window.location.href);
    const urlParams = new URLSearchParams(url.search);
    
    // Extract params including auth code
    setParams({
      gameId: urlParams.get('gameId'),
      roomCode: urlParams.get('roomCode'),
      authCode: urlParams.get('code') // Get auth code from URL
    });
    
    // If we have an auth code, Auth Kit middleware will automatically 
    // handle the authentication by setting the appropriate cookies
    // This happens server-side in your Cloudflare Worker
  }, []);
  
  // Connect to game state if we have params
  const { gameState, loading, error } = useGameState(
    params?.gameId, 
    params?.roomCode
  );
  
  if (!params) return <div>Loading...</div>;
  if (loading) return <div>Connecting to game...</div>;
  if (error) return <div>Error connecting to game: {error.message}</div>;
  
  return (
    <div className="tv-display">
      {/* Game display optimized for TV */}
      <GameDisplay gameState={gameState} isTvMode={true} />
    </div>
  );
}
```

This approach ensures that:

1. When a user initiates casting, their authentication status is preserved
2. The Chromecast receiver gets an authenticated session via the web auth code
3. The auth state is properly handled on the server side through Auth Kit's middleware
4. The Cast Router maintains session information to keep the connection alive

### Server-Side Setup (Cloudflare Workers)

The server-side component requires Cloudflare Workers with Workflows enabled. For optimal performance, initialize the router once when the worker is instantiated.

#### 1. Configure wrangler.toml

```toml
name = "your-game-cast-router"
main = "src/worker.ts"
compatibility_date = "2023-12-01"

# KV Namespace for storing session data (optional - you can use any storage solution)
[[kv_namespaces]]
binding = "CAST_SESSIONS"
id = "your-kv-namespace-id"  # Create this in the Cloudflare dashboard

# Environment variables
[vars]
RENDER_HOST = "https://renderer.triviajam.tv"  # Ask for access or host your own

# Workflow definition
[[workflows]]
name = "cast-session-workflow"
binding = "CAST_SESSION_WORKFLOW"
```

#### 2. Implement the Worker Using the Singleton Pattern

```typescript
// src/worker.ts
import { createCastRouter } from '@open-game-collective/cast-kit/server';
import { createKVStorageHooks } from '@open-game-collective/cast-kit/server/storage';
import { CastSessionWorkflow } from '@open-game-collective/cast-kit/server/workflows';

// Define proper typing for the Workflow binding
interface WorkflowBinding<T> {
  create: (payload: T) => Promise<WorkflowInstance>;
  get: (id: string) => Promise<WorkflowInstance | null>;
}

interface WorkflowInstance {
  id: string;
  status: () => Promise<string>;
  terminate: () => Promise<void>;
}

interface CastSessionParams {
  sessionId: string;
  gameUrl: string;
  options?: Record<string, any>;
}

export interface Env {
  CAST_SESSIONS: KVNamespace;
  RENDER_HOST: string;
  CAST_SESSION_WORKFLOW: WorkflowBinding<CastSessionParams>;
}

// Export the workflow class for Cloudflare Workflows
export { CastSessionWorkflow };

// Use the WorkerEntrypoint pattern for better performance
export default class Worker {
  // Create the cast router once when the worker is instantiated
  private castRouter: ReturnType<typeof createCastRouter>;
  
  constructor(env: Env) {
    // Create storage hooks using Cloudflare KV (this is just one storage option)
    const storageHooks = createKVStorageHooks(env.CAST_SESSIONS);
    
    // Create the cast router once 
    this.castRouter = createCastRouter({
      renderHost: env.RENDER_HOST,
      storageHooks, // You can use any storage implementation here
      workflowBinding: env.CAST_SESSION_WORKFLOW
    });
  }
  
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    
    // Handle cast routes
    if (url.pathname.startsWith('/api/cast/')) {
      return this.castRouter(request);
    }
    
    // Handle other routes
    // For example, integrate with other systems like actor-kit if needed
    
    return new Response('Not Found', { status: 404 });
  }
}
```

#### Performance Benefits

Using the singleton pattern to initialize the router once offers several advantages:

1. **Reduced Overhead**: Avoids recreating the router and its dependencies on every request
2. **Faster Response Times**: Eliminates initialization time from request handling
3. **Resource Efficiency**: Creates storage hooks and other resources only once
4. **Better Integration**: Easier to integrate with other routers and services

### Broadcast Page

The broadcast page is what viewers will see on the TV. This page should be optimized for TV display and connect to your game's state management system.

```jsx
// pages/cast.tsx or similar
import { useEffect, useState } from 'react';
import { useGameState } from '../hooks/useGameState';

export default function CastPage() {
  // Get game parameters from URL
  const [params, setParams] = useState(null);
  
  useEffect(() => {
    const urlParams = new URLSearchParams(window.location.search);
    setParams({
      gameId: urlParams.get('gameId'),
      roomCode: urlParams.get('roomCode')
    });
  }, []);
  
  // Connect to game state if we have params
  const { gameState, loading, error } = useGameState(params?.gameId, params?.roomCode);
  
  if (!params) return <div>Loading...</div>;
  if (loading) return <div>Connecting to game...</div>;
  if (error) return <div>Error connecting to game: {error.message}</div>;
  
  return (
    <div className="tv-display">
      {/* Game display optimized for TV */}
      <GameDisplay gameState={gameState} isTvMode={true} />
    </div>
  );
}
```

## API Reference

### Client SDK

```typescript
interface CastClient {
  // Create a new broadcast session
  createBroadcastSession(options: {
    gameUrl: string,      // URL to your game's broadcast page
    sessionData?: any     // Optional data to include with the session
  }): Promise<CastSession>;
  
  // Get the current session
  getSession(): CastSession | null;
  
  // End the current session
  endSession(): Promise<void>;
  
  // Get the current state
  getState(): CastState;
  
  // Subscribe to state changes
  subscribe(listener: (state: CastState) => void): () => void;
}

interface CastState {
  isCreatingSession: boolean;
  error: Error | null;
  session: CastSession | null;
}

interface CastSession {
  sessionId: string;
  status: 'created' | 'connecting' | 'active' | 'error' | 'terminated';
  createdAt: string;
  error?: string;
}
```

### React Components

```typescript
// CastButton props
interface CastButtonProps {
  onCast?: () => void;        // Called when the button is clicked
  disabled?: boolean;         // Disable the button
  className?: string;         // CSS class
  label?: string;             // Button text
}

// CastStatus props
interface CastStatusProps {
  className?: string;         // CSS class
  showEndButton?: boolean;    // Show the "End Casting" button
  onEnd?: () => void;         // Called when end button is clicked
}
```

### Server SDK

```typescript
interface CastRouterOptions {
  renderHost: string;         // URL to the Renderer service
  storageHooks?: StorageHooks; // Custom storage implementation
  workflowBinding: any;       // Cloudflare Workflow binding
}

// Create a router for handling cast requests
function createCastRouter(options: CastRouterOptions): (request: Request) => Promise<Response>;

// Create storage hooks using Cloudflare KV
function createKVStorageHooks(namespace: KVNamespace): StorageHooks;
```

## Advanced Usage

### Custom Storage

Cast Kit is completely unopinionated about storage solutions. You need to provide a storage implementation that matches the `StorageHooks` interface:

```typescript
import { StorageHooks } from '@open-game-collective/cast-kit/server';

// Implement your own storage solution
const customStorage: StorageHooks = {
  saveSession: async (session) => {
    // Save session to your storage
  },
  
  getSession: async (sessionId) => {
    // Get session from your storage
  },
  
  updateSession: async (sessionId, updates) => {
    // Update session in your storage
  },
  
  deleteSession: async (sessionId) => {
    // Delete session from your storage
  },
  
  listActiveSessions: async () => {
    // List active sessions from your storage
  }
};

// Use your custom storage with the router
const castRouter = createCastRouter({
  renderHost: env.RENDER_HOST,
  storageHooks: customStorage, // Your custom implementation
  workflowBinding: env.CAST_SESSION_WORKFLOW
});
```

We provide a KV implementation for convenience, but you can use any storage solution:

```typescript
// Using the KV implementation
import { createKVStorageHooks } from '@open-game-collective/cast-kit/server/storage';

// Create storage hooks using Cloudflare KV
const kvStorage = createKVStorageHooks(env.CAST_SESSIONS);

// Using Durable Objects (example - implementation not included in Cast Kit)
const durableObjectStorage = createDurableObjectStorageHooks(env.CAST_SESSION_DO);

// Using a database (example - implementation not included in Cast Kit)
const dbStorage = createDatabaseStorageHooks({
  connectionString: env.DATABASE_URL
});
```

You can choose the storage solution that best fits your needs and infrastructure.

### Debugging

Cast Kit includes debugging tools to help troubleshoot cast issues:

```typescript
// Enable verbose logging
const castClient = createCastClient({
  host: 'triviajam.tv',
  debug: true
});

// Access logs
const logs = castClient.getLogs();
console.table(logs);

// Monitor session events
castClient.subscribe((state) => {
  console.log('Cast state updated:', state);
});
```

## Game Setup Considerations

### 1. Create a TV-Optimized Game View

- Design a fullscreen layout suitable for TV viewing
- Use larger text and UI elements
- Ensure good visibility from a distance
- Optimize for 16:9 aspect ratio
- Test on actual TV devices

### 2. Implement State Synchronization

Your broadcast page needs to connect to your game's state management system:

```typescript
// Example connection to a game server
function connectToGameState(gameId, roomCode) {
  const socket = new WebSocket(`wss://triviajam.tv/api/game/${gameId}/room/${roomCode}`);
  
  socket.onmessage = (event) => {
    const gameState = JSON.parse(event.data);
    // Update UI with new game state
    updateGameDisplay(gameState);
  };
  
  return () => socket.close(); // Return cleanup function
}
```

### 3. Performance Optimization

Ensure your broadcast page performs well:

- Limit animations and effects that might cause performance issues
- Optimize rendering for smooth frame rates
- Minimize network requests
- Use efficient canvas rendering when possible
- Test on lower-end devices

## Troubleshooting

### Common Issues

#### Cast Button Doesn't Work

- Ensure your Chromecast device is on the same network
- Check browser compatibility (Chrome is recommended)
- Verify that the Google Cast extension is enabled
- Try restarting your Chromecast device

#### Streaming Issues

- Check network connectivity and stability
- Ensure your broadcast page loads correctly in a standalone browser
- Verify WebRTC is not blocked by firewalls or security settings
- Test with a simpler game view to isolate performance issues

#### Server Integration Problems

- Confirm Cloudflare Workers is properly configured
- Verify KV namespace is correctly set up
- Check that Workflows is enabled on your account
- Ensure the Renderer service URL is correct

#### Error Handling

Cast Kit provides detailed error information:

```typescript
castClient.subscribe((state) => {
  if (state.error) {
    console.error('Cast error:', state.error);
    // Show user-friendly error message
    showErrorToUser(state.error.message);
  }
});
```

## Limitations

- **Browser Support**: Cast Kit works best in Chrome and Chrome-based browsers
- **Cloudflare Requirement**: Server-side components require Cloudflare Workers with Workflows
- **Network Dependencies**: Stable network connection is required for smooth streaming
- **WebRTC Constraints**: Performance depends on the capabilities of the broadcasting and receiving devices

## FAQ

### Q: Do I need to implement my own Chromecast receiver app?
A: No. Cast Kit uses a universal receiver that works with any game. You just need to create a broadcast page on your domain.

### Q: How do I handle player input while casting?
A: Your game client becomes a controller when casting is active. Use your existing input handling code, but display the controller UI instead of the game UI.

### Q: Is there a cost to using Cast Kit?
A: Cast Kit itself is free and open source. You'll need a Cloudflare Workers account (which has a generous free tier) and access to the Renderer service (which can be self-hosted or used as a service).

### Q: How many concurrent cast sessions can I support?
A: This depends on your Cloudflare Workers limits and the Renderer service capacity. The default Renderer configuration supports 50 concurrent sessions.

### Q: Can I use Cast Kit with any game?
A: Yes, Cast Kit works with any web-based game that can expose a broadcast view. Games using canvas, WebGL, or DOM-based rendering are all supported.

## Support and Community

- GitHub Issues: [https://github.com/open-game-collective/cast-kit/issues](https://github.com/open-game-collective/cast-kit/issues)
- Discord Community: [Join our Discord](https://discord.gg/opengamecollective)
- Email Support: [support@triviajam.tv](mailto:support@triviajam.tv)

## License

Cast Kit is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

Built with ❤️ by the [Open Game Collective](https://github.com/open-game-collective) 
