## Responses to the [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for the [Sub Apps API](https://github.com/explainers-by-googlers/sub-apps)


1. **What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

    This API exposes a list of all currently installed sub-apps for the parent application, mapping each sub-app's manifest ID to its `appName` via list().
    This exposure is necessary for the parent application to list, display, and manage its sub apps. No information is exposed to third parties, as the API is restricted to the parent context.

2. **Do features in your specification expose the minimum amount of information necessary to enable their intended uses?**

    Yes. The `list()` method only returns the sub-app `ManifestId` and `appName`, which is the minimum metadata required for the parent application to list and manage them. The `add()` and `remove()` methods return granular success and failure lists keyed by the paths/IDs provided in the method call.

3. **How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**

    The API does not interpret, expose, or store any personal information, PII, or derivative information.

4. **How do the features in your specification deal with sensitive information?**

    The list of installed sub-apps is not shared globally or cross-origin; it is strictly limited to sub-apps installed by the calling parent application under the same origin. The API does not deal with sensitive user data.

5. **Does data exposed by your specification carry related but distinct information that may not be obvious to users?**

    No. The manifest data (e.g. name, icons, protocol handlers) is declared by the developer and processed by the user agent and host OS.

6. **Do the features in your specification introduce new state for an origin that persists across browsing sessions?**

    Yes. Installing a sub-app registers it with the host operating system's application launcher and the browser's registry. This state persists across browsing sessions and system restarts until the sub-app is uninstalled (either via `subApps.remove()` or by the user from the OS). However, since sub-apps are bound to the parent IWA's origin, the state is cleared or removed when the parent application itself is uninstalled.

7. **Do the features in your specification expose information about the underlying platform to origins?**

    The API exposes whether the OS application launcher registration succeeded or failed (raising `OperationError` on system/database failures). It does not expose other characteristics of the underlying OS or launcher.

8. **Does this specification allow an origin to send data to the underlying platform?**

    Yes. When calling `subApps.add()`, the parent app provides relative paths to pages that reference Web Manifests. The user agent fetches and parses these manifests, sending application metadata (such as names, icons, protocol handlers, file handlers, and launch handlers) to the OS launcher and application registry to register the sub-app. This is mitigated by strictly parsing inputs as Web Manifests according to the W3C App Manifest specification, and requiring OS-level user confirmation/defaults before integrations (e.g. protocol and file handlers) are activated.

9. **Do features in this specification enable access to device sensors?**

    No.

10. **Do features in this specification enable new script execution/loading mechanisms?**

    No.

11. **Do features in this specification allow an origin to access other devices?**

    No.

12. **Do features in this specification allow an origin some measure of control over a user agent’s native UI?**

    Yes. The parent app can programmatically install sub-apps, which adds launcher icons/shortcuts to the OS and permits the sub-app to run in its own standalone native window. This is mitigated by displaying a unified installation dialog to the user listing all requested sub-apps (names and icons) to obtain explicit user consent. The API is restricted to signature-verified IWAs to prevent spoofing, and a hard limit of 20 installed sub-apps per parent is enforced to prevent launcher spam.

13. **What temporary identifiers do the features in this specification create or expose to the web?**

    None. The API uses stable, persistent manifest identifiers (`ManifestId`) to refer to installed sub-apps.

14. **How does this specification distinguish between behavior in first-party and third-party contexts?**

    The Sub Apps API is restricted to the parent context (the primary application context). The methods reject with `NotSupportedError` if called from a sub-app document itself.

15. **How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**

    It does not. This API is intended to be used in Isolated Web Apps (which do not support either mode).

16. **Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**

    Yes. The specification includes a comprehensive, normative 'Security and Privacy Considerations' section. It details threats such as permission inheritance, identity spoofing, OS integration extension risk, and registry quota exhaustion, and defines normative requirements for user agents to mitigate them via explicit user consent prompts and isolation checks.

17. **Do features in your specification enable origins to downgrade default security protections?**

    No.

18. **How does your feature handle non-"fully active" documents?**

    There's no special handling for non-"fully active" documents.

19. **Does your spec define when and how new kinds of errors should be raised?**

    Yes. The spec defines several granular DOMExceptions:
    - `SecurityError`: Permissions policy check fails.
    - `NotSupportedError`: Called within a sub-app frame.
    - `TypeError`: Invalid relative paths provided.
    - `QuotaExceededError`: Limit of 20 sub-apps exceeded.
    - `NotAllowedError`: User declines installation consent.
    - `ConstraintError`: Manifest scopes overlap, or path points to parent manifest.
    - `DataError`: Manifest parsing/fetching fails.
    - `InvalidStateError`: Sub-app already installed.
    - `NotFoundError`: Sub-app not found during removal.
    - `OperationError`: Generic platform/system registry error.

20. **Does your feature allow sites to learn about the user’s use of assistive technology?**

    No.

21. **What should this questionnaire have asked?**

    N/A.
