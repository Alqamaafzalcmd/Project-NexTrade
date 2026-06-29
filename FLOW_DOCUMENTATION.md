# NexTrade - Implemented Application Flow Documentation

## Scope

This documentation is based on the currently implemented code inside these active folders:

- `frontend` - public landing website and authentication pages
- `dashboard` - authenticated trading dashboard
- `backend` - Express + MongoDB API

Note: The duplicate folders `NexTrade-Frontend`, `NexTrade-Dashboard`, and `NexTrade-Backend` were not used as the main source for this document because the active workspace implementation is under `frontend`, `dashboard`, and `backend`.

---

## 1. Overall Architecture Flow

```text
User Opens NexTrade
        |
        v
Landing App Loads (`frontend`)
        |
        v
sends a request to backend verify
   token exist (user exist )
  .Verifies Cookie via `/auth`
              |
  ____________|___________________________
  |                                      |
  v                                      |
valid Cookie                             |
  |                                      |
  |                                      |
  v                                      |
Redirect to dashboard                    |
                                         |
             ____________________________|                                    
             |
             v
User Can Browse Public Pages
Home / About / Product / Pricing / Support
        |
        v
User Chooses Signup or Login
        |
        v
Frontend Calls Backend Auth APIs
        |
        v
Backend Creates JWT Cookie on Success
        |
        v
Frontend Redirects User to Dashboard App
        |
        v
Dashboard App Loads (`dashboard`)
        |
        v
Dashboard Verifies Cookie via `/auth`
        |
   +----+-----------------+
   |                      |
   v                      v
Valid                  Invalid
Cookie                 Cookie
   |                     |
   v                     v
 Load                 Redirect to
Dashboard           Landing Website
        |
        v
User Views Watchlist / Summary / Orders / Holdings / Positions / Funds / Profile
        |                                                           |          |
        v                                                           |          |
User Can Buy, Sell                                                  |          |_________ 
        |                                                       Add Funds,               |
        v                                                      Withdraw Funds            |
Backend Updates MongoDB Collections                                                    Logout
        |
        v
Dashboard Reloads Data from APIs
```

---

## 2. Backend Startup Flow

```text
Run Backend Server
        |
        v
Load Environment Variables
        |
        v
Create Express App
        |
        v
Enable CORS for Frontend and Dashboard URLs
        |
        v
Enable JSON Parsing + Cookie Parsing
        |
        v
Configure Mongo Session Store
        |
        v
Connect to MongoDB
        |
        v
Register Feature Routers
Holdings / Positions / Orders / Watchlist / Users / Auth
        |
        v
Start Express Listener
        |
        v
Run Stock Price Refresh Loop Every 15 Seconds
        |
        v
Fetch Quotes from Finnhub
        |
        v
Upsert Stock Documents in MongoDB
```

### What actually happens

- `backend/index.js` boots the Express server.
- CORS allows requests from `FRONTEND_URL` and `DASHBOARD_URL` with credentials enabled.
- Sessions and flash messages are configured, although authentication itself is handled with JWT cookies.
- MongoDB is connected through Mongoose.
- A repeating `setInterval(updateStockPrices, 15000)` fetches prices for symbols from `backend/symbols.js`.
- Each refresh updates the `Stock` collection with latest quote data like `currentPrice`, `previousClose`, `change`, and `changePercent`.

---

## 3. Frontend Landing Website Flow

```text
User Opens Frontend App
        |
        v
React Router Loads Public Website
        |
        v
Navbar Renders
        |
        v
Selected Page Component Renders
        |
        v
Footer Renders
```

### Implemented public routes

- `/` -> Home page
- `/about` -> About page
- `/product` -> Product page
- `/pricing` -> Pricing page
- `/support` -> Support page
- `/signup` -> Signup form
- `/login` -> Login form
- `*` -> Not Found page

### Behavior

- The landing site is a separate React application from the dashboard.
- It is mainly informational until the user opens `Signup` or `Login`.
- Toast notifications are enabled globally through `react-toastify`.
- For flash message across same route --> react-toastify ,
  diff route like frontend, dashboard --> Flash (express-session, connect-flash is used)

---

## 4. User Signup Flow

```text
User Opens `/signup`
        |
        v
Signup Form Renders
        |
        v
User Enters Username + Email + Password
        |
        v
Frontend Sends POST `/auth/signup`
        |
        v
Backend Validates Required Fields
        |
   +----+----+
   |         |
   v         v
Missing    Present
Fields     Fields
   |         |
   v         v
Return     Check Existing User by Email
400             |
                v
           +----+----+
           |         |
           v         v
         Exists   Not Exists
           |         |
           v         v
      Return 409   Create User Document
                        |
                        v
              Pre-save Hook Hashes Password
                        |
                        v
                 Save User in MongoDB
                        |
                        v
                 Generate JWT Token
                        |
                        v
                 Set `token` Cookie
                        |
                        v
               Return Success Response
                        |
                        v
         Frontend Redirects to Dashboard URL
```

### Implemented details

- Frontend file: `frontend/src/landing/users/Signup.jsx`
- API endpoint: `POST /auth/signup`
- Backend controller: `backend/controllers/auth.js`
- Password hashing happens in the Mongoose `pre("save")` hook inside `backend/schemas/usersSchema.js`.
- If signup succeeds, the frontend redirects directly to `VITE_DASHBOARD_URL`.

---

## 5. User Login Flow

```text
User Opens `/login`
        |
        v
Login Form Renders
        |
        v
User Enters Username or Email + Password
        |
        v
Frontend Sends POST `/auth/login`
        |
        v
Backend Reads `username_email` + `password`
        |
        v
Backend Searches User by Username or Email
        |
   +----+----+
   |         |
   v         v
No User    User Found
   |         |
   v         v
Return     Compare Password
401             |
                v
           +----+----+
           |         |
           v         v
        Invalid    Valid
        Password  Password
           |         |
           v         v
      Return 401   Generate JWT
                        |
                        v
                 Set `token` Cookie
                        |
                        v
               Return Success Response
                        |
                        v
         Frontend Shows Success Alert
                        |
                        v
         Frontend Redirects to Dashboard URL
```

### Implemented details

- Frontend file: `frontend/src/landing/users/Login.jsx`
- API endpoint: `POST /auth/login`
- Backend compares passwords using `bcrypt.compare`.
- On failure, the frontend shows the backend error message in a toast.
- On success, SweetAlert is shown briefly before redirecting.

---

## 6. Dashboard Entry and Authentication Check Flow

```text
User Reaches Dashboard App
        |
        v
Dashboard `Home` Component Mounts
        |
        v
`useEffect` Calls GET `/auth`
with Credentials
        |
        v
Browser Sends JWT Cookie
        |
        v
Backend `userVerification` Middleware Runs
        |
   +----+----+
   |         |
   v         v
No/Bad     Valid
Token      Token
   |         |
   v         v
Return 401  Attach User to `req.user`
   |         |
   v         v
Dashboard   Auth Controller Returns Success
Redirects        |
to Frontend      v
             Render Dashboard Layout
```

### Implemented details

- Dashboard entry file: `dashboard/src/components/Home.jsx`
- Verification endpoint: `GET /auth`
- Auth middleware: `backend/middlewares/authorization.js`
- Middleware reads the `token` cookie, verifies JWT with `TOKEN_KEY`, loads the user from MongoDB, then allows the request.
- If auth fails, the dashboard immediately redirects to `VITE_FRONTEND_URL`.

---

## 7. Dashboard Layout Flow

```text
Dashboard Auth Succeeds
        |
        v
General Context Provider Wraps App
        |
        v
TopBar Renders
        |
        v
Menu Renders
        |
        v
Dashboard Routes Render Main Screen
        |
        v
Watchlist Panel Always Renders on Side
```

### Implemented dashboard routes

- `/` -> Summary
- `/orders` -> Orders
- `/holdings` -> Holdings
- `/positions` -> Positions
- `/funds` -> Funds
- `/apps` -> Apps

### Implemented details

- `dashboard/src/components/GeneralContext.jsx` manages modal state for:
  - buy dialog
  - sell dialog
  - add funds dialog
  - withdraw funds dialog
  - profile popover
- `dashboard/src/components/Menu.jsx` also fetches user info to display username initials.

---

## 8. Watchlist and Live Market Data Flow

```text
Dashboard Loads Watchlist
        |
        v
WatchList Component Mounts
        |
        v
GET `/watchlist`
        |
        v
Backend Reads All Stock Documents
        |
        v
Return Stock List
        |
        v
Frontend Renders Watchlist Rows
        |
        v
Frontend Starts 50-Second Refresh Interval
        |
        v
Side Doughnut Chart Rebuilds from Current Prices
```

### Implemented details

- Dashboard file: `dashboard/src/components/WatchList.jsx`
- API endpoint: `GET /watchlist`
- Backend controller returns all `Stock` documents.
- Each watchlist item exposes Buy and Sell buttons on hover.
- Stock data is indirectly kept fresh because:
  - backend updates prices every 15 seconds
  - dashboard re-fetches watchlist every 50 seconds

---

## 9. Summary Screen Flow

```text
User Opens Dashboard Summary
        |
        v
Summary Component Mounts
        |
        v
GET `/users/info`
        |
        v
Backend Loads User + Holdings + Positions
        |
        v
Backend Calculates:
funds
usedMargin
holdingValue
positionValue
holdingCount
investment
pnl
pnlPercent
        |
        v
Return Aggregated Portfolio Snapshot
        |
        v
Frontend Displays Margin and Holdings Overview
```

### Implemented calculations

- `holdingValue` = sum of holding quantity multiplied by current stock price
- `investment` = sum of holding quantity multiplied by average buy price
- `pnl` = `holdingValue - investment`
- `pnlPercent` = `(pnl / investment) * 100` when investment is greater than 0
- `positionValue` = sum of position quantity multiplied by current stock price

---

## 10. Holdings Screen Flow

```text
User Opens Holdings Page
        |
        v
Holdings Component Mounts
        |
        v
GET `/holdings`
        |
        v
Backend Loads User Holdings
        |
        v
Backend Loads Matching Stock Records
        |
        v
Backend Joins Holding Data with Market Data
        |
        v
Return Table-ready Holding Objects
        |
        v
Frontend Renders Table + Chart
```

### Returned holding fields

- `instrument`
- `qty`
- `avgCost`
- `ltp`
- `currentValue`
- `pnl`
- `netChange`
- `dayChange`

### Backend logic

- Holdings are filtered by `customer = req.user.id`.
- For each holding, backend matches a stock by `name`.
- It calculates current value and P&L before sending response.

---

## 11. Positions Screen Flow

```text
User Opens Positions Page
        |
        v
Positions Component Mounts
        |
        v
GET `/positions`
        |
        v
Backend Loads User Positions
        |
        v
Backend Loads Matching Stock Records
        |
        v
Backend Combines Position + Market Data
        |
        v
Return MIS Position Objects
        |
        v
Frontend Renders Positions Table + Chart
```

### Returned position fields

- `product`
- `instrument`
- `qty`
- `avgCost`
- `ltp`
- `currentValue`
- `pnl`
- `netChange`
- `dayChange`

### Important implementation note

- Positions are treated as `MIS` product in the returned response.
- The current implementation re-fetches positions repeatedly because the React effect depends on `allPositions`. This creates a refresh loop in practice.

---

## 12. Orders Screen Flow

```text
User Opens Orders Page
        |
        v
Orders Component Mounts
        |
        v
GET `/orders`
        |
        v
Backend Loads Orders for Logged-in User
        |
        v
Return Order History
        |
        v
Frontend Renders Orders Table
```

### Order data stored

- stock name
- quantity
- price
- mode (`BUY` or `SELL`)
- status (`Completed` or `Rejected` in current logic)
- creation timestamp
- customer id

### Important implementation note

- The current Orders page also re-fetches continuously because its React effect depends on `allOrders`.

---

## 13. Buy Stock Flow

```text
User Hovers Watchlist Item
        |
        v
User Clicks Buy
        |
        v
Buy Dialog Opens from Global Context
        |
        v
User Enters Quantity + Price
        |
        v
User Selects Product
    CNC or MIS
        |
        v
Frontend Chooses API Destination
        |
   +----+----+
   |         |
   v         v
  CNC       MIS
   |         |_____________
   |                       |
   v                       v
  POST                    POST
`/holdings/add`     `/positions/add`
        |
        v
Backend Verifies Logged-in User
        |
        v
   `addOrders` Middleware Creates Order Record
        |
   +----+----+
   |         |
   v         v
Enough     Not Enough
Funds      Funds
   |         |
   v         v
Order      Order
Status     Status
Completed  Rejected
        |
        v
  Controller Runs
        |
   +----+----+
   |         |
   v         v
Holding    Position
Controller Controller
   |         |
   v         v
Deduct      Deduct Funds
Funds       and Increase Used Margin
   |         |
   v         v
Create or   Create or Update
Update      Position
Holding
        |
        v
Return Success or Error
        |
        v
Frontend Shows Toast
        |
        v
Dialog Closes
```

### CNC buy implementation

- Endpoint: `POST /holdings/add`
- Middleware: `userVerification` -> `addOrders`
- Controller: `addHolding`
- Funds are reduced from `user.funds`.
- If holding already exists, quantity and weighted average are updated.
- Otherwise a new holding document is created.

### MIS buy implementation

- Endpoint: `POST /positions/add`
- Middleware: `userVerification` -> `addOrders`
- Controller: `addPosition`
- Funds are reduced from `user.funds`.
- The same amount is also added to `user.usedMargin`.
- If position already exists, quantity and weighted average are updated.
- Otherwise a new position document is created.

### Important implementation note

- `addOrders` checks `user.funds < req.body.qty * req.body.price`.
-  If yes execute order, else send 400 status and a message which further shown by react-toastify as message.
-  The holding and position controllers also currently  have the same logic.


---

## 14. Sell Stock Flow

```text
User Hovers Watchlist Item
        |
        v
User Clicks Sell
        |
        v
Sell Dialog Opens
        |
        v
User Enters Quantity + Price + Product
        |
        v
Frontend Chooses API Destination
        |
   +----+----+
   |         |
   v         v
  CNC        MIS
   |         |
   v         v
POST       POST
`/holdings/sell` `/positions/sell`
        |
        v
Backend Verifies User
        |
        v
`sellOrders` Middleware Creates Sell Order
        |
   +----+----+
   |         |
   v         v
Enough     Not Enough
Qty        Qty / No Asset
   |         |
   v         v
Order      Order
Completed  Rejected
        |
        v
Controller Validates Asset Exists
        |
        v
Controller Validates Quantity
        |
        v
Reduce Holding or Position Quantity
        |
        v
Delete Document if Quantity Becomes 0
        |
        v
Credit User Funds
        |
        v
For MIS Also Reduce Used Margin
        |
        v
Return Success or Error
        |
        v
Frontend Shows Toast and Closes Dialog
```

### CNC sell implementation

- Endpoint: `POST /holdings/sell`
- Controller checks whether holding exists and quantity is sufficient.
- User funds increase by `price * qty`.
- Holding quantity decreases or holding document is deleted.

### MIS sell implementation

- Endpoint: `POST /positions/sell`
- Controller checks whether position exists and quantity is sufficient.
- User funds increase by `price * qty`.
- `usedMargin` decreases by `qty * position.avg`.
- Position quantity decreases or position document is deleted.

---

## 15. Add Funds Flow

```text
User Opens Funds Page
        |
        v
User Clicks Add Funds
        |
        v
Add Funds Dialog Opens
        |
        v
User Enters Amount
        |
        v
Frontend Sends POST `/users/addfunds`
        |
        v
Backend Verifies User
        |
        v
Backend Validates Amount > 0
        |
   +----+----+
   |         |
   v         v
Invalid    Valid
Amount     Amount
   |         |
   v         v
Return     Add Amount to
400        `user.funds`
                |
                v
             Save User
                |
                v
          Return Success
                |
                v
      Frontend Shows Success Toast
                |
                v
           Dialog Closes
```

### Implemented details

- UI component: `dashboard/src/components/AddFunds.jsx`
- API endpoint: `POST /users/addfunds`
- Controller reads the amount from `req.body.field`.

---

## 16. Withdraw Funds Flow

```text
User Opens Funds Page
        |
        v
User Clicks Withdraw
        |
        v
Withdraw Dialog Opens
        |
        v
User Enters Amount
        |
        v
Frontend Sends POST `/users/withdrawfunds`
        |
        v
Backend Verifies User
        |
        v
Validate Amount > 0
        |
        v
Check Available Funds
        |
   +----+----+
   |         |
   v         v
Insufficient  Enough
Funds         Funds
   |            |
   v            v
Return 400   Subtract Amount
                  |
                  v
               Save User
                  |
                  v
            Return Success
                  |
                  v
      Frontend Shows Toast and Closes Dialog
```

---

## 17. Funds Screen Flow

```text
User Opens Funds Page
        |
        v
Funds Component Mounts
        |
        v
GET `/users/info`
        |
        v
Backend Returns User Portfolio Snapshot
        |
        v
Frontend Displays:
Available Cash
Holdings Value
Used Margin
Positions Value
Total Portfolio
        |
        v
User Can Open Add or Withdraw Dialog
```

### Important implementation note

- The current Funds page re-fetches continuously because its React effect depends on `info`.

---

## 18. Profile and Logout Flow

```text
User Clicks Profile Section in Menu
        |
        v
 Profile Card Opens
        |
        v
Profile Component Fetches `/users/info`
        |
        v
Username and Email Render
        |
        v
User Clicks Logout
        |
        v
Frontend Calls GET `/auth/logout`
        |
        v
Backend Clears `token` Cookie
        |
        v
Return Success Response
        |
        v
Frontend Shows Alert
        |
        v
Redirect User to Frontend Landing Site
```

### Implemented details

- Logout endpoint is currently `GET /auth/logout`.
- Backend clears the cookie with `httpOnly`, `secure`, and `sameSite: "none"`.

---

## 19. Protected API Flow

```text
Frontend or Dashboard Calls Protected API
                 |
                 v
         Browser Sends Cookie Automatically
                 |
                 v
         `userVerification` Middleware Reads `req.cookies.token`
                 |
                 v
         Verify JWT with `TOKEN_KEY`
                 |
            +----+----+
            |         |
            v         v
          Invalid    Valid
           JWT        JWT
            |         |
            v         v
         Return 401  Extract User ID
                          |
                          v
                    Find User in MongoDB
                          |
                     +----+----+
                     |         |
                     v         v
                  No User    User Found
                     |         |
                     v         v
                Return 401    Attach `req.user`
                                  |
                                  v
                            Continue to Controller
```

### Protected routes currently implemented

- `GET /auth`
- `GET /users/info`
- `POST /users/addfunds`
- `POST /users/withdrawfunds`
- `GET /holdings`
- `POST /holdings/add`
- `POST /holdings/sell`
- `GET /positions`
- `POST /positions/add`
- `POST /positions/sell`
- `GET /orders`

### Public route currently implemented

- `GET /watchlist`

---

## 20. MongoDB Data Flow

```text
 User Signs Up
        |
        v
Create User Document

User Buys CNC
        |
        v
Create Order Document
        |
        v
Create or Update Holding Document
        |
        v
Update User Funds



User Buys MIS
        |
        v
Create Order Document
        |
        v
Create or Update Position Document
        |
        v
Update User Funds and Used Margin



   User Sells
        |
        v
Create Order Document
        |
        v
Update or Delete Holding / Position
        |
        v
Update User Funds

Backend Market Sync
        |
        v
Create or Update Stock Documents Repeatedly
```

### Main collections

- `User`
- `Order`
- `Holding`
- `Position`
- `Stock`

---

## 21. Overall Flow of App

```text
User Opens Landing Website
        |
        v
Browses Public Pages
        |
        v
Signs Up or Logs In
        |
        v
Backend Sets JWT Cookie
        |
        v
User Redirected to Dashboard
        |
        v
Dashboard Verifies Authentication
        |
        v
Watchlist Loads Current Market Data
        |
        v
Summary Loads Portfolio Snapshot
        |
        v
User Can:
- View Holdings
- View Positions
- View Orders
- Manage Funds
- Open Profile
        |
        v
User Places Buy or Sell Orders
        |
        v
Backend Stores Order and Updates Portfolio
        |
        v
User Views Updated Portfolio Screens
        |
        v
   User Logs Out
        |
        v
Cookie Cleared and User Returns to Landing Website
```

---

## 22. Important Implementation Observations

These are not theoretical flows. They reflect the current implementation behavior in the repo:

1. The project is split into 3 separate runnable apps: landing frontend, dashboard frontend, and backend API.
2. Authentication is implemented with a JWT stored in a cookie named `token`.
3. Password hashing is implemented in the Mongoose user schema hook.
4. Stock market data is refreshed server-side every 15 seconds using Finnhub.
5. Watchlist is public and does not require authentication.
6. Buy and sell actions are initiated only from the dashboard watchlist action buttons.
7. Orders are always logged through middleware before holding/position updates.
8. CNC trades update holdings, while MIS trades update positions.
9. Funds and positions/holdings values shown in the UI are computed from live stock prices stored in MongoDB.
10. Some dashboard pages currently re-fetch continuously because of React effect dependency choices:
    - `Positions.jsx`
    - `Orders.jsx`
    - `Funds.jsx`

---
## 23. Packages Used 

### Backend

1. `bcryptjs` - used for password hashing and password comparison during authentication.
2. `body-parser` - used to parse incoming request bodies.
3. `connect-flash` - used to store one-time flash messages in session.
4. `connect-mongo` - used to store Express sessions in MongoDB.
5. `cookie-parser` - used to read cookies from incoming requests.
6. `cors` - used to allow frontend and dashboard cross-origin requests with credentials.
7. `dotenv` - used to load environment variables from `.env`.
8. `express` - used to create the backend API server and routes.
9. `express-session` - used for server-side session management.
10. `jsonwebtoken` - used to sign and verify JWT authentication tokens.
11. `mongoose` - used to define MongoDB schemas and interact with MongoDB.
12. `nodemon` - used in development to restart the backend automatically on file changes.


### Frontend

1. `react` - core library for building the frontend UI.
2. `react-dom` - used to render React components in the browser.
3. `react-router-dom` - used for client-side routing between pages.
4. `axios` - used to make API requests to the backend.
5. `@mui/material` - used for frontend UI components.
6. `@mui/icons-material` - used for Material UI icons.
7. `react-toastify` - used for toast notifications.
8. `react-hot-toast` - used to show lightweight toast notifications.
9. `sweetalert2` - used for modal alerts and feedback popups.


### Dashboard

1. `react` - core library for building the dashboard UI.
2. `react-dom` - used to render the dashboard React app in the browser.
3. `react-router-dom` - used for dashboard route navigation.
4. `axios` - used to make authenticated API requests.
5. `@mui/material` - used for dashboard UI components.
6. `@mui/icons-material` - used for Material UI icons.
7. `chart.js` - used to build dashboard charts.
8. `react-chartjs-2` - React wrapper around Chart.js.
9. `react-toastify` - used for toast notifications in dashboard flows.
10. `react-hot-toast` - used to show lightweight toast notifications.
11. `sweetalert2` - used for popup alerts and action feedback.

---

# 💻 Made by Alqama Afzal
