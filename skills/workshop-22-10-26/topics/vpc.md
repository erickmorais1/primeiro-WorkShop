# VPC — Virtual Private Cloud

> **In one sentence:** a VPC is **your private, isolated network inside AWS**, where you decide who talks to whom and who has access to the internet.

> **Docs:** [What is Amazon VPC?](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) · [VPC product page](https://aws.amazon.com/vpc/)

---

## 0. The VPC and the static site

**The static site (S3 + CloudFront) does not run inside a VPC.** S3 and CloudFront are managed services outside your VPC.

So why study VPC? Because it is the foundation of practically everything that comes next in commerce: **EC2, RDS, Lambda with database access, ECS/EKS**. In short:

> *"For the static site, a VPC is not required. But as soon as the e-commerce gets a backend, an API or a database, it will live in a VPC."*

That is why the VPC does not appear in the site demo, but it is the base of everything that comes after.

---

## 1. Analogy to remember

A VPC is a **gated community**:

| Gated community | AWS |
|---|---|
| The land and its size | **VPC** and the **CIDR block** |
| Streets/blocks of the community | **Subnets** |
| Main gate to the public street | **Internet Gateway** |
| Street map that says "to go downtown, go through the gate" | **Route table** |
| A doorman at each house (remembers who came in, lets them out) | **Security Group** |
| A guardhouse at the block entrance (separate in/out rules, no memory) | **Network ACL** |
| Houses that only leave through the service gate and receive no outside visitors | **Private subnet + NAT Gateway** |

---

## 2. What a VPC is and why it matters

- A virtual network **logically isolated** from other VPCs and accounts.
- You control: the IP range, subnets, routing, gateways and security rules.
- **Scope:** it belongs to **one region** and **spans the AZs** (Availability Zones) of that region.
- **Why it matters:**
  - **Isolation and security:** the database does not need to be reachable from the internet.
  - **Control:** you decide what is public and what is private.
  - **Compliance and multi-tier architecture** (web, app, data).
- Every account already comes with a **default VPC** (CIDR `172.31.0.0/16`, public subnets). It is good for getting started fast, but for real projects create your own.

> **Docs:** [How Amazon VPC works](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) · [Default VPCs](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html)

---

## 3. CIDR block (the size of the network)

**CIDR** (*Classless Inter-Domain Routing*) is the `IP/prefix` notation, for example `10.0.0.0/16`.

- The number after the slash says **how many fixed bits** the network has. **The smaller the number, the bigger the network.**

| CIDR | Total addresses | Typical use |
|---|---|---|
| `/16` | 65,536 | The whole **VPC** |
| `/20` | 4,096 | A large subnet |
| `/24` | 256 | A typical subnet |
| `/28` | 16 | Smallest allowed subnet |

**AWS rules:**
- A VPC accepts **/16 up to /28** (IPv4).
- Use **private** ranges (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
- In each subnet, AWS **reserves 5 IPs** (the first 4 and the last one). A `/24` therefore has 251 usable.
- Each **subnet lives in a single AZ** and its CIDR is a **slice** of the VPC CIDR, with no overlap.
- **Plan with room to grow and avoid overlap** with other VPCs or with the on-premises network. It is hard to connect them later (peering, VPN).

**Design example:**

```
VPC: 10.0.0.0/16  (region us-east-1)
 ├─ Public   AZ-a : 10.0.1.0/24
 ├─ Public   AZ-b : 10.0.2.0/24
 ├─ Private  AZ-a : 10.0.11.0/24
 └─ Private  AZ-b : 10.0.12.0/24
```

Using **two AZs** gives high availability: if one AZ goes down, the other keeps running.

> **Docs:** [VPC CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html) · [Subnet sizing (reserved IP addresses)](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html) · [Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html) · [VPC peering](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-peering.html)

---

## 4. Public vs private subnet

> The difference **is not a label**: it is the **route table**.

| | **Public** | **Private** |
|---|---|---|
| Route `0.0.0.0/0` | → **Internet Gateway (IGW)** | **No** route to the IGW |
| Do resources need a public IP? | Yes, to be reached from the internet | No |
| Typical | Load balancer, NAT Gateway, bastion | Database, application servers |

**Rule:** a subnet is public when it has a **route to an Internet Gateway** in its route table. A NAT Gateway is **not** a requirement (in fact, it is used for *private* subnets).

> **Docs:** [Route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) · [Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)

---

## 5. How to reach the internet and public resources

Connectivity components:

- **Route table:** a list of rules "for this destination, go this way". Every VPC has the `local` route (internal communication).
- **Internet Gateway (IGW):** **bidirectional** entry and exit door between the VPC and the internet. It is managed, scales by itself and has no cost of its own.
- **NAT Gateway:** lets resources in the **private** subnet **go out** to the internet (e.g. to download updates) **without** being reachable from outside. It sits in a **public** subnet.
  - ⚠️ **It has a cost:** you pay for each hour the NAT Gateway is available **plus** each GB it processes. Remember to **delete** it in study accounts.
  - A *zonal* NAT Gateway sits in a **public** subnet (as above). AWS also offers a *regional* NAT Gateway that expands across AZs automatically.
- **VPC Endpoints:** **private** access to AWS services (such as S3) without going through the internet.
  - *Gateway endpoint* (S3 and DynamoDB, **no hourly or data-processing charge**) and *Interface endpoint* (PrivateLink, other services).
  - Handy to remember: "a private server reaches S3 without a NAT".

**Flows to keep in mind:**

```
User on the internet -> IGW -> PUBLIC subnet (load balancer/server with a public IP)

PRIVATE server -> NAT Gateway (in the public one) -> IGW -> internet   (outbound only)

PRIVATE server -> VPC Endpoint -> S3                                   (no internet)
```

> A public IPv4 address is also charged by the hour on AWS, whether it is in use or idle. Another good reason to keep resources in private subnets whenever possible.

> **Docs:** [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) · [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) · [What is AWS PrivateLink? (VPC endpoint concepts)](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html) · [Gateway endpoints for Amazon S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html) · [Pricing for NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html) · [Regional NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html) · [VPC pricing](https://aws.amazon.com/vpc/pricing/)

---

## 6. Layered security: Security Group vs NACL

Two different "firewalls". **It is the most important difference between the two.**

| | **Security Group (SG)** | **Network ACL (NACL)** |
|---|---|---|
| Level | **Instance / network interface (ENI)** | **Subnet** |
| **Stateful?** | **Yes**: if the outbound leg is allowed, the return is automatic | **No (stateless)**: outbound and return both need a rule |
| Rule types | **Allow** only | **Allow and Deny** |
| Evaluation | All rules together | In **numeric order** (lowest number first); the first match wins |
| Default | Denies inbound, **allows** outbound | The **default** NACL allows everything |
| Can reference another SG? | Yes (great for "only the app SG reaches the database") | No (CIDR only) |

**In short:** the **Security Group** works at the instance level and is **stateful**.

**How to explain "stateful" in one sentence:** *"the SG has memory: if it let the request in, it lets the response out."* On the NACL, which has no memory, you must allow the **ephemeral ports** (1024–65535) for the return traffic.

**Good practice:** the SG is the main day-to-day defense. The NACL is an extra layer, useful to **block** a specific IP (since the SG cannot deny).

> **Docs:** [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html) · [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html) · [Compare security groups and network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html)

---

## 7. Overview (diagram)

```
                          Internet
                              |
                       [Internet Gateway]
                              |
   ┌──────────────────────── VPC 10.0.0.0/16 ───────────────────────┐
   │                                                                 │
   │  PUBLIC subnet 10.0.1.0/24           (route: 0.0.0.0/0 -> IGW)  │
   │   ├─ Load Balancer  [SG: 443 open]                              │
   │   └─ NAT Gateway                                                │
   │            |                                                    │
   │  PRIVATE subnet 10.0.11.0/24         (route: 0.0.0.0/0 -> NAT)  │
   │   ├─ App / Lambda   [SG: accepts only from the LB]              │
   │   └─ Database       [SG: accepts only from the App]             │
   │                                                                 │
   │  (the NACL protects each subnet as an extra layer)              │
   └─────────────────────────────────────────────────────────────────┘
```

> **Docs:** [AWS Reference Architecture Diagrams](https://docs.aws.amazon.com/reference-architecture-diagrams/)

---

## 8. Common beginner pitfalls

- Thinking "public" is a checkbox on the subnet: it is **route table + public IP**.
- Forgetting that the SG **only allows** (it has no deny rule).
- Forgetting the ephemeral ports on the NACL and breaking the return traffic.
- Choosing a CIDR that is too small or that overlaps another network.
- Leaving a **NAT Gateway** running in the study account (cost).
- Putting a database in a public subnet.

---

## 9. Hands-on practice (study account)

1. In the console, use **"Create VPC" → "VPC and more"** and pick 2 AZs, 2 public, 2 private, **without** a NAT Gateway (to avoid cost). Read the generated *preview*.
2. Open the public subnet's **route table** and find the route `0.0.0.0/0 → igw-...`. Compare it with the private one.
3. Create a **Security Group** allowing HTTPS (443) and notice that outbound traffic is already allowed.
4. Compare the **default NACL** with an **SG**: where is there a *deny* rule? Which one has rule numbers?
5. When you finish, **delete the VPC** in the console (it removes the associated resources).

> **Docs:** [Create a VPC](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html)

---

## 10. Self-check questions

- How many usable IPs does a `/24` subnet have on AWS? (251)
- What makes a subnet public?
- What is the NAT Gateway for, and in which subnet does it sit?
- SG or NACL: which one is stateful? Which one accepts *deny*?
- Does a VPC belong to a region or to an AZ? And a subnet?

---

## References

- VPC product page: https://aws.amazon.com/vpc/
- What is Amazon VPC?: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html
- Default VPCs: https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html
- VPC CIDR blocks: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html
- Subnet sizing (reserved IPs): https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html
- Subnets: https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html
- Route tables: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html
- Internet gateways: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html
- NAT gateways: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
- VPC endpoints (PrivateLink concepts): https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html
- Gateway endpoints for S3: https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html
- Security groups: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html
- Network ACLs: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html
- Compare security groups and network ACLs: https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html
- VPC peering: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-peering.html
- Create a VPC: https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html
- Pricing for NAT gateways: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html
- Regional NAT gateways: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html
- VPC pricing: https://aws.amazon.com/vpc/pricing/
- AWS Reference Architecture Diagrams: https://docs.aws.amazon.com/reference-architecture-diagrams/
