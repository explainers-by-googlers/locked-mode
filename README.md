# Explainer for Locked Mode API

This proposal is an early design sketch by ChromeOS Web Apps APIs to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- ChromeOS Web Apps APIs

## Participate

- https://github.com/explainers-by-googlers/locked-mode/issues


## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->



<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

In the education sector, there is a demand for applications to be able to serve students a low-stakes test, where the whole operating system is put in a restricted environment, preventing the student from using any other apps or OS features outside of the test itself. It is not currently possible for a web app to trigger this mode.

We’d like to propose a new Web feature: Locked Mode API where a site or app prompts the user to enter a locked-down fullscreen mode, and the user cannot exit this mode without notifying the site or app.

High-stakes testing is a non-goal to the API as it cannot stay secure against all adversaries, and it is up to each browser to implement their mitigations against exploits and abuses should they wish to pursue such use cases.


## Description of the API

The use cases for this API include:



*   A teacher may want students to be restricted from accessing other apps during a low-stakes assessment, e.g. in-class quiz. They can configure the assessment to enter locked mode, preventing the student from accessing other apps for its duration.

This API is most useful in managed environments[^1] (i.e. the device belongs to a school or another organization) where abuse is more preventable and the integrity of the API can be guaranteed more easily, e.g. by disabling Extensions, DevTools, bookmarklets and executing JavaScript in the URL bar. Unmanaged environments are out of scope of this explainer, but it’s important to note that this does not mean the API cannot work in unmanaged environments: it merely means it’s up to the individual implementation to decide whether or not to support unmanaged mode.

```JavaScript
// Used by sites/apps to make a request to enter Locked Mode for a given window/tab.
//
// Returns a Promise<void> that:
// * Rejects on failure.
// * Otherwise, enters Lock Mode and reloads the page (to prevent tampering).
//
// In the reloaded page, the site/app could query Locked Mode state
// using `navigator.inLockedMode`.
nagivator.requestLockedMode()

// Exit Locked Mode for a given window/tab.
//
// Returns a Promise<void> that:
//  * Resolves upon exiting locked mode successfully.
//  * Rejects on failure.
nagivator.exitLockedMode()

// Boolean attribute indicating whether the window is in Locked Mode.
nagivator.inLockedMode
```

When the API is called, the browser must:



*   Decide whether Locked Mode is supported, then ask for user confirmation before entering Locked Mode.
*   Lock the test site/app in fullscreen such that the user cannot switch to any other tabs.
*   Prevent the user from opening a new tab/window while staying inside Locked Mode.
*   Provide the user with a way to exit the mode (similar to the [hold-to-exit UI](https://developer.mozilla.org/en-US/docs/Web/API/Keyboard/lock) for the keyboard lock API), unless management policy specifically disables this feature.
*   Provide a way for the site/app to get notified if Locked Mode is forcefully exited, e.g. via an event listener (see section below).

Additionally, the browser should take other measures to prevent other parts of the system from interfering with the test environment, for example:



*   Discard other tabs (i.e. evict their state from memory so they can’t communicate/play audio in the background) and disable non-allowed extensions.
*   Shut down any running service workers for other origins and prevent them from starting up again for the duration of Locked Mode.
*   Prevent tampering against the test site/app by reloading after entering Locked Mode, e.g. via DevTools, extensions, bookmarklets and URL bar JavaScript execution.

Furthermore, the browser should communicate to the operating system to enter what it considers "Locked Mode". What "Locked mode" means will vary per operating system, but in general it should:



*   Lock the test site/app fullscreen such that the user cannot switch to any other windows or access any unauthorized desktop elements.
*   Disable all non-allowed keyboard shortcuts.
*   Close other running applications, preventing them from communicating/playing audio in the background.
*   Clear the clipboard buffer upon entering/exiting the Locked Mode.


#### Event listener

The API must provide a way for the site/app to get notified if Locked Mode is forcefully exited. We propose adding a `Navigator: lockedmodechange` event, that fires whenever Locked mode is entered or exited. It is worth noting that there are situations where it is impossible to detect forced exits by the user, e.g. when they power cycle the device. This event should fire on detecting locked mode changes on a best-effort basis, e.g. when the API is called or when the user exits via the provided UI, and the fact that this event hasn’t been fired alone should not serve as a guarantee that the test session hasn’t been interrupted.


### Threat model

This API has unusual security properties because in addition to the primary security concern of protecting the user from the remote site, Locked Mode also has to guard against abuse by the user (the student) against the test site/app. Here we enumerate various ways that this could occur:



1. The user infiltrates data to the test app (e.g. by preparing notes on the clipboard then pasting them in; having a call in the background that speaks the answers).
2. The user exfiltrates data from the test app (e.g. by copying quiz content to the clipboard before exiting Locked Mode; sharing contents of the screen over the internet via screen capture)
3. The user escapes the mode during the test without alerting the test app (e.g. by tampering with event handlers and calling `exitLockedMode()` using DevTools/extensions/bookmarklets/URL bar JavaScript execution; dumping contents of the RAM and then rebooting to an unrestricted operating environment before restoring RAM to its previous state when the device was in Locked Mode).
4. The user tries to convince the app that it is in locked mode when in fact it is not (e.g. by overwriting the `requestLockedMode()` method with their own that simply returns true at runtime).

While the proposed API design attempts to mitigate some of the aforementioned attacks (e.g. prevent pasting in notes / pasting out quiz content by clearing the clipboard, prevent unauthorized communications by closing background apps), it is important to note that it is very difficult, if not impossible, to mitigate some of these attacks (e.g. dumping and restoring RAM contents, tampering with the API at a system level or at runtime), especially from the standpoint of the API alone. Considerations against these types of attacks are out of scope of this API, but generally, for the API to work as intended it is a good idea to make sure:



1. the API implementation itself in the system is correct and complete,
2. the user is not able to tamper with the API at runtime, e.g. by executing their own JavaScript code or installing extensions in the context of the test site/app.


#### Considerations for developers

To ensure the API works as intended, the developers / test providers could consider the following:



*   Keep track of the number of times the student has launched the quiz.
*   Prevent users from executing their own JavaScript code (e.g. via DevTools, bookmarklets, URL bar) or installing extensions in the context of the test site/app or installing extensions.


### Alternatives considered


#### Implementing the [Smarter Balanced Assessment Consortium](http://www.smarterbalanced.org/) (SBAC)’s Secure Browser [specifications](https://www.smarterapp.org/documents/SecureBrowserRequirementsSpecifications_0-3.pdf)

Pros:



*   Specification implemented by Microsoft and other providers, e.g. Cambium, Pearson, etc, in their customized testing browsers.
*   Would allow existing test apps designed for those browsers to work without modification in a browser implementing this standard.

Cons:



*   The API uses the `browser` namespace which is inconsistent with the rest of the web platform. In addition, the relevant methods are in the `browser.security` namespace which would be inappropriate for a W3C standard.
*   Defines a wide variety of (albeit optional) methods that have security and privacy characteristics that might not be appropriate for the Web in general e.g. e.g. `getProcessList()`, `getIPAddressList()`.
*   We could consider standardizing a subset of SBAC specs / aligning method call names to it (e.g. `enableLockDown(boolean)`), but the benefits of doing so are not sufficient to outweigh the fact it wouldn’t provide an optimal shape to our proposed API. It is also possible to create a polyfill to provide compatibility to existing test sites/apps instead.


#### Having a low-level "just lock the app to fullscreen" API and separate APIs to lock out other functionalities

Pros:



*   In line with the [Extensible Web Manifesto](https://extensiblewebmanifesto.org/) which recommends designing low level APIs.

Cons:



*   Increases development costs significantly compared to existing platform solutions by requiring test app developers to call the correct combination of APIs in the correct order.
*   Operating systems don’t necessarily expose the individual low level capabilities required to implement Locked Mode this way, e.g. disabling Siri/Cortana/Google Assistant, while both Windows and macOS expose a high level API to enter a locked mode.
*   Places responsibility for plugging every hole onto the app developer, rather than the browser/OS manufacturer. Since every OS would have different features that need to be disabled, this would require the app developer to consider the individual features of every OS, and so would not be able to guarantee the lockdown of the OS features in a portable way.


#### Extending the <code>[requestFullScreen](https://developer.mozilla.org/en-US/docs/Web/API/Element/requestFullscreen)</code> API

Pros:



*   We are extending an existing Web standard.

Cons:



*   Does not allow developers to detect the Locked Mode feature by checking the existence of a separate API.
*   Locked Mode is a much more involved feature than fullscreen mode because it affects the whole session. They also serve very different purposes, and thus it makes more sense to separate them out.
*   Allows requesting Locked Mode for any HTML element, which is not currently a desired feature as we only want to be able to put the whole document into Locked Mode; adds complexity to design considerations, e.g. how do we handle putting a radio button into Locked Mode (as it isn’t expected to be a valid use case of the API)?


#### Extending the <code>[Document: fullscreenchange](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreenchange_event)</code> event so that it fires whenever Locked Mode is entered or exited

Pros:



*   We are extending an existing Web standard.

Cons:



*   fullscreenchange is fired on `window.navigator.document` whereas the proposed API is fired on `window.navigator`.


### Existing platform solutions


#### ChromeOS

Today there is a “Locked Mode” feature as part of Google for Education, limited to managed ChromeOS devices and Google Forms. Forms locks down devices this way by calling a Chrome extension API[^2] through a broker extension installed on managed devices, but other sites/apps cannot take advantage of this mechanism[^3].

Alternatively, ChromeOS can be configured to boot in [kiosk mode](https://support.google.com/chrome/a/answer/3273084?hl=en#zippy=%2Coption-school-sets-up-chromebook-as-a-single-app-kiosk-running-the-exam-app:~:text=Create%20and%20deploy%20Chrome%20kiosk%20apps) where a single site or app is locked into fullscreen without any other system UI.

ChromeOS does not yet allow third-party secure testing in a regular user session, or with a user switch without requiring reboot.


#### Apple (macOS/iOS)

Apple’s assessment mode is a locked mode for macOS/iOS suitable for running secure tests and is available on both managed and unmanaged devices. Approved developers can use the [Automatic Assessment Configuration](https://developer.apple.com/documentation/automaticassessmentconfiguration) API in their apps to put the device into assessment mode.

The Automatic Assessment Configuration API is only available to applications that have its [entitlement](https://developer.apple.com/documentation/bundleresources/entitlements). Getting the entitlement requires Apple’s approval.

The Automatic Assessment Configuration prevents access to desktop elements in macOS/iOS like:



*   The Dock
*   The Application Menu Bar
*   Mission Control (task switching)
*   Notification Center
*   Spaces other than the current one
*   Other apps, except those that the developer selectively allows

Additionally, it:



*   Prevents screen recording and screen capture
*   Disables Siri (voice commands)
*   Stops media playing
*   Allows network access for only the test provider’s app
*   Disables Handoff (cross-device desktop workspace)
*   Clears the clipboard buffer when starting and stopping the session


#### Windows

Windows 10/11 provides a UWP [Take a Test](https://learn.microsoft.com/en-us/windows/uwp/apps-for-education/take-a-test-api) app on managed devices using [Secure Assessment mode](https://learn.microsoft.com/en-us/education/windows/take-tests-in-windows), and on unmanaged devices using [kiosk mode](https://learn.microsoft.com/en-us/education/windows/edu-take-a-test-kiosk-mode?tabs=win). Take a Test is a specialized web browser that navigates to a predefined test site/app. This browser implements the [Smarter Balanced Assessment Consortium](http://www.smarterbalanced.org/) (SBAC) [specifications](https://www.smarterapp.org/documents/SecureBrowserRequirementsSpecifications_0-3.pdf) that exposes a JavaScript API to allow the test site/app to lock down the computer. When locked down using Secure Assessment mode, users are unable to:



*   Print, use screen capture, or text suggestions (unless allowlisted),
*   Access other applications,
*   Change system settings, such as display extension, notifications, updates,
*   Access Cortana (voice commands),
*   Access content copied to the clipboard.


#### Other solutions

Some testing providers also distribute their own secure testing browsers. However, Take a Test app is explicitly mentioned as an equivalent substitute (e.g. [Cambium](https://guides.cambiumast.com/Supported_Browsers/California/Content/Portal_OSs/Sec_Browsers/Windows_10_Take_a_Test.htm), [Pearson [page 4]](https://tn.mypearsonsupport.com/resources/schoolnet/Secure%20Tester%20Installation%20and%20User%20Guide.pdf)).


### Implementation in browsers


#### Chrome-on-ChromeOS

For Chrome-on-ChromeOS, there is an existing extension API which locks the device to fullscreen for testing, which makes it possible for the Web API implementation to plug into the same architecture.


#### Browsers on other OSes

Similarly, browsers on other OSes can implement this API by communicating and coordinating with the operating system. Theoretically, browsers vendors could leverage administrator/sudo privileges and lower-level system APIs on Windows/macOS to lock down the device as best as they could, but it is not a guarantee that they will be able to properly implement Locked Mode without support from Microsoft/Apple, where they could provide an OS-wide API that can properly lock down the device for assessment (this API already exists in macOS as mentioned in a previous section). In practice, both Windows and macOS would need to authorize uses of this OS-wide API for browsers/apps other than their own, and this might create some friction for third party developers hoping to implement the Locked Mode API. To ease the transition, existing specialized “secure testing” browsers such as Microsoft Take-a-Test, Cambium and Pearson could expose this new W3C standard in addition to SBAC which they currently use.


<!-- Footnotes themselves at the bottom. -->
## Notes

[^1]:
     Per [minutes](https://docs.google.com/document/d/1brQFdsFsLtdhcMDmnyhqkpK5TTpDkAIsKVzYvkbs8Lw/edit) for standardizing managed user agent behavior at TPAC 2023, this is something we'd like to be able to propose in W3C standards going forward.

[^2]:
     When a Chrome extension with the correct permission calls `chrome.windows.update` and requests the "locked-fullscreen" state, the <code>[SetLockedFullscreenState](https://docs.google.com/document/d/1_DJ8vwxkyICFYk1tS2yAsSSUAxM2EdXY_MKz8GphMIE/edit#heading=h.z85wi9k9fc51)</code> function is used. Currently it does not do anything on OSes other than ChromeOS.

[^3]:
     The Google Forms extension calls `chrome.windows.update` to set the window to a "locked-fullscreen" state, but the browser only allows this state to be entered from a allowlisted extension. Anybody can write an extension that calls this API, but it wouldn't do anything unless their extension had permission.
