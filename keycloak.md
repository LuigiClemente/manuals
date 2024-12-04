### **1. Integrate Zammad with Keycloak via SAML for User Authentication**

Zammad supports SAML-based SSO, allowing users to authenticate through Keycloak. To set this up:

- **Configure Keycloak**:
  - Add Zammad as a client in Keycloak.
  - Set the **Client Protocol** to `saml`.
  - Configure mappers to pass necessary user attributes (e.g., email, first name, last name) to Zammad.

- **Configure Zammad**:
  - Navigate to **Settings > Security > Third-Party Applications > Authentication via SAML**.
  - Enter the Identity Provider (IdP) details from Keycloak, including the SSO target URL and IdP certificate.

For detailed instructions, refer to Zammad's documentation on SAML integration. 

---

### **2. Authenticate Zammad API Requests Using Keycloak**



#### **Approach B: Implement OAuth2 Authentication via Keycloak**

To use Keycloak-issued tokens for API authentication:

1. **Configure OAuth2 in Zammad**:
   - Zammad doesn't natively support OAuth2 for API authentication.
   - However, you can implement a custom middleware or proxy that validates Keycloak tokens and translates them into Zammad API requests.

2. **Set Up Keycloak for OAuth2**:
   - In Keycloak, configure a client for your application with OAuth2 settings.
   - Ensure the client can issue tokens with the necessary scopes for API access.

3. **Develop Middleware for Token Translation**:
   - Create a service that intercepts API requests, validates the Keycloak token, and uses a corresponding Zammad API token to perform actions on behalf of the user.

This approach requires additional development but allows seamless integration using Keycloak's OAuth2 capabilities.

---
To implement **OAuth2 Authentication via Keycloak** for a frontend that uses Zammad APIs with SAML authentication, while your frontend integrates with **BoxyHQ**, here’s how you can proceed:

---

### **1. Key Components in the Integration**
- **Keycloak**: Issues OAuth2 tokens for authenticating users.
- **BoxyHQ**: Handles SAML-to-modern authentication for your frontend.
- **Zammad**: Uses SAML for user authentication but requires API tokens for backend interactions.
- **Custom Middleware**: Validates Keycloak tokens and translates them into Zammad API tokens.

---

### **2. Implementation Steps**

#### **Step 1: Configure Keycloak for OAuth2**
1. **Create a Client in Keycloak**:
   - Log in to Keycloak admin console.
   - Go to **Clients > Create Client**.
   - Set the client protocol to `openid-connect` (OAuth2).
   - Configure:
     - **Client ID**: A unique name for your application (e.g., `zammad-integration`).
     - **Access Type**: `confidential` (if backend authentication is required).
     - **Redirect URI**: URL where users are redirected after successful login (e.g., BoxyHQ’s SSO endpoint or your middleware).

2. **Enable Scopes**:
   - Ensure the client has the scopes required for the API.
   - Example: `profile`, `email`, `roles`.

3. **Generate Client Secrets**:
   - After creating the client, go to the **Credentials** tab.
   - Copy the client secret for use in your middleware.

---

#### **Step 2: BoxyHQ and Frontend Integration**
1. **Integrate BoxyHQ with Keycloak**:
   - Use BoxyHQ's SAML configuration to connect your frontend to Keycloak.
   - Ensure users are authenticated with Keycloak via BoxyHQ before accessing the frontend.

2. **Obtain OAuth2 Token**:
   - After successful authentication via BoxyHQ, your frontend should:
     - Request an OAuth2 token from Keycloak.
     - Use this token to authenticate API requests to your middleware.

---

#### **Step 3: Develop Middleware for Token Translation**
1. **Create Middleware Service**:
   - Implement a middleware service that:
     1. **Validates Keycloak Tokens**:
        - Use Keycloak’s token introspection endpoint to validate the incoming token.
        - Example:
          ```bash
          POST http://<keycloak-domain>/realms/<realm>/protocol/openid-connect/token/introspect
          ```
        - Include the client secret in the request.

     2. **Exchange for Zammad API Token**:
        - Use a pre-configured Zammad API token to make authenticated requests on behalf of the user.
        - Zammad API tokens can be generated for specific users.

2. **Example Middleware Logic**:
   - A simple Node.js example:
     ```javascript
     const express = require('express');
     const axios = require('axios');

     const app = express();

     app.use(express.json());

     app.post('/proxy', async (req, res) => {
       const { token, apiPath } = req.body;

       try {
         // Validate Keycloak token
         const introspectResponse = await axios.post(
           'http://<keycloak-domain>/realms/<realm>/protocol/openid-connect/token/introspect',
           `token=${token}&client_id=<client-id>&client_secret=<client-secret>`,
           { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
         );

         if (!introspectResponse.data.active) {
           return res.status(401).json({ error: 'Invalid token' });
         }

         // Make Zammad API request using API token
         const zammadResponse = await axios.get(
           `http://<zammad-url>/api/v1/${apiPath}`,
           { headers: { Authorization: `Token token=<zammad-api-token>` } }
         );

         res.json(zammadResponse.data);
       } catch (err) {
         res.status(500).json({ error: 'Internal Server Error', details: err.message });
       }
     });

     app.listen(3000, () => console.log('Middleware running on port 3000'));
     ```

---

#### **Step 4: Update Frontend to Use Middleware**
1. **Send Keycloak Token to Middleware**:
   - After authenticating with Keycloak via BoxyHQ, your frontend sends the token and API path to the middleware:
     ```javascript
     fetch('http://localhost:3000/proxy', {
       method: 'POST',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify({
         token: '<keycloak-token>',
         apiPath: '<zammad-api-path>'
       })
     })
       .then(res => res.json())
       .then(data => console.log(data));
     ```

2. **Receive Data from Zammad**:
   - The middleware validates the token and forwards the request to Zammad, ensuring secure communication.

---

### **3. Key Benefits of This Approach**
- **Seamless Integration**:
  - Users authenticate once with Keycloak via BoxyHQ, and the middleware handles interactions with Zammad transparently.
- **Centralized Authentication**:
  - All authentication and user management are handled by Keycloak.
- **Secure Token Translation**:
  - Middleware ensures Zammad API tokens are not exposed to the frontend.

---
