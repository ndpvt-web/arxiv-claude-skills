---
name: "fullstack-agent-enhancing-agentic-fullstack"
description: "Build production-grade full-stack web applications using a three-agent pipeline (Planning, Backend, Frontend) with development-oriented testing and structured debugging. Triggers: 'build a full-stack app', 'create a web app with backend and database', 'full-stack website with API and database', 'build me a CRUD app', 'create a web application with user authentication and data storage', 'scaffold a complete web project with frontend and backend'"
---

# FullStack-Agent: Production-Grade Full-Stack Web Application Development

This skill enables Claude to build complete, production-level full-stack web applications by following a structured three-agent pipeline derived from the FullStack-Agent framework. Instead of generating superficial frontend-only pages that mask the absence of real data processing, this approach enforces genuine backend logic, database persistence, and validated data flow by decomposing development into Planning, Backend Coding, and Frontend Coding phases — each with integrated development-oriented testing that catches bugs through automated API probing and GUI-level interaction validation.

## When to Use

- When the user asks to build a full-stack web application with both frontend and backend components
- When the user needs a web app that stores and retrieves data from a database (e.g., "build me a task manager with user accounts")
- When the user requests a CRUD application with API endpoints and a connected UI
- When the user asks to create an interactive website that requires server-side processing (e.g., "build a dashboard that aggregates sales data")
- When the user wants to scaffold a project with proper data flow between frontend, backend, and database layers
- When the user asks to debug or fix a full-stack application where the bug could be in the API, the database schema, or the frontend integration
- When the user requests a web app with authentication, file uploads, or any feature requiring server-side validation

## Key Technique

The FullStack-Agent framework addresses the core problem that most AI-generated web apps are frontend-only illusions — they render mock data with visual effects but lack genuine server-side logic and database persistence. The paper's key insight is to decompose full-stack development into three sequential agent phases with specialized tooling: a **Planning Agent** that produces a structured architecture plan (page layouts, components, API endpoints, database schema, and data flow), a **Backend Coding Agent** that implements all server-side logic first, and a **Frontend Coding Agent** that builds the UI against the actual running backend APIs.

The critical innovation is **development-oriented testing** embedded directly in the coding loop. Rather than writing tests after the fact, each coding agent has access to specialized debugging tools: a **Backend Test Tool** (analogous to Postman) that sends HTTP requests to API endpoints and returns both the response and backend console output, and a **Frontend Test Tool** that launches the full application, runs a GUI-agent interaction process, and monitors both terminal and browser console outputs. These tools allow the agent to iteratively detect and fix bugs during development, not after — reducing debugging iterations significantly (the paper reports average iterations dropping from 115.5 to 74.9 with the backend debugging tool alone).

The three-layer testing approach (FullStack-Bench) validates correctness at every level: frontend tests check that UI interactions produce correct visual results and trigger proper backend calls; backend tests verify that API endpoints return correct responses for given inputs; database tests extract table schemas and row snapshots to confirm data was actually persisted correctly. This ensures no layer can "fake" functionality.

## Step-by-Step Workflow

1. **Analyze the user request and produce a structured development plan.** Write a JSON or markdown plan that specifies: (a) page layouts and components for each view, (b) backend API endpoints with methods, URLs, request/response schemas, (c) database tables with column names, types, and relationships, (d) data flow between frontend actions, API calls, and database operations. This plan is the contract all subsequent work follows.

2. **Select the technology stack and initialize the project.** Based on the plan, choose appropriate frameworks (e.g., React/Vue + Express/FastAPI + SQLite/PostgreSQL). Create the project directory structure with separate frontend and backend directories. Install dependencies and configure the development environment.

3. **Implement the database layer first.** Create schema definitions, migration files, and seed data. Verify the schema by running migrations and inspecting the resulting tables. Confirm that column names, types, and constraints match the plan.

4. **Build all backend API endpoints.** Implement each endpoint from the plan: route handlers, request validation, business logic, and database queries. Follow RESTful conventions. Include proper error responses with meaningful status codes and messages.

5. **Test each backend endpoint immediately after implementation.** For every endpoint, run a concrete HTTP request using `curl` or a test script. Verify: (a) the response status code and body match expectations, (b) the backend console shows no errors, (c) the database state changed correctly (query the DB and inspect rows). Fix any issues before moving to the next endpoint.

6. **Generate an API summary document.** List every endpoint with its URL, method, expected request body, and response format. This becomes the contract the frontend coding phase uses to integrate with the backend.

7. **Build the frontend against the live backend.** Implement each page/component from the plan, wiring UI actions to actual API calls using the API summary. Use the real backend URL — never hardcode mock data. Handle loading states, errors, and edge cases in the UI.

8. **Test the full application through GUI-level interaction.** Start both frontend and backend servers. Walk through every user flow described in the plan: click buttons, fill forms, submit data, navigate between pages. After each interaction, verify: (a) the UI updated correctly, (b) the backend received the correct request (check server logs), (c) the database contains the expected data (query and inspect).

9. **Validate database persistence end-to-end.** After completing all user flows, extract the database contents (table schemas + sample rows) and verify they match expected state. This catches silent failures where the UI appears to work but data is lost or corrupted.

10. **Fix any issues found during testing by localizing the bug layer.** When a test fails, determine whether the fault is in the frontend (wrong API call, incorrect payload), the backend (wrong logic, missing validation), or the database (wrong schema, missing migration). Fix at the correct layer and re-run the relevant tests.

## Concrete Examples

**Example 1: Build a Task Manager with User Authentication**

User: "Build me a full-stack task manager app where users can sign up, log in, create tasks with due dates, mark them complete, and filter by status."

Approach:
1. Plan: Define pages (Login, Register, Task Dashboard), API endpoints (POST /auth/register, POST /auth/login, GET /tasks, POST /tasks, PATCH /tasks/:id, DELETE /tasks/:id), database tables (users: id/email/password_hash/created_at, tasks: id/user_id/title/due_date/status/created_at).
2. Initialize project: React frontend with Vite, Express backend, SQLite with better-sqlite3.
3. Implement DB schema with users and tasks tables. Run migration. Verify tables exist with correct columns.
4. Build auth endpoints with bcrypt password hashing and JWT tokens. Build CRUD task endpoints with user_id foreign key filtering.
5. Test: `curl -X POST localhost:3001/auth/register -H 'Content-Type: application/json' -d '{"email":"test@test.com","password":"pass123"}'` — verify 201 response and user row in DB. Test login, then use the JWT to create/read/update/delete tasks.
6. Document API summary: all endpoints, auth header format, request/response shapes.
7. Build React pages: LoginForm, RegisterForm, TaskList with filter dropdown, TaskForm modal, TaskItem with complete toggle. Wire to real API with fetch + JWT in Authorization header.
8. Start both servers. Register a user, log in, create 3 tasks with different due dates, mark one complete, filter by "completed" — verify only the completed task shows. Check DB has all 3 tasks with correct statuses.
9. Query SQLite: `SELECT * FROM tasks` — confirm 3 rows, one with status="completed".
10. If filtering fails, check: Is the frontend sending the correct query parameter? Is the backend SQL WHERE clause correct? Fix at the right layer.

Output structure:
```
task-manager/
  backend/
    server.js          # Express app with auth middleware
    db.js              # SQLite connection and migrations
    routes/auth.js     # Register and login endpoints
    routes/tasks.js    # CRUD task endpoints
  frontend/
    src/
      App.jsx          # Router with auth context
      pages/Login.jsx
      pages/Register.jsx
      pages/Dashboard.jsx
      components/TaskList.jsx
      components/TaskForm.jsx
      api/client.js    # Fetch wrapper with JWT
```

**Example 2: Build an E-Commerce Product Catalog with Cart**

User: "Create a simple e-commerce site with a product catalog, shopping cart, and checkout that saves orders to a database."

Approach:
1. Plan: Pages (Product Listing, Product Detail, Cart, Checkout, Order Confirmation), API endpoints (GET /products, GET /products/:id, POST /cart/items, GET /cart, DELETE /cart/items/:id, POST /orders), DB tables (products: id/name/price/description/image_url/stock, cart_items: id/session_id/product_id/quantity, orders: id/session_id/total/status/created_at, order_items: id/order_id/product_id/quantity/price).
2. Initialize: Next.js frontend, FastAPI backend, PostgreSQL (or SQLite for simplicity).
3. Create tables and seed 10 sample products with realistic data.
4. Build product listing/detail endpoints. Build cart endpoints using session-based tracking. Build order creation endpoint that moves cart items to order_items and clears the cart.
5. Test each endpoint: GET /products returns 10 items. POST /cart/items adds a product. POST /orders creates an order row and order_items rows, clears cart_items.
6. API summary: document all endpoints including session cookie handling.
7. Build product grid with images and prices, detail page with "Add to Cart" button, cart page with quantity adjustment, checkout form that POSTs to /orders.
8. Full flow test: Browse products, add 2 items to cart, go to cart, adjust quantity, checkout. Verify order confirmation shows correct total.
9. Database check: orders table has 1 row with correct total. order_items has 2 rows. cart_items is empty. products stock decremented.
10. Common bug: cart total doesn't match order total — trace from frontend calculation to backend order creation logic.

**Example 3: Debugging a Full-Stack Bug**

User: "My app shows data on the frontend but when I refresh it's gone. The save button seems to work but data doesn't persist."

Approach:
1. Identify the data flow: What happens when the user clicks Save? Trace: frontend onClick -> API call -> backend handler -> database INSERT.
2. Check the frontend: Open browser dev tools Network tab (or add logging). Is the API call actually being sent? What's the response status?
3. Check the backend: Add logging to the route handler. Is the request arriving? Is the DB query executing without error?
4. Check the database: Query the table directly after a save action. Are rows actually being inserted?
5. Common culprits: (a) Frontend is updating local state but not calling the API. (b) Backend returns 200 but the DB write is in a transaction that's never committed. (c) The app uses an in-memory SQLite database that resets on server restart. (d) The frontend reads from a different endpoint than it writes to.
6. Fix at the correct layer and verify end-to-end: save, refresh the page, confirm data reappears.

## Best Practices

- **Do:** Always implement and test the backend before building the frontend. This ensures the frontend integrates with real, validated APIs rather than imagined contracts.
- **Do:** Test each API endpoint immediately after writing it with a concrete HTTP request. Don't batch all testing to the end — bugs compound and become harder to localize.
- **Do:** Validate database state directly after key operations. Check that rows exist, values are correct, and foreign keys point to valid records. The UI can lie; the database cannot.
- **Do:** Produce a written API summary (endpoints, methods, request/response shapes) before starting frontend work. This prevents mismatches between what the frontend expects and what the backend provides.
- **Avoid:** Generating frontend-only applications that use hardcoded data or localStorage as a substitute for a real backend. If the user asks for a full-stack app, deliver genuine server-side logic and database persistence.
- **Avoid:** Writing all code before testing anything. The development-oriented testing approach specifically calls for test-as-you-go to catch bugs when they're cheap to fix.
- **Avoid:** Debugging at the wrong layer. When something breaks, first determine whether the fault is in the frontend, backend, or database before changing code. Blindly modifying the frontend when the bug is a missing database migration wastes effort.

## Error Handling

| Problem | Diagnosis | Fix |
|---|---|---|
| API returns 500 but no clear error | Backend lacks error middleware or swallows exceptions | Add global error handler that logs full stack traces; check that async route handlers have try/catch or express-async-errors |
| Frontend shows stale data after mutation | Cache invalidation issue or missing refetch | After POST/PATCH/DELETE, explicitly refetch the relevant GET endpoint or invalidate the query cache |
| Database table missing columns | Migration not run or schema out of sync | Re-run migrations; compare DB schema against plan; add the missing ALTER TABLE |
| CORS errors blocking API calls | Backend missing CORS headers for frontend origin | Configure CORS middleware with the correct frontend URL and allowed methods |
| Data saves but relationships are broken | Foreign key constraints missing or wrong IDs | Add proper FOREIGN KEY constraints; verify IDs are passed correctly in API requests |
| App works in dev but fails in production | Hardcoded localhost URLs or missing env vars | Use environment variables for all URLs and secrets; verify .env files exist in production |

## Limitations

- This workflow is designed for small-to-medium full-stack applications (1-20 pages, <50 API endpoints). For large enterprise applications with microservices, message queues, or complex deployment pipelines, the single-project approach needs to be adapted.
- The sequential Backend-then-Frontend ordering works well for CRUD apps but may need adjustment for real-time applications (WebSockets) or applications where frontend and backend are tightly co-designed.
- Claude cannot run persistent servers across tool calls. Testing requires starting and stopping servers within individual bash sessions, which limits the ability to do true end-to-end GUI testing. Workaround: use curl/httpie for API testing and verify frontend correctness through code review and browser console output analysis.
- Complex authentication flows (OAuth, SAML) and third-party integrations require credentials and external services that may not be available in the development environment.
- The approach assumes a monorepo or co-located frontend/backend structure. For separate deployment targets (e.g., Vercel + Railway), additional configuration steps are needed.

## Reference

**Paper:** [FullStack-Agent: Enhancing Agentic Full-Stack Web Coding via Development-Oriented Testing and Repository Back-Translation](https://arxiv.org/abs/2602.03798v1) — Lu et al., 2026. Key insight: decompose full-stack development into Planning/Backend/Frontend phases with embedded debugging tools (Postman-like API tester + GUI-agent browser tester) that validate correctness at every layer during development, not after.