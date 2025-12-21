# Testing Instructions for Docker Support Team

## Quick Start

1. **Clone the repository**
   ```bash
   git clone [REPO_URL]
   cd docker-dns-reproduction
   ```

2. **Create environment file**
   ```bash
   cp .env.example .env
   # No need to change values, defaults are fine for testing
   ```

3. **Start the services**
   ```bash
   docker compose up -d
   ```

4. **Wait for PostgreSQL to be healthy**
   ```bash
   docker compose ps
   # Wait until dns-test-postgres shows "healthy"
   ```

5. **Check n8n logs for the DNS error**
   ```bash
   docker compose logs dns-test-n8n
   # You should see connection errors mentioning "dns-test-postgres"
   ```

## Demonstrating the DNS Failure

### Test 1: Ping from n8n container

```bash
# Enter the n8n container
docker compose exec dns-test-n8n sh

# Try to ping postgres (will fail)
ping dns-test-postgres
# Expected output: "ping: bad address 'dns-test-postgres'"

# But ping by IP works (if you can find it)
docker inspect dns-test-postgres | grep IPAddress
ping [THE_IP_ADDRESS]  # This works
```

### Test 2: DNS lookup

```bash
# Still in n8n container
nslookup dns-test-postgres
# Should fail to resolve

# Check what DNS server is being used
cat /etc/resolv.conf
# Should show Docker's embedded DNS (127.0.0.11)
```

### Test 3: Network connectivity

```bash
# From host machine
docker network inspect docker-dns-test_test_network

# Check if both containers are on the network
# They should both be listed with IP addresses
```

## What You Should See

1. **PostgreSQL**: Starts successfully and becomes healthy
2. **n8n**: Starts but cannot connect to database
3. **Logs**: Show connection errors with hostname resolution failures
4. **Ping test**: Fails with "bad address"
5. **nslookup**: Fails to resolve service names
6. **Network inspect**: Shows both containers ARE on the same network

## Expected Result (What Should Happen)

- n8n should connect to postgres automatically
- Hostname `dns-test-postgres` should resolve to the container's IP
- No configuration beyond service names should be needed
- This is how Docker Compose has worked for years

## System Information

Please collect this information when testing:

```bash
# Docker version
docker version

# Docker Compose version
docker compose version

# Docker Desktop version (if applicable)
# Check in Docker Desktop UI -> Settings -> About

# OS Information
uname -a
cat /etc/os-release

# Network driver information
docker network ls
docker network inspect docker-dns-test_test_network
```

## Clean Up

```bash
# Stop and remove everything
docker compose down -v

# Remove volumes
docker volume rm docker_dns_test_postgres_data docker_dns_test_n8n_data
```

## Questions for Docker Team

1. Is this a known issue with recent Docker Desktop releases?
2. Was there a change to embedded DNS behavior in December 2024?
3. Are custom bridge networks affected specifically?
4. Is there a setting we should check in Docker Desktop preferences?
5. Are there any workarounds besides static IPs and manual /etc/hosts?

## Timeline

- **Working**: Before December 15, 2024
- **Broken**: After Docker Desktop update around December 15-16, 2024
- **Platform**: Docker Desktop on Linux (Ubuntu/Debian based systems)
- **Impact**: Complete service discovery failure on custom networks

## Contact

[Your contact information or GitHub issue link]
