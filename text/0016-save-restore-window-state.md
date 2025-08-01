# RFC 
- Start Date: 2025-03-20
- RFC PR: 
- Electron Issues: Related to [electron/electron/issues/526](https://github.com/electron/electron/issues/526)
- Reference Implementation: https://github.com/electron/electron/pull/47425/files
- Status: **Proposed**

# Save/Restore Window State API

Currently, Electron does not have any built-in mechanism for saving and restoring the state of BaseWindows, but this is a very common need for apps that want to feel more native.

## Summary

This proposal aims to implement a save/restore window state API for Electron by providing a simple but powerful configuration object `windowStatePersistence` that handles complex edge cases automatically. This approach offers a declarative way to configure window persistence behavior while maintaining flexibility for different application needs. The object would be optional in the `BaseWindowConstructorOptions`.

## Motivation

**Why should we do this?**

Window state persistence represents a core functionality that many production Electron apps implement. By elevating it to a first-class feature, Electron acknowledges its essential nature and provides a standardized approach directly in the framework.

**What use cases does it support?** 
 
It's useful for developers who want to implement save/restore window state for their apps reliably. Different apps have different needs. An example use case for this API would be for when apps want to preserve window position (with display overflow) and size across multi-monitor setups with different display scales and resolutions.

`windowStatePersistence` aims to cover most save/restore use cases while providing room for future extensions.

**What is the expected outcome?**

Electron provides an option to save and restore window state out of the box with multiple configurations which will cover almost all application needs. 

While established applications with custom implementations may continue using their existing solutions, this feature will most likely see adoption by many smaller projects. 

## Implementation Details  

Before diving into the API specifications which is the next section, I’d like to outline the approach for saving a window’s state in Electron.  

We can use Chromium's [PrefService](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/pref_service.h) to store window state as a JSON object into the already existing Local State folder inside *app.getPath('userData')*.

We schedule an async write to disk every time a window is moved or resized and batch write every 10 seconds (I will explain why later). We also write to disk on app quit synchronously. **This continuous writing approach would enable us to restore the window state even when there are crashes or improper app exits.**

**Questions that might arise:**

**Why 10 seconds?**
The default time window used by Chromium's PrefService is 10 seconds [[reference](https://source.chromium.org/chromium/chromium/src/+/main:base/files/important_file_writer.cc;l=48;drc=ff37fb5e8b02e0c123063b89ef2ac3423829b010)].
I think reusing the existing implementation and retaining the 10-second commit interval would be a simple and effective approach.

**What if the app crashes in the 10 second window?** The window could possibly restore where it was a few seconds ago (worst case 10 seconds ago). This wouldn't be the worst user experience in the world.

#### *Why PrefService?*  
- Atomic writes using ImportantFileWriter [[reference](https://source.chromium.org/chromium/chromium/src/+/main:base/files/important_file_writer.h;l=28-29)]
- Async writes which means no blocking of the UI thread, preventing jank during window state updates [[reference](https://docs.google.com/document/d/1rlwl_GvvokatMiRUUkR0vGNcraqe4weFGNilTsZgiXA/edit?tab=t.0#bookmark=id.49710km71rd7)]
- Synchronous writes on demand [[reference](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/pref_service.h;l=199-209;drc=51cc784590d00e6a95b48e1e1bf5c3fe099edf64)]
- Offers batched scheduled writes (using ImportantFileWriter) every 10 second window [[reference](https://source.chromium.org/chromium/chromium/src/+/main:base/files/important_file_writer.cc;l=48;drc=ff37fb5e8b02e0c123063b89ef2ac3423829b010)].
- Already implemented in Electron to preserve the devTools bounds [[reference](https://github.com/electron/electron/blob/73a017577e6d8cf67c76acb8f6a199c2b64ccb5d/shell/browser/ui/inspectable_web_contents.cc#L509)]
- Provides identical behavior across Windows, macOS, and Linux, simplifying implementation and maintenance
- Falls back to default values automatically when corrupted data is encountered [[reference](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/pref_service.h;l=233-235;drc=51cc784590d00e6a95b48e1e1bf5c3fe099edf64)]
- Safe failing to ensure apps don't crash on data corruption [[reference](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/json_pref_store.cc;l=91-95;drc=ccea09835fd67b49ef0d4aed8dda1a5f22a409c8)]

<br/>

Here's the schema I propose for the `windowStatePersistence` object. This is simply for reference as I have not explained what each field means yet. I will explain each field in the API specification section which is next.

<br/>

BaseWindowConstructorOptions schema - This would be passed by the developer in the BaseWindowConstructorOptions/BrowserWindowConstructorOptions

```json
{
    "name": string,
    "windowStatePersistence": {
        "bounds": boolean,
        "displayMode": boolean
    }
}
```

> [!NOTE]
> The entire window state (bounds, maximized, fullscreen, etc.) would be saved internally if name and windowStatePersistence are passed.
> We would use the saved state and enforce the chosen config during restoration the next time the window is constructed.

#### **State Persistence Mechanism**  

Firstly, all the window states with their `windowStatePersistence` would be to loaded from disk synchronously during the startup process just like other preferences in Electron. Doing so would allow us the have the states in memory during window creation with minimal performance impact. 

Secondly, once the states are loaded into memory, we can use them to restore the window state with the `windowStatePersistence` rules in place during window creation. An algorithm that handles all the edges would be required for this. Chromium's [WindowSizer::AdjustBoundsToBeVisibleOnDisplay](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/window_sizer/window_sizer.cc;drc=0ec56065ba588552f21633aa47280ba02c3cd160;l=350) seems like a good reference point that handles some of the edge cases. Also, the default options provided in the BaseWindowConstructorOptions/BrowserWindowConstructorOptions constructor options would be overridden (if we are restoring state).

We can respect the min/max height/width, fullscreenable, maximizable, minimizable properties set inside `BaseWindowConstructorOptions` if applicable. Meaning these properties would take a higher priority during the restoration of a window.

Each time a window is **moved** or **resized**, we schedule a write using Chromium's ScopedDictPrefUpdate [[example](https://github.com/electron/electron/blob/4ad20ccb396a3354d99b8843426bece2b28228bf/shell/browser/ui/inspectable_web_contents.cc#L837-L841)].

Here's a reference to all the variables we would saving be internally using PrefService https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/chrome_views_delegate.cc;drc=1f9f3d7d4227fc2021915926b0e9f934cd610201;l=85

#### Developer-Facing Events

I’m also considering emitting these events: `will-save-window-state`, `saved-window-state`, `restored-window-state`. **However, I’m unsure about their practical usefulness for developers and hope to finalize them through this RFC.**

I'm thinking we could make these events simple telemetry events. Simple telemetry events would provide observability without the performance cost of running the custom logic for hundreds of events per second during window manipulation. Adding these events with the ability to intercept and modify values that are going to be saved, ability to intercept with custom restoration logic, would increase the implementation complexity and maintainability.

The following events will be emitted for window state persistence process:

#### Event: 'will-save-window-state'

Emitted before window state is saved. This event is triggered when the window is closed, resized, moved or when `win.savePreferences()` is called.

**Parameters:**
- `event` Event - The event object with `preventDefault()` method

```js
win.on('will-save-window-state', (event) => {
  console.log('About to save window state');
  
  // Prevent the save operation if needed
  if (someCondition) {
    event.preventDefault();
  }
});
```

#### Event: 'saved-window-state'

Emitted after window state has been successfully written to disk.

```js
win.on('saved-window-state', () => {
  console.log('Window state successfully saved');
});
```

#### Event: 'restored-window-state'

Emitted immediately after the window constructor completes with the restored state applied.

```js
win.on('restored-window-state', () => {
  console.log('Window state restored');
});
```


## API Specification

Here's how it would land in the electron documentation.

### BaseWindowConstructorOptions

* `name` string (optional) - An identifier for the window that enables features such as state persistence.

* `windowStatePersistence` ([WindowStatePersistence]() | Boolean) (optional) - Configures or enables the persistence of window state (position, size, maximized state, etc.) across application restarts. Has no effect if window `name` is not provided. _Experimental_

### WindowStatePersistence Object

* `bounds` boolean (optional) - Whether to persist window position and size across application restarts. Defaults to `true` if not specified.

* `displayMode` boolean (optional) - Whether to persist display modes (fullscreen, kiosk, maximized, etc.) across application restarts. Defaults to `true` if not specified. 

We could add a tutorial on how the new constructor option can be used effectively.

```js
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  ...existingOptions, 
  windowStatePersistence: {
    displayMode: false
  },
})
```

### API Design Rationale:
- Everything is set to true by default so that the developer can set whatever they want to exclude to false. This means less lines of code as more options are added to the config.
- Displays are considered the same if they span over the same work_area dimensions.
- A window can never be restored out of reach. I am presuming no apps would want this behavior.
- If window (width, height) reopens on a different display and does not fit on screen auto adjust to fit and resave the value. This would reduce the number of edge cases significantly and I highly doubt that any app would want to preserve an overflow when opened on a different display.
- Not handling scaling as Windows, macOS, and Linux support multimonitor window scaling by default.

### Additional APIs
I also plan on adding synchronous methods to `BaseWindow` to save, restore, clear, and get the window state or rather window preferences. Down below is the API spec for that.

Here's an exhaustive list of the options that can be saved. It would provide a way to save additional properties on the window apart from the bounds itself.

>[!NOTE]
> Restoring these properties would be set in order passed by in the options object. It would be the equivalent of calling the instance methods on BrowserWindow in the same order as provided by the developer. For example, win.setAutoHideMenuBar(true).

#### `win.savePreferences([options])`

* `options` Object
  
  * `autoHideMenuBar` boolean (optional) - Save window's current autoHideMenuBar state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>Linux</kbd></span>
  
  * `focusable` boolean (optional) - Save window's current focusable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `visibleOnAllWorkspaces` boolean (optional) - Save window's current visibleOnAllWorkspaces state. Default: `false`<span>&nbsp;&nbsp;  <kbd>macOS</kbd></span>

  * `shadow` boolean (optional) - Save window's current shadow state. Default: `false`

  * `menuBarVisible` boolean (optional) - Save window's current menuBarVisible state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>Linux</kbd></span>

  * `representedFilename` boolean (optional) - Save window's current representedFilename. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `title` boolean (optional) - Save window's current title. Default: `false`

  * `minimizable` boolean (optional) - Save window's current minimizable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

   * `maximizable` boolean (optional) - Save window's current maximizable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `fullscreenable` boolean (optional) - Save window's current fullscreenable state. Default: `false`

  * `resizable` boolean (optional) - Save window's current resizable state. Default: `false`

  * `closable` boolean (optional) - Save window's current closable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `movable` boolean (optional) - Save window's current movable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `excludedFromShownWindowsMenu` boolean (optional) - Save window's current excludedFromShownWindowsMenu state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `accessibleTitle` boolean (optional) - Save window's current accessibleTitle. Default: `false`

  * `backgroundColor` boolean (optional) - Save window's current backgroundColor. Default: `false`

  * `aspectRatio` boolean (optional) - Save window's current aspectRatio. Default: `false`

  * `minimumSize` boolean (optional) - Save window's current minimumSize (width, height). Default: `false`

  * `maximumSize` boolean (optional) - Save window's current maximumSize (width, height). Default: `false`

  * `hiddenInMissionControl` boolean (optional) - Save window's current hiddenInMissionControl state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `alwaysOnTop` boolean (optional) - Save window's current alwaysOnTop state. Default: `false`

  * `skipTaskbar` boolean (optional) - Save window's current skipTaskbar state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `opacity` boolean (optional) - Save window's current opacity. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `windowButtonVisibility` boolean (optional) - Save window's current windowButtonVisibility state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `ignoreMouseEvents` boolean (optional) - Save window's current ignoreMouseEvents state. Default: `false`

  * `contentProtection` boolean (optional) - Save window's current contentProtection state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `autoHideCursor` boolean (optional) - Save window's current autoHideCursor state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `vibrancy` boolean (optional) - Save window's current vibrancy state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `backgroundMaterial` boolean (optional) - Save window's current backgroundMaterial state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd></span>

  * `windowButtonPosition` boolean (optional) - Save window's current windowButtonPosition state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `titleBarOverlay` boolean (optional) - Save window's current titleBarOverlay state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>Linux</kbd></span>

  * `zoomLevel` boolean (optional) - Save window webcontent's current zoomLevel. Default: `false`

  * `audioMuted` boolean (optional) - Save window webcontent's current audioMuted state. Default: `false`

  * `isDevToolsOpened` boolean (optional) - Save window webcontent's current isDevToolsOpened state. Default: `false`

  * `devToolsTitle` boolean (optional) - Save window webcontent's current devToolsTitle. Default: `false`

  * `ignoreMenuShortcuts` boolean (optional) - Save window webcontent's current ignoreMenuShortcuts state. Default: `false`

  * `frameRate` boolean (optional) - Save window webcontent's current frameRate. Default: `false`

  * `backgroundThrottling` boolean (optional) - Save window webcontent's current backgroundThrottling state. Default: `false`
  

Returns `boolean` - Whether the state was successfully saved. Relevant events would be emitted.

```js
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  ...existingOptions,
  name: '#1230',
  windowStatePersistence: true
});

// Save additional properties
win.savePreferences({ 
  autoHideMenuBar: true,
  focusable: true,
  visibleOnAllWorkspaces: true,
  shadow: true,
  menuBarVisible: true,
  representedFilename: true,
});
```

#### `win.restorePreferences()`

Returns `boolean` - Whether the preferences were successfully restored and applied to the window. Relevant events would be emitted.
Preferences will be restored in the order of the options object passed during savePreferences.
```js
// Restore the previously saved preferences
const success = win.restorePreferences()
console.log(success) // true if state was successfully restored
```

#### `BaseWindow.clearState([name])`

Static function over BaseWindow.

Clears the saved preferences for the window **including bounds**.
Returns `boolean` - Whether the state was successfully cleared. Returns true if name was not found.

```js
// Clear the entire saved state for the window
const success = BaseWindow.clearState('#1230');
console.log(success) // true if state was successfully cleared
```


### Algorithm for saving/restoring the window state
As mentioned before, we would take a continuous approach to saving window state if `name` and `windowStatePersistence` are passed through the constructor.

For the preferences it will be important to store the order of the properties as well when win.savePreferences is called due to possible conflicting properties. We can restore them in the same order and it will be as though the properties were set via JavaScript APIs in the exact same order.

The algorithm would be simple two step approach:

**Calculate new bounds:**
This step involves checking if the window is reopened on the same display by comparing the current work_area to the saved work_area, setting minimum height and width for the window (100, 100), ensuring proper fallback behaviour for when window is restored out of bounds or with overflow
There are some tricks we can use from [WindowSizer::AdjustBoundsToBeVisibleOnDisplay](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/window_sizer/window_sizer.cc;drc=0ec56065ba588552f21633aa47280ba02c3cd160;l=350)

**Set window state:**
Once we have the width, height, x, y, and displayModes we can set the window state. `displayMode` would take precedence over the saved bounds.

## Guide-level explanation

*Explain the feature as if it were already implemented in Electron and you were teaching it to
an Electron app developer.*

Electron is introducing `windowStatePersistence`, an optional object/boolean in the `BaseWindowConstructorOptions` and would also be available in `BrowserWindowConstructorOptions`.
It can be used to save and restore window state with multiple configurations.

`windowStatePersistence` schema - Everything is optional. Developers can choose what they want to preserve and how they want to restore it or simply set it to true. The default behaviour in this case would be `bounds` and `displayMode` both are restored.

```json
"name": string,
"windowStatePersistence": true
```
OR
```json
"name": string, 
"windowStatePersistence": {
    "bounds": boolean,
    "displayMode": boolean
  }
```

Here's an example that would let you restore bounds (x, y, width, height) but not the display mode (maximized, fullscreen etc)
```js
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  ...existingOptions, 
  name: '#1230',
  windowStatePersistence: {
    displayMode: false
  }
})
```

*Discuss how this impacts the ability to read, understand, and maintain Electron code. Will the
proposed feature make Electron code more maintainable? How difficult is the upgrade path for
existing apps?*

It would be the same code for Windows, macOS, and Linux using Chromium's PrefService. I think it would be easy to read, understand, and maintain the new Electron code.

The path to upgrade for apps would be for developers to remove their existing implementation and use this new API if they want to. If their implementation is on the JavaScript side it will always execute their restore logic after window construction which means our implementation would be overridden.


## Reference-level explanation

Covered in the [Implementation Details](#implementation-details) section.


## Drawbacks
- Writing to disk everytime the window is moved or resized in a batched 10 second window might not be necessary and better to write to disk only on window close synchronously.
- Similar outcomes could be achieved via JavaScript APIs with miniscule performance difference. The only issue being the window state is not set at the right time in the window lifecycle.

Why should we *not* do this?
- It's not a critical issue.
- Adds maintenance burden for the Electron team to support this feature long-term.

## Rationale and alternatives

*Why is this design the best in the space of possible designs?* 
  
  Overall, providing a constructor option is the best design in my opinion. It provides maximum flexibility and future-proofing for different requests in the future. It also sets the window properties at the right time in the window lifecycle. Although not perfect right now, it can be improved by the community based on different use cases quite easily. We're also saving the window state in a continuous manner so it can be restored even after crashes.

*What other designs have been considered and what is the rationale for not choosing them?*
  
  I've considered a JavaScript API that could be used. But it's not the best option since it would be setting the window state again after creation.

*What is the impact of not doing this?*
  
It's not a critical issue. Apps might vanish on rare occasions.

*If this is an API proposal, could this be done as a JavaScript module or a native Node.js add-on?*
  
The real value proposition isn't that this functionality can't be implemented in JavaScript - it absolutely can and has been attempted through various community libraries. Rather, the value lies in providing a standardized, well-maintained solution integrated directly into Electron's core. There's also a miniscule performance benefits as it would avoid extra ipc calls while restoring window state. 

## Prior art

Electron devTools persists bounds using PrefService. Implementation can be seen here [inspectable_web_contents.cc](https://github.com/electron/electron/blob/73a017577e6d8cf67c76acb8f6a199c2b64ccb5d/shell/browser/ui/inspectable_web_contents.cc#L509). 
It also seems likely Chrome uses PrefService to store their window bounds [reference](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/prefs/README.md).

From my research, I found out native applications built directly with the operating system's own tools and frameworks often get window state persistence for free (macOS, Windows).
I thought it would be inappropriate to enforce such rules on Electron apps. Thus, `windowStatePersistence` to provide flexibility and the choice to opt-in to this behavior.

## Unresolved questions
*What parts of the design do you expect to resolve through the RFC process before this gets merged?*
- Variable names and the entire API Spec.

## Future possibilities

Introduce custom behavior under `fallbackBehaviour` and `openBehaviour` parameters based on community requests.

* `openBehaviour` string (optional) - Special behavior when the window is opened.

  * `openOnLaunchedDisplay` - Opens the window on the display that it was launched from always.

  * `openOnPrimaryDisplay` - Opens the window on the primary display always.


* `fallbackBehaviour` string (optional) - Fallback behaviour when position is out of bounds.

  * `"snapToNearestDisplayEdge"` - Snaps the window to the nearest display edge.

  * `"centerOnNearestDisplay"` - Centers the window on the nearest display.

  * `"centerOnPrimaryDisplay"` - Centers the window on the primary display.


We would need an algorithm that calculates bounds based on these parameters. Many things would be needed to be taken into consideration to accomplish this.

The algorithm to restore window state with the newly introduced options `fallbackBehaviour` and `openBehaviour` is detailed [here](https://gist.github.com/nilayarya/48d24e38d8dbf67dd05eef9310f147c6#algorithm-for-savingrestoring-the-window-state). One particularly cool feature would be to provide an option to restore the window on closest available display/space dynamically.

The `bounds` property that is suggested above could possibly accept an object or boolean (as suggest as of now). The object would allow more configurability and control over reopening windows on different monitors with different dpi scaling and resolution.

We could add `allowOverflow` property inside the `bounds` object to control the restore overflow behaviour (some apps would specifically like to not restore in an overflown state). In our current implementation we won't be considering this and can have something like [this](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/window_sizer/window_sizer.cc;drc=0ec56065ba588552f21633aa47280ba02c3cd160;l=402) for the time being.

APIs to allow changes in `windowStatePersistence` during runtime for apps that want to let users decide save/restore behaviour.

APIs to allow changes to the saved window state on disk such as `BrowserWindow.getWindowState([name])` and `BrowserWindow.setWindowState([stateObj])` might be useful for cloud synchronization of window states as suggested by this comment https://github.com/electron/rfcs/pull/16#issuecomment-2983249038

Overall, the `windowStatePersistence` is very configurable so I think it's future-proof. More configurations can be added based on community requests.