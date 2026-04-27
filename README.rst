H5P plugin for `Tutor <https://docs.tutor.edly.io>`__
######################################################

This plugin integrates `H5P <https://h5p.org>`__ interactive content into
`Open edX <https://openedx.org>`__ by installing the
`h5pxblock <https://github.com/edly-io/h5pxblock>`__ XBlock into the LMS.

Beyond the XBlock itself, the plugin handles the additional configuration
required to make H5P work correctly in Kubernetes environments where
`S3 is the storage backend <https://github.com/cleura/tutor-contrib-s3>`__:
it rewrites H5P media URLs to go through the LMS host instead of hitting the
S3 bucket directly, avoiding CORS and CSRF embedding issues that arise when
browser requests cross from the LMS domain to the S3 domain.  Both
**path-style** and **virtual-hosted (bucket-style)** S3 URL formats are
supported.


How it works
************

When S3 storage is configured, the plugin applies two changes:

1. **LMS Django settings** — ``H5PXBLOCK_STORAGE`` is configured to use
   ``S3Boto3Storage`` with ``custom_domain`` set to ``LMS_HOST``.  This
   causes the XBlock to generate media URLs that point to the LMS host
   instead of the S3 bucket.

2. **Caddy reverse proxy** — A ``/h5pxblockmedia/*`` route is added to the
   LMS Caddy configuration.  Requests that arrive at this path are rewritten
   and proxied transparently to the S3 backend.  The rewrite logic depends on
   the configured URL style:

   - **Virtual-hosted style** (default) — the bucket name is used as a
     subdomain of the S3 host
     (e.g. ``mybucket.s3.us-east-1.amazonaws.com``).
   - **Path style** — the bucket name is prepended as a path segment
     (e.g. ``s3.us-east-1.amazonaws.com/mybucket``).


Requirements
************

- `Tutor <https://docs.tutor.edly.io>`__ >= 21.0 (Sumac)
- For S3 storage: `tutor-contrib-s3 <https://github.com/cleura/tutor-contrib-s3>`__
  must be installed and properly configured before enabling this plugin.


Installation
************

.. code-block:: bash

    pip install tutor-contrib-h5p


Usage
*****

Enable the plugin and apply the configuration:

.. code-block:: bash

    tutor plugins enable h5p


Then rebuild the Open edX image and restart your environment:

.. code-block:: bash

    tutor images build openedx
    tutor local launch   # or: tutor k8s start


Configuration
*************

All settings can be changed with ``tutor config save --set H5P_<KEY>=<value>``.

.. list-table::
   :header-rows: 1
   :widths: 30 15 55

   * - Setting
     - Default
     - Description
   * - ``H5P_BUCKET``
     - ``S3_STORAGE_BUCKET``
     - S3 bucket that stores H5P media files.  Defaults to the value of
       ``S3_STORAGE_BUCKET`` set by tutor-contrib-s3.
   * - ``H5P_ENDPOINT``
     - *(empty)*
     - Custom S3-compatible endpoint hostname (e.g. ``minio.example.com``).
       When empty, the plugin falls back to ``S3_HOST`` / ``S3_PORT`` from
       tutor-contrib-s3 or constructs the standard AWS endpoint from
       ``S3_REGION``.
   * - ``H5P_URL_STYLE``
     - ``virtual``
     - S3 URL style.  Use ``virtual`` for bucket-as-subdomain
       (``bucket.host/key``) or ``path`` for bucket-as-path
       (``host/bucket/key``).
   * - ``H5P_USE_SSL``
     - ``true``
     - Whether to use HTTPS when proxying requests to S3.
   * - ``H5P_PATH``
     - *(empty)*
     - Optional key prefix inside the bucket where H5P files are stored.


S3 URL style examples
=====================

Given bucket ``mybucket``, region ``us-east-1``, and a media key
``h5pxblockmedia/course/file.h5p``:

**Virtual-hosted style** (``H5P_URL_STYLE=virtual``, default):

.. code-block:: text

    Proxy target: https://mybucket.s3.us-east-1.amazonaws.com/h5pxblockmedia/course/file.h5p

**Path style** (``H5P_URL_STYLE=path``):

.. code-block:: text

    Proxy target: https://s3.us-east-1.amazonaws.com/mybucket/h5pxblockmedia/course/file.h5p


License
*******

This software is licensed under the terms of the AGPLv3.