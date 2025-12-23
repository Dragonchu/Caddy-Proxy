---

## 🚀 Quick Deploy

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.com/new/template/caddy-proxy)

---

## ⚙️ Configuration

### Required Environment Variables

**`PROXY_TARGET`** - The backend service to proxy to

### Optional Environment Variables

- **`PORT`** - Port to listen on (Railway provides this automatically)
- **`LOG_LEVEL`** - Logging level: DEBUG, INFO, WARN, ERROR (default: INFO)

---

## 🔧 Usage Examples

### 1. Proxy to Internal Railway Service (Private Network)

Most common use case - proxying to another service in your Railway project:

```bash
# Using Railway's private networking (recommended)
PROXY_TARGET=http://api.railway.internal:3000

# Or using reference variables (also recommended)
PROXY_TARGET=http://${{api.RAILWAY_PRIVATE_DOMAIN}}:3000
```

**Important Notes:**
- Private networking uses `.railway.internal` domain
- Always specify the port your backend service listens on
- Use `http://` (not `https://`) for internal connections
- Private network is isolated per project/environment

### 2. Proxy to External Service

```bash
PROXY_TARGET=https://api.external-service.com
```

### 3. Open Notebook (Jupyter)

```bash
PROXY_TARGET=http://notebook.railway.internal:8888
```

### 4. Huly Platform

```bash
PROXY_TARGET=http://huly.railway.internal:8080
```

### 5. Multiple Backend Services (Custom Caddyfile)

For complex routing, create a custom Caddyfile:

```caddyfile
:{$PORT:8080} {
    # Route API requests
    handle /api/* {
        reverse_proxy http://api.railway.internal:3000
    }
    
    # Route WebSocket connections
    handle /ws/* {
        reverse_proxy http://websocket.railway.internal:8080
    }
    
    # Default to frontend
    handle {
        reverse_proxy http://frontend.railway.internal:5173
    }
}
```

---

## 🌐 Networking on Railway

### Private Networking (Service-to-Service)

**When to use:** Communication between services in the same Railway project

- Use `servicename.railway.internal:PORT` format
- Only accessible within your project/environment
- IPv4 and IPv6 supported (new environments)
- Legacy environments (pre-Oct 2025) are IPv6 only

**Example Configuration:**
```bash
# Direct reference
PROXY_TARGET=http://backend.railway.internal:8080

# Using Railway variables
PROXY_TARGET=http://${{backend.RAILWAY_PRIVATE_DOMAIN}}:${{backend.PORT}}
```

### Public Networking (Internet Access)

**When to use:** Making your Caddy proxy accessible from the internet

1. **Generate Railway Domain:**
   - Go to service settings → Networking → Public Networking
   - Click "Generate Domain"
   - Railway will provide a domain like `yourapp.up.railway.app`

2. **Add Custom Domain (Optional):**
   - Click "+ Custom Domain" in service settings
   - Add your domain (e.g., `proxy.yourdomain.com`)
   - Create a CNAME record pointing to Railway's provided value
   - Railway automatically issues SSL certificate

**Important:** 
- Caddy listens on the `PORT` variable (Railway provides this)
- Set `auto_https off` in Caddyfile (Railway handles SSL)
- Use `http://` for internal connections, Railway handles `https://` externally

---

## 🎯 Common Patterns

### Pattern 1: API Gateway

Expose internal microservices through a single public endpoint:

```
Internet → Caddy (public) → Internal Services (private)
  ↓
https://api.yourdomain.com
  ├─ /users    → http://users-service.railway.internal:3000
  ├─ /orders   → http://orders-service.railway.internal:3001
  └─ /products → http://products-service.railway.internal:3002
```

### Pattern 2: Frontend + Backend

Serve frontend and proxy API requests:

```
Internet → Caddy (public)
  ↓
  ├─ /          → http://frontend.railway.internal:3000
  └─ /api/*     → http://backend.railway.internal:8080
```

### Pattern 3: WebSocket Proxy

Proxy WebSocket connections for real-time apps:

```
PROXY_TARGET=http://realtime-app.railway.internal:8080
```

Caddy automatically handles WebSocket upgrades.

---

## ✨ Features

- ✅ **Private Network Support** - Seamless Railway service-to-service communication
- ✅ **Auto SSL** - Railway provides HTTPS automatically
- ✅ **WebSocket Support** - Real-time applications work out of the box
- ✅ **Compression** - Automatic gzip compression
- ✅ **Security Headers** - XSS, clickjacking, and content-type protection
- ✅ **CORS Ready** - Pre-configured CORS headers
- ✅ **Health Checks** - Built-in `/health` endpoint
- ✅ **Request Logging** - JSON formatted logs
- ✅ **IPv4/IPv6** - Dual-stack support

---

## 🔍 Monitoring & Debugging

### Health Check

Access the health check endpoint:
```bash
curl https://your-caddy-service.up.railway.app/health
```

### View Logs

```bash
railway logs
```

### Common Issues

**1. "Connection refused" or "dial tcp: lookup failed"**
- Verify backend service is running
- Check `PROXY_TARGET` format: `http://service.railway.internal:PORT`
- Ensure backend service is in the same project/environment
- Confirm backend is listening on the correct port

**2. "502 Bad Gateway"**
- Backend service may not be listening on `::` (all interfaces)
- Check backend service logs
- Verify backend health endpoint responds

**3. WebSocket not working**
- Ensure `Connection` and `Upgrade` headers are preserved
- Backend must support WebSocket protocol
- Check if backend is properly handling WebSocket handshake

**4. Custom domain not working**
- Verify CNAME record is correctly configured
- Wait up to 72 hours for DNS propagation
- Check Railway dashboard shows green checkmark
- If using Cloudflare, set SSL/TLS mode to "Full"

---

## 🔐 Security Best Practices

1. **Private Services** - Keep backend services on private network only
2. **CORS Configuration** - Restrict allowed origins in production
3. **Rate Limiting** - Add rate limiting for public endpoints (see advanced config)
4. **Security Headers** - Template includes basic security headers
5. **Environment Variables** - Never commit sensitive values to git

---

## 📚 Advanced Configuration

### Rate Limiting

Add rate limiting to your Caddyfile:

```caddyfile
:{$PORT:8080} {
    handle {
        rate_limit {
            zone static {
                key static
                events 1000
                window 1m
            }
            zone dynamic {
                key {remote_host}
                events 100
                window 1m
            }
        }
        reverse_proxy {$PROXY_TARGET}
    }
}
```

### Custom Error Pages

```caddyfile
handle_errors {
    @404 {
        expression {http.error.status_code} == 404
    }
    rewrite @404 /404.html
    
    @5xx {
        expression {http.error.status_code} >= 500
    }
    rewrite @5xx /500.html
    
    file_server
}
```

### Path-Based Routing

```caddyfile
:{$PORT:8080} {
    # Strip /api prefix before proxying
    handle /api/* {
        uri strip_prefix /api
        reverse_proxy http://backend.railway.internal:8080
    }
    
    # Keep path as-is
    handle /webhooks/* {
        reverse_proxy http://webhook-handler.railway.internal:3000
    }
}
```

---

## 📖 References

- [Railway Public Networking](https://docs.railway.com/guides/public-networking)
- [Railway Private Networking](https://docs.railway.com/guides/private-networking)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [Railway Variables](https://docs.railway.com/guides/variables)

---

## 🤝 Contributing

Issues and pull requests welcome! Feel free to customize this template for your needs.

---

## 📝 License

MIT License - Feel free to use this template for any project.
