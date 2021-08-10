# Explainer

Authors: 
* Current:
  * Darwin Huang &lt;<huangdarwin@chromium.org>&gt;
* Former:
  * Eric Willigers &lt;<ericwilligers@chromium.org>&gt;<br>
  * Jay Harris &lt;<harrisjay@chromium.org>&gt;<br>
  * Raymes Khoury &lt;<raymes@chromium.org>&gt;

## Motivation

This proposal gives installed web applications a way to register their ability to handle (read, stream, edit) files with given MIME types and/or file extensions. This then allows an operating system's file manager or other operating system flows to select a PWA to handle the file using a PWA, similar to how it would handle the file using some other associated installed app.

This has many use cases. For example:
* A document editor could display and edit a number of document formats, like `.txt`, `.md`, `.csv`, and `.docx`.
* A movie player could play a video format.
* A Javascript development environment could interactively step through Javascript programs.
* A data visualizer could generate graphs from .CSV files.

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
          }
        }
      ]
    }
```
## Manifest

The new manifest API surface is contained in the `file_handlers` list, where each entry in this list is a dictionary describing the file handler, with fields like `action`, `name`, `accept`, and `icons`.

`action` must be inside the app scope, and specifies the URL after the origin, later navigated to in the launch event when handling a file.

Each `accept` entry is a dictionary mapping MIME types to extensions. This ensures applications will work on operating systems that only support one of the two (example: Linux, which [only supports MIME types](https://stackoverflow.com/a/31836/3260044)). This also allows applications to register custom file types.

On a system that does not use file extensions but associates files with MIME types, user agents can match on the `"text/csv"`, `"image/svg+xml"`, `"application/vnd.grafr.graph"` and `"application/vnd.alternative-graph-app.graph"` MIME types. If the web application accepts all text and image formats that the browser supports, `"text/*"` and `"image/*"` could be used, i.e. `"*"` may appear in place of a subtype. `"*/*"` can be used if all files are accepted. On platforms that only use file extensions to describe file types, user agents can match on the extensions `".csv"`, `".svg"`, `".grafr"`, `".graf"` and `".graph"`.

Icons will be registered for each handler based on the optional `icons` entry, which expects [`ImageResource`](https://www.w3.org/TR/image-resource/)s to specify the images. If an icon is provided in an `icons` entry, that image will be registered in the operating system for that file type. Otherwise, the image registered will be the web application's icon as specified in the manifest.

After the user [installs](https://w3c.github.io/manifest/#installable-web-applications) Grafr, the operating system UX flows, including file managers and "Open with..." dialogs, should list Grafr as an application that may be used to open files of this file type. In this listing, the `"name"`should be listed as an option, to be associated with the registered icon corresponding to this file type. The user can then right click on CSV or SVG files in the operating system's file browser, and choose to open the files with the Grafr web application. When the user uninstalls this web application, registered file handlers and icons will be properly uninstalled as well.

## Launch

Choosing to open the file would create a new top level browsing context, navigating to the `action` URL (resolved against the manifest URL). Assuming the user opened `graph.csv` in Grafr the URL would be `https://grafr.com/open-csv/`.

To access launched files, a site should specify a consumer for a `launchQueue` object attached to `window`. Launches are queued until they are handled by this consumer, which is invoked exactly once for each launch. In this manner, we can ensure every launch is handled, regardless of when the consumer was specified. The application will have read access through the [File System Access](https://github.com/WICG/file-system-access/blob/master/EXPLAINER.md) API, but will need to separately request write access from the user to edit the file.

`launchQueue`, `launchParams`, and the DOM Window they're used on, are specified as follows:
```
// DOM Window Launch Queue.
partial interface Window {
    readonly attribute LaunchQueue launchQueue;
};

// Launch Queue.
[Exposed=Window] interface LaunchQueue {
  void setConsumer(LaunchConsumer consumer);
};

callback LaunchConsumer = any (LaunchParams params);

// Launch Params.
[Exposed=Window] interface LaunchParams {
  readonly attribute FrozenArray<FileSystemHandle> files;
};
```

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
An application could then choose to handle these files however it chose. For example, it could save the file handle to disk and create a URL to address the launched file, allowing users to navigate back to a file.

The `launchParams` property API design is discussed in more detail in the [Launch Events](https://github.com/WICG/sw-launch/blob/main/explainer.md) explainer [issue](https://github.com/WICG/sw-launch/issues/20).

For more advanced use cases, such as opening a file in an existing window or displaying a notification, applications can add a [launch event handler](https://github.com/WICG/sw-launch/blob/master/explainer.md).

## Permissions API Integration

A `"file-handling"` [permission](https://www.w3.org/TR/permissions/) will be defined to protect the user from accidentally allowing a PWA to view files. This permission prompt may asynchronously show right after the user chooses to use a PWA to open a file of any of the file types the PWA tries to associate with, so that the permission context is more understandable and relevant. This permission prompt should also show after the PWA has had an opportunity to load. The prompt should clearly show the PWA's origin, so that the identity of the PWA requesting file read access for the handled file is clear to the user. Launched files should only be queued into the `launchQueue` after the user accepts the permission and allows the PWA to handle the relevant file types. Alternatively, if the user denies the permission, the `launchQueue` consumer should still be notified of the launch but `launchParams.files` should be empty. Thus, apps can use emptiness of `launchParams.files` to infer the permission prompt was denied. If the denial is permanent (the permission status changed from `prompt` to `denied`), user agents should thereafter block file access without prompting, and further to that, make a best effort to prevent the user from launching the PWA as a file handler. User agents may retain control over default settings and how (or if) they are exposed to the user.

## Differences with Similar APIs on the Web

The proposed method of getting launched files differs from some similar web APIs:

* Web Share Target: All relevant data is contained in the POST request to the page.
* registerProtocolHandler: Relevant data is contained in the query string the page navigates to.

In contrast, when we perform a navigation to the file-handling URL, the files are not available as part of a request. We briefly considered encoding the `FileSystemFileHandles` in the query string in a blob-like format (e.g. `file-handle://<GUIDish>`). However, this presents some problems:
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

# Security and Privacy Considerations

There is a large category of attack vectors that are opened up by allowing websites access to files. These are dealt with in the [file-system-access](https://github.com/WICG/file-system-access/blob/main/EXPLAINER.md#proposed-security-models) explainer.

The additional security-pertinent capability that this specification provides over the file-system-access API is the ability to grant access to certain files through the operating system UI, as opposed to through a file picker shown by a web application. Any restrictions as to the files and folders that can be opened via the picker may also be applied to the files and folders opened via the operating system.

There is still a risk that users may unintentionally grant a web application access to a file by opening it. However, it is generally understood that opening a file allows the application it is opened with to read and/or manipulate that file. Therefore, a user's explicit choice to open a file in an installed application, such as via an “Open with...” context menu, can be read as a signal of trust in the application. In conjunction with a permission prompt, this can be viewed as sufficient trust to let the web application view the file.

More details are also available in the [Security and Privacy Questionnaire](https://github.com/WICG/file-handling/blob/main/PRIVACY_AND_SECURITY.md)

### Accidental default association

The exception to this is where there are no existing applications installed on the host operating system capable of handling a given file type. In this case, some host OSes may automatically promote the newly registered handler to become the default handler for that file type, silently and without any intervention by the user. Currently, this can occur in all desktop OSes. This would mean if the user double-clicks a file of that type, it would automatically open in the registered web app. On such host OSes, when the user agent determines that there is no existing default handler for the file type, an explicit permission prompt might be necessary, to avoid accidentally sending the contents of a file to a web application without the user's consent.

### Mitigations

The following mitigations are recommended.
* User agents should not register every site that can handle files as a file handler. Instead, registration should be gated behind installation. Users expect installed applications to be more deeply integrated with the OS. 
* Users agents should not register web applications as the default file handler for any file type without explicit user confirmation.
* A permissions prompt should be displayed before registering a web application as the default handler for a type where that registration would otherwise happen without the user's intervention (such as in the situation discussed above). Alternatively, a permissions prompt could be displayed the first time (or every time) the user opens a file with the automatically-registered default handler.

## References:
* [Ballista (earlier, related proposal) explainer](https://github.com/chromium/ballista/blob/master/docs/explainer.md).
* Chrome Design Documents:
  * [File Handling design document](https://docs.google.com/document/d/1SpLwK0sQ3CUuuG-T9pFBqlm1Ae-OGwi4MsP5X2bCBow/)
  * [File Handling Icons design document](https://docs.google.com/document/d/1OAkCvMwTVAf5KuHHDgAeCA3YwcTg_XmujZ7ENYq01ws).
  * [File Handling Security Model](https://docs.google.com/document/d/1pTTO5MTSlxuqxpWL3pFblKB8y8SR0jPao8uAjJSUTp4).
