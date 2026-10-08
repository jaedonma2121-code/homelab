# Minecraft Server

The lab includes a dedicated Minecraft Java Edition server hosted as an isolated service workload.

The public documentation intentionally omits live IP addresses, hostnames, container IDs, usernames, overlay addresses, firewall rules, routing entries, and other environment-specific identifiers.

## Service Role

| Component | Responsibility |
|---|---|
| Minecraft Java Server | Dedicated game-server workload |
| Linux service manager | Starts, stops, restarts, and enables the server at boot |
| tmux | Provides a persistent interactive server console |
| Private service network | Isolates the workload from the upstream network |
| Authenticated overlay network | Provides approved remote-client connectivity |
| Network routing layer | Forwards approved game traffic to the Minecraft service |

## High-Level Architecture

~~~text
Approved remote client
        |
        | authenticated overlay network
        v
Existing routing peer
        |
        | controlled private-network forwarding
        v
Minecraft service
        |
        v
Java Edition server
~~~

The Minecraft service is reached through the existing private routing architecture rather than through a public router port.

It is intentionally **not** placed behind the HTTP reverse proxy because Minecraft Java uses its own game protocol rather than HTTP/HTTPS.

## Service Management

The server is managed as a systemd service.

Conceptually:

~~~text
System boot
    |
    v
systemd
    |
    v
Minecraft service
    |
    v
tmux session
    |
    v
Minecraft Java process
~~~

The service is configured to start automatically and restart according to its service policy.

## Interactive Console

`tmux` provides a persistent console without requiring RCON or a web administration panel.

~~~text
Minecraft Java process
        |
        v
   tmux session
        ^
        |
 administrator attaches
~~~

Administrators can attach to the console when commands such as whitelist management, player administration, saving, or graceful shutdown are required.

Detaching from tmux leaves the Minecraft server running.

## Security Model

The public architecture exposes only the service pattern:

- Minecraft remains on the private service network.
- Remote access requires authenticated overlay-network membership.
- Network policy controls which clients can reach the game service.
- The HTTP reverse proxy is not used for Minecraft traffic.
- No public router port-forwarding is required.
- Administrative access remains separate from the game protocol.
- Live network addresses, routes, policies, and client identifiers are excluded from the repository.

## Engineering Highlights

- Dedicated Minecraft Java workload
- Isolated private-network service
- systemd service management
- Persistent tmux-based administration
- Authenticated overlay networking
- Policy-controlled remote access
- Separation of game traffic from HTTP reverse-proxy traffic
- No public router port exposure
- Security-conscious infrastructure documentation
