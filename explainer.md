## Explainer

Author: Eric Willigers &lt;<ericwilligers@chromium.org>&gt;<br>
Author: Jay Harris &lt;<harrisjay@chromium.org>&gt;

### Motivation

This proposal gives web applications a way to register their ability to handle (read, stream, edit) files with given MIME types and/or file extensions.

This has many use cases. An image editor might be able to display and edit a number of image formats. A movie player might understand a video format. A data visualizer might generate graphs from .CSV files. A Logo environment might interactively step through Logo programs.

Some more use cases and motivations are listed in the [Ballista explainer](https://github.com/chromium/ballista/blob/master/docs/explainer.md).

There has historically been no standards-track API for MIME type handling. For some years, Chrome packaged apps have been able to register one or more file handlers using a [Chrome-specific API](https://developer.chrome.com/apps/manifest/file_handlers) where each handler may handle specific MIME types and/or file extensions. This provides some insight into the demand for such functionality. As of August 2018, 619 file handlers handle MIME types, while 509 file handlers handle file extensions. Some of the packages apps have more than one file handler. Overall, 580 packaged apps handle MIME types and 337 packaged apps handle file extensions. This usage, and the use cases above, demonstrate the value of being able to associate web applications with MIME types and/or file extensions.

### Example

The following web application declares in its manifest that is handles CSV and SVG files.

```json
    {
      "name": "Grafr",
      "file_handler": [
        {
          "name": "raw",
          "accept": [".csv", "text/csv"]
        },
        {
          "name": "graph",
          "accept": [".svg", "image/svg+xml"]
        }
      ]
    }
```

Each accept entry is a sequence of MIME types and/or file extensions.

On platforms that only use file extensions to describe file types, user agents can match on the extensions ".csv" and ".svg".

On a system that does not use file extensions but associates files with MIME types, user agents can match on the "text/csv" and "image/svg+xml" MIME types. If the web application accepts all text and image formats, "text/\*" and "image/\*" could be used, i.e. "\*" may appear in place of a subtype. "\*/\*" can be used if all files are accepted.

The user can right click on CSV or SVG files in the operating system's file browser, and choose to open the files with the Grafr web application. (This option would only be presented if Grafr has been [installed](https://w3c.github.io/manifest/#installable-web-applications).)

A LaunchEvent containing a sequence<[FileSystemFileHandle](https://github.com/WICG/writable-files/blob/master/EXPLAINER.md)> is then sent to the service worker, allowing the web application to decide where to open the files (i.e. in a new or existing client).

```js
  self.addEventListener('file', event => {
    event.waitUntil(async () => {
      const allClients = await clients.matchAll();
      // If there isn't one available, open a window.
      if (allClients.length === 0) {
        const client = clients.openWindow('/');
        client.postMessage(event.files);
        client.focus();
        return;
      }

      const client = allClients[0];
      client.postMessage(event.files);
      client.focus();
    }());
  });
```

`event` has the following shape.

```webidl
  interface LaunchEvent : ExtendableEvent {
    readonly attribute DOMString name;
    sequence<FileSystemFileHandle> files;
  }
```

`name` is the name of the file handler, as defined in the web app manifest. If the user selects files files with different handlers (e.g. a CSV and SVG file, in the case of Graphr), one LaunchEvent will be fired for each handler (though a handler could receive multiple files).


A [FileSystemFileHandle](https://github.com/WICG/writable-files/blob/master/EXPLAINER.md) allows reading and writing the file. (An earlier proposed API for file reading and writing was [FileEntry](https://www.w3.org/TR/2012/WD-file-system-api-20120417/#the-fileentry-interface) but work on that proposal has discontinued.).

### Single Tab Application (context for the LaunchEvent name)

Suppose the user keeps the Grafr web application open, and now switches to the operating system's file browser and chooses more files to open in Grafr. Grafr might prefer for the new LaunchEvent, with the additional files, to be sent to the existing Grafr window.

The desire for single tab applications also arises in contexts that don't involve file handling. A general proposal for supporting single tab applications, and much more, is discussed in [SW-Launch](https://github.com/WICG/sw-launch/blob/master/explainer.md).

Suppose Grafr is using SW-Launch. Whenever the user navigates to Grafr from another web site, and Grapr is already open in another window, that window would receive the focus, and there would be a LaunchEvent with the incoming Request (url, form parameters etc.).

Similarly, if the user right clicks on CSV or SVG files in the operating system's file browser, and requests they be opened in Grafr, the existing Grapr window would receive the focus, and there would be a LaunchEvent with the new files.

Grafr need not receive two separate events: one for the open_url Request and one with the files.

### ServiceWorker Event Vs. Client Event

This proposal (with new service worker events) was presented to the Service Worker and Web Platform working groups at TPAC 2018. There was concern that executing more code in the service worker might affect performance, and it may not be clear to users which web application is running code, if no client window has been given focus yet. There is further discussion in issue #3.

The below tables compares handling launch events in the service worker and in a client.
|          | ServiceWorker                                                                                                                                                                          | Launch Event                                                                                                                               |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| The Good | - There is an existing API for managing clients<br>- Can choose what window to open file in<br>- Can handle event without having to open a client (e.g. torrent file, read file later) | - Something will always happen<br>- As fast as possible                                                                                    |
| The Bad  | - Could silently swallow a launch event<br>- Might be slow (a la fetch)<br>- Not clear which service worker should handle the event                                                    | - Have to proxy through the service worker to handle event in a different client<br>- A client has to be opened before handling can begin. |
Based on this table, it seems that service workers are a more natural place for the launch event handler to live, the main objections concerning speed and bad behavior by a handler, rather than ergonomics. Fortunately, it should be possible to mitigate these concerns through careful API design.

#### Mitigating Slowness
A small timeout could be applied to the launch event (i.e. one second), to force sites to provide meaningful feedback as quickly as possible. However, this only addresses half the concern. There are worries that going through the ServiceWorker is inherently slow, due to all the IPC calls. However, for one off launch events, it seems likely that this time will be negligible when compared to the time required to start/load a new instance of the site, and to perform I/O to open the file.

#### Providing Feedback
not-a-great-experience.com (after being installed) could register itself as a handler for a number of files, and not do anything when they are opened, essentially failing silently. We could completely remove this case by enforcing that either:

1. A notification is shown and preventDefault is called (for the case of handling the file in the background) or
2. A client is focused

If neither of these conditions are met, the user agent could focus an existing client, or instantiate a new one. This way, we ensure that a user gets some feedback about what is happening. In addition a user agent might choose to blacklist the offending application). Issue #8 is related to this.

#### Which Service Worker
An application can potentially have multiple service workers installed under its scope, which makes it difficult to determine which worker should receive the launch event. For example:

`app.com/sw-top.js`<br>
`app.com/pwa/manifest.json` ⇐ App scope, controlled by sw-top.js<br>
`app.com/pwa/sub-app/sw-sub-app.js` <br>
`app.com/pwa/sub-app/subber-app/sw-subber-app.js`

There are two solutions which are being considered for this problem:
1. Use whichever service worker manages the launch url for the app
2. Allow an `open_url` to be specified in the file handlers. The service worker that manages `open_url` will receive the launch event. This could be extended to allow specifying an open url per file handler.

Of the two, we're currently leaning towards 1), as it feel more natural (it’s also very similar to what we do in the PWA installability check). In addition, we might not know that there is a ServiceWorker available form `open_url` in 2) unless we have already visited the page. For discussion, see issue #10.

