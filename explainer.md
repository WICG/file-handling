## Explainer

Author: Eric Willigers &lt;<ericwilligers@chromium.org>&gt;<br>
Author: Jay Harris &lt;<harrisjay@chromium.org>&gt;<br>
Author: Raymes Khoury &lt;<raymes@chromium.org>&gt;

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


A [FileSystemFileHandle](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md) allows reading and writing the file. (An earlier proposed API for file reading and writing was [FileEntry](https://www.w3.org/TR/2012/WD-file-system-api-20120417/#the-fileentry-interface) but work on that proposal has discontinued.).

### Launch Events

The intention of the launch events discussed in this explainer is that they be built on top of the more general [sw-launch](https://github.com/WICG/sw-launch/blob/master/explainer.md) proposal, as part of a unified system for handling application launches.

### Concerns

#### Will the current application behavior be supported?
Below is a not-at-all scientific collection of how a few common apps handle files being opened. SW Launch refers to the case where we fire a launch event on the service worker, while client launch refers to a theoretical event handler on the client window.

App     | SW Launch   | Client Launch   | Description
------  | ----------- | --------------- | ---------
VS Code | Yes         | No              | VSCode opens individual files in the last active window (fine for client launch events), unless a parent directory of the file is open in as a workspace, in which case, the file will be opened in the editor for that workspace. This cannot be handled by client events without undesirable focusing of some arbitrary client.
Paint   | Yes         | Yes             | Always opens a new window.
TextEdit| Yes         | No              | Opens files in a new window, if they aren't already open, otherwise focus the window that has the file open.
Sublime | Yes         | Yes             | Configurable. Either always open in new window, or never open in new window.
Chrome  | Yes        | Yes             | Open in last active window.

It seems clear that there are at least some cases where it is useful for applications to be able to inspect already open clients and decide where they want a file to be opened. 

Particularly interesting is that in some cases, the application exposes settings for what to do when new file is opened. We briefly considered a declarative API (e.g. Paint says to always open files in a new window in its manifest). This, however, would indicate that this is unlikely to be a workable approach.

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

Chrome Apps are in the process of being [deprecated](https://arstechnica.com/gadgets/2017/12/google-shuts-down-the-apps-section-of-the-chrome-web-store/) in favour of PWAs. The API was never intended to be a web standard. This API is only available in Chrom(e|ium), and is only available to apps published in the Chrome App Store.

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
