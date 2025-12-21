# Docker Desktop DNS Bug - Summary for Support

## Repository for Docker Desktop Team

**GitHub Repository**: [To be created - docker-dns-reproduction]

This is a minimal reproduction case for a critical DNS resolution bug in Docker Desktop that appeared after updates in mid-December 2024.

## The Bug in One Sentence

Service name resolution fails on custom bridge networks, returning "bad address" despite containers being on the same network.

## Impact

- **Severity**: Critical - Breaks service discovery completely
- **Workaround**: Exists but ugly (static IPs + manual /etc/hosts)
- **Scope**: All custom bridge networks affected
- **First Observed**: December 15-16, 2024 (after Docker Desktop update)

## Quick Reproduction

```bash
# Clone the repo
git clone [REPO_URL]
cd docker-dns-reproduction

# Copy environment file
cp .env.example .env

# Start services
docker compose up -d

# Wait 30 seconds for postgres to be healthy
sleep 30

# Check n8n logs - you'll see connection errors
docker compose logs dns-test-n8n

# Prove DNS is broken
docker compose exec dns-test-n8n sh -c "ping dns-test-postgres"
# Result: "ping: bad address 'dns-test-postgres'"
```

## What's Included in the Repo

1. **docker-compose.yml** - Minimal reproduction (no workarounds)
2. **README.md** - Issue description and architecture
3. **TEST-INSTRUCTIONS.md** - Detailed testing steps for Docker team
4. **WORKAROUND.md** - The ugly hack we had to implement
5. **.env.example** - Configuration template

## Technical Details

### What Fails
- `ping dns-test-postgres` → "bad address"
- `nslookup dns-test-postgres` → Resolution fails
- Service-to-service communication by hostname

### What Works
- Ping by IP address (if you manually look it up)
- Communication via hardcoded IPs
- Everything worked fine before December 15, 2024

### Network Configuration
- Custom bridge network (standard setup)
- No special IPAM configuration
- Default Docker DNS (127.0.0.11)
- Service names should auto-resolve (but don't)

## Environment

```
OS: Linux (Ubuntu/Debian based)
Docker Desktop: Version from December 2024 updates
Docker Compose: v2.x
Network Driver: bridge (custom network)
DNS: Docker embedded DNS (127.0.0.11)
```

## Our Production Workaround (Not in Test Repo)

In our actual project, we had to implement this monstrosity:

```yaml
networks:
  app_network:
    ipam:
      config:
        - subnet: 172.22.0.0/24

services:
  postgres:
    networks:
      app_network:
        ipv4_address: 172.22.0.5

  n8n:
    extra_hosts:
      - "postgres:172.22.0.5"  # Manual DNS bypass
      - "mcp:172.22.0.3"
```

This defeats the entire purpose of Docker's service discovery.

## Timeline

- **Before Dec 15**: Everything worked perfectly
- **Dec 15-16**: Docker Desktop update rolled out
- **Dec 17**: DNS resolution completely broken
- **Dec 17-19**: Implemented ugly workaround with static IPs
- **Dec 21**: Created this reproduction case for Docker support

## Expected vs Actual Behavior

### Expected (Docker Documentation)
```yaml
services:
  app:
    environment:
      DB_HOST: database  # Should "just work"
    networks:
      - my_network
  
  database:
    networks:
      - my_network
```

### Actual (Current Broken State)
```yaml
services:
  app:
    environment:
      DB_HOST: 172.22.0.5  # Hardcoded IP required
    extra_hosts:
      - "database:172.22.0.5"  # Manual /etc/hosts entry
    networks:
      my_network:
        ipv4_address: 172.22.0.10

  database:
    networks:
      my_network:
        ipv4_address: 172.22.0.5

networks:
  my_network:
    ipam:
      config:
        - subnet: 172.22.0.0/24
```

## Questions for Docker Desktop Team

1. Is this a known regression from the December 2024 updates?
2. Was there an intentional change to embedded DNS behavior?
3. Are there diagnostic tools we should run?
4. Is there a Docker Desktop setting that might affect this?
5. What's the ETA on a fix?
6. Should we file a GitHub issue on the docker/for-linux repo?

## How to Use This Repo with Docker Support

1. Share the GitHub repo link
2. Ask them to follow TEST-INSTRUCTIONS.md
3. They'll see the DNS failure in real-time
4. Point them to WORKAROUND.md to see what we had to do
5. Emphasize this broke existing working code

## Screenshots to Capture

When testing, capture these screenshots for Docker support:

1. `docker compose ps` showing all services running
2. `docker compose logs dns-test-n8n` showing connection errors
3. `docker compose exec dns-test-n8n sh -c "ping dns-test-postgres"` showing "bad address"
4. `docker network inspect docker-dns-test_test_network` showing both containers listed
5. Docker Desktop version from Settings → About

## Contact Information

This reproduction case was created to help Docker Desktop team identify and fix a critical regression in service discovery.

**Main Project**: n8n + PostgreSQL + MCP Server (private repo)  
**Test Case**: docker-dns-reproduction (public repo - this one)  
**Date Created**: December 21, 2025  
**Status**: Awaiting Docker Desktop team response

---

**Note for Docker Team**: This is not an edge case or misconfiguration. This is standard Docker Compose usage that worked for years and broke suddenly. We've tested on multiple machines and can reproduce consistently.
