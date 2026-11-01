# Setting Up a DNS Server: CoreDNS Architecture and Mechanics

A client machine uses `/etc/resolv.conf` to query a DNS server on port 53. To serve those requests, an actual service must listen on port 53, parse incoming DNS queries, resolve them against a backend or database of records, and return DNS response packets.

While several implementations exist (such as BIND9, Unbound, or Dnsmasq), **CoreDNS** is the modern standard in cloud-native infrastructure and serves as the default cluster DNS provider for **Kubernetes**.

Below is a structured technical breakdown of how CoreDNS operates, how it is deployed as a standalone binary, and how its plugin pipeline is configured.

---

## 1. What is CoreDNS?

CoreDNS is an authoritative, fast, and flexible DNS server written in Go. Its core architecture is built around a **plugin chaining model**:

```text
Incoming DNS Query (Port 53 UDP/TCP)
                 │
                 ▼
       ┌──────────────────┐
       │   CoreDNS Core   │
       └────────┬─────────┘
                │
                ▼ (Plugin Chain)
       ┌──────────────────┐
       │      errors      │  Logs errors
       └────────┬─────────┘
                │
                ▼
       ┌──────────────────┐
       │       log        │  Logs DNS queries
       └────────┬─────────┘
                │
                ▼
       ┌──────────────────┐
       │      cache       │  Answers from memory if cached
       └────────┬─────────┘
                │
                ▼
       ┌──────────────────┐
       │      hosts       │  Matches against local /etc/hosts file
       └────────┬─────────┘
                │ (fallthrough if not matched)
                ▼
       ┌──────────────────┐
       │     forward      │  Routes unmatched queries to external DNS
       └──────────────────┘
```

CoreDNS itself does very little native processing; almost all functionality (reading hosts files, forwarding to upstream resolvers, caching, rewriting headers, logging, and Kubernetes service discovery) is executed by **plugins**.

---

## 2. Installing and Running CoreDNS Manually

CoreDNS is distributed as a single statically compiled binary. It does not require external runtime dependencies.

### Step 1: Download and Extract the Binary
Using `curl` or `wget` to retrieve the release archive from GitHub:

```bash
# Download the compressed binary archive
curl -LO https://github.com/coredns/coredns/releases/download/v1.15.0/coredns_1.15.0_linux_amd64.tgz

# Extract the archive
tar -zxf coredns_1.12.4_linux_amd64.tgz
```

Extraction yields a single executable named `coredns`.

### Step 2: Execution and Default Port
Running the binary directly:

```bash
./coredns
```

* **Default Port:** DNS servers listen on **UDP port 53** (for standard queries under 512 bytes) and **TCP port 53** (for responses exceeding 512 bytes, DNSSEC, or zone transfers).
* **Privilege Requirement:** In Linux, ports below 1024 are privileged. Running CoreDNS on port 53 requires either `root` privileges (`sudo ./coredns`) or assigning the capability to the binary:
  ```bash
  sudo setcap 'cap_net_bind_service=+ep' ./coredns
  ```

---

## 3. Configuration File: The `Corefile`

When CoreDNS starts, it looks for a configuration file named **`Corefile`** in the working directory (or the path specified by the `-conf` flag).

The `Corefile` uses **Server Blocks** to define which zones to handle, on which ports, and which plugins to activate for each zone.

### Anatomy of the Configuration

Consider the standard configuration file:

```corefile
.:53 {
    # Use /etc/hosts to resolve hostnames
    hosts /etc/hosts {
        reload 1m
        fallthrough
    }

    # Forward unmatched queries to the host's resolver
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }

    cache 30
    log
    errors
}
```

Breaking down each directive and its operational purpose:

---

### Directive 1: Zone Definition (`.:53`)
* **`.` (Root Zone):** Defines the zone this block serves. The root dot means this server block matches **all** incoming domain queries (e.g., `db`, `web.internal`, `google.com`).
* **`:53`:** Specifies the listening port. CoreDNS binds to both UDP and TCP on port 53.

---

### Directive 2: The `hosts` Plugin
The `hosts` plugin instructs CoreDNS to read records from a standard `/etc/hosts` format file instead of maintaining a complex RFC-compliant DNS zone file.

```corefile
hosts /etc/hosts {
    reload 1m
    fallthrough
}
```

* **`/etc/hosts`:** The file path CoreDNS reads to construct its internal in-memory DNS database. If `/etc/hosts` on the DNS server has:
  ```ini
  192.168.1.11    db.internal db
  192.168.1.12    web.internal web
  ```
  CoreDNS will respond to queries for `db` or `db.internal` with `192.168.1.11`.
* **`reload 1m`:** Instructs CoreDNS to check the file every 1 minute for changes. When entries are added or modified in `/etc/hosts`, CoreDNS reloads them into memory without needing a process restart.
* **`fallthrough` (Critical Parameter):**
  * By default, if the `hosts` plugin receives a query for a name not found in `/etc/hosts`, it immediately terminates the chain and returns an `NXDOMAIN` (Non-Existent Domain) error.
  * Specifying `fallthrough` tells CoreDNS: *"If the queried domain does not match any entry in this file, do not fail. Pass the query down to the next plugin in the chain (the `forward` plugin)."*

---

### Directive 3: The `forward` Plugin
When a client requests an external domain (such as `google.com` or `github.com`), the query misses the `hosts` plugin and falls through to `forward`.

```corefile
forward . /etc/resolv.conf {
    max_concurrent 1000
}
```

* **`.`:** Applies this forwarding rule to all domains.
* **`/etc/resolv.conf`:** Instead of hardcoding static IP addresses (like `8.8.8.8`), this instructs CoreDNS to read the upstream nameservers from the host machine's own `/etc/resolv.conf`. If the host is configured to query an enterprise upstream resolver or public DNS, CoreDNS proxies the query there.
  * Alternatively, explicit upstream IPs can be listed: `forward . 8.8.8.8 1.1.1.1`.
* **`max_concurrent 1000`:** Sets the maximum number of concurrent outbound queries to avoid resource exhaustion under heavy query loads.

---

### Directive 4: Support Plugins (`cache`, `log`, `errors`)

* **`cache 30`:** Caches DNS responses in memory for up to 30 seconds. If multiple clients query `google.com` within this window, CoreDNS answers immediately from memory without generating outbound traffic to upstream servers.
* **`log`:** Prints every received DNS query, client IP, query type (A, AAAA, MX), and response code to standard output (`stdout`). Essential for troubleshooting name resolution issues.
* **`errors`:** Prints any internal processing errors or plugin failures to standard error (`stderr`).

---

## 4. End-to-End Query Lifecycle

Once CoreDNS is running on host `192.168.1.100`, client systems configure their `/etc/resolv.conf` to target it:

```ini
# On Client Host A (/etc/resolv.conf)
nameserver 192.168.1.100
```

### Scenario A: Resolving an Internal System (`db`)
1. Client Host A runs `ping db`.
2. Host A generates a DNS query: `A db?` and sends it to `192.168.1.100:53`.
3. CoreDNS receives the query and processes it through the plugin chain:
   * `errors` and `log`: Logs the incoming query.
   * `cache`: Checks if `db` is in memory. If not:
   * `hosts`: Finds `192.168.1.11 db` inside `/etc/hosts`.
4. CoreDNS constructs an A-record response: `db -> 192.168.1.11`, writes it to cache, and sends it back to Host A.

### Scenario B: Resolving an External System (`github.com`)
1. Client Host A runs `curl https://github.com`.
2. Host A sends a query: `A github.com?` to `192.168.1.100:53`.
3. CoreDNS processes the query:
   * `hosts`: Does not find `github.com`. Because **`fallthrough`** is active, it passes the query to `forward`.
   * `forward`: Relays the query to the upstream nameserver listed in the server's own `/etc/resolv.conf` (e.g., `8.8.8.8`).
4. The upstream nameserver returns GitHub's public IP.
5. CoreDNS logs the transaction, stores it in its 30-second `cache`, and sends the resolved IP back to Host A.

---

## 5. CoreDNS in Kubernetes (Preview)

In Kubernetes, manual maintenance of `/etc/hosts` or static record files is replaced entirely. 

Instead of the `hosts` plugin, CoreDNS uses the **`kubernetes` plugin**:
* CoreDNS connects directly to the **Kubernetes API Server**.
* It watches cluster events (creation and deletion of `Services` and `Pods`).
* As Services change, CoreDNS dynamically updates its in-memory records without modifying any flat files, allowing pods to resolve service names like `my-service.default.svc.cluster.local`.