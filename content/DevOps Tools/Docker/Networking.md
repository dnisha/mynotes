---
longform:
  format: single
  title: Docker Networking
title: Networking
---

# Docker Networking Explained with Diagrams

## **1. Host Networking**
```
┌─────────────────────────────────────────┐
│           Docker Host                    │
├─────────────────────────────────────────┤
│  ┌─────────┐     ┌─────────┐           │
│  │   App   │     │   App   │           │
│  │ (Port 80)│     │(Port 8080)│         │
│  └─────────┘     └─────────┘           │
│       │               │                 │
│  ┌─────────────────────────────┐       │
│  │       Host Network Stack     │       │
│  │   (eth0: 192.168.1.100)      │       │
│  └─────────────────────────────┘       │
│                 │                       │
└─────────────────┼───────────────────────┘
                  │
           ┌──────┴──────┐
           │   Network   │
           │   Switch    │
           └─────────────┘
```

- **No isolation** - Container uses host's network directly
- **No NAT** - Port conflicts can occur
- **Command**: `docker run --network=host nginx`

---

## **2. Bridge Networking (Default)**
```
┌─────────────────────────────────────────┐
│           Docker Host                    │
│  ┌─────────┐     ┌─────────┐           │
│  │ Container│     │ Container│          │
│  │ 172.17.0.2│    │ 172.17.0.3│         │
│  │ Port 80   │    │ Port 80   │         │
│  └─────┬─────┘    └─────┬─────┘         │
│        │                │                │
│  ┌─────┴────────────────┴─────┐         │
│  │    docker0 Bridge           │         │
│  │    172.17.0.1/16            │         │
│  └─────────────┬───────────────┘         │
│                │                          │
│  ┌─────────────┴─────────────┐           │
│  │    Host Network Stack      │           │
│  │   eth0: 192.168.1.100      │           │
│  └─────────────┬─────────────┘           │
└────────────────┼──────────────────────────┘
                 │
           ┌─────┴─────┐
           │  Router   │
           │  /Switch  │
           └───────────┘
```

- **Default network** (`docker0` bridge)
- **NAT enabled** - Containers get private IPs
- **Port mapping**: `docker run -p 8080:80 nginx`
- **Isolated** from host network

---

## **3. Custom Bridge Network**
```
┌─────────────────────────────────────────┐
│           Docker Host                    │
│                                          │
│  ┌─────────┐     ┌─────────┐           │
│  │   Web   │     │   DB    │           │
│  │ 10.0.0.2│     │10.0.0.3 │           │
│  │  my-app │     │ my-app  │           │
│  │ network │     │ network │           │
│  └─────┬───┘     └────┬────┘           │
│        │              │                 │
│  ┌─────┴──────────────┴─────┐         │
│  │    Custom Bridge          │         │
│  │     my-app (10.0.0.1/24)  │         │
│  └─────────────┬─────────────┘         │
│                │                        │
│  ┌─────────────┴─────────────┐         │
│  │    docker0 Bridge          │         │
│  └─────────────┬─────────────┘         │
│                │                        │
│  ┌─────────────┴─────────────┐         │
│  │    Host Network Stack      │         │
│  └─────────────────────────────┘         │
└──────────────────────────────────────────┘
```

- **Better isolation** than default bridge
- **Automatic DNS resolution** between containers
- **Command**: 
  ```bash
  docker network create my-app
  docker run --network=my-app nginx
  ```

---

## **4. Overlay Network (Swarm Mode)**
```
┌──────────────────┐     ┌──────────────────┐
│   Docker Node 1  │     │   Docker Node 2  │
│ ┌─────────────┐  │     │  ┌─────────────┐ │
│ │  Service A  │  │     │  │  Service B  │ │
│ │ 10.0.0.2    │◄─┼─────┼─►│  10.0.0.3   │ │
│ │ overlay-net │  │     │  │ overlay-net │ │
│ └─────────────┘  │     │  └─────────────┘ │
│        │         │     │         │        │
│ ┌──────┴──────┐  │     │  ┌──────┴──────┐ │
│ │   VXLAN     │  │     │  │   VXLAN     │ │
│ │  Tunnel     │  │     │  │  Tunnel     │ │
│ └──────┬──────┘  │     │  └──────┬──────┘ │
│        │         │     │         │        │
│ ┌──────┴──────┐  │     │  ┌──────┴──────┐ │
│ │   Physical  │  │     │  │   Physical  │ │
│ │   Network   │  │     │  │   Network   │ │
│ └─────────────┘  │     │  └─────────────┘ │
└──────────────────┘     └──────────────────┘
         │                        │
         └──────────┬─────────────┘
                    │
            ┌───────┴───────┐
            │   Network     │
            │ (Data Center) │
            └───────────────┘
```
- **Multi-host networking** for Docker Swarm
- **Encrypted** by default
- **Service discovery** across nodes
- **Command**: `docker network create --driver overlay overlay-net`

---

## **5. Macvlan Network**
```
┌─────────────────────────────────────────┐
│           Docker Host                    │
│                                          │
│  ┌─────────┐     ┌─────────┐           │
│  │Container│     │Container│           │
│  │MAC: A   │     │MAC: B   │           │
│  │IP:      │     │IP:      │           │
│  │192.168.1.101│ │192.168.1.102│       │
│  └─────┬───┘     └─────┬───┘           │
│        │               │                │
│  ┌─────┴───────────────┴──────┐        │
│  │   Physical Network (eth0)   │        │
│  │   192.168.1.100/24          │        │
│  │   MAC: AA:BB:CC:DD:EE:FF    │        │
│  └─────────────┬───────────────┘        │
└────────────────┼─────────────────────────┘
                 │
           ┌─────┴─────┐
           │  Router   │
           │192.168.1.1│
           └───────────┘
```

- **Containers appear as physical devices**
- **No NAT** - Direct network access
- **Each container gets own MAC & IP**
- **Command**: 
  ```bash
  docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 macvlan-net
  ```

---

## **6. None Network**
```
┌─────────────────────────────────────────┐
│           Docker Host                    │
│                                          │
│  ┌─────────────────────────────┐       │
│  │         Container            │       │
│  │    ┌──────────────────┐     │       │
│  │    │    No Network    │     │       │
│  │    │    Interface     │     │       │
│  │    └──────────────────┘     │       │
│  └─────────────────────────────┘       │
│                                          │
│  ┌─────────────────────────────┐       │
│  │    Host Network Stack        │       │
│  └─────────────────────────────┘       │
└─────────────────────────────────────────┘
```
- **Complete network isolation**
- **Only loopback interface** (127.0.0.1)
- **Useful for** security-sensitive containers
- **Command**: `docker run --network=none alpine`

---

## **Network Comparison Table**

| Network Type | Isolation | NAT | Performance | Use Case |
|-------------|-----------|-----|-------------|----------|
| **Host** | None | No | Best | High-performance apps |
| **Bridge** | Medium | Yes | Good | Single-host apps |
| **Custom Bridge** | High | Optional | Good | Multi-container apps |
| **Overlay** | High | No | Medium | Multi-host/Swarm |
| **Macvlan** | Low | No | Best | Legacy apps needing real IPs |
| **None** | Complete | N/A | N/A | Security-sensitive |

---

## **Key Commands**

```bash
# List networks
docker network ls

# Inspect network
docker network inspect bridge

# Create custom bridge
docker network create --driver bridge my-network

# Connect container to network
docker network connect my-network container-name

# Disconnect container
docker network disconnect my-network container-name

# Remove network
docker network rm my-network
```

## **Practical Example: Multi-Tier App**
```
┌─────────────────────────────────────────┐
│           Docker Host                    │
│                                          │
│  ┌─────────┐      ┌─────────┐          │
│  │   Web   │──────►   App   │          │
│  │ (nginx) │      │ (Node.js)│         │
│  │front-tier│      │ app-tier │         │
│  └─────┬───┘      └────┬────┘          │
│        │               │                │
│  ┌─────┴───────────────┴──────┐        │
│  │       Database              │        │
│  │       (PostgreSQL)          │        │
│  │       db-tier               │        │
│  └─────────────────────────────┘        │
│                                          │
└──────────────────────────────────────────┘
```

**Setup**:
```bash
# Create networks
docker network create front-tier
docker network create app-tier
docker network create db-tier

# Run services on specific networks
docker run -d --network db-tier --name db postgres
docker run -d --network app-tier --link db:db --name app node-app
docker run -d --network front-tier -p 80:80 --link app:app --name web nginx
```