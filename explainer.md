# Explainer

Authors: 
* Current:
  * Evan Stade &lt;<estade@chromium.org>&gt;
* Former:
  * Darwin Huang &lt;<huangdarwin@chromium.org>&gt;
  * Eric Willigers &lt;<ericwilligers@chromium.org>&gt;<br>
  * Jay Harris &lt;<harrisjay@chromium.org>&gt;<br>
  * Raymes Khoury &lt;<raymes@chromium.org>&gt;

## Motivation

This proposal gives installed web applications a way to register their ability to handle (read, stream, edit) files with given MIME types and/or file extensions. This then allows an operating system's file manager or other operating system flows to use a PWA to handle opening the file, similar to how it would open the file using a native app.

This has many use cases. For example:
* A document editor could display and edit a number of document formats, like `.txt`, `.md`, `.csv`, and `.docx`.
* A movie player could play a video format.
* A Javascript development environment could interactively step through Javascript programs.
* A data visualizer could generate graphs from `.csv` files.
* A creative tool could generate files in a proprietary format and become the default handler for such files (opening when a user double clicks on `.foo` in the system file browser).

There has historically been no standards-track API for MIME type handling. For some years, [Chrome packaged apps](https://developer.chrome.com/docs/extensions/apps/) have been able to register one or more file handlers using a [Chrome-specific API](https://developer.chrome.com/apps/manifest/file_handlers) where each handler may handle specific MIME types and/or file extensions. As of August 2018, 619 file handlers handled MIME types, while 509 file handlers handled file extensions. Some packaged apps had more than one file handler. Overall, 580 packaged apps handled MIME types and 337 packaged apps handled file extensions. This usage, and the use cases above, demonstrates the value of being able to associate web applications with MIME types and/or file extensions.

This will be implemented by a new `"file_handlers"` manifest entry, which specifies file handlers to register onto the target system, as well as a new `window.launchQueue` method, that enables the opened page to handle files queued by the underlying operating system.

## Example Manifest

The following web application declares in its manifest that it can handle CSV and SVG files, as well as a hypothetical file format that this application uses called GRAF.

```json
    {
      "name": "Grafr",
      "file_handlers": [
        {
          "action": "/open-csv",
          "name": "Comma-separated Value",
          "accept": {
            "text/csv": [ ".csv" ]
          },
          "icons": [
            {
              "src": "/csv-file.png",
              "sizes": "144x144"
            }
          ]
        },
        {
          "action": "/open-svg",
          "name": "Image",
          "accept": {
            "image/svg+xml": ".svg"
          },
          "icons": [
            {
              "src": "/svg-file.png",
              "sizes": "144x144"
            }
          ]
        },
        {
          "action": "/open-graf",
          "name": "Grafr",
          "accept": {
            "application/vnd.grafr.graph": [
              ".grafr", ".graf"
            ],
            "application/vnd.alternative-graph-app.graph": ".graph"
          },
          "launch_type": "multiple-clients"
        }
      ]
    }
```
## Manifest

The new manifest API surface is contained in the `file_handlers` list, where each entry in this list is a dictionary describing the file handler, with fields like `action`, `name`, `accept`, and `icons`.

`action` must be inside the app scope, and specifies the URL after the origin that is the navigation destination for file handling launches (analogous to `start_url` for typical app launches). `launch_type` defines whether multiple files should be opened in a single client or multiple. If omitted, this defaults to `single-client`. See also `launchParams.files`.

Each `accept` entry is a dictionary mapping MIME types to extensions. Depending on context, both MIME type and file extension may be important. For example, `xdg-utils` on Linux [associates files to apps via MIME types](https://stackoverflow.com/a/31836/3260044), and thus (a) will match files to apps based on MIME type, and (b) requires all file extensions to map to a known MIME type (custom file formats like the `.grafr` example need a MIME type). Other systems map directly from file extension to app. For consistency across platforms, the user agent will enforce that the app can only open files with specified extensions. App authors should expect that both the MIME type *and* file extension must match for a file to be openable by an app.

`name` and `icons` are a string and [`ImageResource`](https://www.w3.org/TR/image-resource/) array, respectively, which describe the file type. These *may* be used to describe the files in file browsers and other contexts, depending on the operating system and other factors such as whether the PWA is the default handler for the file type.

## Launch

Choosing to open the file would create a new top level browsing context, navigating to the `action` URL (resolved against the manifest URL). Assuming the user opened `graph.csv` in Grafr the URL would be `https://grafr.com/open-csv/`.

To access launched files, a site should specify a consumer for a `launchQueue` object attached to `window`. See: [`launch_handler`](https://github.com/WICG/sw-launch/blob/main/launch_handler.md). The application will have read/write access through the [File System Access](https://github.com/WICG/file-system-access/blob/master/EXPLAINER.md) API.

Below is a basic example of using `launchQueue` and `launchParams` to receive the file handles.
```js
// In grafr.com/open-csv
if ('launchQueue' in window) {
  launchQueue.setConsumer(launchParams => {
    if (!launchParams.files.length)
      return;

    const fileHandle = launchParams.files[0];
    // Handle the file:
    // https://github.com/WICG/file-system-access/blob/master/EXPLAINER.md#example-code
  });
}
```
Note that if the user opens multiple files, and the file handler has been notated with `multiple-clients` as its `launch_type`, the files array will always have just one element. This can be useful, for example, if an editing app wishes to make use of the browser's tabbing via `"display_mode": "browser"` rather building its own way of simultaneously displaying or opening multiple documents. On the other hand, a zip archiving app would likely prefer the files all be passed to a single client, so would not set `launch_type`. Note that this applies separately to each `file_handlers` entry. It is not possible to coalesce multiple files that match different `file_handler` entries into a single launch.

# Security and Privacy Considerations

There is a large category of attack vectors that are opened up by allowing websites access to files. These are dealt with in the [file-system-access](https://github.com/WICG/file-system-access/blob/main/EXPLAINER.md#proposed-security-models) explainer.

The additional security-pertinent capability that this specification provides over the file-system-access API is the ability to grant access to certain files through the operating system UI, as opposed to through a file picker shown by a web application. Any restrictions as to the files and folders that can be opened via the picker may also be applied to the files and folders opened via the operating system.

As with native apps, an app's ability to handle a specified file type will be registered with the OS at install time. However, as an additional protection to safeguard against accidental sharing of sensitive information contained in local files, the user agent may initially verify that the action is intentional. As a run-time confirmation, this step follows a flow similar to most web Permissions. The verification prompt should appear before the PWA is launched and the launch should be aborted if the user denies the action. User agents may retain control over default settings and how (or if) they are exposed to the user.

See the [File Handling Security Model](https://docs.google.com/document/d/1pTTO5MTSlxuqxpWL3pFblKB8y8SR0jPao8uAjJSUTp4) for more discussion of the threat model. Additional details are also available in the [Security and Privacy Questionnaire](https://github.com/WICG/file-handling/blob/main/PRIVACY_AND_SECURITY.md).

# Addendum

## Differences to Similar APIs on the Web

The proposed method of getting launched files differs from some similar web APIs:

* Web Share Target: All relevant data is contained in the POST request to the page.
* registerProtocolHandler: Relevant data is contained in the query string the page navigates to.

In contrast, when we perform a navigation to the file-handling URL, the files are not available as part of a request. Since many apps that make use of the File Handling API will also make use of the [File System Access API](https://github.com/WICG/file-system-access/blob/main/EXPLAINER.md), reusing the `FileSystemFileHandle` part of that API should enable greater code reuse.

We briefly considered encoding the `FileSystemFileHandles` in the query string in a blob-like format (e.g. `file-handle://<GUIDish>`). However, this presents some problems:
* The page would have to parse file handles from the URL itself, adding boilerplate.
* Lifetimes for blob-like URLs are complicated, as we can't predict when/where the handle will be used.

In addition, were we designing the existing APIs again today, there is a good chance we might take this approach for them too.

## Previous Solutions
There are a few similar, non-standard APIs, which it may be useful to compare this with.

### [registerContentHandler](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/registerContentHandler)

Register content handler was [deprecated](https://blog.chromium.org/2016/08/from-chrome-apps-to-web.html) due to the lack of two interoperable implementations and the implementation that was available [did not conform to that standard](https://github.com/whatwg/html/commit/b143dbc2d16f3473fcadee377d838070718549d3). This API was only available in Firefox.

Example usage, as per MDN
```js
navigator.registerContentHandler(
    "application/vnd.mozilla.maybe.feed",
    "http://www.example.tld/?foo=%s",
    "My Feed Reader"
);
```

Presumably this API provided readonly access to the file.

#### Why not resurrect this now more vendors are interested?
`registerContentHandler` was not designed to handle files being opened, rather it was designed to handle link clicks that resolve to certain mime types (see [Example 17](https://www.w3.org/TR/html52/webappapis.html#custom-scheme-and-content-handlers-the-registerprotocolhandler-and-registercontenthandler-methods) from the spec).

For Example:
1. It requires file handles to be encodable as a string (this will not be possible with the [file-system-access](https://github.com/WICG/file-system-access/blob/main/EXPLAINER.md) API, which is required to write to files).
2. It can only register handlers for content types. This is a problem because some operating systems only support file extensions and we don't want to spec and maintain a mapping. See this [issue](https://github.com/WICG/web-share-target/issues/74) for more context.

### [Chrome Apps File Handlers](https://developer.chrome.com/apps/manifest/file_handlers)

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

  // File handling integration here.
  // launchData.items[0] is the first file.
}
```

### [WinJS File Handlers](https://msdn.microsoft.com/en-us/windows/desktop/hh452684)

The WinJS API is similar to that of Chrome Apps, except that the registration was done in XAML and the name of the event is different. The API is intended to provide file handling integration to UWP Web Apps, available through the Microsoft store. This API is only available in Edge, isn't accessible from the general web.

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
    // Handle file activation here. 

    // The number of files received is eventArgs.detail.files.size 
    // The first file is eventArgs.detail.files[0] 
} 
```

## References:
* [Ballista (earlier, related proposal) explainer](https://github.com/chromium/ballista/blob/master/docs/explainer.md).
* Chrome Design Documents:
  * [File Handling design document](https://docs.google.com/document/d/1SpLwK0sQ3CUuuG-T9pFBqlm1Ae-OGwi4MsP5X2bCBow/)
  * [File Handling Icons design document](https://docs.google.com/document/d/1OAkCvMwTVAf5KuHHDgAeCA3YwcTg_XmujZ7ENYq01ws).
  * [File Handling Security Model](https://docs.google.com/document/d/1pTTO5MTSlxuqxpWL3pFblKB8y8SR0jPao8uAjJSUTp4).
