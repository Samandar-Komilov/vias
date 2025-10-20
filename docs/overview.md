# Project Overview

## Overall Idea

The system works by establishing a persistent, long-lived connection, called a **tunnel**, between three components:

1.  **The Server (Gateway):** A publicly accessible machine (e.g., a VPS) with a fixed IP address and DNS name. This is the heart of your service.
2.  **The Client (Agent):** A small application run by the user on their local machine, where their private service is running (e.g., `http://localhost:8080`).
3.  **The Web User/Requester:** The person/system making a request from the public internet.

### The Process

1.  **Tunnel Establishment:** The **Client** initiates a connection (usually a **WebSocket** or **TCP connection**) *outbound* to the **Server**. This connection is kept alive and acts as the tunnel.
2.  **Subdomain Assignment:** The **Server** assigns a unique, public-facing URL (e.g., `user-name.yourdomain.io`) to this new tunnel.
3.  **Public Request:** A **Web User** sends an HTTP request to `user-name.yourdomain.io`.
4.  **Reverse Proxying:** The **Server** receives the request, identifies the corresponding tunnel based on the subdomain, and forwards the entire request data *down* the established persistent connection to the **Client**.
5.  **Local Forwarding:** The **Client** receives the request, forwards it to the user's local service (e.g., `http://localhost:8080`), gets the response, and sends it back *up* the tunnel to the **Server**.
6.  **Response Delivery:** The **Server** receives the response from the tunnel and delivers it back to the **Web User**.

---

## Essential Features to Implement

To be consistent and effective like jprq.io, your service needs robust features split between the Client and the Server.

### Server (Gateway) Features

| Feature | Description | Implementation Detail |
| :--- | :--- | :--- |
| **Tunnel Handler** | Listens for incoming client connections and keeps them alive. | Use Go's `net/http` for HTTP/S requests and a library for **WebSockets** (or raw **TCP**) for the persistent tunnel connection. |
| **Subdomain Routing** | Maps incoming public requests based on the subdomain to the correct client tunnel. | Requires a **Trie** or a **Hash Map** to quickly look up the active tunnel based on the subdomain in the `Host` header. |
| **Domain Management** | Handles the public-facing domain (e.g., `yourdomain.io`). Requires a **Wildcard DNS record** (`*.yourdomain.io`) pointing to the Server's IP. | Use Let's Encrypt for automatic **SSL/TLS** certificate management for all subdomains. |
| **Rate Limiting** | Protects the server and prevents abuse from any single tunnel/client. | Implement basic limits on concurrent connections or requests per second. |
| **Health Checks/Cleanup** | Periodically checks if client tunnels are still alive and cleans up dead connections and their assigned subdomains. | Use Go's `time` package to manage timeouts and heartbeats. |

### Client (Agent) Features

| Feature | Description | Implementation Detail |
| :--- | :--- | :--- |
| **Connection Persistence** | Establishes and automatically **re-establishes** the persistent tunnel connection to the Server if it drops. | Use an **exponential backoff** strategy for retries to avoid overwhelming the server during outages. |
| **Local Proxying** | Receives forwarded public requests from the Server and forwards them to the user's specified local port (e.g., `localhost:8080`). | Use Go's built-in `net/http/httputil.ReverseProxy` on the client side to simplify local forwarding. |
| **Custom Subdomain** | Allow the user to request a specific subdomain (e.g., `my-project.yourdomain.io`). | The client needs to send the requested subdomain name in the initial tunnel establishment handshake with the Server. |
| **Status/Logging** | Provides clear, real-time output to the user about the public URL, connection status, and incoming requests. | Print logs to the console showing the public URL and details of each request and response. |