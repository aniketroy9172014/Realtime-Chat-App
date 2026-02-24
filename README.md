# Realtime Chat App — Internal Architecture & Flow

**Overview**
- **Purpose:** Real-time 1:1 chat app with image attachments, auth, and online presence.
- **Stack:** Node/Express + Socket.IO + MongoDB (backend) and React + Vite + Zustand (frontend).

**Backend — High Level**
- **Entry:** [backend/src/index.js](backend/src/index.js) — configures Express, CORS, routes, and starts the HTTP server returned by the Socket.IO wrapper.
- **DB:** [backend/src/lib/db.js](backend/src/lib/db.js) — connects to MongoDB via Mongoose.
- **Socket layer:** [backend/src/lib/socket.js](backend/src/lib/socket.js) — creates `io` and stores a `userSocketMap` that maps userId -> socketId.
- **Routes / Controllers:**
  - Auth: [backend/src/routes/auth.route.js](backend/src/routes/auth.route.js) -> [backend/src/controllers/auth.controller.js](backend/src/controllers/auth.controller.js)
  - Messages: [backend/src/routes/message.route.js](backend/src/routes/message.route.js) -> [backend/src/controllers/message.controller.js](backend/src/controllers/message.controller.js)
- **Auth middleware:** [backend/src/middleware/auth.middleware.js](backend/src/middleware/auth.middleware.js) — verifies JWT from cookie and sets `req.user`.
- **Cloud uploads:** [backend/src/lib/cloudinary.js](backend/src/lib/cloudinary.js) — Cloudinary client used by controllers to upload base64 images.

**Frontend — High Level**
- **Entry:** [frontend/src/main.jsx](frontend/src/main.jsx) and [frontend/src/App.jsx](frontend/src/App.jsx).
- **HTTP client:** [frontend/src/lib/axios.js](frontend/src/lib/axios.js) — axios instance with `withCredentials: true` so cookies are sent.
- **Global state:** Zustand stores in [frontend/src/store/](frontend/src/store)
  - `useAuthStore` ([frontend/src/store/useAuthStore.js](frontend/src/store/useAuthStore.js)) — handles auth API calls, maintains `authUser`, `socket`, and `onlineUsers`, and manages socket connect/disconnect.
  - `useChatStore` ([frontend/src/store/useChatStore.js](frontend/src/store/useChatStore.js)) — loads sidebar users, fetches messages, sends messages, and subscribes to incoming socket `newMessage` events.
- **UI:** Pages and components under [frontend/src/pages](frontend/src/pages) and [frontend/src/components](frontend/src/components) (notably `Sidebar`, `ChatContainer`, `MessageInput`, `ChatHeader`).

**Authentication Flow**
- **Signup / Login:** frontend calls `/api/auth/signup` or `/api/auth/login` using `axiosInstance`.
- **Token Creation:** [backend/src/controllers/auth.controller.js](backend/src/controllers/auth.controller.js) calls `generateToken` in [backend/src/lib/utils.js](backend/src/lib/utils.js) which creates a JWT and sets an HTTP-only cookie named `jwt`.
- **Protected APIs:** `protectRoute` middleware in [backend/src/middleware/auth.middleware.js](backend/src/middleware/auth.middleware.js) reads `req.cookies.jwt`, verifies it and attaches user object to `req.user` for controllers to use.
- **Frontend check:** `useAuthStore.checkAuth()` calls GET `/api/auth/check` to verify session and then calls `connectSocket()` when `authUser` is set.

**Socket / Presence Flow**
- **Client connect:** `useAuthStore.connectSocket()` creates a socket.io client with a `query` containing `userId` (the logged-in user's `_id`). See [frontend/src/store/useAuthStore.js](frontend/src/store/useAuthStore.js).
- **Server handling:** when a socket connects, [backend/src/lib/socket.js](backend/src/lib/socket.js) reads `socket.handshake.query.userId` and stores `userSocketMap[userId] = socket.id`.
- **Online broadcast:** after connecting / disconnecting the server emits `getOnlineUsers` (array of userIds) to all clients so frontend updates `onlineUsers`.

**Message Send — End-to-End Flow (when a user sends a message)**
- **1 — Compose:** user enters text and/or selects an image in `MessageInput` ([frontend/src/components/MessageInput.jsx](frontend/src/components/MessageInput.jsx)). If an image is chosen it's read as a base64 string.
- **2 — POST to server:** `useChatStore.sendMessage()` posts to `POST /api/messages/send/:id` (where `:id` is the selected user's `_id`) via `axiosInstance`. See [frontend/src/store/useChatStore.js](frontend/src/store/useChatStore.js).
- **3 — Server processing:** `sendMessage` in [backend/src/controllers/message.controller.js](backend/src/controllers/message.controller.js):
  - If there's a base64 image, upload it to Cloudinary via [backend/src/lib/cloudinary.js](backend/src/lib/cloudinary.js) and get `secure_url`.
  - Create and save a new `Message` document ([backend/src/models/message.model.js](backend/src/models/message.model.js)) with `senderId`, `receiverId`, `text`, and `image`.
- **4 — Realtime notify:** server calls `getReceiverSocketId(receiverId)` (from [backend/src/lib/socket.js](backend/src/lib/socket.js)). If a socketId exists for that user, server emits `newMessage` to that socket with the saved message payload.
- **5 — Client receive:** front-end socket listener set in `useChatStore.subscribeToMessages()` listens for `newMessage` and — if the message `senderId` matches the currently selected chat — appends it to the chat messages list. The UI scrolls to the bottom in `ChatContainer`.
- **6 — Local echo:** after successful POST, `useChatStore.sendMessage()` sets the returned message into local `messages` so sender sees message immediately.

**Other notable flows**
- **Get users for Sidebar:** `useChatStore.getUsers()` -> GET `/api/messages/users` -> `getUsersForSidebar` controller returns all users except logged-in user.
- **Get conversation messages:** `useChatStore.getMessages(userId)` -> GET `/api/messages/:id` -> `getMessages` controller queries `Message` collection for both (sender, receiver) combinations.
- **Profile update:** `useAuthStore.updateProfile()` calls `PUT /api/auth/update-profile` -> controller uploads new profile image to Cloudinary and updates `User.profilePic`.

**Key Files (quick links)**
- Backend entry: [backend/src/index.js](backend/src/index.js)
- Socket server: [backend/src/lib/socket.js](backend/src/lib/socket.js)
- Auth controller: [backend/src/controllers/auth.controller.js](backend/src/controllers/auth.controller.js)
- Message controller: [backend/src/controllers/message.controller.js](backend/src/controllers/message.controller.js)
- Auth middleware: [backend/src/middleware/auth.middleware.js](backend/src/middleware/auth.middleware.js)
- DB connect: [backend/src/lib/db.js](backend/src/lib/db.js)
- Frontend entry: [frontend/src/main.jsx](frontend/src/main.jsx) and [frontend/src/App.jsx](frontend/src/App.jsx)
- Axios client: [frontend/src/lib/axios.js](frontend/src/lib/axios.js)
- Auth store: [frontend/src/store/useAuthStore.js](frontend/src/store/useAuthStore.js)
- Chat store: [frontend/src/store/useChatStore.js](frontend/src/store/useChatStore.js)
- Message UI: [frontend/src/components/MessageInput.jsx](frontend/src/components/MessageInput.jsx)
- Chat UI: [frontend/src/components/ChatContainer.jsx](frontend/src/components/ChatContainer.jsx)

**Sequence Summary (short)**
1. Browser authenticates -> server sets `jwt` cookie.
2. Frontend calls `checkAuth()` -> sets `authUser` -> `connectSocket()`.
3. Socket connection registers user's socketId on server; server broadcasts online users.
4. User selects contact -> frontend loads messages (GET `/api/messages/:id`).
5. User sends message -> POST `/api/messages/send/:id` -> server saves -> emits `newMessage` to receiver's socket.
6. Receiver's client listens to `newMessage` and appends it to active conversation.

**Environment / Run Notes**
- The backend expects `MONGODB_URI`, `JWT_SECRET`, and Cloudinary env vars (`CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`). See [backend/src/lib/cloudinary.js](backend/src/lib/cloudinary.js) and [backend/src/lib/db.js](backend/src/lib/db.js).
- `axiosInstance` in [frontend/src/lib/axios.js](frontend/src/lib/axios.js) uses `withCredentials: true` so the `jwt` cookie is included automatically.

If you want, I can also:
- Add a sequence diagram or Mermaid chart.
- Generate a compact architecture diagram.
- Create a CONTRIBUTING or DEV_RUN.md with exact start commands.
