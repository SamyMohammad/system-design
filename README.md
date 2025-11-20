# Mobile System Design for Flutter Developers
**A Guide by a Senior System Architect**

In the backend world, System Design is about throughput, load balancing, and database sharding. In the **Mobile/Flutter world**, System Design is about **Constraint Management**. You are designing for a device with limited battery, unreliable networks, finite memory, and an impatient human touching the screen.

---

## I. The Core Concepts: Mobile System Design

### 1. Overview of the Concept
Mobile system design moves beyond "Does it compile?" to "Does it scale and survive?" It involves defining the relationships between your data, your UI, and the outside world. The goal is to build an app that is **resilient** (works when things go wrong) and **reactive** (responds immediately to the user).

### 2. Specific Application to Mobile Clients
Unlike servers, mobile clients are "Stateful, Unreliable Nodes."
* **The Network is a Lie:** You must assume the internet is broken by default.
* **The Lifecycle is Chaos:** The OS can kill your app anytime to save memory.
* **The Single Source of Truth:** Is it the Server? Or the Local Database? (Hint: For a great UX, it's often the Local DB).

### 3. Best Practices for Flutter Architecture
You need a separation of concerns. The industry standard is **Clean Architecture** combined with a state management solution.

* **The Layers:**
    1.  **Presentation:** UI Widgets + State Managers (BLoC/Riverpod).
    2.  **Domain (The Heart):** Pure Dart. Entities and Abstract Use Cases. No Flutter dependencies here.
    3.  **Data:** Repositories (implementation), Data Sources (API/Local DB), and DTOs (Models).
* **State Management Choice:**
    * **Riverpod:** Excellent for dependency injection and compile-time safety. Highly recommended for scalable apps.
    * **BLoC/Cubit:** Great for strict event-driven architecture. Forces a unidirectional data flow which makes debugging easy.

### 4. Offline-First Strategies
This is the hallmark of a senior engineer.
* **Strategy:** "Local-First, Sync-Later." The UI *never* waits for the API. It renders data from the Local DB.
* **The Stack:** Use **Isar** or **Drift (SQLite)** for structured data, **Hive** for key-value.
* **Conflict Resolution:**
    * *Last-Write-Wins:* Simplest, based on timestamps.
    * *Optimistic UI:* Update the UI immediately. If the API fails later, rollback the change and notify the user.

### 5. Real-Time Architecture
* **WebSockets:** Use `web_socket_channel`.
* **Lifecycle Management:** You **must** disconnect sockets when the app enters the background (using `AppLifecycleListener`) to save battery, and reconnect when foregrounded.
* **Heartbeats:** Implement client-side "pings" to detect "zombie connections" (where the wifi icon is on, but data isn't flowing).

### 6. API Efficiency
Mobile data is expensive and slow.
* **BFF (Backend for Frontend):** Don't make the mobile app call 5 different microservices. Have one API endpoint that aggregates data specifically for the mobile view.
* **GraphQL:** Excellent for avoiding over-fetching (getting data you don't need) and under-fetching (needing 10 requests to build one screen).
* **Debouncing:** Don't search on every keystroke. Wait 300ms.

### 7. Performance Optimizations
* **Rendering:** Use `RepaintBoundary` around complex widgets (like Maps or Charts) to stop them from repainting when the rest of the UI updates.
* **Isolates:** JSON parsing in Dart is synchronous. If you parse a 5MB JSON on the main thread, the UI janks. Spawn an **Isolate** (`compute` function) to parse heavy data.
* **Image Caching:** Use `cached_network_image` with strict cache-control headers.

### 8. Retry Logic & Exponential Backoff
Never retry immediately in a loop.
1.  Request Fails.
2.  Wait 1s. Retry.
3.  Wait 2s. Retry.
4.  Wait 4s. Retry.
* **Jitter:** Add random noise (e.g., wait 4.1s instead of exactly 4s) so all clients don't hit the server at the exact same millisecond.

### 9. Secure Mobile Authentication
* **OAuth2 with PKCE:** Standard OAuth is for servers with secrets. Mobile apps cannot keep secrets (APKs can be decompiled). Use **PKCE (Proof Key for Code Exchange)** to verify the login flow dynamically.
* **Token Storage:** Never use `SharedPreferences` for tokens. Use `flutter_secure_storage` (KeyChain on iOS, Keystore on Android).
* **Refresh Logic:** Use a `Dio` or `Http` interceptor. If you get a 401, lock all requests, refresh the token, retry the failed request, and unlock the queue.

### 10. Mobile-Specific Failure Handling
* **Graceful Degradation:** If the recommendation engine fails, show "Popular Items" (cached) instead of an error screen.
* **Queueing:** If a POST request fails (e.g., sending a message), queue it locally in the DB with a status `pending_sync`. A background "Sync Worker" tries to send it later.

---

## II. Architecture Diagrams

### 1. The Clean Architecture Flow (Flutter)

```text
[ UI Layer ]                 [ Domain Layer ]           [ Data Layer ]
+----------------+           +----------------+         +-------------------------+
|                |  Event    |                |         |                         |
|  Widget (View) | --------> |    UseCase     | <------ |  Repository (Impl)      |
|                |           | (Business Logic)|         |                         |
+-------+--------+           +----------------+         +----+---------------+----+
        ^                             ^                      |               |
        | State                       | Result               |               |
+-------+--------+                    |                  +---v---+       +---v---+
|                |                    |                  | Remote|       | Local |
| State Manager  | <------------------+                  | Source|       | Source|
| (BLoC / Prov)  |                                       | (API) |       | (DB)  |
+----------------+                                       +-------+       +-------+
