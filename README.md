Project Documentation — Real-Time Order Updates

What is this project?
This is a web application that shows live order updates on a dashboard.
When someone adds, edits, or deletes an order — every person who has the dashboard open will automatically see the change on their screen within 1 second. No need to refresh the page. No need to keep clicking anything.
Think of it like a live cricket scoreboard — the score updates on your screen by itself without you doing anything.

The Problem It Solves
Normally, to see new data on a webpage, you either:

Refresh the page manually, OR
The page keeps asking the server every few seconds "anything new?" (this is called polling)

Both of these are wasteful and slow.
This project solves it differently — the server tells the client when something changes, instead of the client repeatedly asking. This approach is faster, more efficient, and works well even with many users.

How It Works — Simple Explanation
You (Browser)                  Server                   Database
     |                            |                          |
     | "Hey, I want live updates" |                          |
     |-------------------------->|                          |
     |                            |  checks DB every second  |
     |                            |------------------------->|
     |                            |<-- "order #5 changed" ---|
     | <-- "order #5 changed" ----|                          |
     |  (your screen updates)     |                          |

Your browser connects to the server once and keeps that connection open
The server quietly checks the database every second for any changes
The moment something changes, the server pushes a message to your browser
Your browser updates the screen automatically


Technologies Used
WhatWhySpring BootThe backend framework that runs the serverMySQLThe database where orders are storedSSE (Server-Sent Events)The technology that lets the server push messages to the browserJPA / HibernateMakes it easy to read and write to the database from JavaHTML + JavaScriptThe browser dashboard the user sees

Project Files — What Does Each File Do?
realtime-orders/
│
├── frontend/
│   └── index.html                ← The dashboard you open in the browser
│
└── src/main/java/com/apt/realtime/
    │
    ├── RealtimeOrdersApplication.java     ← Starting point of the application
    │
    ├── model/Order.java                   ← Describes what an "order" looks like
    │                                        (id, customer name, product, status)
    │
    ├── repository/OrderRepository.java    ← Talks to the database
    │                                        (save, find, delete orders)
    │
    ├── dto/OrderDtos.java                 ← The shapes of data sent/received
    │                                        (what a create request looks like,
    │                                         what a response looks like)
    │
    ├── service/
    │   │
    │   ├── OrderService.java              ← Handles create, update, delete logic
    │   │
    │   ├── SseEmitterService.java         ← Manages all connected browsers
    │   │                                    (keeps track of who is connected,
    │   │                                     sends updates to all of them)
    │   │
    │   └── OrderChangePollingService.java ← The "watcher"
    │                                        (checks DB every second,
    │                                         detects what changed,
    │                                         tells SseEmitterService to broadcast)
    │
    ├── controller/OrderController.java    ← Handles incoming requests from browser
    │                                        (create order, get orders, stream)
    │
    └── config/
        ├── WebConfig.java                 ← Technical settings (timeouts, CORS)
        └── GlobalExceptionHandler.java    ← Handles errors nicely

The Database Table
The application uses one table called orders:
ColumnTypeDescriptionidnumberUnique ID for each order (auto-generated)customer_nametextName of the customerproduct_nametextName of the product orderedstatustextCurrent state: PENDING, SHIPPED, or DELIVEREDupdated_attimestampWhen this order was last changedcreated_attimestampWhen this order was first created
The updated_at column has a database index on it. This is like an index at the back of a book — it makes finding recently changed orders extremely fast without scanning the entire table.

The Key Idea — Watermark Polling
Every second, the server runs this query:
sqlSELECT * FROM orders WHERE updated_at > last_checked_time
This is very efficient because:

It only fetches rows that actually changed — not all rows
The index on updated_at makes this query fast even with millions of rows
If nothing changed, the query returns nothing and the server does nothing

After each check, the server remembers the latest timestamp it saw. Next time it only looks for rows newer than that. This "last seen time" is called the watermark.

How the Browser Gets Updates — SSE
SSE stands for Server-Sent Events. It works like this:

Your browser opens a connection to /api/orders/stream
The server keeps this connection open (instead of closing it like normal)
Whenever there's a change, the server writes an event into this open connection
The browser receives it and updates the page

It's a one-way channel — server talks, browser listens. This is perfect for live dashboards, notifications, and feeds.
In JavaScript, connecting to an SSE stream is just one line:
javascriptconst source = new EventSource('http://localhost:8080/api/orders/stream');

What Happens for Each Type of Change
When an order is CREATED:

The polling query finds a new row (an ID that wasn't seen before)
Server sends event with eventType: "CREATED" to all browsers
Browser adds a new row to the table with a green flash

When an order is UPDATED:

The polling query finds a row with a newer updated_at than before
Server sends event with eventType: "UPDATED" to all browsers
Browser updates that row with a blue flash

When an order is DELETED:

The server periodically compares all known IDs with what's in the DB
If an ID is missing, it was deleted
Server sends event with eventType: "DELETED" to all browsers
Browser removes that row with a fade-out animation


How to Set Up and Run
What you need installed

Java 17 or newer
Maven (for building the project)
MySQL 8

Step 1 — Set up the database
Add MySQL to your terminal path (Windows):
bashexport PATH=$PATH:"/c/Program Files/MySQL/MySQL Server 8.0/bin"
Create the database and table:
bashmysql -u root -p < src/main/resources/schema.sql
Step 2 — Add your database password
Open src/main/resources/application.properties and change:
propertiesspring.datasource.password=YOUR_PASSWORD_HERE
Step 3 — Start the server
bashmvn clean package -DskipTests
mvn spring-boot:run
You should see: Tomcat started on port 8080
Step 4 — Open the dashboard
Double-click frontend/index.html in File Explorer. You will see the live orders dashboard with a green Live dot in the corner.

How to Test It
Basic test — create an order
Open a terminal and run:
bashcurl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Alice","productName":"Laptop"}'
Watch the browser — the new order appears instantly.
Update an order
bashcurl -X PATCH http://localhost:8080/api/orders/1 \
  -H "Content-Type: application/json" \
  -d '{"status":"SHIPPED"}'
The status badge on the dashboard changes to SHIPPED in real time.
Delete an order
bashcurl -X DELETE http://localhost:8080/api/orders/1
The row disappears from the dashboard within 5 seconds.
The most impressive test — write directly to MySQL
bashmysql -u root -p orders_db
sqlINSERT INTO orders (customer_name, product_name, status, updated_at, created_at)
VALUES ('Direct Insert', 'Test Product', 'PENDING', NOW(), NOW());
Watch the browser — the new row appears even though you never touched the API. This proves the system monitors the database directly, not just API calls.
Watch the raw SSE stream
bashcurl -N http://localhost:8080/api/orders/stream
Every change will print as raw text in this terminal — this is exactly what the browser receives.

Available API Endpoints
ActionMethodURLBodySubscribe to live updatesGET/api/orders/streamnoneGet all ordersGET/api/ordersnoneGet one orderGET/api/orders/{id}noneCreate an orderPOST/api/orders{"customerName":"...", "productName":"..."}Update an orderPATCH/api/orders/{id}{"status":"SHIPPED"}Delete an orderDELETE/api/orders/{id}noneCheck server healthGET/api/statsnone

Common Issues and Fixes
Green dot shows "Reconnecting" instead of "Live"
→ Make sure the Spring Boot server is running on port 8080
"mysql: command not found" in terminal
→ Run: export PATH=$PATH:"/c/Program Files/MySQL/MySQL Server 8.0/bin"
Application fails to start
→ Check that MySQL is running and the password in application.properties is correct
Orders don't appear on the dashboard
→ Make sure you opened frontend/index.html after the server started, so the SSE connection could be established

Summary
This project demonstrates:

Real-time data delivery using Server-Sent Events — no page refresh needed
Efficient database monitoring using indexed watermark queries — not brute force
Clean separation of concerns — detection, broadcasting, and API handling are all separate
Production-ready patterns — connection cleanup, heartbeats, error handling, and connection pooling
