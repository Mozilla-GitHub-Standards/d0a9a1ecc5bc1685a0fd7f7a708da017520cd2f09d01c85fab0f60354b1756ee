.. _tutorial-attachments:

Work with Attachments
=====================

Goals
-----

* Publish large or binary content

Prerequisites
-------------

This guide assumes that you have already installed the following commands:

- cURL
- uuidgen

And that you are familiar with the Remote Settings API, at least on the dev server.

We'll refer the running instance as ``$SERVER`` (eg. ``https://kinto.dev.mozaws.net/v1``).


Introduction
------------

Files can be attached to records. When a record has a file attached to it, it has an ``attachment`` attribute, which contains the file related information (url, hash, size, mimetype, etc.).

The Remote Settings client API is **not in charge** of downloading the remote files.

During synchronization, only the records that changed are fetched. Depending on your implementation, attachments may have to be redownloaded completely even if only a few bytes were changed.

.. note::

    Some day we'd like to have a reusable helper for attachment. Tracking happens in `Bug 1501214 <https://bugzilla.mozilla.org/show_bug.cgi?id=1501214>`_.


Publish records with attachments
--------------------------------

Files can be attached to existing records or records can be created when uploading the attachment.

Suppose that we want to attach a file (``/home/mathieu/DAFSA.bin``) to the existing record ``bec3e95c-4d28-40c1-b486-76682962861f``:

.. code-block:: bash

    BUCKET=main-workspace # (or just ``main`` in Dev)
    COLLECTION=public-suffix-list
    RECORD=bec3e95c-4d28-40c1-b486-76682962861f
    FILEPATH=/home/mathieu/DAFSA.bin

    curl -X POST ${SERVER}/buckets/${BUCKET}/collections/${COLLECTION}/records/${RECORD}/attachment \
         -H 'Content-Type:multipart/form-data' \
         -F attachment=@$FILEPATH \
         -u user:pass

And in order to create a record with both attributes and attachment, you'll have a generate a record id yourself.

.. code-block:: bash

    RECORD=`uuidgen`

    curl -X POST ${SERVER}/buckets/${BUCKET}/collections/${COLLECTION}/records/${RECORD}/attachment \
         -H 'Content-Type:multipart/form-data' \
         -F attachment=@$FILEPATH \
         -F 'data={"name": "Mac Fly", "age": 42}' \
         -u user:pass

.. note::

    Since the dev server is open to anyone and runs on ``.mozaws.net``, certain types of files are not allowed (eg. ``.js``, ``.html``, ...)


Synchronize attachments
-----------------------

The location attribute in records only contains a relative path. In order to build the full URL, some metadata has to be obtained from the server root URL.

.. code-block:: bash

    curl -s {$SERVER}/ | jq .capabilities.attachments.base_url

Using JavaScript, a **naive** implementation to download records attachments can look like this:

.. code-block:: JavaScript

    ChromeUtils.import("resource://gre/modules/XPCOMUtils.jsm");
    ChromeUtils.defineModuleGetter(this, "OS", "resource://gre/modules/osfile.jsm");
    XPCOMUtils.defineLazyGlobalGetters(this, ["fetch"]);

    XPCOMUtils.defineLazyGetter(this, "baseAttachmentsURL", async () => {
      const server = Services.prefs.getCharPref("services.settings.server");
      const serverInfo = await (await fetch(`${server}/`)).json();
      const { capabilities: { attachments: { base_url } } } = serverInfo;
      return base_url;
    });

    RemoteSettings("public-suffix-list").on("sync", async (event) => {
      const {
        data: { created, updated, deleted }
      } = event;

      // Remove every removed attachment.
      const toRemove = deleted.concat(updated.map(u => u.old));
      await Promise.all(
        toRemove.map(async record => {
          const { attachment: { location, filename } } = record;

          const path = OS.Path.join(OS.Constants.Path.profileDir, filename);
          return OS.File.remove(path, { ignoreAbsent: true });
        })
      );

      // Download every new/updated attachment.
      const toDownload = created.concat(updated.map(u => u.new));
      await Promise.all(
        toDownload.map(async record => {
          const { attachment: { location, filename } } = record;

          const resp = await fetch((await baseAttachmentsURL) + location);
          const buffer = await resp.arrayBuffer();
          const bytes = new Uint8Array(buffer);

          const path = OS.Path.join(OS.Constants.Path.profileDir, filename);
          return OS.File.writeAtomic(path, bytes, { tmpPath: path + ".tmp" });
        })
      );
    });

.. important::

    Downloading attachments can introduce complexity, particularly:

    - check available disk space
    - preserve bandwidth
    - resume failed downloads
    - verify integrity (md5sum) regularly
    - redownload corrupt files


About compression
-----------------

The server does not compress the files.

We plan to enable compression at the HTTP level (`Bug 1339114 <https://bugzilla.mozilla.org/show_bug.cgi?id=1339114>`_) for when clients fetch the attachment using the ``Accept-Encoding: gzip`` request header.

In the admin tool
-----------------

The Remote Settings administration tool supports attachments as well. If a collection has a record schema and attachments are "enabled" for that collection, then editors will be able to upload attachments as part of editing records.

The controls for attachments in a given collection are in the ``attachment`` field in the collection metadata (probably located in the `remote-settings-permissions <https://github.com/mozilla-services/remote-settings-permissions>`_ repo). The ``attachment`` attribute should be an object and it can have the following properties:

- ``enabled``: boolean, true to enable attachments for this collection
- ``required``: boolean, true if records in this collection must have an attachment
