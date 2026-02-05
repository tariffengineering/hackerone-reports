Properties
ID: [[#622122]]

Severity: [[Medium]] (6.8)

Bounty: $9,700

Weakness: [[Uncontrolled Resource Consumption]] (Web Cache Poisoning)

Reporter: [[albinowax]]

Company: [[PayPal]]

Asset: [[paypal.com]] / [[www.paypalobjects.com]]

Disclosure Date: October 23, 2019

Overview
This vulnerability is known as "Web Cache Poisoning."

Websites use "Cache Servers" to temporarily store frequently used files like images and JavaScript to speed up loading times. Normally, when a user requests a file, the cache server provides the stored version.

In this case, the attacker sent a request containing an "invalid Transfer-Encoding header" that the server could not process. The origin server responded with an error message: 501 Not Implemented. Unfortunately, the cache server mistakenly identified this error message as the "correct content" for the requested file and stored it.

As a result, when legitimate users tried to load that file (such as essential JavaScript required for the site to function), they received the error message instead of the actual code. This resulted in a Denial of Service (DoS) state, breaking core website functionality.

"Analogy" Explanation
Imagine an "Automatic Bread Vending Machine" (the Cache Server):

Normal State: A customer presses the "Anpan" button. The machine gets an Anpan from the factory, delivers it, and keeps a few more in its internal stock for the next customers.

The Attack: A malicious person presses the button while shoving a "note written in an incomprehensible alien language" (the invalid header) into the coin slot.

Factory Reaction: The factory sees the gibberish note and sends back a "Rejection Slip" (the Error) saying, "I don't understand this request!"

The Machine's Mistake: The vending machine mistakenly thinks the "Rejection Slip" is the product and stocks its shelves with it.

The Damage: Every normal customer who pushes the "Anpan" button from then on receives a "Rejection Slip" instead of bread. No one can eat.

Steps to Reproduce (Summary)
The reporter, [[albinowax]], used Burp Suite to manipulate request headers to trigger the poisoning.

1. Burp Suite Observation (Predicted Traffic) By inserting a non-standard value into the Transfer-Encoding header, the origin server is forced to return a 501 error.

HTTP
GET /en_US/i/scr/pixel.gif HTTP/1.1
Host: www.paypalobjects.com
Transfer-Encoding: invalid_value
User-Agent: Mozilla/5.0 ...
The cache server sees this response and associates it with the URL, caching the 501 error for all subsequent users.

2. Python PoC (Predicted Script) A script like the following could be used to automate the poisoning across different files.

Python
import requests

target_url = "https://www.paypalobjects.com/path/to/script.js"
headers = {
    "Transfer-Encoding": "completely_invalid",
    "User-Agent": "CachePoisoner/1.0"
}

def poison_cache():
    # Trigger 501 error to overwrite the cache
    response = requests.get(target_url, headers=headers)
    if response.status_code == 501:
        print(f"Successfully poisoned: {target_url}")
    else:
        print("Poisoning failed.")

if __name__ == "__main__":
    poison_cache()
Key Discussion Summary
The negotiation focused on the technical impact and the severity of a "Responsible DoS."

Severity Adjustments: The report was initially submitted as "High (7.5)," dropped to "Medium (5.3)" by [[PayPal]], and eventually settled at "Medium (6.8)."

Negotiation Points:

The reporter argued that this was a sophisticated "Web Cache Poisoning" technique that could disrupt core services by disabling JavaScript.

[[PayPal]] acknowledged the impact on paypalobjects.com, which serves static assets for the main site.

Bounty Rationale: The $9,700 bounty is exceptionally high for a [[Medium]] severity report. This reflects the critical nature of the [[paypal.com]] domain and the fact that disabling JavaScript assets can effectively take down the entire payment flow.

Mitigation for Site Owners
Website owners with similar architectures should implement the following defenses:

Do Not Cache Error Responses: Ensure that the cache server (CDN) is configured NOT to cache responses like 501 Not Implemented or 400 Bad Request.

Ignore/Reject Invalid Headers: Configure the edge/cache layer to strip or reject malformed Transfer-Encoding or other hop-by-hop headers before they reach the origin server.

Strict Cache Key Design: Be mindful of which headers are included in the "Cache Key." If a header changes the response, it must be part of the key; otherwise, the response should not be cached.

CDN-Origin Consistency: Regularly audit the interaction between your CDN and your origin server to ensure they interpret HTTP headers in the same way, preventing "Request Smuggling" or "Cache Poisoning" scenarios.
