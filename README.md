# snehal-ugale-Networkwalks-B084D-week2-footprint-scanning
Week 2: Footprinting, Reconnaissance & Network Scanning


> For educational purposes only. All activity was performed from my own Kali Linux lab against targets named in the course tasks. Nothing here is a vulnerability assessment of any organisation.

---

## 1. Summary

Week 2 covers the first two phases of an attack lifecycle: **reconnaissance (footprinting)** and **scanning**. I completed the minimum required set of three modules, plus the final report.

| Module | Type | Tool(s) | Target | Result in one line |
|---|---|---|---|---|
| [W2-PM1](W2-PM1-Footprinting-Multiple-Kali-Tools) | Elective | whois, whatweb, nslookup, curl, wafw00f, dnsrecon | networkwalks.com (course lab target) | Mapped registrar, hosting, web stack, DNS/mail setup and WAF |
| [W2-PM4](W2-PM4-Footprinting-theHarvester) | Elective | theHarvester | microsoft.com (passive, public sources) | Baidu returned nothing; all-sources run found ~10,000 hostnames, 41 IPs, 6 ASNs, 3 emails |
| [W2-PM5](W2-PM5-Network-Scanning-Zenmap) | Essential | Zenmap / Nmap | My Kali lab subnet (10.0.0.0/24) | 2 live hosts found; topology saved as PDF |
| W2-PM-FINAL | Essential | n/a | n/a | This report |

**Not attempted:** W2-PM2 (GHDB-based footprinting) and W2-PM3 (Maltego-based footprinting). The task allows any one elective as a minimum, and this report covers only the modules I completed.

---

## 2. Repository structure

```
NETWORKWALKS-IFEANYI-B084-WK2-FOOTPRINTING-AND-SCANNING/
├── README.md                                  <- this report (W2-PM-FINAL)
├── W2-PM1-Footprinting-Multiple-Kali-Tools/
│   ├── outputs/        01-whois.txt ... 06-dnsrecon.txt
│   ├── screenshots/    01 ... 10
│   └── docs/           lab notes (.docx) + WhatWeb troubleshooting report (.docx, .md)
├── W2-PM4-Footprinting-theHarvester/
│   ├── outputs/        theharvester-baidu.txt, theharvester-all-sources.txt
│   └── screenshots/    01 ... 05
└── W2-PM5-Network-Scanning-Zenmap/
    ├── outputs/        nmap-ping-scan.xml, topology.pdf
    └── screenshots/    01 ... 04
```

**A note on the output files.** The six W2-PM1 text files and the Baidu file were taken from the command transcripts in my lab Word document, with blank lines removed. The all-sources theHarvester file is the saved terminal output, unedited. The screenshots are the primary evidence.

---

## 3. Lab environment

| Item | Detail |
|---|---|
| Platform | Kali Linux virtual machine (VirtualBox), NAT Network 10.0.0.0/24 from Week 1 |
| Kali address | 10.0.0.2/24 on eth0 |
| WhatWeb | 0.6.4 (Ruby 3.3.8) |
| wafw00f | 2.4.2 |
| theHarvester | 4.11.1 |
| Nmap / Zenmap | 7.991 |
| Dates | Tool runs between 8 and 10 November 2026 |

### Passive vs. active techniques

Understanding which tools touch the target matters for both ethics and detection.

| Tool | Style | What the target can see |
|---|---|---|
| whois, nslookup | Passive / public records | Registry and DNS lookups only; nothing sent to the web server |
| theHarvester | Passive OSINT | Queries third-party sources (search engines, certificate and DNS datasets), not the target |
| dnsrecon | Light active | Queries to the target's DNS servers |
| curl -I, whatweb, wafw00f | Active (low volume) | Normal-looking HTTP requests appear in the web server logs |
| Zenmap ping scan | Active | Probes only my own lab subnet |

---

## 4. W2-PM1: Footprinting with multiple Kali tools

Target: **networkwalks.com**, the site named in the task.

### 4.1 Findings at a glance

| Tool | What it revealed |
|---|---|
| whois | Registrar GoDaddy; created 2019-11-06; expires 2027-11-06; registrant hidden behind a privacy service (Domains By Proxy); DNSSEC unsigned; HostGator name servers listed |
| whatweb | Apache; WordPress 7.1.4; WordPress Download Manager 3.3.76; jQuery 3.7.1; Google Tag Manager; public contact email; server IP 192.232.216.135 |
| nslookup | networkwalks.com resolves to 192.232.216.135 |
| curl -I | HTTP/2 200; `Server: Apache`; WordPress REST API advertised in a `Link` header; Secure + HttpOnly session cookie; hosting cache headers |
| wafw00f | Site is behind **ModSecurity (SpiderLabs)** WAF |
| dnsrecon | SOA and 2 NS (HostGator); MX `mail.networkwalks.com`; A record; SPF and Google verification TXT records; 8 cPanel autodiscover SRV records |

### 4.2 Task 1: whois

```
whois networkwalks.com
```

Output: [`outputs/01-whois.txt`](W2-PM1-Footprinting-Multiple-Kali-Tools/outputs/01-whois.txt)

![whois](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/01-whois-scan.png)

- **Registrar:** GoDaddy.com, LLC (IANA ID 146), with an abuse contact published.
- **Dates:** created 6 Nov 2019, last updated 12 Nov 2025, expires 6 Nov 2027.
- **Registrant:** "Registration Private" via Domains By Proxy, LLC, so no personal owner details are exposed. This is good practice.
- **Status flags:** all four `client*Prohibited` locks (delete, renew, transfer, update), which protect against unauthorised transfer.
- **Name servers:** the registry (Verisign) record lists two HostGator servers (`NS6135/NS6136`). The registrar's own record also lists two GoDaddy servers (`NS29/NS30.DOMAINCONTROL.COM`). I noted the difference but did not investigate which set is authoritative in practice.
- **DNSSEC:** unsigned.

**Attacker's view:** the name servers point to HostGator, so the hosting provider is known immediately. Dates and abuse contacts help with social engineering and with planning.

### 4.3 Task 2: whatweb

The assigned command, `whatweb networkwalks.com`, **did not work on my machine** (see section 4.8). The result below came from a workaround command:

```
echo "192.232.216.135 networkwalks.com" | sudo tee -a /etc/hosts
time whatweb -v -U "Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0" https://networkwalks.com
```

Output: [`outputs/02-whatweb.txt`](W2-PM1-Footprinting-Multiple-Kali-Tools/outputs/02-whatweb.txt)

![whatweb scan](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/04-whatweb-scan.png)

| Detected | Value |
|---|---|
| Status / title | 200 OK, "Networkwalks Academy" |
| Web server | Apache |
| CMS | WordPress 7.1.4 (from the meta generator tag) |
| Plugin | WordPress Download Manager 3.3.76 |
| JavaScript | jQuery 3.7.1, Google Tag Manager |
| Cookie | `__wpdm_client` (HttpOnly) |
| Exposed email | info@networkwalks.com |
| Uncommon headers | permissions-policy, link, upgrade, referrer-policy, x-endurance-cache-level, x-nginx-cache |

**Caveat:** WhatWeb also reported `Bootstrap[7.1.4]`. That is almost certainly a **false positive**, because Bootstrap does not use that version number and it matches the WordPress version string. Version numbers in generator tags can also be hidden or altered, so they are indicators, not proof.

**Attacker's view:** exact software and plugin versions can be looked up in vulnerability databases to find known exploits. I did not do that, as it is outside the task.

### 4.4 Task 3: nslookup

```
nslookup networkwalks.com
```

Result: non-authoritative answer **192.232.216.135** via the local resolver. Output: [`outputs/03-nslookup.txt`](W2-PM1-Footprinting-Multiple-Kali-Tools/outputs/03-nslookup.txt)

![nslookup](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/06-nslookup.png)

### 4.5 Task 4: curl -I

```
curl -I https://networkwalks.com
```

Output: [`outputs/04-curl-headers.txt`](W2-PM1-Footprinting-Multiple-Kali-Tools/outputs/04-curl-headers.txt)

![curl](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/07-curl-headers.png)

- `HTTP/2 200`, `server: Apache`.
- A `link` header advertises the WordPress REST API (`/wp-json/`), a clear WordPress fingerprint.
- `set-cookie: __wpdm_client` is marked `secure; HttpOnly`, which is good.
- `x-endurance-cache-level` and `x-nginx-cache: WordPress` reveal a caching layer in front of the site. I associate the Endurance header with HostGator-style shared hosting, but did not confirm it.
- **Not present in the captured response:** `Strict-Transport-Security`, `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`. This is an observation from a single HEAD request, not a full header audit.

### 4.6 Task 5: wafw00f

```
wafw00f networkwalks.com
```

Result: *"The site https://networkwalks.com is behind ModSecurity (SpiderLabs) WAF"* after 2 requests. Output: [`outputs/05-wafw00f.txt`](W2-PM1-Footprinting-Multiple-Kali-Tools/outputs/05-wafw00f.txt)

![wafw00f](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/08-wafw00f.png)

This fits the 406 Not Acceptable response I got when WhatWeb sent a short user agent (section 4.8), which is consistent with basic request filtering. I did not prove that link.

### 4.7 Task 6: dnsrecon

```
dnsrecon -d networkwalks.com
```

Output: [`outputs/06-dnsrecon.txt`](W2-PM1-Footprinting-Multiple-Kali-Tools/outputs/06-dnsrecon.txt)

![dnsrecon](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/09-dnsrecon.png)

| Record | Value |
|---|---|
| SOA | ns6135.hostgator.com |
| NS | ns6135.hostgator.com, ns6136.hostgator.com |
| MX | mail.networkwalks.com (same IP as the web server) |
| A | networkwalks.com -> 192.232.216.135 |
| TXT (SPF) | `v=spf1 +a +mx +ip4:50.87.144.87 +include:websitewelcome.com ~all` |
| TXT | Google site verification token |
| SRV | 8 `_autodiscover._tcp` records pointing to cPanel email discovery |

Observations:
- Both name servers disclose their **BIND version (9.16.23-RH)**.
- Web and mail share one IP, which suggests shared hosting.
- SPF ends in `~all` (soft fail), which is more lenient than `-all`.
- The DNSSEC query returned no answer, consistent with whois showing DNSSEC unsigned.
- I did **not** attempt a zone transfer or brute-force enumeration; only the standard enumeration ran.

### 4.8 WhatWeb troubleshooting (deviation from the task)

The plain `whatweb networkwalks.com` command failed repeatedly with `execution expired` and `no address for networkwalks.com`. I tested and ruled out general DNS failure, network/HTTPS access, IPv6, the default user agent being blocked, and an outdated package. A complete scan only succeeded with a browser-style user agent and the site's IP pinned in `/etc/hosts`, and it took **1,027.82 seconds (about 17 minutes)**, almost all of it CPU time.

![whatweb error](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/02-whatweb-error.png)
![hosts entry](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/03-hosts-entry-pinned.png)
![runtime](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/05-whatweb-runtime-17min.png)

**Still unconfirmed:** why the scan was so slow, whether the hosts entry was truly needed (one scan succeeded before I added it), and why failures were intermittent. I have recorded that honestly rather than guess a cause.

The temporary `/etc/hosts` line was removed afterwards (screenshot below). IPv6 settings I briefly changed were also restored.

![cleanup](W2-PM1-Footprinting-Multiple-Kali-Tools/screenshots/10-hosts-cleanup.png)

Full write-up: [`docs/WhatWeb-Troubleshooting-Report.md`](W2-PM1-Footprinting-Multiple-Kali-Tools/docs/WhatWeb-Troubleshooting-Report.md)

---

## 5. W2-PM4: Footprinting with theHarvester

Target: **microsoft.com**, a large public domain chosen by the task. theHarvester queries third-party public sources, so the target is not contacted directly.

![theHarvester menu](W2-PM4-Footprinting-theHarvester/screenshots/01-open-theharvester.png)
![usage guide](W2-PM4-Footprinting-theHarvester/screenshots/02-usage-guide.png)

### 5.1 Task 1: Baidu, limit 1000

```
theHarvester -d microsoft.com -l 1000 -b baidu
```

Output: [`outputs/theharvester-baidu.txt`](W2-PM4-Footprinting-theHarvester/outputs/theharvester-baidu.txt)

![baidu](W2-PM4-Footprinting-theHarvester/screenshots/03-baidu-no-results.png)

**Result: nothing.** No IPs, emails, people or hosts were found. The task notes that results vary because sources change their behaviour. I did not investigate why Baidu returned nothing, so the cause (blocking, changed page format, or no data) is unknown.

### 5.2 Task 2: all sources, limit 50

```
theHarvester -d microsoft.com -l 50 -b all
```

Output: [`outputs/theharvester-all-sources.txt`](W2-PM4-Footprinting-theHarvester/outputs/theharvester-all-sources.txt) (about 10,200 lines)

![all sources](W2-PM4-Footprinting-theHarvester/screenshots/04-all-sources-run.png)
![details](W2-PM4-Footprinting-theHarvester/screenshots/05-all-sources-details.png)

| Category | Count |
|---|---|
| Hosts / subdomains | **9,963** |
| IP addresses | 41 (36 IPv4, 5 IPv6) |
| ASNs | 6 (AS13335, AS133618, AS14779, AS206834, AS40034, AS8075) |
| Email addresses | 3 (`dotnet-docker-bot@`, `opencode@`, `secure@` at microsoft.com) |
| "Interesting" URLs | 1 (a Microsoft login OAuth authorize URL) |
| LinkedIn users / people | 0 |

Observations:
- **The `-l 50` limit did not cap the total.** Almost 10,000 hosts came back, so the limit appears to apply only to some source types. I did not test which.
- **Many sources were skipped.** The log shows 34 missing-API-key messages (for example BeVigil, BuiltWith, Censys, DNSDumpster, FullHunt, SecurityScorecard), and 25 sources show a "Searching" line. Adding API keys would likely increase coverage.
- **Hostnames reveal structure.** Names in the sample include environment markers such as `uat`, `prod` and `daily`. A defender should know that non-production hostnames in public data are visible to anyone.
- **The emails are role or bot-style addresses**, not employee names, so the phishing value is limited in this particular run.

**Why it matters:** each email is a potential phishing target, and each subdomain is a door a defender may have forgotten. Defenders can run the same tool on their own domain to see what an attacker would see.

---

## 6. W2-PM5: Network scanning with Zenmap

Target: **my own Kali lab subnet**, 10.0.0.0/24.

### 6.1 Deviation from the task guide

The guide uses Windows, `ipconfig`, and an instructor subnet with 4 live hosts. I ran Zenmap on **Kali Linux** using `ip a`, on an isolated NAT network, so my result has fewer hosts. The method is identical.

### 6.2 Tasks

| Task | Result |
|---|---|
| 1. Install Zenmap | Zenmap (Nmap 7.991) running on Kali |
| 2. Local IP and subnet | **10.0.0.2/24** (`ip a`, interface eth0, broadcast 10.0.0.255) |
| 3. Live hosts | `_gateway` and `kali.local` |
| 4. Number of live hosts | **2** (including my own VM) |
| 5. IP addresses | 10.0.0.1 and 10.0.0.2 |
| 6. MAC addresses | 52:54:00:12:35:00 (gateway, reported by Nmap as "QEMU virtual NIC") and 08:00:27:5a:87:bc (my Kali VM, read with `ip link`) |
| 7. Topology saved as PDF | [`outputs/topology.pdf`](W2-PM5-Network-Scanning-Zenmap/outputs/topology.pdf) |

Nmap does not print the scanning host's own MAC address, so I read the Kali MAC from the `eth0` interface with `ip link` (screenshot below).

```
nmap -sn 10.0.0.2/24      # Zenmap profile: Ping scan
```

Result: **256 addresses scanned, 2 hosts up, about 3.2 seconds.** Raw XML: [`outputs/nmap-ping-scan.xml`](W2-PM5-Network-Scanning-Zenmap/outputs/nmap-ping-scan.xml). The XML records a second run of the same scan at 10:26; the Zenmap screenshot was taken during the earlier run at 10:05, which is why the timings differ slightly (3.26 s vs 3.22 s).

![network config](W2-PM5-Network-Scanning-Zenmap/screenshots/01-network-config-annotated.png)
![ping scan](W2-PM5-Network-Scanning-Zenmap/screenshots/02-ping-scan.png)
![topology](W2-PM5-Network-Scanning-Zenmap/screenshots/03-topology-and-legend.png)
![ip link](W2-PM5-Network-Scanning-Zenmap/screenshots/04-mac-address-ip-link.png)

A ping scan (`-sn`) only discovers live hosts; it does not scan ports, so no services or operating systems were identified in this module.

---

## 7. Connecting the modules

1. **Recon (PM1, PM4)** built a profile from public information: who hosts the site, what software runs, how DNS and mail are set up, whether a WAF is present, and (for a large organisation) how big the public footprint is.
2. **Scanning (PM5)** is the step that turns that picture into live-host discovery. In a real engagement, the recon findings (IP ranges, hosting, WAF) would decide what to scan and how carefully.
3. **Everything above is public or low-impact**, which is why recon is so hard to detect, and why it is worth running against your own assets first.

### Defensive takeaways (from what I observed)

- Keep WordPress and plugins patched, and consider hiding version strings in generator tags.
- Add standard security headers (HSTS, CSP, X-Content-Type-Options, X-Frame-Options) where missing.
- Consider tightening SPF from `~all` to `-all` once all legitimate senders are covered. I did not check DMARC.
- Ask the hosting provider whether DNS server version banners can be hidden.
- Review which subdomains, especially non-production ones, are publicly listed.
- Keep WHOIS privacy and registrar locks enabled (already in place here).

---

## 8. Limitations

- One run of each tool; results can change as sources and sites change.
- WhatWeb needed a workaround and its slow runtime is unexplained.
- Baidu returned nothing and its cause was not investigated.
- theHarvester ran without API keys for many sources.
- The Zenmap scan covers my small lab network only and does not include port or OS detection.
- Vulnerability lookups, exploitation and zone transfers were out of scope and not performed.

## 9. Skills demonstrated

Passive and active reconnaissance, DNS and WHOIS analysis, web technology fingerprinting, HTTP header analysis, WAF detection, OSINT collection, host discovery with Nmap/Zenmap, structured troubleshooting, and honest technical reporting.


⚖️ *For educational purposes only. Only public information and a network used with permission.*

**Snehal Ugale** · Networkwalks Cybersecurity Internship · Batch B084

Thanks to **Networkwalks** for the internship.

</div>
