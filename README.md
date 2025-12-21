# Docker Desktop DNS Resolution Bug - Reproduction Case

## Issue Summary

Docker Desktop (version from December 2024 updates) fails to resolve service hostnames on custom bridge networks, returning "bad address" errors despite services being on the same network.

## Expected Behavior

Containers on the same Docker network should be able to resolve each other by service name:
```bash
# From n8n container
ping postgres  # Should resolve and respond
```

## Actual Behavior

DNS resolution completely fails:
```bash
# From n8n container
ping postgres
# Returns: "ping: bad address 'postgres'"
```

## Environment

- **OS**: Linux (Ubuntu/Debian based)
- **Docker Desktop**: Version with December 2024 updates
- **Docker Compose**: v2.x

## Reproduction Steps

1. Clone this repository
2. Run `docker compose up -d`
3. Wait for all services to be healthy
4. Exec into the n8n container: `docker compose exec dns-test-n8n sh`
5. Try to ping postgres: `ping dns-test-postgres`
6. Observe the DNS failure

## What Works

- Containers can ping each other by IP address
- Containers can communicate via hardcoded IPs
- Docker's built-in bridge network works fine
- Custom networks with static IPs + extra_hosts workaround works

## What Doesn't Work

- Standard service name resolution (e.g., "postgres", "mcp")
- Custom bridge networks with automatic DNS
- Service discovery that Docker Compose is supposed to provide

## Architecture

This is a minimal n8n + PostgreSQL setup that demonstrates the issue:

```
┌─────────────┐
│  dns-test-  │  Tries to connect to "dns-test-postgres"
│    n8n      │  ❌ DNS resolution fails
└──────┬──────┘
       │
       ▼ (should connect here)
┌─────────────┐
│  dns-test-  │  IP: Assigned by Docker
│  postgres   │  Hostname: Should resolve but doesn't
└─────────────┘
```

## Files in This Repo

- `docker-compose.yml` - Minimal reproduction setup (NO workarounds)
- `.env.example` - Required environment variables
- `README.md` - This file
- `WORKAROUND.md` - The hacky solution we had to implement

## For Docker Desktop Team

This is a clean reproduction case. We've confirmed this works with static IPs and `extra_hosts` entries, but that's not a proper solution. We need normal DNS resolution to work as documented.

Our production workaround (in a separate private repo) uses:
- Static IPs on custom networks
- Manual `extra_hosts` entries in docker-compose.yml
- This is brittle and defeats the purpose of service discovery

## Contact

If you need more information or have questions, please open an issue in this repository.

## Related Issues

This appears to be related to Docker Desktop updates from mid-December 2024 that affected DNS resolution on custom bridge networks.
