# CloudFront — the AWS CDN

> **In one sentence:** CloudFront is a **CDN** (*Content Delivery Network*) that keeps copies of your content in **edge locations** around the world and serves the user from the closest one, making the site faster, safer and cheaper to serve.

> **Docs:** [What is Amazon CloudFront?](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) · [How CloudFront delivers content](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html) · [CloudFront product page](https://aws.amazon.com/cloudfront/)

> **Course (Udemy path, Week 1):** section *CloudFront*. **Live demo from the workshop:** https://dhdwvoyc20mo5.cloudfront.net/ (try `curl -I` on it and read `X-Cache`, `Via` and `X-Amz-Cf-Pop`).

---

## 1. Analogy to remember

- **S3** = the store's **central warehouse** (a single address).
- **CloudFront** = **branches/mini-stockrooms** spread across cities.

A customer in Porto Alegre, Brazil, does not wait for a truck to come from Virginia (USA): they pick the product up at the branch near them. If the branch **does not have** the item, it fetches it from the warehouse, delivers it and **keeps a copy** for the next customer.

---

## 2. What it is (and what it is NOT)

**It is:** a **CDN** that delivers content from edge locations close to the user.

**It is not:**
- Managed DNS → that is **Route 53**.
- A load balancer for EC2 → that is **ELB/ALB**.
- A database → that is RDS/DynamoDB.

**Benefits:**
- **Lower latency** (content close to the user).
- Free and easy **HTTPS** (certificates via ACM).
- **Less load and cost** on the origin (S3 is only queried on a *cache miss*).
- **Security:** hides the bucket, and integrates with **AWS WAF** and **Shield Standard** (basic DDoS protection at no extra cost).
- Global **availability and scale** without you managing anything.

> **Docs:** [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html) · [AWS Shield (DDoS protection)](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-overview.html) · [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)

---

## 3. Essential vocabulary

| Term | Meaning |
|---|---|
| **Edge location** | Point of presence (PoP) close to the user that holds the cache |
| **Regional edge cache** | An intermediate cache layer between the edges and the origin |
| **Distribution** | The CloudFront "configuration": defines origins, behaviors, HTTPS, domain. Generates an address like `dxxxxxxxx.cloudfront.net` |
| **Origin** | Where the original content comes from: an **S3 bucket**, an ALB, an HTTP server |
| **Behavior** | Rules per path (`/api/*` → one origin, `*` → another) and cache policy |
| **Cache hit / miss** | Hit = the edge already had it. Miss = it had to fetch from the origin |
| **TTL** | How long the object stays in cache |
| **Invalidation** | A request to **remove** objects from the cache before the TTL (e.g. `/*` after a deploy) |
| **Default root object** | The file served at `/` (usually `index.html`) |

### Flow of a request

```
User --> Edge location (cached?)
           ├─ HIT  --> answers directly (fast)
           └─ MISS --> (regional edge cache) --> Origin (S3) --> stores in cache --> answers
```

> **Docs:** [Manage how long content stays in the cache (expiration)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html) · [Invalidate files](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html) · [Specify a default root object](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html) · [Distribution settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-values-specify.html)

---

## 4. CloudFront + S3 (the heart of the workshop)

### The problem
If the bucket is **public** so that the site works, anyone can reach the files directly, without your own HTTPS, without cache and without control.

### The solution: **private** S3 + **OAC**
**OAC (Origin Access Control)** is the mechanism that lets **only CloudFront** (your distribution) read the private bucket.

```
Browser ──HTTPS──> CloudFront ──(OAC: signed request)──> PRIVATE S3
                                                  ✗ direct bucket access: Access Denied
```

> **OAC vs OAI:** **OAI** (Origin Access Identity) is the **old/legacy** method. **OAC** is the current, recommended one (it supports more scenarios, such as SSE-KMS). If you see "OAI" in old material, know that it has been superseded.

### Step by step (this is what the demo shows)

1. **Create the S3 bucket** (private, Block Public Access **on**) and upload the site (`index.html`, `css/`, `js/`, `img/`).
2. **Create the distribution** in CloudFront:
   - **Origin:** the S3 bucket (the bucket's REST endpoint, **not** the "website" endpoint).
   - **Origin access:** **Origin Access Control (OAC)**. The console offers to create the OAC.
   - **Default root object:** `index.html`.
   - **Viewer protocol policy:** *Redirect HTTP to HTTPS*.
3. **Update the S3 bucket policy:** the console generates the ready-made text. It allows `s3:GetObject` to `cloudfront.amazonaws.com` **with the condition** that it is your distribution (`AWS:SourceArn`). See the example in `s3.md`.
4. **Wait for the distribution to deploy** (a few minutes) and open `https://dxxxx.cloudfront.net`.
5. **Prove the block:** the direct S3 object URL must return **Access Denied**.
6. **Update the site:** upload new files and create an **invalidation** (`/*`) to see the change right away.

> **Docs:** [Restrict access to an Amazon S3 origin (OAC)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) · [Require HTTPS between viewers and CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-viewers-to-cloudfront.html)

### Custom domain and HTTPS (just mention)
- Custom domain → **Alternate domain name (CNAME)** on the distribution + a record in **Route 53** (or another DNS).
- Certificate → **AWS Certificate Manager (ACM)**. ⚠️ For CloudFront, the certificate **must be in the `us-east-1` region** (N. Virginia).

> **Docs:** [Use alternate domain names](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html) · [Requirements for using HTTPS with alternate domain names](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html) · [Route traffic to a CloudFront distribution (Route 53)](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-cloudfront-distribution.html)

---

## 5. Final picture: everything together

```
                     ┌────────────── AWS ──────────────┐
 User ──HTTPS──>     │  CloudFront (closest edge)      │
                     │        │  OAC + bucket policy   │
                     │        v                        │
                     │   S3 (private, Standard)        │
                     └─────────────────────────────────┘

 IAM : who creates/manages, and the bucket policy that only accepts CloudFront
 S3  : stores the site     |   CloudFront: delivers fast and securely
 VPC : not part of the static site (comes in when there is a backend)
```

> **Docs:** [AWS Reference Architecture Diagrams](https://docs.aws.amazon.com/reference-architecture-diagrams/)

---

## 6. Useful details

- **Cache and deploys:** files with a **versioned name** (`app.3f9a.js`) avoid depending on invalidation. For `index.html`, use a short TTL or invalidate.
- **Invalidation has a cost** beyond a free monthly quota: the first 1,000 paths per month are free, then each additional path is charged (confirm the current price on the pricing page).
- **SPA (React/Vue):** when the user opens `/product/123`, S3 returns 403/404. Fix: a **custom error response** returning `/index.html` with status 200.
- **Subfolders with OAC:** `/products/` does not become `/products/index.html` by itself; use a **CloudFront Function** or explicit links.
- **Price:** CloudFront is *pay-as-you-go* (data transfer out and requests), with an **Always Free** monthly allowance (1 TB of data transfer out and 10,000,000 requests at the time of writing). It also offers **flat-rate plans**: a fixed monthly price with no overage charges, starting with a free plan. Check the current values and plans at https://aws.amazon.com/cloudfront/pricing/ before quoting numbers.
- **Price class:** you can limit the edges used (e.g. North America/Europe) to reduce cost.
- **Not only static:** with *behaviors* the same CloudFront can point `/api/*` to an API Gateway/ALB.

> **Docs:** [Customize error responses](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GeneratingCustomErrorResponses.html) · [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html) · [Distribution settings (price class, behaviors)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-values-specify.html) · [CloudFront pay-as-you-go pricing](https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/) · [Flat-rate pricing plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html) · [CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/)

---

## 7. Common beginner pitfalls

- Using the S3 **website** endpoint as the origin (it does not work with OAC and requires a public bucket).
- Forgetting to **update the bucket policy** after creating the OAC → `403 Access Denied`.
- Changing the site and **not seeing the change** (cache!). Missing invalidation or a short TTL.
- Creating the ACM certificate in the wrong region (it must be `us-east-1`).
- Forgetting the **Default root object**.
- Thinking CloudFront only serves static content.

---

## 8. Hands-on practice (study account)

1. Follow the step by step in section 4 with a test `index.html` (a "Hello, store!" page).
2. Open `https://dxxxx.cloudfront.net` and confirm it works.
3. Try the direct S3 URL. It must return **Access Denied**.
4. Edit `index.html`, upload it, refresh the page (you may see the old version). Create an **invalidation** `/*` and refresh again.
5. Look at the `X-Cache` response header (DevTools → Network): `Hit from cloudfront` or `Miss from cloudfront`.
6. Compare your response headers with the workshop live demo: `curl -I https://dhdwvoyc20mo5.cloudfront.net/` (look at `X-Cache`, `Via`, `X-Amz-Cf-Pop` and `Server: AmazonS3`).
7. At the end, **disable and delete** the distribution, then empty and delete the bucket.

---

## 9. Self-check questions

- What is an edge location and what is an origin?
- What does OAC solve? What is the difference from OAI?
- What happens on a *cache miss*?
- Why did I update the file in S3 and the site is still old?
- In which region must the ACM certificate be for CloudFront?

---

## References

- CloudFront product page: https://aws.amazon.com/cloudfront/
- What is Amazon CloudFront?: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html
- How CloudFront delivers content: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html
- Restrict access to an S3 origin (OAC): https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html
- Expiration (TTL): https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html
- Invalidation: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
- HTTPS between viewers and CloudFront: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-viewers-to-cloudfront.html
- Alternate domain names: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html
- HTTPS requirements for alternate domain names (ACM in us-east-1): https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html
- Route 53 to CloudFront: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-cloudfront-distribution.html
- AWS Certificate Manager: https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html
- Default root object: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html
- Custom error responses: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GeneratingCustomErrorResponses.html
- CloudFront Functions: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html
- Pricing: https://aws.amazon.com/cloudfront/pricing/
- Pay-as-you-go pricing (invalidation, Always Free): https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/
- Flat-rate pricing plans: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html
- Workshop live demo (S3 + CloudFront): https://dhdwvoyc20mo5.cloudfront.net/
- Course learning path, Week 1 (login required): https://aircompany.udemy.com/learning-paths/12612441/
- AWS Reference Architecture Diagrams: https://docs.aws.amazon.com/reference-architecture-diagrams/
