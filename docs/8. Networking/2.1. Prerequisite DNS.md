# DNS Architecture: Local Resolution, DNS Servers, and Name Resolution Mechanics

In networked systems, computers communicate with each other using IP addresses. However, human operators and modern application architectures require human-readable names. 

The translation between hostnames and IP addresses is called **Name Resolution**. This process evolved from static local mapping files to distributed, hierarchical DNS servers. Below is a complete technical breakdown of how name resolution works, how the Linux operating system handles it, and how each configuration file operates.

---

## 1. Local Name Resolution: The `/etc/hosts` File

When two machines are placed on the same network (`192.168.1.10` and `192.168.1.11`), they can communicate using IP addresses directly:

```bash
ping 192.168.1.11
```

If the system administrator wants to refer to `192.168.1.11` by a name such as `db`, running `ping db` immediately fails with:

```text
ping: db: Name or service not known
```

The operating system cannot locate a destination IP for the string `db`.

### How `/etc/hosts` Works
The oldest and simplest mechanism to resolve names in Unix-like systems is the `/etc/hosts` file. It is a static, local lookup table stored on the machine.

To map `192.168.1.11` to the name `db`, an entry is appended to `/etc/hosts`:

```ini
# /etc/hosts
127.0.0.1       localhost
192.168.1.11    db
```

Now, any application running on this host (such as `ping`, `ssh`, or a database client) that queries the hostname `db` will read this file, extract `192.168.1.11`, and dispatch packets to that IP.

### Operating Principles and Caveats of `/etc/hosts`
* **Local Source of Truth:** The operating system trusts `/etc/hosts` unconditionally. It does not verify whether the remote machine's actual OS hostname is `db` (even if running `hostname` on that target machine returns `host2`).
* **Aliases:** A single IP can map to multiple names, or an existing public domain name can be overridden locally:
  ```ini
  192.168.1.11    db database.local www.google.com
  ```
  If this entry exists, navigating to `www.google.com` on this local machine sends traffic to `192.168.1.11`.
* **The Scaling Problem:** In an infrastructure with hundreds or thousands of nodes, maintaining `/etc/hosts` on every host is impossible. If a server's IP changes, every single host in the entire network must be manually updated.

---

## 2. Centralized Resolution: The DNS Server

To eliminate manual updates on individual hosts, name resolution is moved to a central service: a **DNS (Domain Name System) Server**.

Instead of storing records on every machine:
1. All hostname-to-IP records are stored in one central database on the DNS server (e.g., at IP `192.168.1.100`).
2. Client hosts send a query across the network to `192.168.1.100` whenever they need to resolve a hostname.
3. If a host's IP changes, only the DNS server's record is updated. All client systems immediately receive the new IP without local configuration changes.

---

## 3. Client Configuration: `/etc/resolv.conf`

To instruct the Linux kernel and local resolver library to use a central DNS server, the server's IP address is defined in `/etc/resolv.conf`.

```ini
# /etc/resolv.conf
nameserver 192.168.1.100
```

* **`nameserver` directive:** Specifies the IPv4 or IPv6 address of the DNS server that the client should query.
* Multiple `nameserver` lines can be defined. The resolver queries them in top-down order as fallbacks if the primary server fails to respond.

---

## 4. Name Resolution Order: `/etc/nsswitch.conf`

A Linux machine can obtain name mappings from both local files (`/etc/hosts`) and remote DNS servers (`/etc/resolv.conf`).

When both contain conflicting entries—for instance, `/etc/hosts` says `db` is `192.168.1.115`, while the DNS server says `db` is `192.168.1.116`—the host must know which source takes precedence.

This precedence is controlled by the **Name Service Switch** configuration file: `/etc/nsswitch.conf`.

Inspect the `hosts:` entry in `/etc/nsswitch.conf`:

```ini
# /etc/nsswitch.conf
hosts:          files dns
```

### The Lookup Flow
1. **`files`:** The system first searches local configuration files (`/etc/hosts`). If a matching hostname is found, the lookup completes immediately using that IP.
2. **`dns`:** If `/etc/hosts` does not contain the queried hostname, the system sends a DNS query to the nameserver specified in `/etc/resolv.conf`.

If the order is changed to `hosts: dns files`, the system queries the DNS server first and only consults `/etc/hosts` if the DNS server returns an NXDOMAIN (non-existent domain) error or fails.

---

## 5. External Resolution and Forwarding

If a host queries a public internet address like `www.facebook.com`:
* The hostname is not in `/etc/hosts`.
* The internal company DNS server (`192.168.1.100`) does not own or host records for `facebook.com`.

Two mechanisms can resolve external queries:

1. **Adding Public Upstream Resolvers to `/etc/resolv.conf`:**
   Public recursive DNS resolvers (such as Google's `8.8.8.8` or Cloudflare's `1.1.1.1`) can be added directly to the client's configuration:
   ```ini
   nameserver 192.168.1.100
   nameserver 8.8.8.8
   ```
2. **DNS Forwarding (Standard Practice):**
   Instead of modifying every client machine, the internal DNS server (`192.168.1.100`) is configured with an **upstream forwarder**. When the internal server receives a request for a domain outside its zone, it forwards the request to public root/recursive servers, retrieves the IP, caches it, and returns it to the client.

---

## 6. Domain Name Hierarchy and the DNS Tree

Internet domain names are structured hierarchically, separated by dots:

```text
                       . (Root Zone)
                     /   \
                 .com     .org    .net    (Top-Level Domains / TLDs)
                 /
            google.com                     (Second-Level Domain)
            /        \
      maps.google.com  apps.google.com     (Subdomains)
```

### Components of a Domain Name:
* **Root Domain (`.`):** The apex of the DNS hierarchy. Represented implicitly by the final dot in a Fully Qualified Domain Name (FQDN) such as `apps.google.com.`. Managed by 13 root server clusters globally.
* **Top-Level Domain (TLD):** The rightmost label (`.com`, `.org`, `.net`, `.edu`, or country codes like `.uk`, `.de`). TLD servers track the authoritative servers for every domain registered under them.
* **Second-Level Domain:** The registered domain name assigned to an organization (e.g., `google` in `google.com`).
* **Subdomain:** Additional namespace partitions created by the domain owner to route traffic to specific services (e.g., `maps`, `drive`, `mail`).

### Iterative DNS Resolution Step-by-Step
When a client requests `apps.google.com`:
1. The client queries the local/recursive resolver.
2. If not cached, the recursive resolver queries a **Root Server (`.`)**. The root server replies: *"I do not know `apps.google.com`, but here are the IPs for the `.com` TLD servers."*
3. The recursive resolver queries the **`.com` TLD Server**. The TLD server replies: *"I do not know the exact host, but here are the Authoritative Nameservers for `google.com`."*
4. The recursive resolver queries **Google's Authoritative Nameserver**. This server holds the actual database zone file and responds: *"The IP for `apps.google.com` is `142.250.180.206`."*
5. The recursive resolver returns the IP to the client and caches it locally based on its **TTL (Time To Live)** value so subsequent requests avoid traversing the entire tree.

---

## 7. Search Domains and FQDNs: The `search` Directive

In corporate and enterprise networks, systems are named with full domain suffixes, such as `web.mycompany.com` or `db.prod.mycompany.com`.

If an engineer runs `ping web`, the DNS server cannot resolve it directly because the record is stored as `web.mycompany.com`, not `web`.

Typing the full domain name every time is inefficient. To resolve short names automatically, the **`search`** directive in `/etc/resolv.conf` is used:

```ini
# /etc/resolv.conf
nameserver 192.168.1.100
search mycompany.com prod.mycompany.com
```

### How the `search` Directive Works:
* When a user runs `ping web`, the resolver determines that `web` is not an FQDN (it does not end with a dot and has fewer dots than the `ndots` threshold).
* The resolver appends the first search domain: `web.mycompany.com` and queries the DNS server.
* If that succeeds, the IP is returned.
* If it returns NXDOMAIN, the resolver appends the second search domain: `web.prod.mycompany.com` and queries again.
* If a fully qualified name is queried directly (e.g., `ping api.external.com`), the resolver skips the search domains and queries the target directly.

---

## 8. Common DNS Record Types

DNS servers store mapping data inside **Zone Files** using specific resource record types:

| Record Type | Full Name | Function / Purpose | Example Entry in Zone File |
| :--- | :--- | :--- | :--- |
| **A** | Address Record | Maps a hostname to an **IPv4 address** (32-bit). | `web.mycompany.com.  IN  A  192.168.1.50` |
| **AAAA** | Quad-A Record | Maps a hostname to an **IPv6 address** (128-bit). | `web.mycompany.com.  IN  AAAA  2001:db8::1` |
| **CNAME** | Canonical Name | Creates an **alias** pointing one hostname to another hostname. | `eat.delivery.com.   IN  CNAME  food.delivery.com.` |
| **PTR** | Pointer Record | Used for **Reverse DNS (rDNS)**: maps an IP address back to a hostname. | `50.1.168.192.in-addr.arpa. IN PTR web.mycompany.com.` |
| **MX** | Mail Exchange | Points to the mail server responsible for accepting emails for the domain. | `mycompany.com.      IN  MX  10 mail.mycompany.com.` |
| **NS** | Name Server | Delegates a DNS zone to use a specific authoritative server. | `mycompany.com.      IN  NS  ns1.mycompany.com.` |
| **TXT** | Text Record | Stores arbitrary text; used for verification (SPF, DKIM, site ownership). | `mycompany.com.      IN  TXT "v=spf1 include:_spf.google.com ~all"` |

---

## 9. DNS Inspection and Troubleshooting Tools

Different utilities query different layers of the resolution stack. Understanding which tool queries what is critical for debugging.

### 1. `ping`
* Uses the **system resolver** (`gethostbyname` / `getaddrinfo`).
* Follows `/etc/nsswitch.conf`. It reads `/etc/hosts` **first** before querying DNS.
* Tests both name resolution and network layer (ICMP) reachability.

### 2. `nslookup`
* Queries the **DNS server directly** (using the nameserver defined in `/etc/resolv.conf` or a custom server specified as an argument).
* **Completely ignores `/etc/hosts`**. If an entry is in `/etc/hosts` but not on the DNS server, `nslookup` will report that the host does not exist.
* Syntax:
  ```bash
  nslookup web.mycompany.com
  # Querying a specific server:
  nslookup web.mycompany.com 192.168.1.100
  ```

### 3. `dig` (Domain Information Groper)
* The industry-standard diagnostic tool for DNS. Like `nslookup`, it queries DNS servers directly and bypasses `/etc/hosts`.
* Returns the raw DNS response, including status flags (`NOERROR`, `NXDOMAIN`, `SERVFAIL`), the authoritative answer section, TTL values, and response time.
* Syntax:
  ```bash
  # Standard query:
  dig web.mycompany.com

  # Short output (IP address only):
  dig web.mycompany.com +short

  # Query a specific record type on a specific DNS server:
  dig @192.168.1.100 web.mycompany.com CNAME
  ```

---

## 10. Summary of Key Files and Directives

| File / Command       | Primary Function                                                   | Key Syntax / Directive                            |
| :------------------- | :----------------------------------------------------------------- | :------------------------------------------------ |
| `/etc/hosts`         | Local, static hostname-to-IP lookup table.                         | `<IP> <hostname> [alias...]`                      |
| `/etc/resolv.conf`   | Configures upstream DNS servers and search domains.                | `nameserver <IP>`<br>`search <domain1> <domain2>` |
| `/etc/nsswitch.conf` | Dictates name resolution order for the system resolver.            | `hosts: files dns`                                |
| `nslookup <host>`    | CLI tool to query DNS records directly (skips local files).        | `nslookup <host> [server]`                        |
| `dig <host>`         | Advanced diagnostic tool showing full DNS query/response metadata. | `dig [@server] <host> [type]`                     |