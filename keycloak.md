### **Options for Token Storage in APISIX**
APISIX supports token storage for:
1. **Caching Tokens**: To reduce the need for repeated validation with the IdP.
2. **Persistent Storage**: For long-term storage of tokens (e.g., refresh tokens or client secrets).

### **1. Configure Token Caching in APISIX**
APISIX can cache validated tokens to avoid repeatedly querying the Keycloak IdP.

#### **Step 1: Enable OpenID Connect Plugin**
Ensure the OpenID Connect plugin is installed and configured.

Example:
```json
{
  "plugins": {
    "openid-connect": {
      "client_id": "apisix-client",
      "client_secret": "<client-secret>",
      "discovery": "http://<keycloak-domain>/realms/<realm>/.well-known/openid-configuration",
      "introspection_endpoint_auth_method": "client_secret_basic",
      "scope": "openid",
      "bearer_only": true
    }
  },
  "uri": "/protected/*",
  "upstream": {
    "nodes": {
      "127.0.0.1:8080": 1
    },
    "type": "roundrobin"
  }
}
```

#### **Step 2: Enable Token Caching**
Add token caching settings to the OpenID Connect plugin configuration:
```json
{
  "plugins": {
    "openid-connect": {
      "client_id": "apisix-client",
      "client_secret": "<client-secret>",
      "discovery": "http://<keycloak-domain>/realms/<realm>/.well-known/openid-configuration",
      "introspection_endpoint_auth_method": "client_secret_basic",
      "scope": "openid",
      "bearer_only": true,
      "cache_tokens": true,
      "cache_ttl": 300
    }
  }
}
```

- **`cache_tokens`**: Enables token caching.
- **`cache_ttl`**: Defines the time-to-live (TTL) for cached tokens in seconds.

This setup ensures validated tokens are cached for 5 minutes (`300 seconds`) before requiring revalidation.

---

### **2. Store Tokens Persistently**
If you need to store tokens persistently (e.g., refresh tokens or client secrets), you can use APISIX’s integration with etcd or other plugins:

#### **Step 1: Use etcd (Default Storage)**
APISIX uses etcd as its default storage backend. Tokens and metadata can be stored as part of the route or plugin configurations.

1. **Save Tokens**:
   You can write tokens to etcd directly using the APISIX admin API:
   ```bash
   curl http://<apisix-admin-url>/apisix/admin/routes/1 -X PUT -d '{
     "uri": "/protected/*",
     "plugins": {
       "openid-connect": {
         "client_id": "apisix-client",
         "client_secret": "<client-secret>",
         "discovery": "http://<keycloak-domain>/realms/<realm>/.well-known/openid-configuration",
         "cache_tokens": true,
         "cache_ttl": 300
       }
     },
     "metadata": {
       "refresh_token": "<refresh-token>"
     }
   }'
   ```

2. **Read Tokens**:
   Tokens stored in etcd can be accessed through the APISIX admin API or directly via etcd commands.

---

### **3. Advanced Token Management with Redis**
If you need a high-performance, external caching solution, integrate **Redis** with APISIX:

#### **Step 1: Install Redis**
Set up a Redis server as your external token storage.

#### **Step 2: Configure APISIX to Use Redis**
Enable the Redis plugin for caching tokens. Update your APISIX configuration:
```yaml
# conf/config.yaml
plugin_attr:
  openid-connect:
    token_cache:
      host: 127.0.0.1
      port: 6379
      timeout: 2000
      ttl: 300
```

#### **Step 3: Configure OpenID Connect Plugin**
Update the OpenID Connect plugin configuration to use Redis for token caching:
```json
{
  "plugins": {
    "openid-connect": {
      "client_id": "apisix-client",
      "client_secret": "<client-secret>",
      "discovery": "http://<keycloak-domain>/realms/<realm>/.well-known/openid-configuration",
      "cache_tokens": true
    }
  },
  "uri": "/protected/*",
  "upstream": {
    "nodes": {
      "127.0.0.1:8080": 1
    },
    "type": "roundrobin"
  }
}
```

---

### **4. Use JWT for Token Validation**
If Keycloak issues **JWT tokens**, you can skip storage and validate tokens directly:

1. **Configure Public Key Validation**:
   Use Keycloak’s `jwks_uri` (from the discovery document) to validate JWT tokens:
   ```json
   {
     "plugins": {
       "jwt-auth": {
         "key": "<public-key>",
         "algorithm": "RS256"
       }
     }
   }
   ```

2. **Skip Token Storage**:
   JWTs are self-contained and do not require caching or storage.

---

### **Best Practices**
- **Minimize Token Storage**: Use token caching or JWT validation wherever possible to avoid persistent storage.
- **Secure Sensitive Data**:
  - Use HTTPS to secure token exchanges.
  - Encrypt stored tokens in etcd or Redis.
- **Set Appropriate TTL**: Balance caching duration with token expiration to optimize performance and security.

