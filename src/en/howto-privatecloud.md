[ ![Juju logo](//assets.ubuntu.com/sites/ubuntu/latest/u/img/logo.png) Juju
](https://juju.ubuntu.com/)

  - Jump to content
  - [Charms](https://juju.ubuntu.com/charms/)
  - [Features](https://juju.ubuntu.com/features/)
  - [Deploy](https://juju.ubuntu.com/deployment/)
  - [Resources](https://juju.ubuntu.com/resources/)
  - [Community](https://juju.ubuntu.com/community/)
  - [Install Juju](https://juju.ubuntu.com/download/)

Search: Search

## Juju documentation

LINKS

#  Set up a Private Cloud using Simplestreams

When Juju bootstraps a cloud, it needs two critical pieces of information:

  1. The uuid of the image to use when starting new compute instances.
  2. The URL from which to download the correct version of a tools tarball.

This necessary information is stored in a json metadata format called
"simplestreams". For supported public cloud services such as Amazon Web
Services, HP Cloud, Azure, etc, no action is required by the end user. However,
those setting up a private cloud, or who want to change how things work (eg use
a different Ubuntu image), can create their own metadata, after understanding a
bit about how it works.

The simplestreams format is used to describe related items in a structural
fashion.([See the Launchpad project lp:simplestreams for more details on
implementation](https://launchpad.net/simplestreams)). Below we will discuss how
Juju determines which metadata to use, and how to create your own images and
tools and have Juju use them instead of the defaults.

## Basic Workflow

Whether images or tools, Juju uses a search path to try and find suitable
metadata. The path components (in order of lookup) are:

  1. User supplied location (specified by tools-metadata-url or image-metadata-url config settings).
  2. The environment's cloud storage.
  3. Provider specific locations (eg keystone endpoint if on Openstack).
  4. A web location with metadata for supported public clouds (https://streams.canonical.com).

Metadata may be inline signed, or unsigned. We indicate a metadata file is
signed by using the '.sjson' extension. Each location in the path is first
searched for signed metadata, and if none is found, unsigned metadata is
attempted before moving onto the next path location.

Juju ships with public keys used to validate the integrity of image and tools
metadata obtained from https://streams.canonical.com. So out of the box, Juju
will "Just Work" with any supported public cloud, using signed metadata. Setting
up metadata for a private (eg Openstack) cloud requires metadata to be generated
using tools which ship with Juju.

## Image Metadata Contents

Image metadata uses a simplestreams content type of "image-ids". The product id
is formed as follows:

    com.ubuntu.cloud:server:<series_version>:<arch>

For Example:

    com.ubuntu.cloud:server:14.04:amd64

Non-released images (eg beta, daily etc) have product ids like:

    com.ubuntu.cloud.daily:server:13.10:amd64

The metadata index and product files are required to be in the following
directory tree (relative to the URL associated with each path component):

    <path_url>
      |-streams
          |-v1
             |-index.(s)json
             |-product-foo.(s)json
             |-product-bar.(s)json

The index file must be called "index.(s)json" (sjson for signed). The various
product files are named according to the Path values contained in the index
file.

# Tools Metadata Contents

Tools metadata uses a simplestreams content type of "content-download". The
product id is formed as follows:

    "com.ubuntu.juju:<series_version>:<arch>"

For example:

    "com.ubuntu.juju:12.04:amd64"

The metadata index and product files are required to be in the following
directory tree (relative to the URL associated with each path component). In
addition, tools tarballs which Juju needs to download are also expected.

      |-streams
      |   |-v1
      |      |-index.(s)json
      |      |-product-foo.(s)json
      |      |-product-bar.(s)json
      |
      |-releases
          |-tools-abc.tar.gz
          |-tools-def.tar.gz
          |-tools-xyz.tar.gz

The index file must be called "index.(s)json" (sjson for signed). The product
file and tools tarball name(s) match whatever is in the index/product files.

## Configuration

For supported public clouds, no extra configuration is required; things work
out-of-the-box. However, for testing purposes, or for non-supported cloud
deployments, Juju needs to know where to find the tools and which image to run.
Even for supported public clouds where all required metadata is available, the
user can put their own metadata in the search path to override what is provided
by the cloud.

### User specified URLs

These are initially specified in the environments.yaml file (and then
subsequently copied to the jenv file when the environment is bootstrapped). For
images, use "image-metadata-url"; for tools, use "tools-metadata-url". The URLs
can point to a world readable container/bucket in the cloud, an address served
by a http server, or even a shared directory which is accessible by all node
instances running in the cloud.

Assume an Apache http server with base URL `https://juju-metadata`, providing
access to information at `<base>/images` and `<base>/tools`. The Juju
environment yaml file could have the following entries (one or both):

    tools-metadata-url: https://juju-metadata/tools
    image-metadata-url: https://juju-metadata/images

The required files in each location is as per the directory layout described
earlier. For a shared directory, use a URL of the form "file:///sharedpath".

### Cloud storage

If no matching metadata is found in the user specified URL, environment's cloud
storage is searched. No user configuration is required here - all Juju
environments are set up with cloud storage which is used to store state
information, charms etc. Cloud storage setup is provider dependent; for Amazon
and Openstack clouds, the storage is defined by the "control-bucket" value, for
Azure, the "storage-account-name" value is relevant.

The (optional) directory structure inside the cloud storage is as follows:

      |-tools
      |   |-streams
      |       |-v1
      |   |-releases
      |
      |-images
          |-streams
              |-v1

Of course, if only custom image metadata is required, the tools directory will
not be required, and vice versa.

Note that if juju bootstrap is run with the `--upload-tools` option, the tools
and metadata are placed according to the above structure. That's why the tools
are then available for Juju to use.

### Provider specific storage

Providers may allow additional locations to search for metadata and tools. For
OpenStack, Keystone endpoints may be created by the cloud administrator. These
are defined as follows:

juju-tools

     the &LT;path_url&GT; value as described above in Tools Metadata Contents
product-streams

    the &LT;path_url&GT; value as described above in Image Metadata Contents

Other providers may similarly be able to specify locations, though the
implementation will vary.

### Central web location
([https://streams.canonical.com](https://streams.canonical.com))

This is the default location used to search for image and tools metadata and is
used if no matches are found earlier in any of the above locations. No user
configuration is required.

# Deploying Private Clouds

There are two main issues when deploying a private cloud:

  1. Image ids will be specific to the cloud.
  2. Often, outside internet access is blocked

Issue 1 means that image id metadata needs to be generated and made available.

Issue 2 means that tools need to be mirrored locally to make them accessible.

Juju tools exist to help with generating and validating image and tools
metadata. For tools, it is often easiest to just mirror
`https://streams.canonical.com/tools`. However image metadata cannot be simply
mirrored because the image ids are taken from the cloud storage provider, so
this needs to be generated and validated using the commands described below.

The available Juju metadata tools can be seen by using the help command:

    juju help metadata

The overall workflow is:

  - Generate image metadata
  - Copy image metadata to somewhere in the metadata search path
  - Optionally, mirror tools to somewhere in the metadata search path
  - Optionally, configure tools-metadata-url and/or image-metadata-url

### Image metadata

Generate image metadata using

    juju metadata generate-image -d <metadata_dir>

As a minimum, the above command needs to know the image id to use and a
directory in which to write the files.

Other required parameters like region, series, architecture etc. are taken from
the current Juju environment (or an environment specified with the -e option).
These parameters can also be overridden on the command line.

The image metadata command can be run multiple times with different regions,
series, architecture, and it will keep adding to the metadata files. Once all
required image ids have been added, the index and product json files can be
uploaded to a location in the Juju metadata search path. As per the
Configuration section, this may be somewhere specified by the `image-metadata-
url` setting or the cloud's storage etc.

Examples:

1. image-metadata-url

  - upload contents of  to `http://somelocation`
  - set image-metadata-url to `http://somelocation/images`

2. Cloud storage

  - Upload contents of  directly to environment's cloud storage
  - Use the validation command to ensure an image id can be discovered for a given scenario (region series, arch): 
      juju metadata validate-images

If run without parameters, the validation command will take all required details
from the current Juju environment (or as specified by -e) and output the image
id it would use to spin up an instance. Alternatively, series, region,
architecture etc. can be specified on the command line to override the values in
the environment config.

### Tools metadata

Generally, tools and related metadata are mirrored from
`https://streams.canonical.com/tools`. However, it is possible to manually
generate metadata for a custom built tools tarball.

First, create a tarball of the relevant tools and place in a directory
structured like this:

    <tools_dir>/tools/releases/

Now generate relevant metadata for the tools by running the command:

    juju generate-tools -d <tools_dir>

Finally, the contents of  can be uploaded to a location in the Juju metadata
search path. As per the Configuration section, this may be somewhere specified
by the tools-metadata-url setting or the cloud's storage path settings etc.

Examples:

1. tools-metadata-url

  - upload contents of  to `http://somelocation`
  - set tools-metadata-url to `http://somelocation/tools`

2. Cloud storage

  - upload contents of  directly to environment's cloud storage

As with image metadata, the validation command is used to ensure tools are
available for Juju to use:

      juju metadata validate-tools

The same comments apply. Run the validation tool without parameters to use
details from the Juju environment, or override values as required on the command
line. See `juju help metadata validate-tools` for more details.

  - ## [Juju](/)

    - [Charms](/charms/)
    - [Features](/features/)
    - [Deployment](/deployment/)
  - ## [Resources](/resources/)

    - [Overview](/resources/overview/)
    - [Documentation](/docs/)
    - [The Juju web UI](/resources/juju-gui/)
    - [The charm store](/docs/authors-charm-store.html)
    - [Tutorial](/docs/getting-started.html#test)
    - [Videos](/resources/videos/)
    - [Easy tasks for new developers](/resources/easy-tasks-for-new-developers/)
  - ## [Community](/community)

    - [Juju Blog](/community/blog/)
    - [Events](/events/)
    - [Weekly charm meeting](/community/weekly-charm-meeting/)
    - [Charmers](/community/charmers/)
    - [Write a charm](/docs/authors-charm-writing.html)
    - [Help with documentation](/docs/contributing.html)
    - [File a bug](https://bugs.launchpad.net/juju-core/+filebug)
    - [Juju Labs](/communiy/labs/)
  - ## [Try Juju](https://jujucharms.com/sidebar/)

    - [Charm store](https://jujucharms.com/)
    - [Download Juju](/download/)

(C) 2013-2014 Canonical Ltd. Ubuntu and Canonical are registered trademarks of
[Canonical Ltd](http://www.canonical.com).
