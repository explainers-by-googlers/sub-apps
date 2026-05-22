# Explainer: Sub Apps API

## Background & Introduction

The [Isolated Web Apps](https://github.com/WICG/isolated-web-apps/blob/main/README.md) (IWA) proposal defines a security model and distribution packaging system (Signed Web Bundles) that enables web applications to run with heightened security privileges and offline robustness. However, a single packaged installation does not always map neatly to the diverse, modular applications a developer needs to present to a user.

Developers frequently want to deliver:
- Suite-like experiences (such as office productivity suites containing an email client, a word processor, and a spreadsheet editor).
- Multi-app streaming portals (where a client hosts multiple distinct virtualized applications like a code editor, a browser, and a terminal).

Under a standard Web App installation, all these modules must share a single launcher icon, application name, and shelf window presence, or they must be installed as entirely separate IWAs (incurring massive bundle overhead, distinct update cycles, and separate storage origins).

The **Sub Apps API** solves this problem by allowing a single parent IWA to programmatically install, list, and remove auxiliary applications (Sub Apps) that:
1. Appear to the OS and user as fully distinct applications (separate launcher icons, distinct taskbar/shelf windows, and individual OS integrations like protocol and file handlers).
2. Share the underlying Signed Web Bundle resources, origin identification, storage, permissions, and update lifecycle of the parent IWA.

This enables seamless modularity without the friction of multiple installation payloads, fragmented storage, or divergent updates.

## Participate
- https://github.com/explainers-by-googlers/sub-apps/issues

## Table of Contents
- [Background & Introduction](#background--introduction)
- [Use cases](#use-cases)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [API Design](#api-design)
- [Examples](#examples)
- [Detailed Design Discussion](#detailed-design-discussion)
  - [Start URL vs. Manifest String](#start-url-vs-manifest-string)
  - [Sub App Identity](#sub-app-identity)
  - [Quota & Limits](#quota--limits)
- [Considered Alternatives](#considered-alternatives)
- [References & acknowledgements](#references--acknowledgements)

## Use cases

### 1. Seamless App Streaming
A virtual desktop application streams remote applications to the client. The user wants to interact with these remote apps (e.g., a terminal, a text editor, a browser) as if they were running natively on their local operating system. Using the Sub Apps API, the parent IWA can programmatically install a sub-app launcher icon for each remote application. When launched, each remote application runs in its own standalone window with its own distinct name, custom icon, and taskbar item, completely disconnected visually from the parent application.

### 2. Office Productivity Suites
An office suite consists of an email client, a document processor, and a spreadsheet editor. Rather than forcing the user to install three separate heavyweight IWAs or bundle them all under a single confusing window/icon, the developer installs one parent IWA. The parent IWA then programmatically installs individual Sub Apps. The user gains distinct entry points in their application launcher and can run multiple document/email windows in parallel, each with their proper visual identity, while sharing a single IndexedDB database and cached bundle.

## Security and Privacy Considerations

Because Sub Apps operate inside the security perimeter of the parent IWA, they are designed with strong guardrails to prevent origin confusion and capabilities leakage.

### Shared Origin identity
A Sub App does not possess a separate security origin. It shares the exact same origin, and local data stores (Cookies, IndexedDB, LocalStorage, and Cache Storage) with the parent IWA. Therefore, standard web security boundaries (such as the Same-Origin Policy) treat the parent and all its Sub Apps as a single entity.

### Permission and Policy Inheritance
All permissions and enterprise policies are shared across the parent and its Sub Apps:
- Granting a permission (e.g., camera, file system access) to a Sub App automatically grants it to the parent, and vice versa.
- Enterprise policies (e.g., blocking or force-allowing specific features) configured for the parent automatically apply to all Sub Apps. It is not possible to apply permission controls or enterprise policies to a Sub App independently from its parent.
- To access the Sub Apps API, the parent IWA must explicitly declare the permission policy `sub-apps` (e.g., `Permissions-Policy: sub-apps=(self)`).

### Explicit User Consent & Gesture Requirements
- **User Activation**: Installing a sub-app requires transient user activation (a user gesture).
- **Installation Prompts**: When `window.subApps.add()` is called, a system permission prompt displays all requested sub-apps in a unified installation dialog. If multiple sub-apps are added at once, they are presented inside a scroll container in a single prompt to avoid dialog spam.

### Identity Spoofing Risks
Since developers can customize the names and icons of Sub Apps, there is a risk that a malicious IWA could create a sub-app that mimics system dialogs or trusted third-party applications.
- **Mitigation**: The Sub Apps API is restricted to Isolated Web Apps, which have integrity and signature verification, also they are only allowed to execute code that is bundled, meaning no dynamic JS invocation is possible.

## API Design

The `subApps` object is exposed on the `Window` interface:

```webidl
[
  Exposed=Window,
  SecureContext,
  IsolatedContext
] partial interface Window {
  [SameObject] readonly attribute SubApps subApps;
};

typedef USVString ManifestId;

dictionary SubAppsListResult {
  required DOMString appName;
};

[
  Exposed=Window,
  SecureContext,
  IsolatedContext
] interface SubApps {
  sequence<Promise<ManifestId>> add(sequence<USVString> install_urls);
  sequence<Promise<ManifestId>> remove(sequence<ManifestId> manifest_ids);
  Promise<record<ManifestId, SubAppsListResult>> list();
};
```

### Detailed API Behavior

#### `add(sequence<USVString> install_urls)`
- **Arguments**: A sequence of relative URLs representing the start URLs of the sub-apps to be installed.
- **Synchronous Throws**:
  - `TypeError`: If invalid URLs are passed.
  - `SecurityError`: If the `sub-apps` permissions policy is missing or not allowed.
  - `NotAllowedError`: If the call was not triggered by a transient user gesture.
- **Asynchronous Rejections**: Returns a sequence of Promises. Upon successful installation, each Promise resolves to the sub-app's `ManifestId`. If an individual sub-app fails to install, its corresponding Promise rejects with a `DOMException`:
  - `NotAllowedError`: The user declined the installation prompt. (Declining the prompt in batch installations rejects all pending promises in the batch).
  - `DataError`: The referenced web manifest was invalid or could not be parsed.
  - `InvalidStateError`: The sub-app is already installed.
  - `QuotaExceededError`: The number of sub-apps exceeds the platform limit.
  - `OperationError`: A generic system/database failure occurred.

#### `remove(sequence<ManifestId> manifest_ids)`
- **Arguments**: A sequence of `ManifestId` values to uninstall.
- **Synchronous Throws**:
  - `TypeError`: If invalid URLs are provided.
  - `SecurityError`: If the `sub-apps` permissions policy is missing or not allowed.
- **Asynchronous Rejections**: Returns a sequence of Promises. Upon successful removal, each Promise resolves to the `ManifestId`. If an individual sub-app fails to remove, its corresponding Promise rejects with a `DOMException`:
  - `NotFoundError`: No sub-app with the given ID is installed under this parent IWA, it belongs to a different parent IWA, or it was already removed.
  - `OperationError`: A generic deletion failure occurred.

#### `list()`
- **Synchronous Throws**:
  - `SecurityError`: If the `sub-apps` permissions policy is missing or not allowed.
- **Asynchronous Rejections**: Returns a Promise. If the operation fails due to an internal system failure, the Promise rejects with a `DOMException`:
  - `OperationError`: A generic failure occurred during the list retrieval from the underlying database.
- **Returned Object**: The function returns a record mapping `ManifestId` to `SubAppsListResult`. This mapping (instead of a sequence/list) makes it convenient for developers to use dictionary operations on the result (e.g., to directly check if it contains a specific `manifest_id`).
  - `SubAppsListResult.appName` is the name of the sub-app extracted from its web manifest.

## Examples

### 1. Installing Sub Apps
To install a sub-app, you call `window.subApps.add()` with one or more relative URLs pointing to the sub-app entry point. Each entry point must be a valid HTML page that links to a web manifest.

```js
async function installCalculator() {
  try {
    // Call requires a user gesture (transient activation)
    const [calculatorPromise] = window.subApps.add(["/calculator_app/index.html"]);
    
    // Wait for the user to approve the installation dialog
    const manifestId = await calculatorPromise;
    console.log(`Successfully installed Sub App with Manifest ID: ${manifestId}`);
  } catch (error) {
    console.error("Failed to install Sub App:", error);
  }
}
```

Example manifest linked by `/calculator_app/index.html` (`calculator.webmanifest`):
```json
{
  "start_url": "/calculator_app/index.html",
  "name": "Calculator",
  "version": "3.0.0",
  "display": "standalone",
  "icons": [
    {
      "src": "/images/calculator.png",
      "type": "image/png",
      "sizes": "512x512",
      "purpose": "any maskable"
    }
  ],
  "protocol_handlers": [
    {
      "protocol": "web+calc",
      "url": "/calculator_app/index.html?query=%s"
    }
  ],
  "launch_handler": {
    "client_mode": "focus-existing"
  }
}
```

### 2. Listing Installed Sub Apps
You can retrieve all currently installed sub-apps associated with the parent app:

```js
async function printInstalledSubApps() {
  try {
    const installedApps = await window.subApps.list();
    
    for (const [manifestId, info] of Object.entries(installedApps)) {
      console.log(`Sub App Manifest ID: ${manifestId}`);
      console.log(`Sub App Name: ${info.appName}`);
    }
  } catch (error) {
    console.error("Error listing sub-apps:", error);
  }
}
```

### 3. Removing a Sub App
To uninstall a sub-app programmatically, call `window.subApps.remove()` with the list of target manifest IDs:

```js
async function uninstallCalculator(calculatorId) {
  try {
    const [removePromise] = window.subApps.remove([calculatorId]);
    const removedId = await removePromise;
    console.log(`Successfully removed Sub App: ${removedId}`);
  } catch (error) {
    console.error("Failed to remove Sub App:", error);
  }
}
```

### 4. Dynamic Sub App Provisioning (Service Worker)
For dynamic app environments, developers can generate HTML pages and web manifests dynamically via a Service Worker. The Service Worker intercepts requests to dynamic start URLs and returns a custom manifest.

```js
// service-worker.js
self.addEventListener("fetch", (event) => {
  const url = new URL(event.request.url);
  
  const htmlMatch = url.pathname.match(/^\/dynamic\/(.+)\/app\.html$/);
  const manifestMatch = url.pathname.match(/^\/dynamic\/(.+)\/app\.webmanifest$/);
  
  if (htmlMatch) {
    const appName = htmlMatch[1];
    const html = getDynamicHtml(appName);
    event.respondWith(
      new Response(html, {
        headers: { "Content-Type": "text/html; charset=utf-8" }
      })
    );
  } else if (manifestMatch) {
    const appName = manifestMatch[1];
    const manifest = getDynamicManifest(appName);
    const json = JSON.stringify(manifest);
    event.respondWith(
      new Response(json, {
        headers: { "Content-Type": "application/manifest+json" }
      })
    );
  }
});

function getDynamicHtml(name) {
  return `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="utf-8" />
        <link rel="manifest" href="/dynamic/${name}/app.webmanifest" />
        <title>${name} App</title>
      </head>
      <body>
        <h1>Dynamic App: ${name}</h1>
      </body>
    </html>
  `;
}

function getDynamicManifest(name) {
  return {
    "name": name.toUpperCase(),
    "start_url": `/dynamic/${name}/app.html`,
    "version": "1.0.0",
    "display": "standalone",
    "icons": [
      {
        "src": "/images/default_icon.png",
        "type": "image/png",
        "sizes": "512x512",
        "purpose": "any maskable"
      }
    ]
  };
}
```

## Detailed Design Discussion

### Start URL vs. Manifest String
During early API design, two distinct options were considered for defining sub-apps:
- **Start URL (Chosen)**: Pass the `start_url` of the sub-app, which references a standard hosted/provided web manifest.
- **Manifest String**: Feed a JSON manifest object directly as an argument to `window.subApps.add()`.

The **Start URL** approach was chosen because it aligns natively with standard web-manifest parsing, validation. Passing raw manifest objects would require introducing custom ID generators, complex update/refresh mechanisms, and completely bypassing standard manifest security audits, creating a high maintenance and security overhead.

### Sub App Identity

Sub apps rely on standard web manifest identity fields, but enforce specific isolation and overlap rules to guarantee they do not conflict with each other or the parent application.

| Identifier | Relationship to Parent / Generation Rule |
| :--- | :--- |
| **Origin** | Identical to the parent IWA. |
| **SignedWebBundleId** | Identical to the parent IWA. |
| **Start URL** | Must be defined in the web manifest. Must be unique between sub-apps and the parent app. |
| **Scope** | Might be defined in the web manifest. Scopes must not overlap between sub-apps. The scope of a sub-app must not overlap with or cover the parent app's scope. Otherwise, the installation/update fails. |
| **URL** | Parent IWA origin + `/` + `sub_app_start_url`. *Example:* `isolated-app://pl2ctdpnkf7ltse22mpjdb376etd3ydo7s72lgspuopgzcwl5tkqaaic/sub/calculator.html` |
| **AppId** | Generated uniquely for each sub-app. |
| **ManifestId** | Unique for each sub-app (represents the `start_url` without the reference/hash fragment). *Example:* `/sub/calculator.html` |

### Quota & Limits
To protect the host operating system and the user's application launcher from potential exhaustion or abuse, the platform enforces a hard limit of **20 installed sub-apps** per parent IWA.
- **Why a total limit**: Malicious or runaway applications could easily flood the user's operating system with hundreds of sub-apps. Enforcing a per-call or per-prompt limit would fail to prevent this, as developers could simply trigger successive permission prompts.
- **Partial Success**: If a parent IWA has 19 sub-apps installed and makes a batch call to install 2 more, the first one will be successfully processed, while the second will fail and its corresponding Promise will reject with a `QuotaExceededError` DOMException.

## Considered Alternatives

### Direct window creation with custom icons (`chrome.app.window.create`)
The legacy Chrome Apps platform allowed applications to open new windows with bespoke titles and window icons. While simple, this approach has severe drawbacks:
- **No OS Integration**: It operates only at the visual level and does not expose full OS capability integrations such as launcher searchability, shelf pinning, custom protocol handlers, or file-type associations.
- **Inconsistent Launcher UX**: Windows opened this way do not behave as individual independent apps in the system app manager or application list, frustrating users who expect standard operating system behaviors.

## References & acknowledgements

This specification builds heavily on the PWA manifest update process and standard web platform patterns.

Many thanks to the following individuals for their advice, feedback, and review:
- Ivan Sandrk
- Andrew Rayskiy
- Edman Anjos
- Dominik Bylica 
- Dan Murphy
