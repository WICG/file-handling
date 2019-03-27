## Explainer

Author: Eric Willigers &lt;<ericwilligers@chromium.org>&gt;

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
  self.addEventListener('launch', event => {
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

### Launch Events

The intention of the launch events discussed in this explainer is that they be built on top of the more general [sw-launch](https://github.com/WICG/sw-launch/blob/master/explainer.md) proposal, as part of a unified system for handling application launches.

The Launch Event would have different properties depending on what caused the event. For example, the FileLaunchEvent would contain a list of the files that should be handled, while the UrlLaunchEvent might have the triggering request.

(NOTE: The below interfaces are highly speculative).

(NOTE: I don't know the correct way of doing this in WebIDL, so the below definition is in TypeScript. However, for the explainer, this gets the idea across).

```ts
// Caused when the application is launched via its shortcut.
interface IconLaunchEvent {
  cause: 'icon';
}

// Caused when the application is launched via opening a file.
interface FileLaunchEvent {
  cause: 'file';
  files: FileSystemBaseHandle[];
}

// Caused when the browser would have navigated to an in-app link.
interface UrlLaunchEvent {
  cause: 'url';
  request: FetchAPIRequest;
}

type LaunchEvent = IconLaunchEvent | FileLaunchEvent | UrlLaunchEvent;

// Example Handler
const launchEventHandler = (event: LaunchEvent) => {
  if (event.cause === 'file') {
    // Do something with files.
  }
};

```

More detail is available on the [sw-launch](https://github.com/WICG/sw-launch/blob/master/explainer.md) repository.

### Concerns

This proposal (with new service worker events) was presented to the Service Worker and Web Platform working groups at TPAC 2018. There was concern that executing more code in the service worker might affect performance, and it may not be clear to users which web application is running code, if no client window has been given focus yet.

A different API, was considered with flow similar to the following:
1. If no window exists, create a new one
2. Fire launch event on first active window

However, this API would require jumping through some hoops in common cases. Consider a document editor:
1. A user is editing document1.doc
2. User then opens document2.doc
   1. A launch event on document1.doc is fired
   2. As the editor doesn't want to lost the changes to document1, a message is posted to the service worker.
      1. The service worker checks to see if document2.doc is already open
         1. If yes, focus that window
         2. If no, create a new client, post it the document and focus the client.

Compare to having the event on the service worker, where we can cut out the first postMessage
1. A user is editing document1.doc
2. User then opens document2.doc
   1. A launch event is fired on the service worker
      1. The service worker checks to see if document2 is already open
         1. If yes, focus that window
         2. If no, create a new client, post it the document and focus the client.

Which API is better depends largely on what the more common case is likely to be:
1. Opening the file in an existing Window
   - Note: This only seems to make sense in apps that can only have one instance, otherwise the existing window the browser picks is likely to be fairly arbitrary.
2. Opening the file in a new window

Arguably, the second case is more common on current desktop operating systems. 
#### Microsoft Office
Opens each doc in a different window

#### MS Paint
Opens each image in a new window

#### Preview on OSX
Only ever has one file open at a time (arguably a special case, and not likely to work on the web)

#### Microsoft Visual Studio
Each project is opens a new instance of Visual Studio

#### VS Code
Files trigger a new tab in the last active window, folders result in a new instance


### Previous Solutions
There are a few similar, non-standard APIs, which it may be useful to compare this with.

#### [registerContentHandler](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/registerContentHandler)

Register content handler was [deprecated](https://github.com/whatwg/html/issues/630) due to the lack of two interoperable implementations and the implementation that was available [did not conform to that standard](https://github.com/whatwg/html/commit/b143dbc2d16f3473fcadee377d838070718549d3). This API was only available in Firefox.

Example usage, as per MDN
```js
navigator.registerContentHandler(
    "application/vnd.mozilla.maybe.feed",
    "http://www.example.tld/?foo=%s",
    "My Feed Reader"
);
```

Presumably this API provided readonly access to the file.

#### [Chrome Apps File Handlers](https://developer.chrome.com/apps/manifest/file_handlers)

Chrome Apps are in the process of being [deprecated](https://arstechnica.com/gadgets/2017/12/google-shuts-down-the-apps-section-of-the-chrome-web-store/) in favour of PWAs. The API was never intended to be a web standard. This API is only available in Chrom(e|ium), and is only to apps published in the Chrome App Store.

Example manifest.json
```json
{
  ...
  "file_handlers": {
    "graph": {
        "extensions": [ "svg" ],
        "include_directories": false,
        "types": [ "image/svg+xml" ],
        "verb": "open_with"
    },
    "raw": {
        "extensions": [ "csv" ],
        "include_directories": false,
        "types": [ "text/csv" ],
        "verb": "open_with"
    }
  }
}
```

This would cause a `chrome.app.runtime.onLaunched` event to be fired. This event could be handled in the background process or in a client.

Example Handler
```js
function(launchData) {
  if (launchData.source !== 'file_handler') return;

  // TODO File handling
  // launchData.items[0] is the first file.
}
```

#### [WinJS File Handlers](https://msdn.microsoft.com/en-us/windows/desktop/hh452684)

The WinJS API is surprisingly similar to that of Chrome Apps, except that the registration was done in XAML and the name of the event is different. The API is intended to provide file handling integration to UWP Web Apps, available through the Microsoft store. This API is only available in Edge, is isn't accessible from the general web.

Example Registration
```xml
<Package xmlns="http://schemas.microsoft.com/appx/2010/manifest" xmlns:m2="http://schemas.microsoft.com/appx/2013/manifest">
   <Applications>
      <Application Id="AutoLaunch.App">
         <Extensions>
            <Extension Category="windows.fileTypeAssociation">
                <FileTypeAssociation Name="alsdk">
                  <DisplayName>SDK Sample File Type</DisplayName>
                  <Logo>images\logo.png</Logo>
                  <InfoTip>SDK Sample tip </InfoTip>
                  <EditFlags OpenIsSafe="true" />
                  <SupportedFileTypes>
                     <FileType ContentType="image/jpeg">.alsdk</FileType>
                  </SupportedFileTypes>
               </FileTypeAssociation>
            </Extension>
         </Extensions>
      </Application>
   </Applications>
</Package>
```

Example Handler
```js
function onActivatedHandler(eventArgs) { 
    if (eventArgs.detail.kind !== Windows.ApplicationModel.Activation.ActivationKind.file)  
      return;
    // TODO: Handle file activation. 

    // The number of files received is eventArgs.detail.files.size 
    // The first file is eventArgs.detail.files[0].name 
} 
```