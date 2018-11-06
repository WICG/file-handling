## Explainer

Author: Eric Willigers &lt;<ericwilligers@chromium.org>&gt;

### Motivation

This proposal gives web applications a way to register their ability to handle (read, stream, edit) files with given MIME types and/or file extensions.

This has many use cases. An image editor might be able to display and edit a number of image formats. A movie player might understand a video format. A data visualizer might generate graphs from .CSV files. A Logo environment might interactively step through Logo programs.

Some more use cases and motivations are listed in the [Ballista explainer](https://github.com/chromium/ballista/blob/master/docs/explainer.md).

There has historically been no standards-track API for MIME type handling. For some years, Chrome packaged apps have been able to register one or more file handlers using a [Chrome-specific API](https://developer.chrome.com/apps/manifest/file_handlers) where each handler may handle specific MIME types and/or file extensions. This provides some insight into the demand for such functionality. As of August 2018, 619 file handlers handle MIME types, while 509 file handlers handle file extensions. Some of the packages apps have more than one file handler. Overall, 580 packaged apps handle MIME types and 337 packaged apps handle file extensions. This usage, and the use cases above, demonstrate the value of being able to associate web applications with MIME types and/or file extensions.

### Example

The following web application declares in its manifest that is handles CSV and SVG files.

    {
      "name": "Grafr",
      "file_handling": {
        "open_url": "/file-open",
        "files": [
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
    }

Each accept entry is a sequence of MIME types and/or file extensions.

On platforms that only use file extensions to describe file types, user agents can match on the extensions ".csv" and ".svg".

On a system that does not use file extensions but associates files with MIME types, user agents can match on the "text/csv" and "image/svg+xml" MIME types. If the web application accepts all text and image formats, "text/\*" and "image/\*" could be used, i.e. "\*" may appear in place of a subtype. "\*/\*" can be used if all files are accepted.

The user can right click on CSV or SVG files in the operating system's file browser, and choose to open the files with the Grafr web application. (This option would only be presented if Grafr has been [installed](https://w3c.github.io/manifest/#installable-web-applications).)

A new top level browsing context is created, navigating to the file handling url, e.g. grafr.com/file-open

A ForegroundEvent containing a sequence<[FileSystemFileHandle](https://github.com/WICG/writable-files/blob/master/EXPLAINER.md)> is then sent to the window, allowing the web application to read and update the files.

### Obsolete Proposal

##### This proposal with new service worker events was presented to the Service Worker and Web Platform working groups at TPAC 2018. #####
##### There was concern that executing more code in the service worker might affect performance, and it may not be clear to users which web application is running code if no client window has been given the focus yet. #####

The following event is sent to a service worker when a user requests that a web application be used to open file(s).

    interface FileEvent : ExtendableEvent {
      readonly attribute DOMString name;
      sequence<FileSystemFileHandle> files;
    }

The "name" is the name of the file handler.

If the user selects two CSV files and three SVG files, and requests that Grafr open them, two FileEvent instances will be created and sent to the service worker: one for the CSV files and one for the SVG files.

Each [FileSystemFileHandle](https://github.com/WICG/writable-files/blob/master/EXPLAINER.md) allows reading and writing the file. (An earlier proposed API for file reading and writing was [FileEntry](https://www.w3.org/TR/2012/WD-file-system-api-20120417/#the-fileentry-interface) but work on that proposal has discontinued.).


The service worker should register a listener to process file events.

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
