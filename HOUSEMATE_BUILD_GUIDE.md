# HouseMate — Complete Vibe-Coding Build Guide
### Stack: Express API + React/Vite PWA · JavaScript · Monorepo

> **How to use this guide:** Work through phases sequentially. Each phase is broken into focused tasks. Give the AI **one task at a time**. Paste relevant context (schemas, existing files) when needed. Never skip a phase — each one builds on the last.

---

## Tech Stack Reference

| Layer | Technology |
|---|---|
| Monorepo Structure | Single repo, `backend/` + `frontend/` folders |
| Backend Framework | Express.js (JavaScript) |
| Backend ORM | Prisma |
| Backend WebSockets | Socket.io |
| Backend Auth | HTTP-only session cookie (custom middleware) |
| Backend File Storage | Local disk (`backend/uploads/`) |
| Frontend Framework | React + Vite (JavaScript) |
| Frontend Routing | React Router v6 (`createBrowserRouter`) |
| Frontend Styling | Tailwind CSS |
| Frontend Components | shadcn/ui |
| Frontend State | Zustand |
| Frontend Data Fetching | TanStack Query (custom hooks) |
| Frontend PWA | Vite PWA Plugin |
| Database | PostgreSQL (Docker) |
| Containerization | Docker + Docker Compose (dev & prod) |

---

## Monorepo Folder Structure (Target)

```
housemate/
├── package.json                  # root: workspace scripts only
├── docker-compose.dev.yml
├── docker-compose.prod.yml
├── .env.example
│
├── backend/
│   ├── Dockerfile.dev
│   ├── Dockerfile.prod
│   ├── package.json
│   ├── nodemon.json
│   ├── .env                      # never committed
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── seed.js
│   ├── uploads/                  # volume-mounted; gitignored content
│   │   └── .gitkeep
│   └── src/
│       ├── index.js              # app entry point
│       ├── app.js                # express app setup
│       ├── socket.js             # socket.io setup & handlers
│       ├── middleware/
│       │   ├── auth.js           # session verification middleware
│       │   └── errorHandler.js
│       ├── lib/
│       │   ├── prisma.js         # prisma singleton
│       │   ├── session.js        # cookie/session helpers
│       │   ├── permissions.js    # permission resolution logic
│       │   └── notifications.js  # notification service
│       └── routes/
│           ├── auth.js
│           ├── families.js
│           ├── roles.js
│           ├── members.js
│           ├── chores.js
│           ├── announcements.js
│           ├── notifications.js
│           └── uploads.js
│
└── frontend/
    ├── Dockerfile.dev
    ├── Dockerfile.prod
    ├── package.json
    ├── vite.config.js
    ├── tailwind.config.js
    ├── postcss.config.js
    ├── index.html
    ├── public/
    │   ├── manifest.json
    │   └── icons/                # PWA icons
    └── src/
        ├── main.jsx
        ├── App.jsx
        ├── router.jsx            # createBrowserRouter config
        ├── api/
        │   └── client.js         # axios base client
        ├── hooks/                # all TanStack Query custom hooks
        │   ├── useAuth.js
        │   ├── useFamilies.js
        │   ├── useRoles.js
        │   ├── useMembers.js
        │   ├── useChores.js
        │   ├── useAnnouncements.js
        │   └── useNotifications.js
        ├── stores/               # Zustand stores
        │   ├── authStore.js
        │   ├── familyStore.js
        │   └── notificationStore.js
        ├── components/
        │   ├── ui/               # shadcn generated components
        │   ├── layout/
        │   │   ├── AppShell.jsx
        │   │   ├── Sidebar.jsx
        │   │   ├── BottomNav.jsx  # mobile nav
        │   │   ├── Header.jsx
        │   │   └── NotificationBell.jsx
        │   ├── chores/
        │   ├── announcements/
        │   └── shared/
        │       ├── EmptyState.jsx
        │       ├── PageLoader.jsx
        │       └── ProtectedRoute.jsx
        └── pages/
            ├── auth/
            │   ├── LoginPage.jsx
            │   └── RegisterPage.jsx
            ├── DashboardPage.jsx
            ├── family/
            │   ├── FamilySetupPage.jsx
            │   └── FamilySettingsPage.jsx
            ├── chores/
            │   ├── ChoresPage.jsx
            │   └── ChoreDetailPage.jsx
            ├── AnnouncementsPage.jsx
            ├── MembersPage.jsx
            └── RolesPage.jsx
```

---

---

# PHASE 0 — Project Setup & Infrastructure

> **Goal:** A running monorepo with both services booting via Docker. No features yet — just the skeleton.

---

### Task 0.1 — Root Monorepo Scaffold

**Prompt the AI:**
```
Create a monorepo project called "housemate" with the following structure.
Use plain JavaScript (no TypeScript).

1. Root `package.json`:
   - No dependencies of its own
   - Scripts section with shortcuts:
     "dev": "docker compose -f docker-compose.dev.yml up --build"
     "prod": "docker compose -f docker-compose.prod.yml up --build"
     "down": "docker compose -f docker-compose.dev.yml down"
     "down:prod": "docker compose -f docker-compose.prod.yml down"
     "shell:backend": "docker exec -it housemate-backend sh"
     "db:migrate": "docker exec housemate-backend npx prisma migrate dev"
     "db:seed": "docker exec housemate-backend node prisma/seed.js"
     "db:studio": "docker exec -it housemate-backend npx prisma studio --hostname 0.0.0.0"

2. Create `backend/` folder with an empty `backend/package.json` (name: "housemate-backend")

3. Create `frontend/` folder with an empty `frontend/package.json` (name: "housemate-frontend")

4. Create a root `.gitignore` covering:
   node_modules, .env, uploads/*, !uploads/.gitkeep, dist, build

5. Create `.env.example` at the root:
   # Backend
   DATABASE_URL=postgresql://housemate:housemate@db:5432/housemate
   SESSION_SECRET=change_this_to_a_long_random_string_min_32_chars
   NODE_ENV=development
   PORT=4000
   FRONTEND_URL=http://localhost:5173
   UPLOAD_DIR=./uploads
   MAX_IMAGE_SIZE_MB=10
   MAX_VIDEO_SIZE_MB=100

   # Database
   POSTGRES_USER=housemate
   POSTGRES_PASSWORD=housemate
   POSTGRES_DB=housemate
```

---

### Task 0.2 — Backend Express Setup

**Prompt the AI:**
```
Set up the Express backend in the `backend/` folder.
Use plain JavaScript (no TypeScript).

Install these dependencies (write them into backend/package.json):
  dependencies:
    express, cors, cookie-parser, @prisma/client,
    bcryptjs, uuid, multer, socket.io, dotenv, zod, cookie
  devDependencies:
    prisma, nodemon

Create these files:

1. `backend/src/app.js`:
   - Import express, cors, cookie-parser, dotenv (call dotenv.config() at the top)
   - Create an Express app
   - Apply middleware: express.json(), express.urlencoded({ extended: true }),
     cors({ credentials: true, origin: process.env.FRONTEND_URL }), cookie-parser()
   - Mount health check: GET /api/health → res.json({ status: "ok" })
   - Export the app (do NOT call app.listen here)

2. `backend/src/index.js`:
   - Import http module and app
   - Create httpServer from app (http.createServer(app))
   - Listen on process.env.PORT (default 4000)
   - Log "Backend running on port X"

3. `backend/nodemon.json`:
   {
     "watch": ["src", "prisma"],
     "ext": "js,json",
     "ignore": ["node_modules"],
     "exec": "node src/index.js"
   }

4. Add to backend/package.json scripts:
   "start": "node src/index.js"
   "dev": "nodemon"

5. `backend/prisma/schema.prisma`:
   - datasource db: provider = "postgresql", url = env("DATABASE_URL")
   - generator: prisma-client-js
   - Add comment: // Models will be added per phase

6. `backend/src/lib/prisma.js`:
   - Export a PrismaClient singleton using the global pattern:
     if (!global.__prisma) global.__prisma = new PrismaClient()
     module.exports = global.__prisma
   - This prevents multiple instances during nodemon restarts

7. Create `backend/uploads/.gitkeep` (empty file to commit the uploads directory)
```

---

### Task 0.3 — Frontend Vite + React Setup

**Prompt the AI:**
```
Set up the React + Vite frontend in the `frontend/` folder.
Use plain JavaScript (no TypeScript, no .tsx files).

Write these dependencies into frontend/package.json:
  dependencies:
    react, react-dom, react-router-dom, @tanstack/react-query,
    zustand, axios, socket.io-client, lucide-react, clsx, tailwind-merge, class-variance-authority
  devDependencies:
    vite, @vitejs/plugin-react, tailwindcss, postcss, autoprefixer, vite-plugin-pwa

Create these files:

1. `frontend/vite.config.js`:
   - Use @vitejs/plugin-react
   - Use VitePWA plugin with:
     registerType: 'autoUpdate'
     manifest: { name: 'HouseMate', short_name: 'HouseMate', theme_color: '#4f46e5', ... }
     workbox: { globPatterns: ['**/*.{js,css,html,ico,png,svg}'] }
   - server.proxy: { '/api': { target: 'http://localhost:4000', changeOrigin: true } }
     (this proxies /api calls in dev to the backend)

2. `frontend/tailwind.config.js`:
   - content: ['./index.html', './src/**/*.{js,jsx}']
   - darkMode: 'class'
   - extend: {} (leave empty for now, shadcn will add to this)

3. `frontend/postcss.config.js`:
   - plugins: tailwindcss, autoprefixer

4. `frontend/index.html`:
   Standard Vite HTML, includes:
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <meta name="theme-color" content="#4f46e5">
   Script src="/src/main.jsx"

5. `frontend/src/index.css`:
   @tailwind base;
   @tailwind components;
   @tailwind utilities;

6. `frontend/src/main.jsx`:
   - Create a QueryClient with defaultOptions: { queries: { retry: 1, refetchOnWindowFocus: false } }
   - Render <QueryClientProvider client={queryClient}><App /></QueryClientProvider> into #root
   - Import './index.css'

7. `frontend/src/App.jsx`:
   - Render <h1 className="text-3xl font-bold">HouseMate is running</h1> as placeholder

8. `frontend/src/api/client.js`:
   - Create an axios instance:
     baseURL: '/api'
     withCredentials: true
   - Export as default
   - Add a response interceptor:
     On 401 → window.location.href = '/login'
     On other errors → re-throw

9. frontend/package.json scripts:
   "dev": "vite"
   "build": "vite build"
   "preview": "vite preview"
```

---

### Task 0.4 — Docker: Dev Environment

**Prompt the AI:**
```
Create Docker configuration for the DEVELOPMENT environment.
Both services must hot-reload when source files change on the host.

1. `backend/Dockerfile.dev`:
   FROM node:20-alpine
   WORKDIR /app
   COPY package.json .
   RUN npm install
   COPY . .
   RUN npx prisma generate
   EXPOSE 4000
   CMD ["npm", "run", "dev"]

2. `frontend/Dockerfile.dev`:
   FROM node:20-alpine
   WORKDIR /app
   COPY package.json .
   RUN npm install
   COPY . .
   EXPOSE 5173
   CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]

3. `docker-compose.dev.yml` at the root:
   version: '3.9'
   services:

     db:
       image: postgres:16-alpine
       container_name: housemate-db
       environment:
         POSTGRES_USER: ${POSTGRES_USER}
         POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
         POSTGRES_DB: ${POSTGRES_DB}
       volumes:
         - postgres_data:/var/lib/postgresql/data
       ports:
         - "5432:5432"
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
         interval: 5s
         timeout: 5s
         retries: 5

     backend:
       container_name: housemate-backend
       build:
         context: ./backend
         dockerfile: Dockerfile.dev
       volumes:
         - ./backend/src:/app/src
         - ./backend/prisma:/app/prisma
         - ./backend/uploads:/app/uploads
         - /app/node_modules        # anonymous volume prevents host overwrite
       ports:
         - "4000:4000"
       depends_on:
         db:
           condition: service_healthy
       env_file:
         - .env

     frontend:
       container_name: housemate-frontend
       build:
         context: ./frontend
         dockerfile: Dockerfile.dev
       volumes:
         - ./frontend/src:/app/src
         - ./frontend/public:/app/public
         - /app/node_modules
       ports:
         - "5173:5173"
       depends_on:
         - backend

   volumes:
     postgres_data:

IMPORTANT: The `/app/node_modules` anonymous volume entry is critical.
Without it, the host volume mount (./frontend/src → /app/src) would also
wipe /app/node_modules. The anonymous volume takes precedence.
```

---

### Task 0.5 — Docker: Prod Environment

**Prompt the AI:**
```
Create Docker configuration for the PRODUCTION environment.

1. `backend/Dockerfile.prod`:
   Stage 1 (deps): FROM node:20-alpine as deps
     WORKDIR /app
     COPY package.json .
     RUN npm install --omit=dev
     RUN npx prisma generate
   
   Stage 2 (runner): FROM node:20-alpine
     WORKDIR /app
     COPY --from=deps /app/node_modules ./node_modules
     COPY . .
     RUN npx prisma generate
     COPY start.sh .
     RUN chmod +x start.sh
     EXPOSE 4000
     CMD ["/bin/sh", "start.sh"]

2. `backend/start.sh`:
   #!/bin/sh
   echo "Running database migrations..."
   npx prisma migrate deploy
   echo "Starting server..."
   node src/index.js

3. `frontend/Dockerfile.prod`:
   Stage 1 (builder): FROM node:20-alpine as builder
     WORKDIR /app
     COPY package.json .
     RUN npm install
     COPY . .
     RUN npm run build

   Stage 2 (runner): FROM nginx:alpine
     COPY --from=builder /app/dist /usr/share/nginx/html
     COPY nginx.conf /etc/nginx/conf.d/default.conf
     EXPOSE 80
     CMD ["nginx", "-g", "daemon off;"]

4. `frontend/nginx.conf`:
   server {
     listen 80;
     root /usr/share/nginx/html;
     index index.html;

     # Proxy API to backend
     location /api/ {
       proxy_pass http://backend:4000;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
     }

     # SPA fallback
     location / {
       try_files $uri $uri/ /index.html;
     }

     # Cache static assets
     location ~* \.(js|css|png|jpg|jpeg|svg|ico|woff2)$ {
       expires 1y;
       add_header Cache-Control "public, immutable";
     }
   }

5. `docker-compose.prod.yml`:
   Similar to dev but:
   - Uses Dockerfile.prod for both services
   - No source volume mounts
   - db: no exposed host port
   - backend: no exposed host port (only internal)
   - frontend: ports: ["80:80"]
   - backend uploads volume still mounted
   - NODE_ENV=production in backend env

6. Add `.dockerignore` in both `backend/` and `frontend/`:
   node_modules
   .env
   dist
   uploads
   *.log
```

---

### Task 0.6 — shadcn/ui Initialization

**Prompt the AI:**
```
Initialize shadcn/ui in the `frontend/` directory.
Since we're using Vite (not Next.js), we need to follow the manual/Vite setup path.

Run inside the frontend container or the frontend directory:
  npx shadcn-ui@latest init

When prompted:
  - Style: New York
  - Base color: Slate (or your preference)
  - CSS variables: Yes
  - Tailwind config location: tailwind.config.js
  - Components location: src/components/ui
  - Utils location: src/lib/utils.js (change from .ts to .js)
  - JSX: Yes

Then install these components (run: npx shadcn-ui@latest add <name>):
  button input card badge dialog sheet tabs select textarea label
  toast avatar dropdown-menu popover separator skeleton alert
  scroll-area switch accordion alert-dialog

After init:
- Verify src/components/ui/ has the component files
- Verify src/lib/utils.js exists with the cn() helper
- Verify tailwind.config.js has the shadcn content and theme extensions
- Verify src/index.css has the CSS variable definitions

Add <Toaster /> from shadcn to src/main.jsx at the root level.
```

---

---

# PHASE 1 — Authentication

> **Goal:** Users can register, log in, and log out. HTTP-only session cookie is set. Auth state lives in Zustand.

---

### Task 1.1 — Auth Schema & Migration

**Prompt the AI:**
```
Add the User and Session models to `backend/prisma/schema.prisma`.
The file currently just has the datasource and generator.

Add:

model User {
  id           String    @id @default(uuid())
  name         String
  email        String    @unique
  passwordHash String
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  sessions     Session[]
}

model Session {
  id        String   @id @default(uuid())
  userId    String
  token     String   @unique @default(uuid())
  expiresAt DateTime
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

Then run the first migration with:
  docker exec housemate-backend npx prisma migrate dev --name init_auth

This should create a migrations/ folder and apply the migration to the database.
Show me what the expected output looks like and how to confirm it worked.
```

---

### Task 1.2 — Session Helpers & Auth Middleware

**Prompt the AI:**
```
Create session utilities and auth middleware for the Express backend.

1. `backend/src/lib/session.js`:

   const COOKIE_NAME = 'housemate_session';
   const SESSION_DURATION_MS = 7 * 24 * 60 * 60 * 1000; // 7 days

   async function createSession(userId, res):
     - Generate token = uuid()
     - Insert Session: { userId, token, expiresAt: new Date(Date.now() + SESSION_DURATION_MS) }
     - res.cookie(COOKIE_NAME, token, {
         httpOnly: true,
         secure: process.env.NODE_ENV === 'production',
         sameSite: 'lax',
         maxAge: SESSION_DURATION_MS,
         path: '/'
       })
     - Return the session record

   async function destroySession(token, res):
     - Delete Session where token = token (ignore if not found)
     - res.clearCookie(COOKIE_NAME, { path: '/' })

   async function getSessionUser(req):
     - Read req.cookies[COOKIE_NAME]
     - If missing: return null
     - Find Session by token, include user (select id, name, email only)
     - If not found: return null
     - If session.expiresAt < new Date(): delete session, return null
     - Return { userId: session.userId, user: session.user }

   module.exports = { createSession, destroySession, getSessionUser }

2. `backend/src/middleware/auth.js`:
   - Import getSessionUser
   - Export async function requireAuth(req, res, next):
     - Calls getSessionUser(req)
     - If null: return res.status(401).json({ error: 'Unauthorized' })
     - Sets req.userId = result.userId and req.user = result.user
     - Calls next()

3. `backend/src/middleware/errorHandler.js`:
   - Export function errorHandler(err, req, res, next):
     - If err.code === 'P2025' (Prisma not found): res.status(404).json({ error: 'Not found' })
     - If err.name === 'ZodError': res.status(400).json({ error: 'Validation error', details: err.errors })
     - Otherwise: console.error(err), res.status(500).json({ error: 'Internal server error' })
   - Mount this in app.js AFTER all routes: app.use(errorHandler)
```

---

### Task 1.3 — Auth API Routes

**Prompt the AI:**
```
Create `backend/src/routes/auth.js` with the following routes,
then mount it in app.js at: app.use('/api/auth', authRouter)

Use Zod for input validation. Import createSession, destroySession, getSessionUser.

POST /register:
  Validate body: { name (min 2), email (valid email), password (min 8) }
  - Check if email already exists → 409 { error: 'Email already in use' }
  - Hash password: bcrypt.hash(password, 10)
  - Create User: { name, email, passwordHash }
  - Call createSession(user.id, res)
  - Return 201 { user: { id, name, email } }

POST /login:
  Validate body: { email, password }
  - Find user by email
  - If not found OR bcrypt.compare fails → 401 { error: 'Invalid credentials' }
  - Call createSession(user.id, res)
  - Return 200 { user: { id, name, email } }

POST /logout:
  - Read token from req.cookies['housemate_session']
  - Call destroySession(token, res)
  - Return 200 { message: 'Logged out' }

GET /me:
  - Apply requireAuth middleware
  - Return 200 { user: req.user }

After adding routes, also update `backend/src/app.js` to:
- Import and mount the auth router
- Mount the error handler middleware at the very bottom
```

---

### Task 1.4 — Zustand Auth Store

**Prompt the AI:**
```
Create the Zustand auth store in the React frontend.

`frontend/src/stores/authStore.js`:

import { create } from 'zustand'

export const useAuthStore = create((set) => ({
  user: null,
  isLoading: true,        // true until the first /me check resolves
  isAuthenticated: false,

  setUser: (user) => set({ user, isAuthenticated: true, isLoading: false }),
  clearUser: () => set({ user: null, isAuthenticated: false, isLoading: false }),
  setLoading: (isLoading) => set({ isLoading }),
}))

This is the single source of truth for authentication state in the frontend.
Components read from this store. Query hooks update it on success/error.
```

---

### Task 1.5 — TanStack Query Auth Hooks

**Prompt the AI:**
```
Create `frontend/src/hooks/useAuth.js` with TanStack Query hooks for auth.

IMPORTANT pattern for this entire project:
- Hooks are defined as exported named functions in src/hooks/
- They use the api client from src/api/client.js
- Components import and call these hooks; they do NOT define useQuery inline

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { useNavigate } from 'react-router-dom'
import api from '../api/client'
import { useAuthStore } from '../stores/authStore'

export function useMe() {
  const { setUser, clearUser } = useAuthStore()
  return useQuery({
    queryKey: ['me'],
    queryFn: () => api.get('/auth/me').then(r => r.data),
    onSuccess: (data) => setUser(data.user),
    onError: () => clearUser(),
    staleTime: Infinity,
    retry: false,
  })
}

export function useLogin() {
  const { setUser } = useAuthStore()
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (credentials) => api.post('/auth/login', credentials).then(r => r.data),
    onSuccess: (data) => {
      setUser(data.user)
      queryClient.invalidateQueries({ queryKey: ['me'] })
      navigate('/dashboard')
    }
  })
}

export function useRegister() {
  const { setUser } = useAuthStore()
  const navigate = useNavigate()
  return useMutation({
    mutationFn: (data) => api.post('/auth/register', data).then(r => r.data),
    onSuccess: (data) => {
      setUser(data.user)
      navigate('/dashboard')
    }
  })
}

export function useLogout() {
  const { clearUser } = useAuthStore()
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: () => api.post('/auth/logout').then(r => r.data),
    onSuccess: () => {
      clearUser()
      queryClient.clear()
      navigate('/login')
    }
  })
}
```

---

### Task 1.6 — Auth Pages, Router & Protected Route

**Prompt the AI:**
```
Create the auth UI pages and routing infrastructure.

1. `frontend/src/pages/auth/LoginPage.jsx`:
   - Vertically and horizontally centered layout (full screen)
   - shadcn Card containing:
     - App name/logo at top
     - Email and password Input fields with Labels
     - Submit Button (shows loading spinner while mutation is pending)
     - Error message (red text) below the button if mutation.error exists
     - Link to /register: "Don't have an account? Register"
   - On submit: calls useLogin() mutation with { email, password }
   - Mobile-friendly: card is full-width on small screens, max-w-sm on larger

2. `frontend/src/pages/auth/RegisterPage.jsx`:
   - Same layout as LoginPage
   - Fields: Name, Email, Password
   - Calls useRegister() mutation
   - Link to /login

3. `frontend/src/components/shared/PageLoader.jsx`:
   - Full-screen div with flex center
   - Animated lucide Loader2 icon (animate-spin class)
   - Optional text prop: "Loading..."

4. `frontend/src/components/shared/ProtectedRoute.jsx`:
   import { Navigate, Outlet } from 'react-router-dom'
   import { useAuthStore } from '../../stores/authStore'
   import PageLoader from './PageLoader'

   export default function ProtectedRoute() {
     const { isAuthenticated, isLoading } = useAuthStore()
     if (isLoading) return <PageLoader />
     if (!isAuthenticated) return <Navigate to="/login" replace />
     return <Outlet />
   }

5. `frontend/src/router.jsx`:
   Use createBrowserRouter with these routes:
   - /login → LoginPage
   - /register → RegisterPage
   - / → redirect to /dashboard (Navigate element)
   - Protected group (element: <ProtectedRoute />):
     - /dashboard → DashboardPage (stub: <h1>Dashboard</h1>)
     - /chores → ChoresPage (stub)
     - /chores/:id → ChoreDetailPage (stub)
     - /announcements → AnnouncementsPage (stub)
     - /members → MembersPage (stub)
     - /roles → RolesPage (stub)
     - /family/setup → FamilySetupPage (stub)
     - /family/settings → FamilySettingsPage (stub)

6. `frontend/src/App.jsx`:
   - Call useMe() at the top to bootstrap auth state on app load
   - Render <RouterProvider router={router} />
   Note: useMe must be called inside the QueryClientProvider (it is, since it's in App)
   but it must also be inside a Router for useNavigate to work in hooks.
   Structure: QueryClientProvider > RouterProvider > App content
   Actually: put the router at the top level in main.jsx, wrap App with it.
   Revise main.jsx to: render <RouterProvider router={router} /> directly,
   and move the useMe() call into a layout component.
```

---

---

# PHASE 2 — Families

> **Goal:** Users can create or join a family. Active family stored in Zustand + localStorage. Superadmin is established.

---

### Task 2.1 — Family Schema & Migration

**Prompt the AI:**
```
Add Family and FamilyMember models to `backend/prisma/schema.prisma`.
Here is the current schema (paste your current schema.prisma content here).

Add these models:

model Family {
  id          String         @id @default(uuid())
  name        String
  inviteCode  String         @unique @default(uuid())
  createdById String
  createdBy   User           @relation("FamilyCreator", fields: [createdById], references: [id])
  members     FamilyMember[]
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt
}

model FamilyMember {
  id           String   @id @default(uuid())
  userId       String
  familyId     String
  isSuperAdmin Boolean  @default(false)
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  family       Family   @relation(fields: [familyId], references: [id], onDelete: Cascade)
  createdAt    DateTime @default(now())

  @@unique([userId, familyId])
}

Also add back-relations to the User model:
  familyMemberships FamilyMember[]
  createdFamilies   Family[]       @relation("FamilyCreator")

Run migration:
  docker exec housemate-backend npx prisma migrate dev --name add_families
```

---

### Task 2.2 — Family API Routes

**Prompt the AI:**
```
Create `backend/src/routes/families.js` and mount it in app.js at /api/families.
All routes require the requireAuth middleware.

POST / (create a family):
  Body: { name } — validate: min 2 chars
  - Create Family: { name, createdById: req.userId }
  - Create FamilyMember: { userId: req.userId, familyId: family.id, isSuperAdmin: true }
  - Return 201 { family }

POST /join:
  Body: { inviteCode }
  - Find family by inviteCode → 404 if not found
  - Check req.userId is not already a member → 409 { error: 'Already a member' }
  - Create FamilyMember: { userId: req.userId, familyId, isSuperAdmin: false }
  - Return 200 { family }

GET /mine:
  - Find all FamilyMember records where userId = req.userId
  - Include family info and a _count of members
  - Return 200 { families }

GET /:familyId:
  - Verify req.userId is a member (helper: getMemberOrFail below)
  - Include all members with user (id, name, email) and their role
  - Return 200 { family }

PUT /:familyId:
  Body: { name }
  - Get member, verify isSuperAdmin → 403 if not
  - Update family name
  - Return 200 { family }

GET /:familyId/invite-code:
  - Verify isSuperAdmin
  - Generate new inviteCode = require('uuid').v4()
  - Update family, return 200 { inviteCode }

Create a helper function at the top of the routes file:
  async function getMemberOrFail(userId, familyId, res):
    - Fetch FamilyMember where { userId, familyId }
    - If not found: res.status(403).json({ error: 'Not a member of this family' }) and return null
    - Return the member
  (Routes call this and return early if null is returned)
```

---

### Task 2.3 — Family Zustand Store & Hooks

**Prompt the AI:**
```
Create the family Zustand store and TanStack Query hooks.

1. `frontend/src/stores/familyStore.js`:

import { create } from 'zustand'

const STORAGE_KEY = 'housemate_active_family'

export const useFamilyStore = create((set, get) => ({
  activeFamilyId: localStorage.getItem(STORAGE_KEY) || null,
  activeFamily: null,

  setActiveFamilyId: (id) => {
    localStorage.setItem(STORAGE_KEY, id)
    set({ activeFamilyId: id })
  },
  setActiveFamily: (family) => set({ activeFamily: family }),
  clearFamily: () => {
    localStorage.removeItem(STORAGE_KEY)
    set({ activeFamilyId: null, activeFamily: null })
  },
}))

2. `frontend/src/hooks/useFamilies.js`:

export function useMyFamilies() {
  // GET /api/families/mine
  // onSuccess: if activeFamilyId is null or not in the returned list,
  //            auto-select data.families[0]?.id via familyStore.setActiveFamilyId
}

export function useFamily(familyId) {
  // GET /api/families/:familyId
  // enabled: !!familyId
  // onSuccess: familyStore.setActiveFamily(data.family)
}

export function useCreateFamily() {
  // POST /api/families
  // onSuccess: setActiveFamilyId(data.family.id), invalidate useMyFamilies
}

export function useJoinFamily() {
  // POST /api/families/join
  // onSuccess: setActiveFamilyId(data.family.id), invalidate useMyFamilies
}

export function useUpdateFamily(familyId) {
  // PUT /api/families/:familyId
  // onSuccess: invalidate ['family', familyId]
}

export function useRegenerateInviteCode(familyId) {
  // GET /api/families/:familyId/invite-code (treat as a mutation with side effects)
  // useMutation: mutationFn calls api.get(...)
}
```

---

### Task 2.4 — Family Setup & Settings Pages

**Prompt the AI:**
```
Create the family pages and update the protected route to handle the no-family state.

1. Update `frontend/src/components/shared/ProtectedRoute.jsx`:
   After auth check, check if the user has at least one family:
   - Call useMyFamilies() (from useFamilies)
   - Show PageLoader while families are loading
   - If families is an empty array → <Navigate to="/family/setup" replace />
   - Otherwise render <Outlet />

2. `frontend/src/pages/family/FamilySetupPage.jsx`:
   Layout: centered page, two sections stacked on mobile / side-by-side on tablet+

   Section A — Create a Family:
   - Heading: "Create a Family"
   - Family name input
   - "Create" button → useCreateFamily() → on success navigate to /dashboard

   Section B — Join a Family:
   - Heading: "Join with Invite Code"
   - Invite code input (paste or type)
   - "Join" button → useJoinFamily() → on success navigate to /dashboard

   Each section in a shadcn Card. Show loading and error states.

3. `frontend/src/pages/family/FamilySettingsPage.jsx`:
   Data: useFamily(activeFamilyId) and useMyFamilies()

   Top: Family name display
   - If superadmin: inline edit (input + save button) → useUpdateFamily()

   Invite Code section:
   - Display invite code in a code block style
   - "Copy" button (copies to clipboard + shows toast "Copied!")
   - "Regenerate" button (superadmin only, AlertDialog confirm) → useRegenerateInviteCode()

   Family Switcher (if user is in multiple families):
   - shadcn Select showing all families
   - On change: familyStore.setActiveFamilyId(selectedId)

   Members List:
   - Simple list: avatar (initials), name, email, superadmin crown icon
   - Role badge if they have one
```

---

---

# PHASE 3 — Roles & Permissions

> **Goal:** Admins can define roles with permissions. Members get assigned roles. Permission checks power all backend routes.

---

### Task 3.1 — Roles & Permissions Schema

**Prompt the AI:**
```
Add Role, Permission, and RolePermission models to `backend/prisma/schema.prisma`.
(Paste your current full schema.prisma here before asking.)

model Permission {
  id    String @id @default(uuid())
  name  String @unique
  label String
  // Will be seeded with 13 system permissions (see seed.js)
  roles RolePermission[]
}

model Role {
  id          String         @id @default(uuid())
  name        String
  familyId    String
  isAdmin     Boolean        @default(false)
  family      Family         @relation(fields: [familyId], references: [id], onDelete: Cascade)
  permissions RolePermission[]
  members     FamilyMember[]
  createdAt   DateTime       @default(now())

  @@unique([name, familyId])
}

model RolePermission {
  id           String     @id @default(uuid())
  roleId       String
  permissionId String
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id])

  @@unique([roleId, permissionId])
}

Update FamilyMember model — add:
  roleId String?
  role   Role?   @relation(fields: [roleId], references: [id], onDelete: SetNull)

Add back-relation to Family model:
  roles Role[]

Run migration:
  docker exec housemate-backend npx prisma migrate dev --name add_roles_permissions

After migration, create `backend/prisma/seed.js`:
  Use prisma.permission.upsert() (by name) to insert all 13 permissions:
  view_chores, create_chore, update_chore, delete_chore,
  assign_chore_all, assign_chore_specific,
  create_role, update_role, delete_role,
  create_member, update_member, delete_member,
  create_announcement

  Use labels like "View all chores", "Create chore", etc.

Run the seed:
  docker exec housemate-backend node prisma/seed.js
```

---

### Task 3.2 — Permission Resolution Library

**Prompt the AI:**
```
Create `backend/src/lib/permissions.js` — the core permissions engine.
ALL backend routes must use this for permission checks. Never check permissions inline.

const prisma = require('./prisma')

async function getMember(userId, familyId) {
  return prisma.familyMember.findUnique({
    where: { userId_familyId: { userId, familyId } },
    include: {
      role: {
        include: {
          permissions: { include: { permission: true } }
        }
      }
    }
  })
}

async function hasPermission(userId, familyId, permissionName) {
  const member = await getMember(userId, familyId)
  if (!member) return false
  if (member.isSuperAdmin) return true
  if (member.role?.isAdmin) return true
  if (member.role?.permissions.some(rp => rp.permission.name === permissionName)) return true
  return false
}

async function getMemberPermissions(userId, familyId) {
  const member = await getMember(userId, familyId)
  if (!member) return []
  if (member.isSuperAdmin || member.role?.isAdmin) {
    const all = await prisma.permission.findMany()
    return all.map(p => p.name)
  }
  return member.role?.permissions.map(rp => rp.permission.name) ?? []
}

// Use in route handlers: if (!await requirePermission(...)) return;
async function requirePermission(userId, familyId, permissionName, res) {
  const ok = await hasPermission(userId, familyId, permissionName)
  if (!ok) {
    res.status(403).json({ error: 'Forbidden' })
    return false
  }
  return true
}

async function requireMembership(userId, familyId, res) {
  const member = await getMember(userId, familyId)
  if (!member) {
    res.status(403).json({ error: 'Not a member of this family' })
    return false
  }
  return true
}

async function isSuperAdmin(userId, familyId) {
  const member = await getMember(userId, familyId)
  return member?.isSuperAdmin ?? false
}

module.exports = { getMember, hasPermission, getMemberPermissions,
                   requirePermission, requireMembership, isSuperAdmin }
```

---

### Task 3.3 — Roles API Routes

**Prompt the AI:**
```
Create `backend/src/routes/roles.js`.
Mount in app.js: app.use('/api/families', rolesRouter)
(Shares the /api/families prefix with the families router.)

All routes: requireAuth. Family-scoped routes: requireMembership.

GET /api/families/:familyId/roles:
  - Any member
  - Include each role's permissions (permission name and label)
  - Return 200 { roles }

POST /api/families/:familyId/roles:
  - requirePermission: create_role
  - Body: { name, isAdmin, permissionIds: [] }
  - Validate name uniqueness within family → 409 if taken
  - Create Role, then create RolePermission records for each permissionId
  - Return 201 { role } with permissions included

GET /api/families/:familyId/roles/:roleId:
  - Any member
  - Include permissions
  - Return 200 { role }

PUT /api/families/:familyId/roles/:roleId:
  - requirePermission: update_role
  - Body: { name?, isAdmin?, permissionIds[]? }
  - Update role fields
  - If permissionIds provided: delete all existing RolePermissions, insert new ones
  - Return 200 { role }

DELETE /api/families/:familyId/roles/:roleId:
  - requirePermission: delete_role
  - Check if any FamilyMember has roleId = this role → 400 { error: 'Role is in use' }
  - Delete role (cascade deletes RolePermissions)
  - Return 204

GET /api/families/:familyId/permissions:
  - Any member
  - Return all Permission records
  - Return 200 { permissions }
```

---

### Task 3.4 — Members Management API Routes

**Prompt the AI:**
```
Create `backend/src/routes/members.js`.
Mount in app.js: app.use('/api/families', membersRouter)

GET /api/families/:familyId/members:
  - requireMembership
  - Include: user (id, name, email), role (id, name, isAdmin), isSuperAdmin
  - Return 200 { members }

PUT /api/families/:familyId/members/:memberId:
  Body: { roleId? } — set to null to remove role
  - requirePermission: update_member
  - Fetch target FamilyMember → 404 if not found
  - If target.isSuperAdmin → 403 { error: 'Cannot modify superadmin' }
  - If roleId is provided: verify the role belongs to this family
  - Update roleId
  - Return 200 { member }

DELETE /api/families/:familyId/members/:memberId:
  - requirePermission: delete_member
  - Fetch target FamilyMember → 404 if not found
  - If target.isSuperAdmin → 403 { error: 'Cannot remove superadmin' }
  - If target.userId === req.userId → 403 { error: 'Cannot remove yourself' }
  - Delete FamilyMember
  - Return 204
```

---

### Task 3.5 — Roles & Members Hooks

**Prompt the AI:**
```
Create TanStack Query hooks for roles and members.

`frontend/src/hooks/useRoles.js`:

export function useRoles(familyId) {
  // GET /api/families/:familyId/roles, enabled: !!familyId
}
export function usePermissions(familyId) {
  // GET /api/families/:familyId/permissions
}
export function useCreateRole(familyId) {
  // POST /api/families/:familyId/roles
  // onSuccess: invalidate ['roles', familyId]
}
export function useUpdateRole(familyId) {
  // PUT /api/families/:familyId/roles/:roleId
  // mutationFn receives { roleId, ...data }
  // onSuccess: invalidate ['roles', familyId]
}
export function useDeleteRole(familyId) {
  // DELETE /api/families/:familyId/roles/:roleId
  // mutationFn receives roleId
  // onSuccess: invalidate ['roles', familyId]
}

`frontend/src/hooks/useMembers.js`:

export function useMembers(familyId) {
  // GET /api/families/:familyId/members
}
export function useUpdateMember(familyId) {
  // PUT /api/families/:familyId/members/:memberId
  // mutationFn receives { memberId, roleId }
  // onSuccess: invalidate ['members', familyId]
}
export function useRemoveMember(familyId) {
  // DELETE /api/families/:familyId/members/:memberId
  // mutationFn receives memberId
  // onSuccess: invalidate ['members', familyId]
}
```

---

### Task 3.6 — Roles & Members Pages

**Prompt the AI:**
```
Create the roles and members management pages.

1. `frontend/src/pages/RolesPage.jsx`:
   Uses: useRoles, usePermissions, useCreateRole, useUpdateRole, useDeleteRole
   
   Layout:
   - Page header: "Roles" title + "Create Role" button
   - Grid of role cards (responsive: 1 col mobile, 2-3 col desktop)
   
   Role card content:
   - Role name (bold)
   - "Admin" badge if isAdmin (with tooltip: "Has all permissions")
   - Permission badges (scrollable row)
   - Edit icon button + Delete icon button
   
   Create/Edit Dialog (shadcn Dialog):
   - Name input
   - "Admin role" Switch with description
   - Permission checkboxes (grouped: Chores, Members, Roles, Other)
     Disabled when isAdmin switch is ON
   - Save button (loading state)
   
   Delete: shadcn AlertDialog confirm before useDeleteRole()
   
   Guard: Show "Create Role" button only if user has create_role permission
   (use useMembers to find current user's member record, check their role)

2. `frontend/src/pages/MembersPage.jsx`:
   Uses: useMembers, useRoles, useUpdateMember, useRemoveMember
   
   Layout: responsive list (table on desktop, cards on mobile)
   
   Each member row/card:
   - Avatar (initials in a circle)
   - Name + email
   - Crown icon if isSuperAdmin (non-interactive)
   - Role selector: shadcn Select with all roles + "No role" option
     Disabled for superadmin rows and for the current user's own row
     On change → useUpdateMember()
   - Remove button (shadcn AlertDialog) — hidden for superadmin rows
   
   Guard: role selector and remove button only shown if user has correct permissions
```

---

---

# PHASE 4 — Chores

> **Goal:** Full chore lifecycle — create, assign, status transitions, completion requests, audit log.

---

### Task 4.1 — Chores Schema & Migration

**Prompt the AI:**
```
Add chore models to `backend/prisma/schema.prisma`.
(Paste your current full schema here first.)

Add Prisma enum:
enum ChoreStatus {
  TO_BE_COMPLETED
  COMPLETED
  DO_OVER
  CANCELLED
}

model Chore {
  id                String                  @id @default(uuid())
  familyId          String
  name              String
  description       String?
  status            ChoreStatus             @default(TO_BE_COMPLETED)
  createdById       String
  family            Family                  @relation(fields: [familyId], references: [id], onDelete: Cascade)
  createdBy         User                    @relation("ChoreCreator", fields: [createdById], references: [id])
  assignees         ChoreAssignee[]
  statusHistory     ChoreStatusHistory[]
  completionRequest ChoreCompletionRequest?
  createdAt         DateTime                @default(now())
  updatedAt         DateTime                @updatedAt
}

model ChoreAssignee {
  id      String @id @default(uuid())
  choreId String
  userId  String
  chore   Chore  @relation(fields: [choreId], references: [id], onDelete: Cascade)
  user    User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([choreId, userId])
}

model ChoreStatusHistory {
  id             String      @id @default(uuid())
  choreId        String
  previousStatus ChoreStatus?
  newStatus      ChoreStatus
  changedById    String
  chore          Chore       @relation(fields: [choreId], references: [id], onDelete: Cascade)
  changedBy      User        @relation(fields: [changedById], references: [id])
  createdAt      DateTime    @default(now())
}

model ChoreCompletionRequest {
  id            String   @id @default(uuid())
  choreId       String   @unique
  submittedById String
  text          String?
  imagePath     String?
  videoPath     String?
  chore         Chore    @relation(fields: [choreId], references: [id], onDelete: Cascade)
  submittedBy   User     @relation(fields: [submittedById], references: [id])
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}

Add back-relations to User and Family as needed.

Run migration:
  docker exec housemate-backend npx prisma migrate dev --name add_chores
```

---

### Task 4.2 — Chores API — CRUD

**Prompt the AI:**
```
Create `backend/src/routes/chores.js`.
Mount: app.use('/api/families', choresRouter)

All routes: requireAuth + requireMembership at the top.

Helper in this file:
  async function getChoreOrFail(choreId, familyId, res):
    Fetch chore where { id: choreId, familyId }
    If not found: res.status(404).json({ error: 'Chore not found' }), return null
    Return chore

GET /api/families/:familyId/chores:
  Query param: ?view=assigned|created|all (default: all)
  - assigned: chores where req.userId is in assignees
  - created: chores where createdById = req.userId
  - all: all chores in this family
  - Include: assignees (with user: id, name), createdBy (id, name), completionRequest existence
  - Return 200 { chores }

POST /api/families/:familyId/chores:
  Body: { name, description?, assigneeIds: [] }
  - requirePermission: create_chore
  - Validate: name required, assigneeIds non-empty
  - For now, require assign_chore_all permission to assign
  - Verify each assigneeId is a family member
  - Create Chore + ChoreAssignee records
  - Create ChoreStatusHistory: { previousStatus: null, newStatus: 'TO_BE_COMPLETED', changedById: req.userId }
  - console.log('NOTIFY: chore assigned', choreId, assigneeIds)  // stub
  - Return 201 { chore }

GET /api/families/:familyId/chores/:choreId:
  - Include: assignees+user, createdBy, statusHistory+changedBy (ordered by createdAt desc), completionRequest+submittedBy
  - Return 200 { chore }

PUT /api/families/:familyId/chores/:choreId:
  Body: { name?, description?, assigneeIds[]? }
  - User must be creator OR have update_chore permission
  - Chore status must be TO_BE_COMPLETED or DO_OVER
  - If assigneeIds provided: delete all ChoreAssignees, recreate
  - Return 200 { chore }

DELETE /api/families/:familyId/chores/:choreId:
  - requirePermission: delete_chore
  - Delete chore
  - Return 204
```

---

### Task 4.3 — Chore Status & Completion Request Routes

**Prompt the AI:**
```
Add sub-routes to `backend/src/routes/chores.js`.

POST /api/families/:familyId/chores/:choreId/status:
  Body: { status: 'COMPLETED' | 'CANCELLED' }
  - Verify user is chore.createdById OR admin/superadmin
  - Allowed transitions only:
    TO_BE_COMPLETED → COMPLETED (only if completionRequest exists)
    TO_BE_COMPLETED → CANCELLED
    All other combinations → 400 { error: 'Invalid status transition' }
  - Update chore status
  - Create ChoreStatusHistory entry
  - console.log('NOTIFY: chore status updated')  // stub
  - Return 200 { chore }

POST /api/families/:familyId/chores/:choreId/do-over:
  - Verify user is chore.createdById OR admin/superadmin
  - Chore must have a completionRequest → 400 if not
  - Delete the completionRequest
  - Set status = DO_OVER, record history
  - Immediately set status = TO_BE_COMPLETED, record history
  - console.log('NOTIFY: do over')  // stub
  - Return 200 { chore }

POST /api/families/:familyId/chores/:choreId/completion-request:
  Body: { text? }
  - User must be in chore.assignees → 403 if not
  - Chore status must be TO_BE_COMPLETED → 400 if not
  - No existing completionRequest → 400 { error: 'Request already exists' } if one exists
  - Create ChoreCompletionRequest: { choreId, submittedById: req.userId, text }
  - console.log('NOTIFY: completion request submitted')  // stub
  - Return 201 { completionRequest }

PUT /api/families/:familyId/chores/:choreId/completion-request:
  Body: { text? }
  - Get chore with completionRequest
  - Verify req.userId === completionRequest.submittedById
  - Update { text }
  - Return 200 { completionRequest }

DELETE /api/families/:familyId/chores/:choreId/completion-request:
  - Get chore with completionRequest
  - Verify req.userId === submittedById OR admin/superadmin
  - Delete completionRequest
  - Return 204
```

---

### Task 4.4 — Chores TanStack Query Hooks

**Prompt the AI:**
```
Create `frontend/src/hooks/useChores.js`.

All hooks follow the same pattern as previous hooks files.
Use the activeFamilyId from useFamilyStore() for the familyId parameter.

export function useChores(familyId, view = 'all') {
  // GET /api/families/:familyId/chores?view=view
  // queryKey: ['chores', familyId, view]
}

export function useChore(familyId, choreId) {
  // GET /api/families/:familyId/chores/:choreId
  // enabled: !!familyId && !!choreId
}

export function useCreateChore(familyId) {
  // POST — onSuccess: invalidate ['chores', familyId]
}

export function useUpdateChore(familyId) {
  // PUT — mutationFn: ({ choreId, ...data }) => ...
  // onSuccess: invalidate ['chores', familyId] and ['chore', familyId, choreId]
}

export function useDeleteChore(familyId) {
  // DELETE — mutationFn: (choreId) => ...
  // onSuccess: invalidate ['chores', familyId]
}

export function useChangeChoreStatus(familyId, choreId) {
  // POST /status — onSuccess: invalidate ['chore', familyId, choreId]
}

export function useDoOver(familyId, choreId) {
  // POST /do-over — onSuccess: invalidate ['chore', familyId, choreId]
}

export function useSubmitCompletionRequest(familyId, choreId) {
  // POST /completion-request — onSuccess: invalidate ['chore', ...]
}

export function useUpdateCompletionRequest(familyId, choreId) {
  // PUT /completion-request
}

export function useDeleteCompletionRequest(familyId, choreId) {
  // DELETE /completion-request — onSuccess: invalidate ['chore', ...]
}
```

---

### Task 4.5 — Chores List Page

**Prompt the AI:**
```
Create `frontend/src/pages/chores/ChoresPage.jsx`.

Layout:
- Mobile-first: full-width cards stacked
- Desktop: 2-3 column card grid

Header:
- "Chores" title
- "Create Chore" button (shown only if user has create_chore permission)
  → Opens shadcn Dialog

Tabs (shadcn Tabs): "My Chores" | "Created by Me"
  - My Chores: useChores(familyId, 'assigned')
  - Created by Me: useChores(familyId, 'created')

Chore card:
- Name (bold)
- Status badge:
  TO_BE_COMPLETED → blue "Pending"
  COMPLETED → green "Completed"
  DO_OVER → amber "Do Over"
  CANCELLED → slate "Cancelled"
- Assigned to: avatar initials (up to 3, then "+N more")
- "View Details" button → navigate('/chores/' + chore.id)

Create Chore Dialog:
- Name input (required)
- Description textarea (optional)
- Assignees: multi-select checkboxes from useMembers(familyId)
  (show name + email, allow selecting multiple)
- Submit button → useCreateChore()

Show loading skeletons while fetching.
Show <EmptyState /> when list is empty.
```

---

### Task 4.6 — Chore Detail Page

**Prompt the AI:**
```
Create `frontend/src/pages/chores/ChoreDetailPage.jsx`.

Use useParams() to get choreId.
Get familyId from useFamilyStore().
Fetch with useChore(familyId, choreId).

Mobile-friendly single-column layout with sections separated by shadcn Separator.

Section 1 — Header:
  - Back button (←) that navigates to /chores
  - Chore name (large, bold)
  - Status badge (colored per status)

Section 2 — Info:
  - Description (or "No description" in muted text)
  - "Created by" with name and time ago
  - "Assigned to" with avatar initials and names

Section 3 — Actions (show based on: user is creator OR admin/superadmin):
  - "Approve Completion" button:
    Show only if: completionRequest exists AND status = TO_BE_COMPLETED
    Calls useChangeChoreStatus({ status: 'COMPLETED' })
  - "Do Over" button:
    Show only if: completionRequest exists AND status = TO_BE_COMPLETED
    Calls useDoOver()
  - "Cancel Chore" button:
    Show only if: status = TO_BE_COMPLETED
    AlertDialog confirm → useChangeChoreStatus({ status: 'CANCELLED' })

Section 4 — Completion Request:
  Case A: user is assignee + status = TO_BE_COMPLETED + no completionRequest:
    "Submit Completion" button → Dialog with textarea → useSubmitCompletionRequest()

  Case B: completionRequest exists:
    Display: submitter name, submitted time, request text
    Show image if imagePath (later in Phase 5)
    If req user is submitter:
      "Edit" button → Dialog pre-filled → useUpdateCompletionRequest()
      "Withdraw" button → AlertDialog → useDeleteCompletionRequest()

Section 5 — Status History (shadcn Accordion, collapsed by default):
  For each history entry (newest first):
  - "[previous] → [new]" (use friendly labels not enum values)
  - "by [name] · [time ago]"

Show PageLoader while fetching. Show error message if fetch fails.
```

---

---

# PHASE 5 — Announcements & File Uploads

> **Goal:** Announcements feature + media upload support for completion requests.

---

### Task 5.1 — Announcements Schema & API

**Prompt the AI:**
```
Add Announcement model to `backend/prisma/schema.prisma`:

model Announcement {
  id        String   @id @default(uuid())
  familyId  String
  title     String
  body      String
  authorId  String
  family    Family   @relation(fields: [familyId], references: [id], onDelete: Cascade)
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

Run migration:
  docker exec housemate-backend npx prisma migrate dev --name add_announcements

Create `backend/src/routes/announcements.js`:
Mount: app.use('/api/families', announcementsRouter)

GET /api/families/:familyId/announcements:
  - requireMembership
  - Order: createdAt desc, limit 50
  - Include author (id, name)
  - Return 200 { announcements }

POST /api/families/:familyId/announcements:
  Body: { title, body }
  - requirePermission: create_announcement
  - Create announcement with authorId = req.userId
  - console.log('NOTIFY: announcement created')  // stub
  - Return 201 { announcement }

GET /api/families/:familyId/announcements/:id:
  - requireMembership, return 200 { announcement }

PUT /api/families/:familyId/announcements/:id:
  Body: { title?, body? }
  - requireMembership
  - User must be author OR admin/superadmin → 403 otherwise
  - Update and return 200 { announcement }

DELETE /api/families/:familyId/announcements/:id:
  - requireMembership
  - User must be author OR admin/superadmin
  - Return 204
```

---

### Task 5.2 — Announcements Hooks & Page

**Prompt the AI:**
```
Create `frontend/src/hooks/useAnnouncements.js`:

export function useAnnouncements(familyId) { /* GET */ }
export function useAnnouncement(familyId, id) { /* GET /:id */ }
export function useCreateAnnouncement(familyId) { /* POST, invalidate list */ }
export function useUpdateAnnouncement(familyId) { /* PUT, mutationFn: ({id, ...data}) */ }
export function useDeleteAnnouncement(familyId) { /* DELETE, mutationFn: (id) */ }

Create `frontend/src/pages/AnnouncementsPage.jsx`:

Layout (mobile-first):
- Header: "Announcements" + "New Announcement" button (if has permission)
- List of announcement cards (newest first):
  - Title (bold)
  - Author name + time ago (muted)
  - Body text (truncated to 3 lines, "Read more" toggle)
  - Edit + Delete icon buttons (shown to author or admin)

New/Edit Announcement Dialog:
  - Title input (required)
  - Body textarea (required, min height 100px)
  - Submit → useCreateAnnouncement() or useUpdateAnnouncement()

Delete: AlertDialog confirm → useDeleteAnnouncement()

EmptyState when no announcements.
Loading skeletons while fetching.
```

---

### Task 5.3 — File Upload Backend

**Prompt the AI:**
```
Set up local disk file uploads using multer in the backend.

1. Create `backend/src/routes/uploads.js`:

   const multer = require('multer')
   const path = require('path')
   const fs = require('fs')
   const { v4: uuid } = require('uuid')

   const UPLOAD_DIR = process.env.UPLOAD_DIR || './uploads'

   // Ensure upload dir exists
   fs.mkdirSync(UPLOAD_DIR, { recursive: true })

   const storage = multer.diskStorage({
     destination: (req, file, cb) => cb(null, UPLOAD_DIR),
     filename: (req, file, cb) => {
       const ext = path.extname(file.originalname).toLowerCase()
       cb(null, `${uuid()}-${Date.now()}${ext}`)
     }
   })

   const fileFilter = (req, file, cb) => {
     const allowed = ['image/jpeg', 'image/png', 'image/webp', 'video/mp4', 'video/webm']
     if (allowed.includes(file.mimetype)) cb(null, true)
     else cb(new Error('File type not allowed'), false)
   }

   const upload = multer({
     storage,
     fileFilter,
     limits: { fileSize: 100 * 1024 * 1024 } // 100MB max
   })

   POST /api/uploads (requireAuth, upload.single('file')):
   - Return 201 { filePath: `/uploads/${req.file.filename}` }

   GET /api/uploads/:filename:
   - Security: if filename contains '..' → 400
   - const filePath = path.join(UPLOAD_DIR, req.params.filename)
   - If file doesn't exist → 404
   - res.sendFile(path.resolve(filePath))

2. Mount in app.js:
   app.use('/api/uploads', uploadsRouter)
   app.use('/uploads', express.static(path.join(__dirname, '..', 'uploads')))

3. Install multer if not already: docker exec housemate-backend npm install multer
```

---

### Task 5.4 — File Upload Frontend Integration

**Prompt the AI:**
```
Add file upload to the completion request flow.

1. `frontend/src/hooks/useUpload.js`:
   export function useUploadFile() {
     return useMutation({
       mutationFn: (file) => {
         const formData = new FormData()
         formData.append('file', file)
         return api.post('/uploads', formData, {
           headers: { 'Content-Type': 'multipart/form-data' }
         }).then(r => r.data)
       }
     })
   }

2. `frontend/src/components/shared/FileUploader.jsx`:
   Props: onUpload(filePath), accept, label, value (current path)
   - Hidden file input triggered by a styled button
   - On file select: immediately call useUploadFile() mutation
   - Show: file name during upload, progress indication, error if failed
   - On success: call onUpload(data.filePath)
   - If value is set and is an image: show thumbnail preview <img src={value}>
   - "Remove" button to clear the value (calls onUpload(null))

3. Update the Submit Completion Dialog in ChoreDetailPage.jsx:
   - Add <FileUploader label="Attach Image" accept="image/*" onUpload={setImagePath} />
   - Add <FileUploader label="Attach Video" accept="video/*" onUpload={setVideoPath} />
   - Track imagePath and videoPath in local state
   - Include them in the submission payload

4. In the Completion Request display in ChoreDetailPage.jsx:
   - If completionRequest.imagePath: <img src={completionRequest.imagePath} className="rounded-lg max-h-64 object-cover" />
   - If completionRequest.videoPath: <video controls src={completionRequest.videoPath} className="w-full rounded-lg" />
```

---

---

# PHASE 6 — Notifications & WebSockets

> **Goal:** Real-time notifications via Socket.io. Notification bell in header. In-app notification list.

---

### Task 6.1 — Notifications Schema & Migration

**Prompt the AI:**
```
Add Notification model to `backend/prisma/schema.prisma`:

enum NotificationType {
  CHORE_ASSIGNED
  COMPLETION_REQUEST_SUBMITTED
  CHORE_STATUS_UPDATED
  ANNOUNCEMENT_CREATED
}

model Notification {
  id        String           @id @default(uuid())
  userId    String
  familyId  String
  type      NotificationType
  data      Json
  isRead    Boolean          @default(false)
  user      User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  family    Family           @relation(fields: [familyId], references: [id], onDelete: Cascade)
  createdAt DateTime         @default(now())
}

Add back-relations to User and Family.

Run migration:
  docker exec housemate-backend npx prisma migrate dev --name add_notifications
```

---

### Task 6.2 — Socket.io Server Setup

**Prompt the AI:**
```
Set up Socket.io on the backend.

1. Update `backend/src/index.js`:
   - Change from app.listen() to:
     const http = require('http')
     const { setupSocket } = require('./socket')
     const httpServer = http.createServer(app)
     setupSocket(httpServer)
     httpServer.listen(PORT, ...)

2. Create `backend/src/socket.js`:

const { Server } = require('socket.io')
const cookie = require('cookie')
const { getSessionUser } = require('./lib/session')
const prisma = require('./lib/prisma')

let ioInstance = null

async function setupSocket(httpServer) {
  const io = new Server(httpServer, {
    cors: {
      origin: process.env.FRONTEND_URL,
      credentials: true
    }
  })

  ioInstance = io

  // Auth middleware
  io.use(async (socket, next) => {
    try {
      const rawCookie = socket.handshake.headers.cookie || ''
      const cookies = cookie.parse(rawCookie)
      const token = cookies['housemate_session']
      if (!token) return next(new Error('Unauthorized'))

      // Adapt getSessionUser to work with a token directly
      const session = await prisma.session.findUnique({
        where: { token },
        include: { user: { select: { id: true, name: true, email: true } } }
      })
      if (!session || session.expiresAt < new Date()) return next(new Error('Unauthorized'))

      socket.data.userId = session.userId
      socket.data.user = session.user
      next()
    } catch (err) {
      next(new Error('Unauthorized'))
    }
  })

  io.on('connection', async (socket) => {
    const userId = socket.data.userId

    // Join personal room
    socket.join(`user:${userId}`)

    // Join all family rooms
    const memberships = await prisma.familyMember.findMany({ where: { userId } })
    for (const m of memberships) {
      socket.join(`family:${m.familyId}`)
    }

    socket.on('disconnect', () => {})
  })

  return io
}

function getIO() {
  if (!ioInstance) throw new Error('Socket.io not initialized')
  return ioInstance
}

module.exports = { setupSocket, getIO }
```

---

### Task 6.3 — Notification Service

**Prompt the AI:**
```
Create `backend/src/lib/notifications.js`.

const prisma = require('./prisma')
const { getIO } = require('../socket')

async function createNotification({ userId, familyId, type, data }) {
  const notification = await prisma.notification.create({
    data: { userId, familyId, type, data, isRead: false }
  })
  try {
    getIO().to(`user:${userId}`).emit('notification:new', notification)
  } catch (e) { /* ignore if socket not ready */ }
  return notification
}

async function notifyChoreAssigned(chore, assigneeIds) {
  for (const userId of assigneeIds) {
    if (userId === chore.createdById) continue  // don't notify yourself
    await createNotification({
      userId, familyId: chore.familyId,
      type: 'CHORE_ASSIGNED',
      data: { choreId: chore.id, choreName: chore.name }
    })
  }
}

async function notifyCompletionRequestSubmitted(chore) {
  await createNotification({
    userId: chore.createdById, familyId: chore.familyId,
    type: 'COMPLETION_REQUEST_SUBMITTED',
    data: { choreId: chore.id, choreName: chore.name }
  })
}

async function notifyChoreStatusUpdated(chore, newStatus) {
  const choreWithAssignees = await prisma.chore.findUnique({
    where: { id: chore.id },
    include: { assignees: true }
  })
  for (const assignee of choreWithAssignees.assignees) {
    await createNotification({
      userId: assignee.userId, familyId: chore.familyId,
      type: 'CHORE_STATUS_UPDATED',
      data: { choreId: chore.id, choreName: chore.name, newStatus }
    })
  }
}

async function notifyAnnouncementCreated(announcement, excludeUserId) {
  const members = await prisma.familyMember.findMany({
    where: { familyId: announcement.familyId }
  })
  for (const member of members) {
    if (member.userId === excludeUserId) continue
    await createNotification({
      userId: member.userId, familyId: announcement.familyId,
      type: 'ANNOUNCEMENT_CREATED',
      data: { announcementId: announcement.id, title: announcement.title }
    })
  }
}

module.exports = { notifyChoreAssigned, notifyCompletionRequestSubmitted,
                   notifyChoreStatusUpdated, notifyAnnouncementCreated }

Now also emit a chore:updated event when chore changes happen:
In chores.js routes (status change, do-over, completion request changes), add:
  getIO().to(`family:${familyId}`).emit('chore:updated', { choreId })

Then REPLACE all console.log notification stubs in:
- chores.js POST → notifyChoreAssigned(chore, assigneeIds)
- completion-request POST → notifyCompletionRequestSubmitted(chore)
- /status POST + /do-over POST → notifyChoreStatusUpdated(chore, newStatus)
- announcements.js POST → notifyAnnouncementCreated(announcement, req.userId)
```

---

### Task 6.4 — Notifications API Routes

**Prompt the AI:**
```
Create `backend/src/routes/notifications.js`.
Mount: app.use('/api/notifications', notificationsRouter)
All routes: requireAuth.

GET /api/notifications:
  Query: ?familyId (required), ?unreadOnly=false
  - Fetch notifications for { userId: req.userId, familyId }
  - If unreadOnly=true: add where { isRead: false }
  - Order: createdAt desc, limit 50
  - Return 200 { notifications }

PUT /api/notifications/:id/read:
  - Find notification, verify notification.userId === req.userId
  - Update isRead: true
  - Return 200 { notification }

PUT /api/notifications/read-all:
  Body: { familyId }
  - updateMany: where { userId: req.userId, familyId, isRead: false }
  - Return 200 { count }

NOTE: The /read-all route must be registered BEFORE /:id/read in Express,
otherwise Express will match "read-all" as an :id parameter.
```

---

### Task 6.5 — Frontend Socket & Notification Store

**Prompt the AI:**
```
Set up real-time notifications on the frontend.

1. `frontend/src/stores/notificationStore.js` (Zustand):

export const useNotificationStore = create((set, get) => ({
  notifications: [],
  unreadCount: 0,

  setNotifications: (list) => set({
    notifications: list,
    unreadCount: list.filter(n => !n.isRead).length
  }),
  addNotification: (n) => set(state => ({
    notifications: [n, ...state.notifications],
    unreadCount: !n.isRead ? state.unreadCount + 1 : state.unreadCount
  })),
  markRead: (id) => set(state => ({
    notifications: state.notifications.map(n =>
      n.id === id ? { ...n, isRead: true } : n
    ),
    unreadCount: Math.max(0, state.unreadCount - 1)
  })),
  markAllRead: () => set(state => ({
    notifications: state.notifications.map(n => ({ ...n, isRead: true })),
    unreadCount: 0
  })),
}))

2. `frontend/src/hooks/useSocket.js`:
   - Creates a socket.io-client connection on mount
   - URL: '/' (same origin, proxied in dev, nginx in prod)
   - { withCredentials: true }
   - Store the socket in a useRef
   - Returns { socket (ref.current), isConnected }
   - Disconnect on unmount

3. `frontend/src/hooks/useNotifications.js`:

export function useNotifications(familyId) {
  const { setNotifications } = useNotificationStore()
  return useQuery({
    queryKey: ['notifications', familyId],
    queryFn: () => api.get(`/notifications?familyId=${familyId}`).then(r => r.data),
    onSuccess: (data) => setNotifications(data.notifications),
    enabled: !!familyId
  })
}

export function useMarkRead() {
  const { markRead } = useNotificationStore()
  return useMutation({
    mutationFn: (id) => api.put(`/notifications/${id}/read`).then(r => r.data),
    onSuccess: (_, id) => markRead(id)
  })
}

export function useMarkAllRead(familyId) {
  const { markAllRead } = useNotificationStore()
  return useMutation({
    mutationFn: () => api.put('/notifications/read-all', { familyId }).then(r => r.data),
    onSuccess: () => markAllRead()
  })
}

4. `frontend/src/components/layout/NotificationListener.jsx`:
   A non-rendering component mounted inside AppShell.
   - Calls useSocket() to get socket
   - Calls useQueryClient()
   - On mount: socket.on('notification:new', (n) => notificationStore.addNotification(n))
   - On mount: socket.on('chore:updated', ({ choreId }) => queryClient.invalidateQueries(['chore', familyId, choreId]))
   - Clean up listeners on unmount with socket.off(...)
```

---

### Task 6.6 — Notification Bell UI

**Prompt the AI:**
```
Create `frontend/src/components/layout/NotificationBell.jsx`.

import { Bell } from 'lucide-react'
import { useNotificationStore } from '../../stores/notificationStore'
import { Popover, PopoverContent, PopoverTrigger } from '../ui/popover'
import { useMarkRead, useMarkAllRead } from '../../hooks/useNotifications'
import { useNavigate } from 'react-router-dom'

Component:
- Trigger: Button variant="ghost" with Bell icon and a red Badge overlay showing unreadCount
  Badge is hidden if unreadCount === 0
- Popover content (w-80, max-h-96 scroll):
  Header row: "Notifications" (bold) + "Mark all read" Button (text/ghost)
  
  Scrollable notification list:
  Each item (cursor-pointer, hover bg):
  - Left accent: 4px blue border if !isRead
  - Icon by type:
    CHORE_ASSIGNED: CheckSquare
    COMPLETION_REQUEST_SUBMITTED: Send
    CHORE_STATUS_UPDATED: RefreshCw
    ANNOUNCEMENT_CREATED: Megaphone
  - Message (generated from type + data):
    CHORE_ASSIGNED: `You were assigned to "${data.choreName}"`
    COMPLETION_REQUEST_SUBMITTED: `Completion submitted for "${data.choreName}"`
    CHORE_STATUS_UPDATED: `"${data.choreName}" is now ${formatStatus(data.newStatus)}`
    ANNOUNCEMENT_CREATED: `New announcement: "${data.title}"`
  - Time ago (small muted text)
  - onClick: useMarkRead(n.id) then navigate to:
    CHORE_*: /chores/:choreId
    ANNOUNCEMENT_*: /announcements

  Empty state: "No notifications" centered muted text

Helper: formatStatus(status) → { TO_BE_COMPLETED: 'Pending', COMPLETED: 'Completed', DO_OVER: 'Do Over', CANCELLED: 'Cancelled' }

Add NotificationBell to Header.jsx.
```

---

---

# PHASE 7 — App Shell, Navigation & Dashboard

> **Goal:** Responsive app shell, sidebar (desktop), bottom nav (mobile), populated dashboard.

---

### Task 7.1 — App Shell & Navigation Components

**Prompt the AI:**
```
Create the responsive app shell and navigation.

1. `frontend/src/components/layout/Sidebar.jsx` (shown on md+ screens):
   Width: 240px, fixed left side
   Sections:
   - Top: "HouseMate" logo text
   - Family switcher: shows active family name, click = shadcn DropdownMenu
     listing all families + "Create / Join Family" option → navigate to /family/setup
   - Nav links (each with icon + label, active state highlighted):
     Home/Dashboard (LayoutDashboard icon) → /dashboard
     Chores (CheckSquare) → /chores
     Announcements (Megaphone) → /announcements
     Members (Users) → /members
     Roles (Shield) → /roles
     Family Settings (Settings) → /family/settings
   - Bottom: User avatar (initials), name, email, Logout button → useLogout()

2. `frontend/src/components/layout/BottomNav.jsx` (mobile only, fixed bottom):
   5 tabs: Dashboard, Chores, Announcements, Members, More
   "More" opens a shadcn Sheet from the bottom with: Roles, Family Settings, Logout
   Active tab: use useLocation() to determine active route, highlight with primary color

3. `frontend/src/components/layout/Header.jsx` (mobile top bar):
   - Family name + dropdown (same as sidebar family switcher)
   - <NotificationBell /> on the right

4. `frontend/src/components/layout/AppShell.jsx`:
   - Render <Sidebar /> (hidden on mobile: hidden md:flex)
   - Render <Header /> (mobile only: md:hidden)
   - Render <Outlet /> as main content (with padding)
   - Render <BottomNav /> (mobile only: md:hidden, fixed bottom)
   - Render <NotificationListener /> (always, non-visual)
   - On mount: load useNotifications(activeFamilyId) to populate the store

5. Update `frontend/src/router.jsx`:
   Wrap all protected routes with AppShell as the layout element:
   { element: <ProtectedRoute />, children: [
     { element: <AppShell />, children: [
       { path: '/dashboard', element: <DashboardPage /> },
       ... all other app routes
     ]}
   ]}
```

---

### Task 7.2 — Dashboard Page

**Prompt the AI:**
```
Create `frontend/src/pages/DashboardPage.jsx`.

Data needed:
- useChores(familyId, 'assigned') → filter to TO_BE_COMPLETED + DO_OVER → "My Pending"
- useAnnouncements(familyId) → latest 3 → "Recent Announcements"
- useNotificationStore() for unread count + recent notifications
- useMembers(familyId) for admin stats

Layout: responsive grid
  Mobile: single column, stacked cards
  Tablet+: 2-column grid

Card 1 — "My Pending Chores":
  - List max 5 chores with name + status badge + "View" link
  - "View all" link to /chores
  - EmptyState if none: "No pending chores 🎉"

Card 2 — "Recent Announcements":
  - Latest 3 announcements: title + author + time ago
  - "View all" link to /announcements
  - EmptyState if none

Card 3 — "Recent Notifications":
  - Last 5 notifications from notificationStore
  - Mark all read button

Card 4 — "Quick Stats" (show only to admin/superadmin):
  - Total members: useMembers().data?.members.length
  - Total chores: useChores(familyId, 'all') counts by status
  - Display as simple stat blocks

Greet the user: "Welcome back, [name]!" at the top of the page.
```

---

### Task 7.3 — Empty States, Error Handling & Polish

**Prompt the AI:**
```
Polish error handling and empty states across the app.

1. `frontend/src/components/shared/EmptyState.jsx`:
   Props: icon (lucide component), title, description, action?: { label, onClick }
   - Centered layout, icon in a soft rounded bg circle
   - Title: medium bold, description: muted text
   - Optional button below

2. `frontend/src/components/shared/ErrorMessage.jsx`:
   Props: message, onRetry?
   - shadcn Alert variant=destructive
   - Optional retry button

3. Audit ALL pages and ensure:
   - While loading → Skeleton cards (shadcn Skeleton)
   - On error → <ErrorMessage message={error.message} onRetry={refetch} />
   - On empty → <EmptyState />

4. In `frontend/src/api/client.js` response interceptor:
   401 responses → authStore.clearUser(), navigate to /login
   (Use a custom event or check window.location instead of useNavigate since we're outside React)

5. Add a catch-all 404 route in router.jsx:
   { path: '*', element: <NotFoundPage /> }
   Create a simple NotFoundPage with "Page not found" and a link to /dashboard.

6. PWA offline page:
   Create frontend/src/pages/OfflinePage.jsx:
   "You're offline. Check your connection and try again."
   Register it in vite.config.js workbox as the offline fallback page.
```

---

---

# PHASE 8 — Advanced Features (User Permission Overrides)

> **Goal:** Per-user permission overrides and assignable member constraints.

---

### Task 8.1 — Override Schema & Migration

**Prompt the AI:**
```
Add user override models to `backend/prisma/schema.prisma`:

model UserPermissionOverride {
  id           String       @id @default(uuid())
  memberId     String
  permissionId String
  member       FamilyMember @relation(fields: [memberId], references: [id], onDelete: Cascade)
  permission   Permission   @relation(fields: [permissionId], references: [id], onDelete: Cascade)
  createdAt    DateTime     @default(now())
  @@unique([memberId, permissionId])
}

model AssignableMember {
  id               String       @id @default(uuid())
  assignerMemberId String
  assigneeMemberId String
  assigner         FamilyMember @relation("Assigner", fields: [assignerMemberId], references: [id], onDelete: Cascade)
  assignee         FamilyMember @relation("Assignee", fields: [assigneeMemberId], references: [id], onDelete: Cascade)
  @@unique([assignerMemberId, assigneeMemberId])
}

Add back-relations to FamilyMember:
  permissionOverrides UserPermissionOverride[]
  assignableTargets   AssignableMember[] @relation("Assigner")
  assignableByUsers   AssignableMember[] @relation("Assignee")

Run migration:
  docker exec housemate-backend npx prisma migrate dev --name add_overrides
```

---

### Task 8.2 — Update Permissions Engine & Routes

**Prompt the AI:**
```
Update `backend/src/lib/permissions.js` to include user-level overrides.

Updated getMember() — also include permissionOverrides:
  permissionOverrides: { include: { permission: true } }

Updated hasPermission() — after role check, before returning false:
  if (member.permissionOverrides.some(o => o.permission.name === permissionName)) return true

Updated getMemberPermissions() — merge role permissions + overrides.

Add new routes to members.js:

GET /api/families/:familyId/members/:memberId/permissions:
  - requireMembership (admin or self)
  - Return: { rolePermissions, overrides, effectivePermissions } 

PUT /api/families/:familyId/members/:memberId/permissions:
  Body: { permissionIds: [] }
  - Requires superadmin or admin
  - Delete all UserPermissionOverride for this memberId, insert new ones
  - Return 200 { overrides }

GET /api/families/:familyId/members/:memberId/assignable:
  - Return list of members this person can assign chores to

PUT /api/families/:familyId/members/:memberId/assignable:
  Body: { memberIds: [] }
  - Requires superadmin or admin
  - Replace AssignableMember records for this assigner
  - Return 200

Update chore creation route in chores.js:
  If user has assign_chore_specific but NOT assign_chore_all:
    Fetch their AssignableMember records (where assignerMemberId = current user's memberId)
    If empty → 403 { error: 'No assignable members configured' }
    Verify all assigneeIds are in their allowed list → 403 if not
```

---

### Task 8.3 — Permission Override UI in Members Page

**Prompt the AI:**
```
Add a permission management Sheet to MembersPage.jsx for each member.

1. New hooks in useMembers.js:
   export function useMemberPermissions(familyId, memberId)
   export function useUpdateMemberPermissions(familyId, memberId)
   export function useMemberAssignable(familyId, memberId)
   export function useUpdateMemberAssignable(familyId, memberId)

2. In MembersPage, add a "Manage Permissions" icon button per member row.
   (Show only to superadmin/admin.)
   Clicking opens a shadcn Sheet from the right.

3. Sheet content — "Permissions: [member name]"

   Section: Role Permissions
   - Read-only permission badges from their role
   - "Inherited from: [roleName]" label

   Section: Permission Overrides
   - Checkbox list of all permissions
   - Pre-checked if user has an override
   - "Save" button → useUpdateMemberPermissions()

   Section: Assignable Members
   - Only shown if member has assign_chore_specific permission
   - Multi-select checkboxes of family members
   - Pre-selected from current assignable list
   - "Save" button → useUpdateMemberAssignable()

4. All save actions show loading state and toast on success/error.
```

---

---

# Appendix A — API Route Map

```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/auth/me

POST   /api/families
POST   /api/families/join
GET    /api/families/mine
GET    /api/families/:id
PUT    /api/families/:id
GET    /api/families/:id/invite-code

GET    /api/families/:id/roles
POST   /api/families/:id/roles
GET    /api/families/:id/roles/:roleId
PUT    /api/families/:id/roles/:roleId
DELETE /api/families/:id/roles/:roleId
GET    /api/families/:id/permissions

GET    /api/families/:id/members
PUT    /api/families/:id/members/:memberId
DELETE /api/families/:id/members/:memberId
GET    /api/families/:id/members/:memberId/permissions
PUT    /api/families/:id/members/:memberId/permissions
GET    /api/families/:id/members/:memberId/assignable
PUT    /api/families/:id/members/:memberId/assignable

GET    /api/families/:id/chores
POST   /api/families/:id/chores
GET    /api/families/:id/chores/:choreId
PUT    /api/families/:id/chores/:choreId
DELETE /api/families/:id/chores/:choreId
POST   /api/families/:id/chores/:choreId/status
POST   /api/families/:id/chores/:choreId/do-over
POST   /api/families/:id/chores/:choreId/completion-request
PUT    /api/families/:id/chores/:choreId/completion-request
DELETE /api/families/:id/chores/:choreId/completion-request

GET    /api/families/:id/announcements
POST   /api/families/:id/announcements
GET    /api/families/:id/announcements/:announcementId
PUT    /api/families/:id/announcements/:announcementId
DELETE /api/families/:id/announcements/:announcementId

GET    /api/notifications
PUT    /api/notifications/read-all        ← register BEFORE /:id routes
PUT    /api/notifications/:id/read

POST   /api/uploads
GET    /api/uploads/:filename
```

---

# Appendix B — Permission Names Reference

| Name | Label |
|---|---|
| `view_chores` | View all chores |
| `create_chore` | Create chore |
| `update_chore` | Update chore |
| `delete_chore` | Delete chore |
| `assign_chore_all` | Assign to any member |
| `assign_chore_specific` | Assign to specific members |
| `create_role` | Create role |
| `update_role` | Update role |
| `delete_role` | Delete role |
| `create_member` | Add member |
| `update_member` | Update member |
| `delete_member` | Remove member |
| `create_announcement` | Post announcements |

---

# Appendix C — Chore Status Flow

```
[Created]
    └──► TO_BE_COMPLETED (default)
              │
              ├──► CANCELLED  (creator/admin — terminal)
              │
              └──► [assignee submits CompletionRequest]
                          │
                          ├──► COMPLETED  (creator/admin approves — terminal)
                          │
                          └──► DO_OVER   (creator/admin rejects)
                                    │
                                    └──► TO_BE_COMPLETED (auto-reset, new submission allowed)
                                              │
                                           [cycle repeats]
```

---

# Appendix D — Docker Command Reference

> Always run these inside the containers, never on the host.

```bash
# Install a backend package
docker exec housemate-backend npm install <package>

# Install a frontend package
docker exec housemate-frontend npm install <package>

# Run any npx command in backend
docker exec -it housemate-backend npx <command>

# Create a Prisma migration
docker exec housemate-backend npx prisma migrate dev --name <name>

# Apply migrations (for prod deploy)
docker exec housemate-backend npx prisma migrate deploy

# Open Prisma Studio (visit localhost:5555)
docker exec -it housemate-backend npx prisma studio --hostname 0.0.0.0

# Run seed
docker exec housemate-backend node prisma/seed.js

# Reset database (dev only — deletes all data!)
docker exec housemate-backend npx prisma migrate reset

# Open a shell
docker exec -it housemate-backend sh
docker exec -it housemate-frontend sh

# View logs
docker compose -f docker-compose.dev.yml logs -f backend
```

---

# Appendix E — Vibe Coding Tips

1. **One task per prompt.** Copy a single task block. Don't combine tasks.

2. **Always paste context.** Before schema tasks, paste the full current `schema.prisma`. Before adding routes, show what's already in `app.js`.

3. **Full error output.** When something breaks, paste the complete stack trace — not just the error message.

4. **Remind the AI of the stack** if it drifts:
   *"Reminder: Express backend + React/Vite frontend, JavaScript only (no TypeScript), monorepo in backend/ and frontend/ folders."*

5. **Never install on host.** All `npm install`, `npx`, and `prisma` commands run via `docker exec`.

6. **TanStack Query hook pattern:** Always define hooks as named exports in `src/hooks/`. Components import and call them — never define `useQuery` or `useMutation` inline in a component file.

7. **Zustand pattern:** Stores live in `src/stores/`. Hooks call store actions in `onSuccess` callbacks. Components read from stores with `useXxxStore()`.

8. **Cookie + CORS in dev:** The Vite proxy (`server.proxy` in vite.config.js) forwards `/api` to the backend. This keeps everything on the same origin so cookies work without CORS issues. In prod, nginx does the same job.

9. **Socket.io in dev:** The Vite proxy does NOT forward WebSocket connections by default. If socket.io can't connect in dev, update vite.config.js to add ws: true to the proxy config:
   `/api`: { target: 'http://localhost:4000', ws: true, changeOrigin: true }
   And connect socket.io-client directly to 'http://localhost:4000' in dev.

10. **After each phase:** Smoke test the core feature before moving on. Don't accumulate broken phases.
