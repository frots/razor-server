# Razor API Overview and Use

## Compatibility, URL Stability, and Ongoing Support

The Razor API adopts the REST notion that hypertext defines the API, rather
than URL templates or clients with special knowledge of the URL structure.

As developers, we promise good compatibility and support for your client if
you follow the simple rule: use navigation, rather than client-side knowledge
of the URL structure.

To do that, implement any action by starting at `http://razor:8080/api`,
rather than anywhere else in the API namespace.  This document then allows you
to navigate -- much like a web browser can navigate a website -- through the
various query options available to you.

While this document contains some example URL's, and client output examples
also include some, you should make no assumptions that the URLs that your
server uses follow the same structure as the ones in this document.

### Stability Warning

The Razor API is not in a stable state yet. While we try our best to not make
any incompatible changes, we can't guarantee that we won't.  This is
supported, in part, by providing clients with well versioned navigation tools
to discover their desired endpoint, or to cleanly discover that it does not
exist any longer.

Even after we declare the API stable, clients will have to be able to deal
with changes to the API: the URL structure, other than the top level
navigation entry point, is not subject to any assurance that it will stay
as-is.  Use hypertext navigation, and normal HTTP caching, to ensure this does
not burn you.

The one hard-coded URL you can use reliably is `/api`, and the document it
returns is intended to be significantly more stable than any other component
of the API.  This is because it is the root of all navigation, and if we break
that no other compatibility assurances matter. ;)

## How to navigate through the document

The type of objects is indicated with a `spec` attribute. The value of the
attribute is an absolute URL underneath
http://api.puppetlabs.com/razor/v1. These URL's are currently not (yet) backed
by any content, and serve solely as a unique identifier.

Two attributes are commonly used to identify objects: `id` is a fully
qualified URL that can be used as a globally unique reference for that
object, and a `GET` request against that URL will produce a representation
of the object. The `name` attribute is used for a short (human readable)
reference to the object, generally only unique amongst objects of the same
type on the same server.

### `/api` document reference

When you fetch `http://razor:8080/api`, you fetch the top level entry point
for navigating through our command and query facilities.  The structure of
this document is a JSON object with the following keys:

 * `commands`: the commands -- mutating operations -- available on this server
 * `collections`: the collections -- read-only queries -- available on this server

Each of those keys contains a JSON array, with a sequence of JSON objects,
which have the following keys:

 * `name`: a human-readable label.  No stability promises.
 * `rel`: a "spec URL" that indicates the type of contained data.  Use this to
          discover the endpoint that you wish to follow, rather than the `name`.
 * `id`: the URL to follow to get at this content.

This document has a reasonable stability promise: you should be prepared to
ignore additional keys at any level, and to treat the value of those keys as
"unspecified" rather than assuming they will be JSON arrays.

If you follow those simple rules (eg: assume that this documents the minimum
content returned, and ignore everything else), you can have good confidence
that you will not need to change your client.

### `/svc` URLs

The `/svc` namespace is an internal namespace, used for communication with the
iPXE client, the Microkernel, and other internal components of Razor.

This namespace is not enumerated under `/api`, and has no stability promises.
If you use this namespace, be aware that operations are designed specifically
for the needs of our internal components rather than as generic query, and
that we make *NO PROMISES* about stability of these calls, or their content,
even over patch releases.

## Commands

The list of commands that the Razor server supports is returned as part of
a request to `GET /api` in the `commands` array. Clients can identify
commands using the `rel` attribute of each entry in the array, and should
make their POST requests to the URL given in the `url` attribute.

Commands are generally asynchronous and return a status code of 202
Accepted on success. The `url` property of the response generally refers to
an entity that is affected by the command and can be queried to determine
when the command has finished.

### Create new repo

There are two flavors of repositories: ones where Razor unpacks ISO's for
you and serves their contents, and ones that are somewhere else, for
example, on a mirror you maintain. The first form is created by creating a
repo with the `iso-url` property; the server will download and unpack the
ISO image into its file system:

    {
      "name": "fedora19",
      "iso-url": "file:///tmp/Fedora-19-x86_64-DVD.iso"
    }

The second form is created by providing a `url` property when you create
the repository; this form is merely a pointer to a resource somehwere and
nothing will be downloaded onto the Razor server:

    {
      "name": "fedora19",
      "url": "http://mirrors.n-ix.net/fedora/linux/releases/19/Fedora/x86_64/os/"
    }

### Delete a repo

The `delete-repo` command accepts a single repo name:

    {
      "name": "fedora16"
    }

### Create installer

Razor supports both installers stored in the filesystem and installers
stored in the database; for development, it is highly recommended that you
store your installers in the filesystem. Details about that can be found
[on the Wiki](https://github.com/puppetlabs/razor-server/wiki/Writing-installers)

For production setups, it is usually better to store your installers in the
database. To create an installer, clients post the following to the
`/spec/create_installer` URL:

    {
      "name": "redhat6",
      "os": "Red Hat Enterprise Linux",
      "os_version": "6",
      "description": "A basic installer for RHEL6",
      "boot_seq": {
        "1": "boot_install",
        "default": "boot_local"
      }
      "templates": {
        "boot_install": " ... ERB template for an ipxe boot file ...",
        "installer": " ... another ERB template ..."
      }
    }

The possible properties in the request are:

name       | The name of the installer; must be unique
os         | The name of the OS; mandatory
os_version | The version of the operating system
description| Human-readable description
boot_seq   | A hash mapping the boot counter or 'default' to a template
templates  | A hash mapping template names to the actual ERB template text

### Create broker

To create a broker, clients post the following to the `create-broker` URL:

    {
      "name": "puppet",
      "configuration": { "server": "puppet.example.org", "version": "3.0.0" },
      "broker-type": "puppet"
    }

The `broker-type` must correspond to a broker that is present on the
`broker_path` set in `config.yaml`.

The permissible settings for the `configuration` hash depend on the broker
type and are declared in the broker type's `configuration.yaml`.

### Create tag

To create a tag, clients post the following to the `/spec/create_tag`
command:

    {
      "name": "small",
      "rule": ["=", ["fact", "processorcount"], "2"]
    }

The `name` of the tag must be unique; the `rule` is a match expression.

### Delete tag

A tag can be deleted by posting its name to the `/spec/delete_tag` command:

    {
      "name": "small",
      "force": true
    }

If the tag is used by a policy, the attempt to delete the tag will fail
unless the optional parameter `force` is set to `true`; in that case the
tag will be removed from all policies that use it and then deleted.

### Update tag

The rule for a tag can be changed by posting the following to the
`/spec/update_tag_rule` command:

    {
      "name": "small",
      "rule": ["<=", ["fact", "processorcount"], "2"],
      "force": true
    }

This will change the rule of the given tag to the new rule. The tag will be
reevaluated against all nodes and each node's tag attribute will be updated
to reflect whether the tag now matches or not, i.e., the tag will be added
to/removed from each node's tag as appropriate.

If the tag is used by any policies, the update will only be performed if
the optional parameter `force` is set to `true`. Otherwise, the command
will return with status code 400.

### Create policy

    {
      "name": "a policy",
      "repo": { "name": "some_repo" },
      "installer": { "name": "redhat6" },
      "broker": { "name": "puppet" },
      "hostname": "host${id}.example.com",
      "root_password": "secret",
      "max_count": "20",
      "line_number": "100"
      "tags": [{ "name": "existing_tag"},
               { "name": "new_tag", "rule": ["=", "dollar", "dollar"]}]
    }

Policies are matched in the order of ascending line numbers.

Tags, brokers, installers and repos are referenced by their name. Tags can
also be created by providing a rule; if a tag with that name already
exists, the rule must be equal to the rule of the existing tag.

Hostname is a pattern for the host names of the nodes bound to the policy;
eventually you'll be able to use facts and other fun stuff there. For now,
you get to say ${id} and get the node's DB id.

The `max_count` determines how many nodes can be bound at any given point
to this policy at the most. This can either be set to `nil`, indicating
that an unbounded number of nodes can be bound to this policy, or a
positive integer to set an upper bound.

### Enable/disable policy

Policies can be enabled or disabled. Only enabled policies are used when
matching nodes against policies. There are two commands to toggle a
policy's `enabled` flag: `enable-policy` and `disable-policy`, which both
accept the same body, consisting of the name of the policy in question:

    {
      "name": "a policy"
    }

### Delete node

A single node can be removed from the database with the `delete-node`
command. It accepts the name of a single node:

    {
      'name': 'node17'
    }

Of course, if that node boots again at some point, it will be automatically
recreated.

### Unbind node

Unbinding a node removes its association with a policy; once unbound, the
node will boot back into the Microkernel and go through discovery, tag
matching and possibly be bound to another policy. Specify which node to
unbind by sending the node's name in the body of the request

    {
      'name': 'node17'
    }

### Set node IPMI credentials

Razor can store IPMI credentials on a per-node basis.  These are the hostname
(or IP address), the username, and the password to use when contacting the
BMC/LOM/IPMI lan or lanplus service to check or update power state and other
node data.

This is an atomic operation: all three data items are set or reset in a single
operation.  Partial updates must be handled client-side.  This eliminates
conflicting update and partial update combination surprises for users.

The structure of a request is:

    {
      'name': 'node17',
      'ipmi-hostname': 'bmc17.example.com',
      'ipmi-username': null,
      'ipmi-password': 'sekretskwirrl'
    }

The various IPMI fields can be null (representing no value, or the NULL
username/password as defined by IPMI), and if omitted are implicitly set to
the NULL value.

You *must* provide an IPMI hostname if you provide either a username or
password, since we only support remote, not local, communication with the
IPMI target.

## Collections

Along with the list of supported commands, a `GET /api` request returns a list
of supported collections in the `collections` array. Each entry contains at
minimum `url`, `spec`, and `name` keys, which correspond respectively to the
endpoint through which the collection can be retrieved (via `GET`), the 'type'
of collection, and a human-readable name for the collection.

A `GET` request to a collection endpoint will yield a list of JSON objects,
each of which has at minimum the following fields:

id   | a URL that uniquely identifies the object
spec | a URL that identifies the type of the object
name | a human-readable name for the object

Different types of objects may specify other properties by defining additional
key-value pairs. For example, here is a sample tag listing:

    [
      {
        "spec": "http://localhost:8080/spec/object/tag",
        "id": "http://localhost:8080/api/collections/objects/14",
        "name": "virtual",
        "rule": [ "=", [ "fact", "is_virtual" ], true ]
      },
      {
        "spec": "http://localhost:8080/spec/object/tag",
        "id": "http://localhost:8080/api/collections/objects/27",
        "name": "group 4",
        "rule": [
          "in", [ "fact", "dhcp_mac" ],
          "79-A8-C3-39-E4-BA",
          "6C-35-FE-B7-BD-2D",
          "F9-92-DF-E0-26-5D"
        ]
      }
    ]

In addition, references to other resources are represented either as an array
of, in the case of a one- or many-to-many relationship, or single, for a one-
to-one relationship, JSON objects with the following fields:

url    | a URL that uniquely identifies the object
obj_id | a short numeric identifier
name   | a human-readable name for the object

If the reference object is in an array, the `obj_id` field serves as a unique
identifier within the array.

## Other things

### The default boostrap iPXE file

A GET request to `/api/microkernel/bootstrap` will return an iPXE script
that can be used to bootstrap nodes that have just PXE booted (it
culminates in chain loading from the Razor server)

The URL accepts the parameter `nic_max` which should be set to the maximum
number of network interfaces that respond to DHCP on any given machine. It
defaults to 4.
