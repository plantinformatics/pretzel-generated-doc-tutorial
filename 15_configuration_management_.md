# Chapter 15: Configuration Management

In [Chapter 14: Web Workers](14_web_workers_.md), we saw how Pretzel can offload intensive tasks to background threads to keep the UI responsive. Like any sophisticated application, Pretzel's frontend and backend components need instructions on how to behave in different environments – how to connect to databases, where to find APIs, which features to enable. How does Pretzel manage these crucial settings?

## Motivation: The Application's Control Panel and Instructions

Imagine deploying Pretzel. The frontend needs to know the URL of the backend API server. The backend needs database credentials, the port number to listen on, email server details (if used), and perhaps settings to control authentication modes. Hardcoding these values directly into the source code is inflexible and insecure.

We need a way to:

*   **Configure for Different Environments:** Use different database credentials for development vs. production. Point the frontend to `localhost` during development but to `api.pretzel.example.com` in production.
*   **Manage Secrets:** Keep sensitive information like database passwords or API keys out of version control.
*   **Control Behavior:** Enable or disable features (like authentication or email verification) based on the deployment context.

Configuration Management acts as the central control panel and set of instruction manuals for Pretzel. It defines how the different parts of the system should connect and operate, allowing administrators to tailor the application to specific deployment needs without modifying the core codebase.

**Our Central Use Case:** When deploying Pretzel, how do we configure the frontend application (built via Ember CLI) to communicate with the backend API running at `https://api.pretzel.example.com`, and how do we configure the backend server (Loopback application) to connect to a production MongoDB database at `mongodb://prod-db.example.com` using specific credentials, listen on port 8080, and enforce authentication (`AUTH=ALL`, `EMAIL_VERIFY=USER`)?

## Key Concepts: Where Settings Live

Configuration in Pretzel is distributed across several key areas:

1.  **Frontend Build-Time Configuration (`config/environment.js`):**
    *   **What:** A JavaScript file within the Ember frontend project (`frontend/config/environment.js`).
    *   **When:** Settings defined here are baked into the frontend application *at build time*.
    *   **How:** Ember CLI executes this file during the build process (`ember build`). It often reads environment variables available *during the build* (e.g., `process.env.API_URL`, `process.env.ROOT_URL`) to populate settings within the `ENV` object.
    *   **What it controls:** API host (`apiHost`), API namespace (`apiNamespace`), application base URL (`rootURL`), addon configurations (like `ember-simple-auth`), feature flags, etc.
    *   **Analogy:** The construction blueprints for the frontend office, specifying fixed details like the address of the main headquarters (API) before the office is even built.

2.  **Backend Environment Variables (`.env`):**
    *   **What:** A file named `.env` typically placed in the root of the backend project (`lb4app/lb3app/.env`). It contains key-value pairs.
    *   **When:** These variables are loaded into the backend server's *runtime environment* when it starts.
    *   **How:** The `dotenv` library (configured in `server/server.js`) reads this file and loads the variables into `process.env`. This file should *not* be committed to version control, as it often contains secrets.
    *   **What it controls:** Database credentials (`DB_HOST`, `DB_USER`, `DB_PASS`, `DB_NAME`), server port (`API_PORT_EXT`), authentication modes (`AUTH`, `EMAIL_VERIFY`), email settings (`EMAIL_HOST`, `EMAIL_USER`, etc.), third-party API keys.
    *   **Analogy:** Secure operating instructions delivered to the backend headquarters just before it opens for business, kept separate from the public building plans.

3.  **Backend Local Configuration Files (`*.local.js`):**
    *   **What:** JavaScript files in the backend's `server/` directory that override default configurations (e.g., `datasources.local.js`, `config.local.js`, `model-config.local.js`). See [Chapter 6: Loopback Application & Server](06_loopback_application___server_.md).
    *   **When:** Read by the Loopback application during its boot sequence.
    *   **How:** These files often contain logic to read values from `process.env` (loaded from `.env` or the system environment) and use them to configure Loopback settings (like datasource connection details or API port). This provides flexibility, allowing environment variables to dictate runtime configuration.
    *   **What it controls:** Database connection specifics (host, port, credentials read from `process.env`), API root path, server host/port (read from `process.env`), model-datasource mappings, enabling/disabling features based on environment variables (e.g., `authentication.js` boot script checking `process.env.AUTH`).
    *   **Analogy:** The headquarters' internal setup procedures that consult the secure operating instructions (`.env`/`process.env`) to configure specific systems like the vault connection (database) or the main phone line (port).

4.  **Backend Runtime Configuration (API/DB):**
    *   **What:** Settings potentially fetched by the frontend *after* it has loaded, usually from a dedicated backend API endpoint.
    *   **How:** The backend might expose a `/api/configurations/runtimeConfig` endpoint (backed by the `Configuration` model - `common/models/configuration.js`) that returns non-sensitive runtime settings needed by the frontend (e.g., a specific license key like `handsOnTableLicenseKey` read from the backend's `process.env`).
    *   **When:** Fetched dynamically by the frontend during its initialization or later.
    *   **What it controls:** Settings needed by the frontend that are determined by the backend's runtime environment (and aren't secrets).
    *   **Analogy:** A public information kiosk in the headquarters lobby providing specific operational details to visitors after they've arrived.

## Solving the Use Case: Configuring for Production

Let's configure Pretzel for the production scenario described:

1.  **Frontend Configuration (`config/environment.js` & Build):**
    *   During the production build (`ember build --environment=production`), set the necessary build-time environment variables:
        ```bash
        export API_URL="https://api.pretzel.example.com"
        export ROOT_URL="/pretzel-app/" # Example if served from sub-directory
        ember build --environment=production
        ```
    *   The `frontend/config/environment.js` file reads these:
        ```javascript
        // frontend/config/environment.js (Simplified)
        module.exports = function (environment) {
          // API_URL set during build overrides default
          const apiHost = process.env.API_URL || 'http://localhost:5000';
          // ROOT_URL set during build overrides default
          const rootURL = process.env.ROOT_URL || '/';
          const ENV = {
            // ...
            apiHost: apiHost,
            rootURL: rootURL,
            locationType: 'history',
            // ... other settings based on environment ...
          };
          // ... (rest of the config) ...
          return ENV;
        };
        ```
        *Explanation:* The build process uses the provided `API_URL` and `ROOT_URL` environment variables to set the `apiHost` and `rootURL` baked into the production frontend assets.

2.  **Backend Configuration (`.env` & Startup):**
    *   Create a `.env` file in the backend's root directory (`lb4app/lb3app/.env`):
        ```dotenv
        # .env (Backend configuration - DO NOT COMMIT)
        DB_HOST=prod-db.example.com
        DB_PORT=27017
        DB_USER=pretzel_prod_user
        DB_PASS=supersecretproductionpassword
        DB_NAME=pretzel_prod_db

        API_PORT_EXT=8080

        AUTH=ALL # Enforce authentication
        EMAIL_VERIFY=USER # Enable user email verification

        # Optional Email Settings (if EMAIL_VERIFY != NONE)
        EMAIL_ACTIVE=true
        EMAIL_HOST=smtp.example.com
        EMAIL_PORT=587
        EMAIL_USER=email_service_user
        EMAIL_PASS=email_service_password
        EMAIL_FROM=noreply@pretzel.example.com
        ```
        *Explanation:* Defines database credentials, the desired API port, and authentication behaviour using environment variables.

    *   When starting the backend (`node server/server.js`), `dotenv` loads these into `process.env`.
    *   Loopback boots and reads local configuration files:
        *   `server/datasources.local.js` reads `process.env.DB_HOST`, `DB_PORT`, etc., to configure the `mongoDs` connection.
        ```javascript
        // lb4app/lb3app/server/datasources.local.js (Excerpt)
        const database = process.env.DB_NAME || 'pretzel';
        var config = {
          "mongoDs": {
            "host": process.env.DB_HOST, // Reads from .env
            "port": process.env.DB_PORT, // Reads from .env
            "database": database,
            "password": process.env.DB_PASS, // Reads from .env
            "user": process.env.DB_USER,     // Reads from .env
            "connector": "mongodb",
            // ... other options
          },
          // ... other datasources (like email) ...
        };
        module.exports = config;
        ```
        *Explanation:* Uses `process.env` values (loaded from `.env`) to set the MongoDB connection parameters.
        *   `server/config.local.js` reads `process.env.API_PORT_EXT` to set the listening port.
        ```javascript
        // lb4app/lb3app/server/config.local.js (Excerpt)
        var config = {
          "restApiRoot": "/api",
          "host": "0.0.0.0",
          // Reads API_PORT_EXT from .env, defaults to 3000 if not set
          "port": process.env.API_PORT_EXT || 3000,
          // ... remoting settings ...
        };
        module.exports = config;
        ```
        *Explanation:* Uses `process.env.API_PORT_EXT` to configure the port the Loopback server listens on.
        *   The boot script `server/boot/authentication.js` reads `process.env.AUTH` to enable/disable authentication.
        ```javascript
        // lb4app/lb3app/server/boot/authentication.js
        'use strict';
        module.exports = function enableAuthentication(server) {
          // Enable authentication unless explicitly NONE
          if (process.env.AUTH !== 'NONE') {
            server.enableAuth();
          }
        };
        ```
        *Explanation:* Checks the `AUTH` environment variable to conditionally enable Loopback's authentication system.
        *   The `server/environment.js` script reads `process.env.EMAIL_VERIFY` to validate settings and potentially configure model options in `model-config.local.js`.

Now, the deployed frontend knows where the production API is, and the backend connects to the correct database, listens on the specified port, and enforces the desired authentication mode, all configured via external settings without changing the source code.

## Internal Implementation: How Configuration is Applied

**Frontend Build:**

1.  `ember build` command is executed with environment variables set (`API_URL`, `ROOT_URL`).
2.  Ember CLI runs `frontend/config/environment.js`, passing the target environment (e.g., 'production').
3.  The script reads `process.env.API_URL`, `process.env.ROOT_URL`, etc.
4.  It populates the `ENV` object with these values.
5.  Ember CLI uses the `ENV` object to configure the application build, embedding values like `apiHost` and `rootURL` into the generated JavaScript assets (`vendor.js`, `app.js`).

**Backend Startup:**

1.  `node server/server.js` command is executed.
2.  `dotenv.config()` is called near the top, reading `lb4app/lb3app/.env` and loading its contents into `process.env`.
3.  Loopback `boot` process starts.
4.  It reads `server/datasources.local.js`. This file executes JavaScript code that accesses `process.env` (e.g., `process.env.DB_HOST`) to get database connection details.
5.  It reads `server/config.local.js`. This file accesses `process.env` (e.g., `process.env.API_PORT_EXT`) to get server settings.
6.  It runs boot scripts like `server/boot/authentication.js`, which can also access `process.env` (e.g., `process.env.AUTH`) to modify the application's behavior (like calling `app.enableAuth()`).
7.  The server starts listening, using the configuration derived from the `.local.js` files, which were populated from `process.env` (originating from `.env` or the system environment).

**Simplified Configuration Flow Diagram:**

```mermaid
graph TD
    subgraph Frontend Build Process
        direction LR
        BuildEnvVars[Build Environment (API_URL, ROOT_URL)] -->|read by| BuildConfig[config/environment.js]
        BuildConfig -->|configures| EmberBuild[Ember Build Output (app.js)]
    end

    subgraph Backend Runtime Startup
        direction TB
        DotEnvFile[.env File] -->|loaded by dotenv| ProcessEnv[Node.js process.env]
        SystemEnvVars[System Environment] -->|populates| ProcessEnv
        ProcessEnv -->|read by| DatasourcesLocal[datasources.local.js]
        ProcessEnv -->|read by| ConfigLocal[config.local.js]
        ProcessEnv -->|read by| AuthBoot[boot/authentication.js]
        DatasourcesLocal -->|configures| DBConnection[Database Connection]
        ConfigLocal -->|configures| ServerPort[Server Port]
        AuthBoot -->|enables/disables| AuthSystem[Authentication System]
    end
```

**Runtime Configuration Endpoint (Optional):**

*   Frontend needs a setting only the backend knows at runtime (e.g., `handsOnTableLicenseKey`).
*   Frontend calls `GET /api/configurations/runtimeConfig`.
*   Backend routes to `Configuration.runtimeConfig` method in `common/models/configuration.js`.
*   This method reads `process.env.handsOnTableLicenseKey` and returns it in the JSON response.
*   Frontend receives the response and uses the setting.

```javascript
// lb4app/lb3app/common/models/configuration.js (Excerpt)
module.exports = function(Configuration) {
  Configuration.runtimeConfig = function (cb) {
    // Read license key from backend environment
    let handsOnTableLicenseKey = process.env.handsOnTableLicenseKey;
    cb(null, { handsOnTableLicenseKey });
  };
  Configuration.remoteMethod('runtimeConfig', { /* config */ });
  // ... version endpoint and ACLs ...
};
```
*Explanation:* Provides a specific API endpoint for the frontend to query non-sensitive, backend-determined runtime settings like license keys read from the backend's environment.

## Conclusion

Configuration management is essential for making Pretzel adaptable and secure across different environments. The frontend relies on build-time configuration via `config/environment.js` (often populated by build environment variables), while the backend uses a combination of runtime environment variables (`.env` loaded into `process.env`) and local configuration files (`*.local.js`) that read these variables to control database connections, ports, authentication, and other runtime behaviors. This separation allows developers and administrators to deploy Pretzel flexibly and securely without modifying the core application code, providing the necessary instructions for each component to function correctly in its specific context.

This concludes our journey through the core concepts of the Pretzel application. We hope this tutorial has provided valuable insights into its architecture and how its various components work together.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)