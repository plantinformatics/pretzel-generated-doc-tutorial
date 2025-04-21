# Chapter 8: Loopback Models (Dataset, Block, Feature, Client, Group)

In [Chapter 7: Ember Services (API & State)](07_ember_services__api___state__.md), we saw how the Pretzel frontend uses Ember Services to manage state and interact with the backend API. This API, built with Loopback ([Chapter 6: Loopback Application & Server](06_loopback_application___server_.md)), needs a structured way to define the data it works with. How does the server know what constitutes a "Dataset" or a "Block"? How does it understand their properties, relationships, and the rules governing them?

## Motivation: The Backend Data Blueprints

Imagine the backend server as the main headquarters responsible for managing all the official data records. Just like a physical office needs well-defined filing systems and forms, the backend needs formal definitions for its data entities. Without these "blueprints," the server wouldn't know:

*   What fields belong to a `Dataset` (e.g., `name`, `description`).
*   How a `Block` relates to a `Dataset` (a Block belongs to one Dataset).
*   What data type each field should be (e.g., `name` is a string, `length` is a number).
*   What rules the data must follow (e.g., a `Block`'s `length` must be positive).
*   How to handle specific operations or enforce custom logic related to the data.

Loopback Models provide these blueprints. They are the server-side definitions of the application's data entities, stored in the backend "main office," dictating how data must be structured and what operations are allowed. This includes not only the genomic data (`Dataset`, `Block`, `Feature`) but also user management (`Client`, `Group`, `ClientGroup`).

**Our Central Use Case:** We need to define the `Block` model on the server. This definition must specify its properties (`name`, `length`, `_meta`), its relationship to `Dataset` (belongs to one) and `Feature` (has many), and potentially add validation rules (like ensuring `length` is positive).

## Key Concepts: Defining Data Structures and Logic

Loopback models are typically defined using two files per model, residing in the `common/models/` directory:

1.  **Model Definition File (`.json`)**:
    *   **What it is:** A JSON file (e.g., `block.json`) defining the model's core structure.
    *   **Role:** Specifies properties and their types, relationships with other models (`belongsTo`, `hasMany`), Access Control Lists (ACLs - see [Chapter 9: Authentication & Authorization](09_authentication___authorization_.md)), and the base model it extends (often `PersistedModel` or a custom base like `Record`).
    *   **Analogy:** The architectural blueprint showing rooms (properties) and connections (relations).

2.  **Model Script File (`.js`)**:
    *   **What it is:** A JavaScript file (e.g., `block.js`) associated with the model.
    *   **Role:** Allows adding custom logic:
        *   **Validation:** Implementing specific validation rules beyond basic type checking.
        *   **Operation Hooks:** Executing code before or after standard operations (e.g., `before save`, `after save`, `observe('access')`). These hooks allow you to modify data, enforce complex rules, or trigger side effects.
        *   **Remote Methods:** Defining custom API endpoints specific to this model that go beyond the standard CRUD operations.
        *   **Initialization:** Running code when the model is loaded by Loopback.
        *   **Attaching Mixins:** Incorporating reusable logic defined in `common/mixins/`.
    *   **Analogy:** The detailed specifications for plumbing, electrical wiring, and custom functionalities within the blueprint.

3.  **Base Models & Mixins (`record.js`, `mixins/`)**:
    *   Often, models share common fields (like `clientId`, `groupId`, `createdAt`, `updatedAt`) and logic (like ownership checks). This shared functionality is often defined in a base model (like `common/models/record.js`, even though direct inheritance isn't explicitly used by all models, its patterns are) or through Loopback Mixins.

4.  **Relations:** Loopback supports standard database relationships defined in the `.json` file:
    *   `belongsTo`: Many-to-one (e.g., a `Block` belongs to one `Dataset`).
    *   `hasMany`: One-to-many (e.g., a `Dataset` has many `Blocks`, a `Block` has many `Features`).
    *   `hasAndBelongsToMany`: Many-to-many (often through a specific join model like `ClientGroup`).

5.  **Model Registration (`server/model-config.local.js`)**:
    *   This crucial configuration file tells Loopback which models exist, which datasource they use (e.g., `"mongoDs"` for MongoDB - see [Chapter 6: Loopback Application & Server](06_loopback_application___server_.md)), and whether they should be exposed via the default REST API (`"public": true`).

## Solving the Use Case: Defining the `Block` Model

Let's define our `Block` model on the server side.

**1. Define Structure (`common/models/block.json`)**

```json
// lb4app/lb3app/common/models/block.json (Simplified)
{
  "name": "Block",
  "base": "PersistedModel", // Standard Loopback base for DB models
  "idInjection": true, // Automatically add an 'id' field
  "properties": {
    "name": {
      "type": "string",
      "required": true
    },
    "length": {
      "type": "number",
      "required": true
    },
    "meta": { // Loopback side uses 'meta', Ember uses '_meta'
      "type": "object"
    },
    "tags": {
      "type": [ "string" ]
    },
    // Properties for ownership/sharing (from Record pattern)
    "clientId": { "type": "string", "index": true },
    "groupId": { "type": "string", "index": true },
    "public": { "type": "boolean", "default": false },
    "readOnly": { "type": "boolean", "default": false },
    "createdAt": { "type": "date" },
    "updatedAt": { "type": "date" }
  },
  "relations": {
    "datasetId": { // Name chosen for clarity, points to Dataset model
      "type": "belongsTo",
      "model": "Dataset",
      "foreignKey": "datasetId_fk" // Explicit foreign key name in DB
    },
    "features": {
      "type": "hasMany",
      "model": "Feature",
      "foreignKey": "blockId" // Foreign key on the Feature model
    }
  },
  "validations": [],
  "acls": [], // Defined in Chapter 9
  "methods": {}
}
```

*Explanation:* This JSON file defines the `Block` model:
*   It inherits from `PersistedModel`.
*   Specifies properties like `name` (required string), `length` (required number), `meta` (object), and ownership fields like `clientId`.
*   Defines a `belongsTo` relationship named `datasetId` pointing to the `Dataset` model.
*   Defines a `hasMany` relationship named `features` pointing to the `Feature` model.

**2. Add Custom Logic/Validation (`common/models/block.js`)**

```javascript
// lb4app/lb3app/common/models/block.js (Simplified)
'use strict';

var acl = require('../utilities/acl');
var recordUtils = require('./record.js'); // Import base logic if needed

module.exports = function(Block) {

  // Add validation: Ensure 'length' is positive
  Block.validatesNumericalityOf('length', { int: true, gt: 0, message: { gt: 'Length must be greater than 0' } });

  // Apply standard access control rules from 'acl' utility
  acl.assignRulesRecord(Block);
  // Limit exposed remote methods using utility functions
  acl.limitRemoteMethods(Block); // Hide generic methods if needed
  acl.limitRemoteMethodsSubrecord(Block);
  acl.limitRemoteMethodsRelated(Block);

  // Add operation hook (Example: Apply common 'record' logic before save)
  Block.observe('before save', function(ctx, next) {
     // Example: Call a function mimicking base model logic
     recordUtils.applyRecordLogic(ctx); // Assumes recordUtils exists
     next();
  });

  // Could add custom remote methods here, e.g., Block.myCustomMethod = ...
};

// Hypothetical utility function in record.js or similar
// module.exports.applyRecordLogic = (ctx) => {
//   var newDate = Date.now();
//   if (ctx.currentInstance) { ctx.instance.updatedAt = newDate; }
//   else if (ctx.isNewInstance) { /* set clientId, createdAt etc */ }
// };
```

*Explanation:* This JS file enhances the `Block` model:
*   `Block.validatesNumericalityOf('length', ...)` adds a specific validation rule ensuring `length` is an integer greater than 0. Loopback provides several built-in validation helpers.
*   `acl.assignRulesRecord(Block)` applies reusable ACL rules defined in the `acl` utility (covered in [Chapter 9: Authentication & Authorization](09_authentication___authorization_.md)).
*   `acl.limitRemoteMethods*` functions might hide certain auto-generated API endpoints for security or simplicity.
*   `Block.observe('before save', ...)` registers an operation hook. This example shows how you might apply common logic (like setting `updatedAt` or `clientId` for new records) before saving a `Block`.

**3. Register the Model (`server/model-config.local.js`)**

Ensure this file maps the `Block` model to the `mongoDs` datasource and makes it public:

```javascript
// lb4app/lb3app/server/model-config.local.js (Excerpt)
'use strict';
var config = {
  // ... other models ...
  "Dataset": {
    "dataSource": "mongoDs",
    "public": true
  },
  "Block": { // Our newly defined model
    "dataSource": "mongoDs", // Use MongoDB
    "public": true         // Expose via REST API
  },
  "Feature": {
    "dataSource": "mongoDs",
    "public": true
  },
  "Client": { // User model
    "dataSource": "mongoDs",
    "public": true
  },
  "Group": { // Group model
     "dataSource": "mongoDs",
     "public": true
  },
  "ClientGroup": { // Join model for Client<->Group
     "dataSource": "mongoDs",
     "public": true
  },
  // ... other models like AccessToken, Role, ACL (often public: false) ...
};
module.exports = config;
```

*Explanation:* This entry tells Loopback to load the `Block` model, connect it to the `mongoDs` datasource defined in `datasources.local.js` ([Chapter 6](06_loopback_application___server_.md)), and automatically generate REST endpoints for it (because `"public": true`).

With these files created and configured, the Loopback server now has a complete blueprint for the `Block` entity, including its structure, relationships, validation, and associated logic.

## Internal Implementation: Model Loading and Request Handling

How does Loopback use these definitions?

**Step-by-Step Model Loading:**

1.  **Server Boot:** The server starts (`node server/server.js`), and the `boot` script runs ([Chapter 6](06_loopback_application___server_.md)).
2.  **Config Reading:** `boot` reads `server/model-config.local.js`.
3.  **Model Discovery:** For each entry (like `"Block"`), Loopback looks for `common/models/block.json` and `common/models/block.js`.
4.  **Model Construction:** It creates an internal representation of the `Block` model:
    *   Parses `block.json` for properties, relations, base model, etc.
    *   Executes `block.js`, attaching validation rules, operation hooks, and remote methods defined within it.
    *   Applies any specified mixins.
5.  **Datasource Linking:** The constructed model object is linked to the specified datasource (`mongoDs`). The datasource connector (e.g., `loopback-connector-mongodb`) provides the bridge to the actual database.
6.  **API Generation:** Because `"public": true`, Loopback automatically generates standard REST endpoints (e.g., `GET /api/blocks`, `POST /api/blocks`, `GET /api/blocks/{id}`, `PUT /api/blocks/{id}`, etc.) mapped to the model's methods (`find`, `create`, `findById`, `upsertWithWhere`, etc.).

**Step-by-Step Request Handling (e.g., `POST /api/blocks`)**

1.  **Request Arrival:** An HTTP `POST` request with block data in the body arrives at `/api/blocks`.
2.  **Routing:** Loopback's REST router maps this to the `Block.create` method.
3.  **Authentication/Authorization:** Middleware checks if the user is authenticated and authorized (ACLs) to create a `Block` ([Chapter 9](09_authentication___authorization_.md)).
4.  **`before save` Hook:** The hook registered in `block.js` runs (e.g., setting `clientId`, `createdAt`, `updatedAt`).
5.  **Validation:** Loopback checks built-in validations (e.g., `name` is present) and custom validations (e.g., `length > 0` from `block.js`). If validation fails, a 422 Unprocessable Entity error is returned.
6.  **Datasource Call:** If valid, the validated data is passed to the `mongoDs` connector's `create` (or equivalent) method.
7.  **DB Operation:** The connector constructs and executes the appropriate MongoDB command (e.g., `db.blocks.insertOne(...)`).
8.  **DB Response:** MongoDB performs the insertion and returns the result (e.g., the new document with its `_id`).
9.  **Datasource Response:** The connector translates the DB result back.
10. **`after save` Hook (if any):** Any `after save` hooks registered for `Block` would run.
11. **API Response:** Loopback formats the created block data as JSON and sends a `200 OK` (or `201 Created`) response back to the client.

**Diagram: Model Definition & Request Flow**

```mermaid
graph TD
    subgraph Server Boot
        direction LR
        BootScript[server.js / boot()] --> ModelConfig[model-config.local.js]
        ModelConfig --> BlockJSON[block.json: Props, Relations]
        ModelConfig --> BlockJS[block.js: Logic, Hooks, Validation]
        BlockJSON & BlockJS --> LoopbackModel[Loopback 'Block' Object]
        ModelConfig --> DataSource[datasources.local.js]
        DataSource --> MongoConnector[MongoDB Connector]
        LoopbackModel -- Attached To --> MongoConnector
        LoopbackModel -- Exposes --> RestAPI[REST API Endpoints]
    end

    subgraph API Request (POST /api/blocks)
        direction TB
        ClientRequest[Client Request (JSON Body)] --> RestAPI
        RestAPI --> Auth[Auth/ACL Check]
        Auth -- Granted --> HooksBefore[block.js: before save Hook]
        HooksBefore --> Validation[Validation: block.js & block.json]
        Validation -- Valid --> MongoConnector[Connector: create]
        MongoConnector --> MongoDB[Database: insertOne]
        MongoDB -- Result --> MongoConnector
        MongoConnector --> HooksAfter[block.js: after save Hook]
        HooksAfter --> Response[API Response (JSON)]
        Validation -- Invalid --> ErrorResponse[422 Error Response]
        Auth -- Denied --> ErrorResponse401[401/403 Error Response]
    end

    Server Boot --> API Request
```

## Other Key Models

Besides `Dataset`, `Block`, and `Feature`, other crucial models define users and groups:

*   **`Client` (`common/models/client.js`, `.json`)**: Represents a user account. Typically extends Loopback's built-in `User` model or uses it as a base. Contains properties like `email`, `password` (hashed), `name`, `institution`, `emailVerified`, etc. The `.js` file often includes logic for email verification (`afterRemote('create')`), password resets (`on('resetPasswordRequest')`), and confirmation hooks (`beforeRemote('confirm')`).
*   **`Group` (`common/models/group.js`, `.json`)**: Represents a user-created group for sharing data. Properties include `name`, `writable` (can members add data?), and relationships like `clientId` (belongsTo the owner `Client`) and `clientGroups` (hasMany join records).
*   **`ClientGroup` (`common/models/clientGroup.js`, `.json`)**: The join model connecting `Client` and `Group` (many-to-many). It typically has `clientId` (belongsTo `Client`) and `groupId` (belongsTo `Group`). The `.js` file might contain remote methods like `addEmail` to add a user to a group by their email address, handling the lookup and creation of the `ClientGroup` record. It also implements logic to ensure only authorized users (group owner or the member themselves, depending on configuration) can remove a member (`observe('before delete')`).

These models work together, using the same Loopback mechanisms (JSON definitions, JS logic, hooks, relations, datasources) to manage user identity and data sharing permissions within Pretzel.

## Conclusion

Loopback Models are the essential server-side blueprints defining the structure, relationships, validation rules, and custom logic for all data entities in Pretzel, including genomic data (`Dataset`, `Block`, `Feature`) and user/group management (`Client`, `Group`, `ClientGroup`). Defined through `.json` and `.js` files, they provide a structured way for the backend to interact with the database and enforce application rules. They are registered via `model-config.local.js` and interact with datasources defined in `datasources.local.js`. Understanding these models is crucial for comprehending how Pretzel manages its data persistence and backend logic.

Now that we understand *what* data structures exist on the backend, the next chapter delves into *who* can access and modify this data.

**Next:** [Chapter 9: Authentication & Authorization](09_authentication___authorization_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)