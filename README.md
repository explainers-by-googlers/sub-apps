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
- [Resource Provisioning & Installation](#resource-provisioning--installation)
- [Detailed Design Discussion](#detailed-design-discussion)
  - [Install URL vs. Manifest String](#install-url-vs-manifest-string)
  - [Returning a promise vs sequence of promises](#returning-a-promise-vs-sequence-of-promises)
  - [Sub App Identity](#sub-app-identity)
  - [Internal Manifest Data](#internal-manifest-data)
  - [OS Integrations](#os-integrations)
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

### Permission inheritance
All permissions are shared across the parent and its Sub Apps:
- Granting a permission (e.g., camera, file system access) to a Sub App automatically grants it to the parent, and vice versa.
- To access the Sub Apps API, the parent IWA must explicitly declare the permission policy `sub-apps` (e.g., `Permissions-Policy: sub-apps=(self)`). Permission policies declared for a sub app have no effect.

### Explicit User Consent
- **Installation Prompts**: When `window.subApps.add()` is called, a system permission prompt displays all requested sub-apps in a unified installation dialog. If multiple sub-apps are added at once, they are presented inside a scroll container in a single prompt to avoid dialog spam.

### Identity Spoofing Risks
Since developers can customize the names and icons of Sub Apps, there is a risk that a malicious IWA could create a sub-app that mimics system dialogs or trusted third-party applications.
- **Mitigation**: The Sub Apps API is restricted to Isolated Web Apps, which have integrity and signature verification, also they are only allowed to execute code that is bundled, meaning no dynamic JS invocation is possible.

### OS Integration Extension Risk
Sub apps have the ability to register their own OS integrations (such as protocol handlers or file type associations). This means an IWA could potentially extend its reach into the OS far beyond what was declared or audited in the parent app's primary manifest, because sub app web manifest can be even generated dynamically using Service Worker provisioning which means our ability to inspect the manifest is limited.
This is mitigated by the fact that most OS integrations require user approval before working. For example if a file handler is specified, it does not mean that the app will instantly handle files by default, first the user will need to select the sub app as the default app and click “Open always with”.

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
typedef USVString InstallURL;

dictionary SubAppsAddResponse {
  record<InstallURL, ManifestId> installedApps;
  record<InstallURL, DOMException> failedApps;
};

dictionary SubAppsRemoveResponse {
  sequence<ManifestId> removedApps;
  record<ManifestId, DOMException> failedApps;
};

dictionary SubAppsListResult {
  required DOMString appName;
};

[
  Exposed=Window,
  SecureContext,
  IsolatedContext
] interface SubApps {
  Promise<SubAppsAddResponse> add(sequence<InstallURL> install_urls);
  Promise<SubAppsRemoveResponse> remove(sequence<ManifestId> manifest_ids);
  Promise<record<ManifestId, SubAppsListResult>> list();
};
```

### Detailed API Behavior

#### `add(sequence<USVString> install_urls)`
- **Arguments**: A sequence of relative URLs (`InstallURL`) representing the start URLs of the sub-apps to be installed. Each install URL is the URL path to the sub-app's `index.html` that must reference a web manifest.
- **Asynchronous Rejections**: Returns a Promise that resolves to a `SubAppsAddResponse`. There are no synchronous exceptions. The entire promise can be rejected with a `DOMException` for global/batch-level failures:
  - `TypeError`: If invalid URLs are passed.
  - `SecurityError`: If the `sub-apps` permissions policy is missing or not allowed.
  - `NotSupportedError`: If the API is called inside of another sub app frame.
  - `NotAllowedError`: The user declined the batch installation prompt.
  - `QuotaExceededError`: The number of sub-apps exceeds the platform limit per parent app.
  - `OperationError`: A generic failure occurred.
- **Returned Object**: `SubAppsAddResponse` contains records mapping each input `InstallURL` provided to the call to either its success or failure state. Developers must use the `InstallURL` key to identify which call failed and which succeeded:
  - `installedApps`: A record mapping the `InstallURL` to the successfully installed sub-app's `ManifestId`. `ManifestId` is a stable identified of a sub app, it is later used in `subApps.remove` and `subApps.list` calls. For what is `ManifestId` exactly, please, consult [Sub App identity](#sub-app-identity) section.
  - `failedApps`: A record mapping the `InstallURL` to a `DOMException` explaining why that individual sub-app failed to install. Possible exceptions include:
    - `DataError`: The referenced web manifest was invalid or could not be parsed.
    - `InvalidStateError`: The sub-app is already installed.
    - `OperationError`: A generic system or database failure occurred during the installation of this specific sub-app.

#### `remove(sequence<ManifestId> manifest_ids)`
- **Arguments**: A sequence of `ManifestId` values to uninstall.
- **Asynchronous Rejections**: Returns a Promise that resolves to a `SubAppsRemoveResponse`. There are no synchronous exceptions. The entire promise can be rejected with a `DOMException` for global/batch-level failures:
  - `TypeError`: If invalid URLs are provided.
  - `SecurityError`: If the `sub-apps` permissions policy is missing or not allowed.
  - `NotSupportedError`: If the API is called inside of another sub app frame.
  - `OperationError`: A generic deletion failure occurred.
- **Returned Object**: `SubAppsRemoveResponse` contains the successfully removed IDs and records mapping any failed `ManifestId` to its failure state:
  - `removedApps`: A sequence (`sequence<ManifestId>`) of successfully removed `ManifestId` values.
  - `failedApps`: A record mapping the input `ManifestId` to a `DOMException` explaining why that individual sub-app failed to remove. Possible exceptions include:
    - `NotFoundError`: No sub-app with the given ID is installed under this parent IWA, it belongs to a different parent IWA, or it was already removed.
    - `OperationError`: A generic deletion failure occurred for this specific sub-app.

#### `list()`
- **Asynchronous Rejections**: Returns a Promise. There are no synchronous exceptions. If the operation fails, the Promise rejects with a `DOMException`:
  - `SecurityError`: If the `sub-apps` permissions policy is missing or not allowed.
  - `NotSupportedError`: If the API is called inside of another sub app frame.
  - `OperationError`: A generic failure occurred.
- **Returned Object**: The function returns a record mapping `ManifestId` to `SubAppsListResult`. This mapping (instead of a sequence/list) makes it convenient for developers to use dictionary operations on the result (e.g., to directly check if it contains a specific `manifest_id`).
  - `SubAppsListResult.appName` is the name of the sub-app extracted from its web manifest.

## Examples

### 1. Installing Sub Apps
To install sub-apps, you call `window.subApps.add()` with one or more relative URLs pointing to the sub-app entry point. Each entry point must be a valid HTML page that links to a web manifest.

```js
async function installSubApps() {
  try {
    const { installedApps, failedApps } = await window.subApps.add(["/calc", "/docs"]);
    
    for (const [installUrl, id] of Object.entries(installedApps)) {
      console.log(`Success: ${installUrl} -> ${id}`);
    }
    for (const [installUrl, exception] of Object.entries(failedApps)) {
      console.error(`Failed: ${installUrl} due to ${exception.name}`);
    }
  } catch (error) {
    console.error("Failed to execute add call:", error);
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

### 3. Removing Sub Apps
To uninstall sub-apps programmatically, call `window.subApps.remove()` with the list of target manifest IDs:

```js
async function uninstallSubApps(calculatorId, docsId) {
  try {
    const { removedApps, failedApps } = await window.subApps.remove([calculatorId, docsId]);
    
    for (const id of removedApps) {
      console.log(`Successfully removed Sub App: ${id}`);
    }
    for (const [id, exception] of Object.entries(failedApps)) {
      console.error(`Failed: ${id} due to ${exception.name}`);
    }
  } catch (error) {
    console.error("Failed to execute remove call:", error);
  }
}
```

## Resource Provisioning & Installation

Sub app resources can be provided in two ways: Static and Dynamic.

1. **Static Provisioning**: The parent IWA packages all resources into itself. The minimum requirement is the start HTML file and the web manifest it references. In this case to update the sub app, the parent IWA must be updated first.
2. **Dynamic Provisioning**: The IWA hosts dynamically generated resources and a web manifest via a Service Worker. The app provides a `start_url` that the Service Worker intercepts. This approach eliminates the need to update the IWA for sub app updates. Even though the sub app is dynamic, it can be implemented in a way that is sufficiently auditable from the parent IWA bundle, because IWA itself cannot execute JS that was not provided in the bundle in the first place.

### Example of Dynamic Sub App

The full example shows how to dynamically install a sub app using a Service Worker.

```ts
// ServiceWorker.js script
declare var self: ServiceWorkerGlobalScope;

self.addEventListener("fetch", (event) => {
    const url = new URL(event.request.url);

    const htmlMatch = url.pathname.match(/^\/dynamic\/(.+)\/app\.html$/);
    const manifestMatch = url.pathname.match(/^\/dynamic\/(.+)\/app\.webmanifest$/);

    if (htmlMatch) {
        const appName = htmlMatch[1];
        const html = appHtml(appName);
        event.respondWith(localResponse("text/html; charset=utf-8", html));
    } else if (manifestMatch) {
        const appName = manifestMatch[1];
        const manifest = appManifest(appName);
        const json = JSON.stringify(manifest);
        event.respondWith(localResponse("application/manifest+json", json));
    }
});


const appHtml = (appName: string) => `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="manifest" href="/dynamic/${appName}/app.webmanifest" />
    <title>Dynamic sub app</title>
  </head>
  <body>
    <h1>Dynamic sub app</h1>
    <p>App: <b>${capitalize(appName)}</b></p>
  </body>
</html>
`;

const appManifest = (name: string) => ({
    "name": capitalize(name),
    "start_url": `/dynamic/${name}/app.html`,
    "version": "0.0.0",
    "display": "standalone",
    "icons": [
        {
            "src": appIcon(name),
            "type": "image/png",
            "sizes": "512x512",
            "purpose": "any maskable"
        }
    ]
});

const appIcon = (appName: string) => "data:image/jpeg;base64,..."// Base64 encoded icon string
```

## Detailed Design Discussion

### Install URL vs. Manifest String
Two distinct options were considered for defining sub-apps:
- **Install URL (Chosen)**: Pass the `install_url` of the sub-app, which references a standard hosted/provided web manifest.
- **Manifest String**: Feed a JSON manifest object directly as an argument to `window.subApps.add()`.

The **Install URL** approach was chosen because it aligns natively with standard web-manifest parsing, validation. Passing raw manifest objects would require introducing custom ID generators, complex update/refresh mechanisms, and completely bypassing standard manifest security audits, creating a high maintenance and security overhead.

### Returning a promise vs sequence of promises
Two distinct options were considered for the return types of the `add()` and `remove()` methods:
- **`Promise<_Response>` (Chosen)**: Returns a single promise that resolves once the batch operation concludes. The resulting response dictionary maps individual items to either successful confirmation identifiers or detailed exception records in the event of partial failures.
- **`sequence<Promise>`**: Returns an array of promises where each promise corresponds to a single sub-app operation. While this approach allows individual calls to resolve slightly earlier if one completes faster than the others, sub-app installations and removals are generally extremely fast, rarely involving network latency or intensive computation.

**`Promise<_Response>`** was chosen primarily for superior API ergonomics and simplicity. Returning a sequence of promises would place an unnecessary burden on developers, requiring them to manage and resolve multiple concurrent promises via loops.

### Sub App Identity

Sub apps rely on standard web manifest identity fields, but enforce specific isolation and overlap rules to guarantee they do not conflict with each other or the parent application.

| Identifier | Relationship to Parent / Generation Rule |
| :--- | :--- |
| **Origin** | Identical to the parent IWA. |
| **SignedWebBundleId** | Identical to the parent IWA. |
| **Start URL** | Must be defined in the web manifest. Must be unique between sub-apps and the parent app. |
| **Scope** | Might be defined in the web manifest. Scopes must not overlap between sub-apps. The scope of a sub-app must not overlap with or cover the parent app's scope. Otherwise, the installation/update fails. |
| **URL** | Parent IWA origin + `/` + `sub_app_start_url`. *Example:* `isolated-app://pl2ctdpnkf7ltse22mpjdb376etd3ydo7s72lgspuopgzcwl5tkqaaic/sub/calculator.html` |
| **AppId** | It is unique between all apps of a particular user (regardless if it is a sub app or usual app). It is derived from ManifestId. |
| **ManifestId** | Unique for each sub app. Defined in the web manifest, if it is empty, it falls back to the `start_url` without the reference/hash fragment. *Example:* `/sub/calculator.html` |

### Internal Manifest Data


`protocol_handlers`, `file_handlers`, and `launch_handler` function individually for each sub app.
Standard web resources (HTML, JS, CSS, images) apply normally.

### Web manifest

Sub apps utilize standard web app manifest properties ([W3C App Manifest](https://www.w3.org/TR/appmanifest/)).

Notable web manifest properties are OS Integrations:
- **Protocol handlers**: Allow a sub app to capture links with custom protocols like `+web_app_protocol://` ([W3C Protocol Handlers](https://www.w3.org/TR/appmanifest/#protocol_handlers-member)).
- **File handlers**: Open specific types of files like `.docx` via a sub app ([MDN File Handlers](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest/Reference/file_handlers)).
- **Launch handler**: Specify launch behavior on app click, whether new window of app will be opened or existing one reused and brought to focus ([Web App Launch](https://wicg.github.io/web-app-launch/)).
- **Scope extensions**: Allow a sub app to capture `https://` links that are associated with developer origin websites ([Scope Extensions](https://wicg.github.io/manifest-incubations/#scope_extensions-member)).

### Quota & Limits
To protect the host operating system and the user's application launcher from potential exhaustion or abuse, the platform enforces a hard limit of **20 installed sub-apps** per parent IWA.
- **Why a total limit**: Malicious or runaway applications could easily flood the user's operating system with hundreds of sub-apps. Enforcing a per-call or per-prompt limit would fail to prevent this, as developers could simply trigger successive permission prompts.
- **Quota Rejection**: If a batch installation call exceeds the platform limit, the entire `add()` call rejects with a `QuotaExceededError` DOMException.

## Considered Alternatives

### Direct window creation with custom icons (`chrome.app.window.create`)
The legacy Chrome Apps platform allowed applications to open new windows with bespoke titles and window icons. While simple, this approach has severe drawbacks:
- **No OS Integration**: It operates only at the visual level and does not expose full OS capability integrations such as launcher searchability, shelf pinning, custom protocol handlers, or file-type associations.
- **Inconsistent Launcher UX**: Windows opened this way do not behave as individual independent apps in the system app manager or application list, frustrating users who expect standard operating system behaviors.

## References & acknowledgements

This specification builds heavily on the PWA manifest update process and standard web platform patterns.

Many thanks to the following individuals for their advice, feedback, and review:
- Dominic Farolino
- Andrew Rayskiy
- Edman Anjos
- Dominik Bylica 
- Dan Murphy
