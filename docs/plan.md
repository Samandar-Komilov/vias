# Project Plan v0

### Phase 1: Core Tunneling and Basic Server (TCP Only)

1.  **Initialize Project:** Run `go mod init <your-repo-path>` to create your `go.mod` file.
2.  **Define Core Tunnel Data Structure (`internal/core/tunnel.go`):** Create a `Tunnel` struct that encapsulates the persistent connection (e.g., `net.Conn` or `websocket.Conn`), the associated **subdomain**, and a channel for sending data.
3.  **Build Simple Server (`cmd/server/main.go`):**
      * Start a basic **TCP listener** on a port (e.g., `:8000`) for the client agents to connect.
      * When a client connects, read the initial message (e.g., requesting a subdomain).
      * Create a new `Tunnel` instance and store it in a global map (in `internal/server/registry.go`).
4.  **Build Simple Client (`cmd/client/main.go`):**
      * Establish an **outbound TCP connection** to the server's tunnel port (`:8000`).
      * Send the initial request message (including the desired local port and subdomain).
      * Start a background goroutine to maintain the connection and handle incoming forwarded requests.

-----

### Phase 2: HTTP Routing and Request Forwarding

5.  **Implement Server Registry (`internal/server/registry.go`):** Create a thread-safe struct (using `sync.RWMutex`) to map requested subdomains (from the `Host` header) to the active `Tunnel` struct.
6.  **Implement Server HTTP Handler (`internal/server/handler.go` and `cmd/server/main.go`):**
      * Start the main **HTTP/S listener** on standard ports (`:80` and `:443`).
      * In the handler, inspect the **`Host` header** of the incoming public request.
      * Look up the corresponding `Tunnel` in the registry.
      * **Serialize** the entire incoming HTTP request (headers and body) into a custom protocol payload and send it down the tunnel connection to the client.
7.  **Implement Client Request Handling (`internal/client/agent.go`):**
      * The client constantly reads data from the tunnel connection.
      * When a request payload arrives, **deserialize** it back into an HTTP request object.
      * Forward this request to the local application using `httputil.ReverseProxy` (e.g., `http://localhost:8080`).
      * Serialize the resulting response and send it back up the tunnel to the server.

-----

### Phase 3: Robustness and Features

8.  **Add Tunnel Protocol:** Move from raw TCP to **WebSockets** for the persistent connection. This simplifies handling HTTP messages over the tunnel and is a standard practice for reverse proxies.
9.  **Implement Connection Heartbeats:** Add a mechanism where the client and server periodically exchange small packets to ensure the tunnel is still active and prevent NAT timeouts.
10. **Implement Client Retries:** Add logic to the client to automatically reconnect to the server with an **exponential backoff** strategy if the tunnel connection is dropped.
11. **Add HTTPS/TLS:** Configure the server to handle TLS termination (port 443). Use a library like `golang.org/x/crypto/acme/autocert` for easy integration with **Let's Encrypt** to automatically provision certificates for all subdomains.

-----

### Phase 4: User Experience and Deployment

12. **Add CLI Configuration (`pkg/config/config.go`):** Use a library like `spf13/cobra` or `urfave/cli` to handle command-line flags on the client (e.g., `-subdomain`, `-port`).
13. **Finalize Client Output:** Ensure the client prints the assigned public URL and logs incoming requests clearly.
14. **Deployment:** Compile the server executable and deploy it on a public VPS with a **wildcard DNS record** (`*.yourdomain.io`) pointing to its IP address. Compile the client for different architectures (Linux, macOS, Windows).