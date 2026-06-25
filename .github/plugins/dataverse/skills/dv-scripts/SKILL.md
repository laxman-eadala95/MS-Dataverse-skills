---
name: dv-scripts
description: >
  Author, upload, and register Dataverse JavaScript web resources (form scripts, common libraries).
  Use when: "add form script", "create web resource", "register JS on form", "common library",
  "jio_common.js", "shared utilities", "form event", "onChange", "onLoad", "setRequired",
  "setVisible", "setDisabled", "field validation", "business logic in JS", "refactor JS",
  "tightly coupled", "constants library", "utility functions", "web resource dependency".
  Do not use when: creating tables or columns (use dv-metadata), writing Python scripts (use dv-data / dv-query).
---

# Skill: JavaScript Web Resources

Covers authoring, uploading, registering, and maintaining Dataverse form scripts and shared JS libraries.

---

## Skill boundaries

| Need | Use instead |
|---|---|
| Create tables, columns, relationships, forms, views | **dv-metadata** |
| Create or update data records | **dv-data** |
| Query or read records | **dv-query** |
| Export or deploy solutions | **dv-solution** |

---

## Architecture: Two-layer JS model

Every project uses exactly **two layers** of JavaScript web resources:

```
jio_common.js          ← Layer 1: constants + shared utilities (loaded first)
jio_<entity>_form.js   ← Layer 2: entity-specific logic (depends on jio_common.js)
```

Never put magic strings or magic numbers in entity-specific scripts.
Never put entity-specific logic in the common library.

---

## Standards

### Language: ES6+
Use `const`/`let`, arrow functions, destructuring, template literals.
Dataverse Unified Interface supports ES6+ natively — no transpilation needed.

### Namespace
All libraries live under the `Jio` global namespace to prevent collisions:
```js
var Jio = Jio || {};   // safe initialisation — works if other files set it first
```

Entity-specific scripts use their own namespaces under a separate global:
```js
var SupportTicket = SupportTicket || {};
```

### Module pattern: IIFE
Every file returns a frozen public API via an IIFE.
Internal helpers are private (not on the return object).

### Constants: grouped by entity, then by type
```js
Jio.Common.Entities.SupportTicket.Fields.CustomerType   = "jio_customertype"
Jio.Common.Entities.SupportTicket.OptionValues.CustomerType.B2B = 200010200
```

Never reference a raw string `"jio_customertype"` or raw number `200010200` outside `jio_common.js`.

---

## Layer 1: jio_common.js

Full canonical template — copy and extend for every new entity:

```js
// jio_common.js
// Shared constants and utility functions for all Dataverse form scripts.
// Must be loaded BEFORE any entity-specific script.
// Web resource name: jio_common.js

var Jio = Jio || {};

Jio.Common = (function () {
    "use strict";

    // ── Entity constants ────────────────────────────────────────────────
    // Add a new block per entity as the project grows.

    const SupportTicket = {
        Fields: {
            TicketId:     "jio_ticketid",
            Subject:      "jio_name",
            Customer:     "jio_customerid",
            CustomerType: "jio_customertype",
            Priority:     "jio_priority",
            Status:       "statuscode",
            Agent:        "jio_agentid",
            Email:        "jio_email",
            Phone:        "jio_phone",
            Description:  "jio_description",
        },
        OptionValues: {
            CustomerType: { B2B: 200010200, B2C: 200010201 },
            Priority:     { Low: 200010000, Medium: 200010001,
                            High: 200010002, Critical: 200010003 },
            Status:       { Open: 200010010, InProgress: 200010011,
                            Resolved: 200010012, Closed: 200010013 },
        },
    };

    // ── Utility functions ───────────────────────────────────────────────
    // All functions accept formContext as first arg so callers never
    // need to reach into Xrm directly.

    const Utils = {
        // Attribute access
        getAttr:   (fc, f) => fc.getAttribute(f),
        getCtrl:   (fc, f) => fc.getControl(f),
        getValue:  (fc, f) => { const a = fc.getAttribute(f); return a ? a.getValue() : null; },
        setValue:  (fc, f, v) => { const a = fc.getAttribute(f); if (a) a.setValue(v); },

        // Required level  — level: "none" | "recommended" | "required"
        setRequired: (fc, f, level) => {
            const a = fc.getAttribute(f);
            if (a) a.setRequiredLevel(level);
        },

        // Visibility / disabled state
        setVisible:  (fc, f, show) => { const c = fc.getControl(f); if (c) c.setVisible(show); },
        setDisabled: (fc, f, lock) => { const c = fc.getControl(f); if (c) c.setDisabled(lock); },

        // Form-level notifications
        notify:      (fc, msg, level, id) => fc.ui.setFormNotification(msg, level, id),
        clearNotify: (fc, id)             => fc.ui.clearFormNotification(id),

        // Register onChange safely (guards against missing attribute)
        onChange: (fc, f, handler) => {
            const a = fc.getAttribute(f);
            if (a) a.addOnChange(handler);
        },
    };

    // ── Public API ──────────────────────────────────────────────────────
    return Object.freeze({
        Entities: Object.freeze({ SupportTicket }),
        Utils:    Object.freeze(Utils),
    });

})();
```

---

## Layer 2: entity-specific script

Template for any entity form script that depends on `jio_common.js`:

```js
// jio_<entity>_form.js
// Depends on: jio_common.js  (registered as a dependency — loads first)
// Web resource name: jio_<entity>_form.js

var <Entity> = <Entity> || {};

<Entity>.Form = (function () {
    "use strict";

    // Destructure from common — no magic strings below this line
    const { Entities, Utils } = Jio.Common;
    const { Fields, OptionValues } = Entities.<Entity>;

    // ── private handlers ──────────────────────────────────────────────

    function someHandler(executionContext) {
        const fc = executionContext.getFormContext();
        // use Utils.* and Fields.* — never raw strings
    }

    // ── onLoad ────────────────────────────────────────────────────────

    function onLoad(executionContext) {
        const fc = executionContext.getFormContext();
        Utils.onChange(fc, Fields.SomeField, someHandler);
        someHandler(executionContext);   // apply on initial load
    }

    // ── public API ────────────────────────────────────────────────────
    return Object.freeze({ onLoad });

})();
```

---

## SupportTicket example (fully decoupled)

```js
// jio_supportticket_form.js
var SupportTicket = SupportTicket || {};

SupportTicket.Form = (function () {
    "use strict";

    const { Entities, Utils } = Jio.Common;
    const { Fields, OptionValues } = Entities.SupportTicket;
    const CT = OptionValues.CustomerType;

    function applyCustomerTypeLogic(executionContext) {
        const fc    = executionContext.getFormContext();
        const value = Utils.getValue(fc, Fields.CustomerType);

        if (value === CT.B2B) {
            Utils.setRequired(fc, Fields.Email, "required");
            Utils.setRequired(fc, Fields.Phone, "none");
        } else if (value === CT.B2C) {
            Utils.setRequired(fc, Fields.Phone, "required");
            Utils.setRequired(fc, Fields.Email, "none");
        } else {
            Utils.setRequired(fc, Fields.Email, "none");
            Utils.setRequired(fc, Fields.Phone, "none");
        }
    }

    function onLoad(executionContext) {
        const fc = executionContext.getFormContext();
        Utils.onChange(fc, Fields.CustomerType, applyCustomerTypeLogic);
        applyCustomerTypeLogic(executionContext);
    }

    return Object.freeze({ onLoad });
})();
```

---

## Uploading web resources via Python

```python
import base64, requests
from auth import load_env, get_token

load_env()
ENV      = os.environ["DATAVERSE_URL"].rstrip("/")
SOLUTION = "AutomationTesting"

def upload_webresource(name, display_name, js_path):
    content_b64 = base64.b64encode(open(js_path, "rb").read()).decode("ascii")
    token = get_token()
    hdrs  = {"Authorization": f"Bearer {token}", "Accept": "application/json",
             "Content-Type": "application/json", "OData-MaxVersion": "4.0",
             "OData-Version": "4.0", "MSCRM.SolutionName": SOLUTION}

    # Check if already exists
    existing = requests.get(
        f"{ENV}/api/data/v9.2/webresourceset?$filter=name eq '{name}'&$select=webresourceid",
        headers=hdrs).json().get("value", [])

    if existing:
        wr_id = existing[0]["webresourceid"]
        requests.patch(f"{ENV}/api/data/v9.2/webresourceset({wr_id})",
                       headers=hdrs, json={"content": content_b64})
        print(f"  Updated: {name}")
    else:
        r = requests.post(f"{ENV}/api/data/v9.2/webresourceset", headers=hdrs, json={
            "name": name, "displayname": display_name,
            "webresourcetype": 3, "content": content_b64,
            "isenabledformobileclient": False,
        })
        wr_id = r.headers["OData-EntityId"].split("(")[-1].rstrip(")")
        print(f"  Created: {name}  ({wr_id})")

    # Add to solution
    requests.post(f"{ENV}/api/data/v9.2/AddSolutionComponent",
        headers={**hdrs, "MSCRM.SolutionName": ""},
        json={"ComponentId": wr_id, "ComponentType": 61,
              "SolutionUniqueName": SOLUTION, "AddRequiredComponents": False})
    return wr_id
```

---

## Registering a library on a form

```python
from xml.etree import ElementTree as ET
import uuid

def new_guid():
    return "{" + str(uuid.uuid4()).upper() + "}"

def register_on_form(form_xml, wr_name, func_name):
    root = ET.fromstring(form_xml)

    # formLibraries
    libs = root.find("formLibraries")
    if libs is None:
        libs = ET.Element("formLibraries")
        root.insert(0, libs)
    if not any(l.get("name") == wr_name for l in libs.findall("Library")):
        lib = ET.SubElement(libs, "Library")
        lib.set("name", wr_name)
        lib.set("libraryUniqueId", new_guid())

    # events / onload handler
    events = root.find("events")
    if events is None:
        events = ET.Element("events")
        root.insert(list(root).index(libs) + 1, events)
    onload = next((e for e in events.findall("event") if e.get("name") == "onload"), None)
    if onload is None:
        onload = ET.SubElement(events, "event")
        onload.set("name", "onload")
        onload.set("application", "false")
        onload.set("active", "false")
    handlers = onload.find("Handlers")
    if handlers is None:
        handlers = ET.SubElement(onload, "Handlers")
    if not any(h.get("functionName") == func_name for h in handlers.findall("Handler")):
        h = ET.SubElement(handlers, "Handler")
        h.set("functionName", func_name)
        h.set("libraryName", wr_name)
        h.set("handlerUniqueId", new_guid())
        h.set("enabled", "true")
        h.set("parameters", "")
        h.set("passExecutionContext", "true")

    return ET.tostring(root, encoding="unicode")
```

---

## Web resource naming convention

| File | Web resource name | Namespace |
|---|---|---|
| Common library | `jio_common.js` | `Jio.Common` |
| Support Ticket form | `jio_supportticket_form.js` | `SupportTicket.Form` |
| Account form | `jio_account_form.js` | `Account.Form` |
| Contact form | `jio_contact_form.js` | `Contact.Form` |

---

## Loading order — dependency declaration in form XML

`jio_common.js` **must appear before** `jio_<entity>_form.js` in `<formLibraries>`.
Always insert the common library first when building the formLibraries list:

```python
LIBS_IN_ORDER = [
    "jio_common.js",          # always first
    "jio_supportticket_form.js",
]
for wr_name in LIBS_IN_ORDER:
    if not any(l.get("name") == wr_name for l in libs.findall("Library")):
        lib = ET.SubElement(libs, "Library")
        lib.set("name", wr_name)
        lib.set("libraryUniqueId", new_guid())
```

---

## Adding a new entity — checklist

1. Add a new constants block to `jio_common.js` under `Entities`
2. Create `jio_<entity>_form.js` using the Layer 2 template
3. Upload both via `upload_webresource()`
4. Register `jio_common.js` first, then `jio_<entity>_form.js` on the form
5. Run `add_to_solution.py` to include the web resources
6. Re-export + unpack + commit + push

---

## After any JS change: pull to repo

```bash
pac solution export --name AutomationTesting --path ./solutions/AutomationTesting.zip --managed false
pac solution unpack --zipfile ./solutions/AutomationTesting.zip --folder ./solutions/AutomationTesting
rm ./solutions/AutomationTesting.zip
git add solutions/ scripts/webresources/
git commit -m "chore: update JS web resources"
git push
```
