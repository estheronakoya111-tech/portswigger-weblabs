# Web Cache Deception (WCD) Labs Notes

This repository contains writeups, concepts, and key takeaways from solving PortSwigger Web Security Academy's Web Cache Deception labs.

---

## 📌 Core Concepts

### What is Web Cache Deception?
Web Cache Deception (WCD) occurs when an intermediate cache proxy and a backend origin server interpret a URL path differently:
* **Origin Server (The Kitchen):** Treats the request as dynamic, uses path delimiters (e.g., `;`, `?`, `%23`) to truncate the path, and returns sensitive user-specific data (e.g., `/my-account`).
* **Cache Server (The Counter):** Inspects the end of the URL path, sees a static file extension (e.g., `.js`, `.css`), assumes the resource is public/static, and saves the sensitive dynamic response in memory.

### Key Terminology
* **Origin Server:** The backend application server executing code, processing database logic, and generating dynamic HTML.
* **Cache Proxy / CDN:** An intermediate server storing static assets close to users to reduce latency and server load.
* **Static Assets:** Unchanging files delivered identically to all users (`.css`, `.js`, `.png`).
* **Dynamic Assets:** User-specific resources generated on-the-fly (`/my-account`, `/inbox`).

---

## 🧪 Lab Breakdown

### Lab 1: Exploiting path mapping for web cache deception
* **Objective:** Capture the admin/victim API key by exploiting path mapping behavior in the caching layer.
* **Core Vulnerability:** Discrepancy between how the origin server routes dynamic paths and how the cache proxy maps static file extensions.
* **Key Steps:**
  1. Identifies that appending static file extensions (e.g., `/my-account/wcd.js`) causes the cache proxy to save the response.
  2. The origin server still maps `/my-account/wcd.js` back to the endpoint for `/my-account`.
  3. Forced a victim to navigate to the crafted URL, causing their authenticated account details to be cached.
  4. Retrieved the cached response to extract the target API key.

---

### Lab 2: Exploiting path delimiters for web cache deception
* **Objective:** Extract Carlos's API key by identifying a valid path delimiter accepted by the origin server.
* **Core Vulnerability:** Path delimiter mismatch where the origin server strips matrix parameters/delimiters while the cache proxy evaluates the full URL string.
* **Key Steps:**
  1. Tested various delimiter candidates (`/my-account;mcp.js`, `/my-account?mcp.js`, `/my-account%3Bmcp.js`).
  2. Identified that `;` acts as a valid path delimiter on the target:
     * **Origin Server:** Strips `;mcp.js` and renders `/my-account` with `200 OK`.
     * **Cache Proxy:** Reads `.js` at the end and stores the response (`X-Cache: hit`).
  3. Delivered the exploit payload to the victim:
     ```html
     <script>
         document.location = "https://<TARGET-LAB-ID>.web-security-academy.net/my-account;mcp.js";
     </script>
     ```
  4. Navigated directly to `/my-account;mcp.js` to view Carlos's cached account page and extract his API key.

---

## 🛠️ Common Pitfalls & Troubleshooting

| Issue / Response | Root Cause | Solution |
| :--- | :--- | :--- |
| `403 Forbidden` (`GET requests cannot contain a body`) | Trailing lines, whitespace, or invalid `Content-Length` headers in Burp Repeater. | Remove `Content-Length` and ensure the request body is completely empty (`0 bytes`). |
| `302 Found` (Redirect to `/login`) | Session cookie expired or missing from the `Cookie:` header. | Log back in via browser, grab a fresh `session=` cookie from HTTP history, and update Repeater. |
| Continuous `X-Cache: miss` | Origin server accepts delimiter, but cache rule ignores the tested extension or requires specific path prefixes. | Rotate file extensions (`.css`, `.png`, `.svg`) or test static directory paths (`/static/`). |

---

## 🛡️ Remediation Strategies

1. **Use Explicit Cache-Control Headers:** Ensure dynamic endpoints return `Cache-Control: no-store, private` so caches never store them regardless of URL appearance.
2. **Standardize Path Parsing:** Align URL parsing rules across proxies and backend servers so path delimiters (like `;`) are handled identically.
3. **Strict Static Routing:** Configure cache proxies to validate that requested static resources actually exist on the origin before caching responses.