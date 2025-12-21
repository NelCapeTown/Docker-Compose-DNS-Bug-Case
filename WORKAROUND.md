# Workaround for Docker Desktop DNS Issue

## The Problem

Docker Desktop's DNS resolution is broken on custom bridge networks. Services cannot resolve each other by hostname.

## The Workaround (Ugly but Functional)

We had to implement a hacky solution using static IPs and manual DNS entries:

### 1. Assign Static IPs

```yaml
services:
  dns-test-postgres:
    networks:
      test_network:
        ipv4_address: 172.23.0.5  # Fixed IP

networks:
  test_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.23.0.0/24
```

### 2. Use `extra_hosts` to Bypass DNS

```yaml
services:
  dns-test-n8n:
    extra_hosts:
      - "dns-test-postgres:172.23.0.5"  # Manual /etc/hosts entry
```

## Why This Is Bad

1. **Brittle**: IP conflicts if subnets overlap
2. **Manual**: Must maintain IP mappings manually
3. **Defeats Service Discovery**: The whole point of Docker networking
4. **Not Scalable**: Imagine doing this for 20 services
5. **Against Docker Best Practices**: Service names should "just work"

## What Should Work (But Doesn't)

This should be all we need:

```yaml
services:
  dns-test-postgres:
    networks:
      - test_network
  
  dns-test-n8n:
    environment:
      DB_HOST: dns-test-postgres  # Should resolve automatically
    networks:
      - test_network

networks:
  test_network:
    driver: bridge
```

## Timeline

- **Before Dec 15, 2024**: Everything worked fine
- **After Docker Desktop Update (Dec 15-16, 2024)**: DNS completely broken
- **Current Status**: Using workaround, waiting for fix

## Our Production Workaround

See our main repository (private) for the full implementation with:
- Static IP assignments for 3+ services
- Complete `extra_hosts` mappings
- Documentation explaining why this monstrosity exists

We shouldn't have to do this in 2024.
