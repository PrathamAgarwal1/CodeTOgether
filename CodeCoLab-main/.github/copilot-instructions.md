# CodeCoLab AI Coding Instructions

## Architecture Overview

**Stack:** MERN (MongoDB, Express, React, Node.js) with LiveKit for video conferencing.

### Core Components

1. **Rooms** - Collaborative workspaces where teams gather (DB model: owner + members array)
   - Socket.IO syncs real-time user presence (`roomUsers` global state)
   - Each room is a self-contained environment for projects, chat, and (now) optional video calls
   
2. **Projects** - Code projects within rooms with their own members subset
   - Created from templates (React App, Node.js API, Static HTML/CSS, Python Script)
   - Uses `templateManager.js` to generate initial files in the DB
   - Members are added/removed via `/api/projects/:id/members`

3. **LiveKit Integration** - Video/audio conferencing
   - Token generation: `/api/livekit/token` (roomName = roomId)
   - VideoGrid component wraps LiveKitRoom from `@livekit/components-react`
   - **CURRENT PATTERN:** Video is auto-started when entering RoomPage; leaving room leaves video

4. **Socket.IO Server** - Real-time sync (index.js ~120 lines)
   - Key events: `joinRoom`, `leaveRoom`, `chatMessage`, `timerUpdate`
   - `roomUsers` object tracks active connections per room
   - Old WebRTC signaling code present but unused (replaced by LiveKit)

### Data Flow: Room Lifecycle

1. User joins room → `RoomPage` mounts → `fetchRoomData()` queries API + initializes socket
2. Socket `joinRoom` adds user to `roomUsers[roomId]` → broadcast to all users
3. `VideoGrid` auto-mounts, fetches LiveKit token using roomId → video/audio streams
4. User leaves room → manual `handleLeaveRoom()` → emits socket `leaveRoom` → navigates to dashboard

## Key Implementation Details

### File References
- [collab-platform/client/src/pages/RoomPage.jsx](collab-platform/client/src/pages/RoomPage.jsx) - Main room UI (508 lines, handles timer, chat, projects, sidebar)
- [collab-platform/client/src/components/rooms/VideoGrid.jsx](collab-platform/client/src/components/rooms/VideoGrid.jsx) - LiveKit wrapper (auto-fetches token, renders video)
- [collab-platform/server/models/Room.js](collab-platform/server/models/Room.js) - Simple schema: name, description, owner, members[]
- [collab-platform/server/index.js](collab-platform/server/index.js) - Socket handlers + express setup
- [collab-platform/server/routes/livekit.js](collab-platform/server/routes/livekit.js) - Token generation with AccessToken SDK

### Critical Patterns

**Socket User Tracking:**
```javascript
// server/index.js
const roomUsers = {}; // { roomId: [ { userId, username, socketId } ] }
socket.on('joinRoom', ({ roomId, user }) => {
    socket.join(roomId);
    roomUsers[roomId].push({ userId, username, socketId });
    io.to(roomId).emit('roomUsers', roomUsers[roomId]); // Broadcast updated list
});
```

**Component State Sync:**
- RoomPage component state (timer, tasks, projects) is socket-synced
- Timer broadcasts via `timerUpdate` event; all users receive same state
- Chat persists to DB via Message model on `chatMessage` event

**Project Creation Flow:**
1. User clicks "New Project" modal → `CreateProjectModal` posts to `/api/projects`
2. Backend creates Project doc + generates files via `templateManager` 
3. Socket emits `room-update` → RoomPage re-fetches projects
4. Optimistic UI: new project appears immediately in sidebar

## Common Workflows

**Add a new Room feature:**
1. Extend RoomSchema if data persistence needed
2. Update RoomPage component state + socket listeners
3. Broadcast changes via `socket.emit('room-update', ...)`
4. Dashboard or other components listen for `room-update`

**Modify Socket Events:**
- Both client (socket.js) and server (index.js) must emit/listen for same event names
- Always emit from one side, listen on the other
- Include `roomId` in payload for room-specific broadcasts

**Add Video Feature:**
- Token is requested with `roomId` as roomName parameter
- LiveKit components (VideoGrid) handle rendering
- Control bar is built-in from `@livekit/components-react`

## Development Notes

- **No linter:** ESLint config exists but not enforced in package.json scripts
- **No tests:** Project lacks test framework; QA via manual testing
- **Env vars:** Server reads from `.env` (MONGO_URI, LIVEKIT_API_KEY/SECRET, PORT)
- **CORS:** Hardcoded to localhost:5173 in server; update for production
- **Git:** .github folder is empty except new instructions file

## Next Steps for New Developers

1. Read [RoomPage.jsx](collab-platform/client/src/pages/RoomPage.jsx) end-to-end to understand room UX
2. Trace socket flow: client socket.js → server index.js handlers
3. Check [server/models](collab-platform/server/models) for data structure assumptions
4. Run `npm install && npm run dev` in client/ and `npm install && node index.js` in server/
