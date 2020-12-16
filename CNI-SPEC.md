# Container Network Interface Specification


## Version

This is CNI **spec** version **1.0.0 pre-release** and is still under active development.

Note that this is **independent from the version of the CNI library and plugins** in this repository (e.g. the versions of [releases](https://github.com/containernetworking/cni/releases)).

#### Released versions

Released versions of the spec are available as Git tags.

| tag                                                                                  | spec permalink                                                                        | major changes                     |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | --------------------------------- |
| [`spec-v0.4.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.4.0) | [spec at v0.4.0](https://github.com/containernetworking/cni/blob/spec-v0.4.0/SPEC.md) | Introduce the CHECK command and passing prevResult on DEL |
| [`spec-v0.3.1`](https://github.com/containernetworking/cni/releases/tag/spec-v0.3.1) | [spec at v0.3.1](https://github.com/containernetworking/cni/blob/spec-v0.3.1/SPEC.md) | none (typo fix only)              |
| [`spec-v0.3.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.3.0) | [spec at v0.3.0](https://github.com/containernetworking/cni/blob/spec-v0.3.0/SPEC.md) | rich result type, plugin chaining |
| [`spec-v0.2.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.2.0) | [spec at v0.2.0](https://github.com/containernetworking/cni/blob/spec-v0.2.0/SPEC.md) | VERSION command                   |
| [`spec-v0.1.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.1.0) | [spec at v0.1.0](https://github.com/containernetworking/cni/blob/spec-v0.1.0/SPEC.md) | initial version                   |

*Do not rely on these tags being stable.  In the future, we may change our mind about which particular commit is the right marker for a given historical spec version.*


## Overview

This document proposes a generic plugin-based networking solution for application containers on Linux, the _Container Networking Interface_, or _CNI_.

For the purposes of this proposal, we define two terms very specifically:
- _container_ can be considered synonymous with a [Linux _network namespace_][namespaces], though the actual isolation technology is not defined by the specification. What unit this corresponds to depends on a particular container runtime implementation.
- _network_ refers to a group of endpoints that are uniquely addressable that can communicate amongst each other. This could be either an individual container (as specified above), a machine, or some other network device (e.g. a router). Containers can be conceptually _added to_ or _removed from_ one or more networks.

This document aims to specify the interface between "runtimes" and "plugins". The key words "must", "must not", "required", "shall", "shall not", "should", "should not", "recommended", "may" and "optional" are used as specified in [RFC 2119][rfc-2119].

[namespaces]: http://man7.org/linux/man-pages/man7/namespaces.7.html 
[rfc-2119]: https://www.ietf.org/rfc/rfc2119.txt



## Summary

The CNI specification defines:

1. A format for administrators to define network configuration.
2. A protocol for container runtimes to make requests to network plugins
3. A procedure for exectuing plugins based on the configuration.
4. A data type for plugins to return their results to the runtime.

## Section 1: Network configuration format

CNI defines a network configuration format for administrators. It contains
directives for both the container runtime as well as the plugins to consume. At
plugin execution time, this configuration format is interpreted by the runtime and 
transformed in to a form to be passed to the plugins.

In general, the network configuration is intended to be static. It can conceptually
be thought of as being "on disk", though the CNI specification does not actually
require this.

### Configuration format

The network configuration "file" consists of a JSON object with the following keys:

- `cniVersion` (string): [Semantic Version 2.0](https://semver.org) of CNI specification to which this configuration list and all the individual configurations conform.
- `name` (string): Network name. This should be unique across all containers on the host (or other administrative domain).  Must start with a alphanumeric character, optionally followed by any combination of one or more alphanumeric characters, underscore , dot (.) or hyphen (-).
- `disableCheck` (boolean): Either `true` or `false`.  If `disableCheck` is `true`, runtimes must not call `CHECK` for this network configuration list.  This allows an administrator to prevent `CHECK`ing where a combination of plugins is known to return spurious errors.
- `plugins` (list): A list of CNI plugins and their configuration, which is a list of plugin configuration objects.

#### Plugin configuration objects:
Plugin configuration objects may contain additional fields than the ones defined here. 
The runtime MUST pass through these fields, unchanged, to the plugin, as defined in
section 4.

**Required keys:**
- `type` (string): Matches the name of the CNI plugin binary on disk

**Optional keys, used by the protocol:**
- `capabilities` (dictionary): TODO 

**Reserved keys, used by the protocol:**
These keys are generated by the runtime at execution time, and thus should not
be used in configuration.
- `runtimeConfig`
- `args`

**Optional keys, well-kown:**
These keys are not used by the protocol, but have a standard meaning to plugins.
Plugins that consume any of these configuration keys should respect their intended semantics.

- `ipMasq` (boolean): If supported by the plugin, sets up an IP masquerade on the host for this network. This is necessary if the host will act as a gateway to subnets that are not able to route to the IP assigned to the container.
- `ipam` (dictionary): Dictionary with IPAM specific values:
    - `type` (string): Refers to the filename of the IPAM plugin executable.
- `dns` (dictionary, optional): Dictionary with DNS specific values:
    - `nameservers` (list of strings, optional): list of a priority-ordered list of DNS nameservers that this network is aware of. Each entry in the list is a string containing either an IPv4 or an IPv6 address.
    - `domain` (string, optional): the local domain used for short hostname lookups.
    - `search` (list of strings, optional): list of priority ordered search domains for short hostname lookups. Will be preferred over `domain` by most resolvers.
    - `options` (list of strings, optional): list of options that can be passed to the resolver

**Other keys:**
Plugins may define additional fields that they accept and may generate an error if called with unknown fields. Runtimes must preserve unknown fields in plugin configuration objects when transforming for execution. 

#### Example configuration
```jsonc
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "plugins": [
    {
      "type": "bridge",
      // plugin specific parameter
      "bridge": "cni0",
      
      "keyA": ["some more", "plugin specific", "configuration"],
      
      "ipam": {
        "type": "host-local",
        // ipam specific
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1"
      },
      "dns": {
        "nameservers": [ "10.1.0.1" ]
      }
    },
    {
      "type": "tuning",
      "capabilities": {
        "mac": true,
      },
      "sysctl": {
        "net.core.somaxconn": "500"
      }
    },
    {
        "type": "portmap",
        "capabilities": {"portMappings": true}
    }
  ]
}
```

## Section 2: Execution Protocol

### Overview

The CNI protocol is based on execution of binaries.  Each CNI plugin is invoked by the container management system (e.g. ContainerD or Podman). CNI defines the protocol between the plugin binary and the executing runtime.

A CNI plugin is responsible for configuring a container's network interface in some manner. Plugins fall in to two broad categories:
* "Interface" plugins, which create a network interface inside the container and ensure it has connectivity.
* "Chained" plugins, which adjust the configuration of an already-created interface (but may need to create more interfaces to do so).

The runtime passes parameters to the plugin via environment variables. It supplies configuration via stdin. The plugin returns
a result on stdout. Configuration and results are encoded in JSON.

Parameters define invocation-specific settings, whereas configuration is, with some exceptions, the same for any given network.

### Parameters

Protocol parameters are passed to the plugins via OS environment variables.

- `CNI_COMMAND`: indicates the desired operation; `ADD`, `DEL`, `CHECK`, or `VERSION`.
- `CNI_CONTAINERID`: Container ID. A unique plaintext identifier for a container, allocated by the runtime. Must not be empty.  Must start with a alphanumeric character, optionally followed by any combination of one or more alphanumeric characters, underscore (), dot (.) or hyphen (-).
- `CNI_NETNS`: A reference to the container's "isolation domain". If using network namespaces, then a path to the network namespace (e.g. `/run/netns/[nsname]`)
- `CNI_IFNAME`: Name of the interface to create inside the container; if the plugin is unable to use this interface name it must return an error.
- `CNI_ARGS`: Extra arguments passed in by the user at invocation time. Alphanumeric key-value pairs separated by semicolons; for example, "FOO=BAR;ABC=123"
- `CNI_PATH`: List of paths to search for CNI plugin executables. Paths are separated by an OS-specific list separator; for example ':' on Linux and ';' on Windows

### CNI operations

CNI defines 4 operations: ADD, DEL, CHECK, and VERSION. These are passed to the plugin via the `CNI_COMMAND` environment variable.

#### `ADD`: Add container to network, or apply modifications

A CNI plugin, upon receiving an ADD command, should either
- create the interface defined by `CNI_IFNAME` inside the container at `CNI_NETNS`, or
- Adjust the configuration of the interface defined by `CNI_IFNAME` inside the container at `CNI_NETNS`.

If the CNI plugin is unable to successfully complete, it MUST return with a non-zero exit code and the "error" result structure as defined below.

If the CNI plugin is successful, it MUST return with a zero exit code and a result structure as defined below.

If an interface of the requested name already exists in the container, the CNI plugin MUST return with an error.

A runtime should not call ADD twice (without an intervening DEL) for the same `(CNI_CONTAINERID, CNI_IFNAME)` tuple. This implies that a given container ID may be added to a specific network more than once only if each addition is done with a different interface name.

**Input:** 

The runtime will provide a json-serialized plugin configuration object (defined below) on standard in.

Required environment parameters:
- `CNI_COMMAND`
- `CNI_CONTAINERID`
- `CNI_NETNS`
- `CNI_IFNAME`

Optional environment parameters:
- `CNI_ARGS`
- `CNI_PATH`

#### `DEL`: Remove container from network, or un-apply modifications

A CNI plugin, upon receiving a DEL command, should either
- Delete the interface defined by `CNI_IFNAME` inside the container at `CNI_NETNS`, or
- Undo any modifications applied int the plugin's `ADD` functionality

Plugins should generally complete a `DEL` action without error even if some resources are missing.  For example, an IPAM plugin should generally release an IP allocation and return success even if the container network namespace no longer exists, unless that network namespace is critical for IPAM management. While DHCP may usually send a 'release' message on the container network interface, since DHCP leases have a lifetime this release action would not be considered critical and no error should be returned. For another example, the `bridge` plugin should delegate the DEL action to the IPAM plugin and clean up its own resources (if present) even if the container network namespace and/or container network interface no longer exist.

Plugins MUST accept multiple `DEL` calls for the same (`CNI_CONTAINERID`, `CNI_IFNAME`) pair, and return sucess if the
interface in question, or any modifications added, are missing.

If the CNI plugin is unable to successfully complete, it must return with a non-zero exit code and the "error" result structure as defined below.

If the CNI plugin is successful, it must return with a zero exit code and a result structure as defined below.

**Input:**

The runtime will provide a json-serialized plugin configuration object (defined below) on standard in. 

Required environment parameters:
- `CNI_COMMAND`
- `CNI_CONTAINERID`
- `CNI_IFNAME`

Optional environment parameters:
- `CNI_NETNS`
- `CNI_ARGS`
- `CNI_PATH`


#### `CHECK`: Check container's networking is as expected
CHECK is a way for a runtime to probe the status of an existing container.

Plugin considerations:
- The plugin must consult the `prevResult` to determine the expected interfaces and addresses.
- The plugin must allow for a later chained plugin to have modified networking resources, e.g. routes, on ADD.
- The plugin should return an error if a resource included in the CNI Result type (interface, address or route) was created by the plugin, and is listed in `prevResult`, but is missing or in an invalid state.
- The plugin should return an error if other resources not tracked in the Result type such as the following are missing or are in an invalid state:
  - Firewall rules
  - Traffic shaping controls
  - IP reservations
  - External dependencies such as a daemon required for connectivity
  - etc.
- The plugin should return an error if it is aware of a condition where the container is generally unreachable.
- The plugin must handle `CHECK` being called immediately after an `ADD`, and therefore should allow a reasonable convergence delay for any asynchronous resources.
- The plugin should call `CHECK` on any delegated (e.g. IPAM) plugins and pass any errors on to its caller.


Runtime considerations:
- A runtime must not call `CHECK` for a container that has not been `ADD`ed, or has been `DEL`eted after its last `ADD`.
- A runtime must not call `CHECK` if `disableCheck` is set to `true` in the [configuration list](#network-configuration-lists).
- A runtime must include a `prevResult` field in the network configuration containing the `Result` of the immediately preceding `ADD` for the container. The runtime may wish to use libcni's support for caching `Result`s.
- A runtime may choose to stop executing `CHECK` for a chain when a plugin returns an error.
- A runtime may execute `CHECK` from immediately after a successful `ADD`, up until the container is `DEL`eted from the network.
- A runtime may assume that a failed `CHECK` means the container is permanently in a misconfigured state.


**Input:**

The runtime will provide a json-serialized plugin configuration object (defined below) on standard in. 

Required environment parameters:
- `CNI_COMMAND`
- `CNI_CONTAINERID`
- `CNI_NETNS`
- `CNI_IFNAME`

Optional environment parameters:
- `CNI_ARGS`
- `CNI_PATH`

All parameters, with the exception of `CNI_PATH`, must be the same as the corresponding ADD for this container.

#### `VERSION`: probe plugin version support
The plugin should output via standard-out a json-serialized version result object (see below).


// TODO move this down
```jsonc
{
    "cniVersion": "1.0.0", // the version of the CNI spec in use for this output
    "supportedVersions": [ "0.1.0", "0.2.0", "0.3.0", "0.3.1", "0.4.0", "1.0.0" ] // the list of CNI spec versions that this plugin supports
}
```

**Input:**

Nothing should be passed on standard in.

Required environment parameters:
- `CNI_COMMAND`


## Section 3: Execution of Network Configurations

This section describes how a container runtime interprets a network configuration (as defined in section 1) and executes plugins accordingly. A runtime may wish to _add_, _delete_, or _check_ a network configuration in a container. This results in a series of plugin `ADD`, `DELETE`, or `CHECK` executions, correspondingly. This section also defines how a network configuration is transformed and provided to the plugin.

The operation of a network configuration on a container is called an _attachment_. An attachment may be uniquely identified by the `(CNI_CONTAINERID, CNI_IFNAME)` tuple.

Runtimes 

### Lifecycle & Ordering

- The container runtime must create a new network namespace for the container before invoking any plugins.
- The container runtime must not invoke parallel operations for the same container, but is allowed to invoke parallel operations for different containers. This includes across multiple attachments.
- Plugins must handle being executed concurrently across different containers. If necessary, they must implement locking on shared resources (e.g. IPAM databases).
- The container runtime must ensure that _add_ is  eventually followed by a corresponding _delete_. The only exception is in the event of catastrophic failure, such as node loss. A _delete_ must still be executed even if the _add_ fails.
- _delete_ may be followed by additional _deletes_.
- The network configuration should not change between _add_ and _delete_.
- The network configuration should not change between _attachments_.

### Attachment Parameters
While a network configuration should not change between _attachments_, there are certain parameters supplied by the container runtime that are per-attachment. They are:

- **Container ID:** A unique plaintext identifier for a container, allocated by the runtime. Must not be empty.  Must start with a alphanumeric character, optionally followed by any combination of one or more alphanumeric characters, underscore (), dot (.) or hyphen (-). During execution, always set as the  `CNI_CONTAINERID` parameter.
- **Namespace**: A reference to the container's "isolation domain". If using network namespaces, then a path to the network namespace (e.g. `/run/netns/[nsname]`). During execution, always set as the `CNI_NETNS` parameter.
- **Container interface name**: Name of the interface to create inside the container. During execution, always set as the `CNI_IFNAME` parameter.
- **Generic Arguments**: Extra arguments, in the form of key-value string pairs, that are relevant to a specific attachment.  During execution, always set as the `CNI_ARGS` parameter as well as provided in the generated configuration, see below.
- **Capability Arguments**: These are also key-value pairs. The key is a string, whereas the value is any JSON-serializable type. The keys and values are defined by [convention](CONVENTIONS.md).

Furthermore, the runtime must be provided a list of paths to search for CNI plugins. This must also be provided to plugins during execution via the `CNI_PATH` environment variable.

### Adding an attachment
To attach create a network attachment, the runtime must first create the isolation domain. 

Then, for every configuration defined in the `plugins` key of the network configuration,
1. Look up the executable specified in the `type` field. If this does not exist, then this is an error.
2. Derive execution configuration from the plugin configuration, with the following parameters:
    - If this is the first plugin in the list, no previous result is provided,
    - For all additional plugins, the previous result is the result of the previous plugins.
3. Execute the plugin binary, with `CNI_COMMAND=ADD`. Provide parameters defined above as environment variables. Supply the derived configuration via standard in.
4. If the plugin returns an error, halt execution and return the error to the user.

The result returned by the final plugin is returned to the user. The runtime must store the returned result persistently, as it is required for _check_ and _delete_ operations.

### Deleting an attachment
Deleting a network attachment is much the same as adding, with a few key differences:
- The list of plugins is executed in **reverse order**
- The previous result provided is always the final result of the _add_ operation.

For every plugin defined in the `plugins` key of the network configuration, *in reverse order*,
1. Look up the executable specified in the `type` field. If this does not exist, then this is an error.
2. Derive execution configuration from the plugin configuration, with the previous result from the initial _add_ operation.
3. Execute the plugin binary, with `CNI_COMMAND=DEL`. Provide parameters defined above as environment variables. Supply the derived configuration via standard in.
4. If the plugin returns an error, halt execution and return the error to the user.

If all deletions were successful

### Checking an attachment
The runtime may also ask every plugin to confirm that a given attachment is still functional. The runtime must use the same attachment parameters as it did for the _add_ operation.

Checking is similar to add with two exceptions:
- the previous result provided is always the final result of the _add_ operation.
- If the network configuration defines `disableCheck`, then always return success to the user.

For every plugin defined in the `plugins` key of the network configuration,
1. Look up the executable specified in the `type` field. If this does not exist, then this is an error.
2. Derive execution configuration from the plugin configuration, with the previous result from the initial _add_ operation.
3. Execute the plugin binary, with `CNI_COMMAND=CHECK`. Provide parameters defined above as environment variables. Supply the derived configuration via standard in.
4. If the plugin returns an error, halt execution and return the error to the user.

If all plugins return success, return success to the user.


### Deriving execution configuration from plugin configuration
The on-disk configuration format needs to be transformed to a version understood by the plugin. This section describes that transformation.

The execution configuration is also JSON. It consists of the plugin configuration, primarily unchanged except for the specified additions and removals.

The following fields must be inserted by the runtime:
- `cniVersion`: taken from the `cniVersion` field of the network configuration
- `name`: taken from the `name` field of the network configuration
- `args`: A JSON object, consisting of the generic arguments provided as an attachment parameter
- `runtimeConfig`: A json object, consisting of the union of capabilities provided by the plugin and requested by the runtime (more detail below)
- `prevResult`: A json object, consisting of the result type returned by the "previous" plugin. The meaning of "previous" is defined by the specific operation (_add_, _delete_, or _check_).

The following fields must be **removed** by the runtime:
- `capabilities`

All other fields should be passed through unaltered.

#### Deriving `runtimeConfig`

Whereas `args` and CNI_ARGS are provided to all plugins, with no indication if they are going to be consumed, _Capability arguments_ need to be declared explicitly in configuration. The runtime, thus, can determine if a given network configuration supports a specific _capability_. Capabilities are not defined by the specification - rather, they are documented [conventions](CONVENTIONS.md).

As defined in section 1, the plugin configuration includes an optional key, `capabilities`. This example shows a that supports the `portMapping` capability:

```json
{ 
  "type": "myPlugin",
  "capabilities": {
    "portMappings": true
  }
}
```

The `runtimeConfig` parameter is derived from the `capabilities` in the network configuration and the _capability arguments_ generated by the runtime. Specifically, any capability supported by the plugin configuration and provided by the runtime should be inserted in the `runtimeConfig`.

Thus, the above example may result in the following being passed to the plugin as part of the execution configuration
```jsonc
{
  "type": "myPlugin",
  "runtimeConfig": {
    "portMappings": [ { "hostPort": 8080, "containerPort": 80, "protocol": "tcp" } ]
  }
  ...
}
```

## Section 4: Result Types

Plugins can return one of three result types:

- success
- error
- version

### Success

Note that IPAM plugins should return an abbreviated `Result` structure as described in [IP Allocation](#ip-allocation).

Plugins must indicate success with a return code of zero and the following JSON printed to stdout in the case of the ADD command. The `ips` and `dns` items should be the same output as was returned by the IPAM plugin (see [IP Allocation](#ip-allocation) for details) except that the plugin should fill in the `interface` indexes appropriately, which are missing from IPAM plugin output since IPAM plugins should be unaware of interfaces.

```
{
  "cniVersion": "1.0.0",
  "interfaces": [                                            (this key omitted by IPAM plugins)
      {
          "name": "<name>",
          "mac": "<MAC address>",                            (required if L2 addresses are meaningful)
          "sandbox": "<netns path or hypervisor identifier>" (required for container/hypervisor interfaces, empty/omitted for host interfaces)
      }
  ],
  "ips": [
      {
          "address": "<ip-and-prefix-in-CIDR>",
          "gateway": "<ip-address-of-the-gateway>",          (optional)
          "interface": <numeric index into 'interfaces' list>
      },
      ...
  ],
  "routes": [                                                (optional)
      {
          "dst": "<ip-and-prefix-in-cidr>",
          "gw": "<ip-of-next-hop>"                           (optional)
      },
      ...
  ],
  "dns": {                                                   (optional)
    "nameservers": <list-of-nameservers>                     (optional)
    "domain": <name-of-local-domain>                         (optional)
    "search": <list-of-additional-search-domains>            (optional)
    "options": <list-of-options>                             (optional)
  }
}
```

`cniVersion` specifies a [Semantic Version 2.0](https://semver.org) of CNI specification used by the plugin. A plugin may support multiple CNI spec versions (as it reports via the `VERSION` command), here the `cniVersion` returned by the plugin in the result must be consistent with the `cniVersion` specified in [Network Configuration](#network-configuration). If the `cniVersion` in the network configuration is not supported by the plugin, the plugin should return an error code 1 (see [Well-known Error Codes](#well-known-error-codes) for details).

`interfaces` describes specific network interfaces the plugin created.
If the `CNI_IFNAME` variable exists the plugin must use that name for the sandbox/hypervisor interface or return an error if it cannot.
- `mac` (string): the hardware address of the interface.
   If L2 addresses are not meaningful for the plugin then this field is optional.
- `sandbox` (string): container/namespace-based environments should return the full filesystem path to the network namespace of that sandbox.
   Hypervisor/VM-based plugins should return an ID unique to the virtualized sandbox the interface was created in.
   This item must be provided for interfaces created or moved into a sandbox like a network namespace or a hypervisor/VM.

The `ips` field is a list of IP configuration information.
See the [IP well-known structure](#ips) section for more information.

The `routes` field is a list of route configuration information.
See the [Routes well-known structure](#routes) section for more information.

The `dns` field contains a dictionary consisting of common DNS information.
See the [DNS well-known structure](#dns) section for more information.

The specification does not declare how this information must be processed by CNI consumers.
Examples include generating an `/etc/resolv.conf` file to be injected into the container filesystem or running a DNS forwarder on the host.

Errors must be indicated by a non-zero return code and the following JSON being printed to stdout:
```
{
  "cniVersion": "1.0.0",
  "code": <numeric-error-code>,
  "msg": <short-error-message>,
  "details": <long-error-message> (optional)
}
```

`cniVersion` specifies a [Semantic Version 2.0](https://semver.org) of CNI specification used by the plugin.
Error codes 0-99 are reserved for well-known errors (see [Well-known Error Codes](#well-known-error-codes) section).
Values of 100+ can be freely used for plugin specific errors. 

In addition, stderr can be used for unstructured output such as logs.

### Network Configuration Lists


Plugins are allowed to modify or suppress all or part of a `prevResult`.
However, plugins that support a version of the CNI specification that includes the `prevResult` field MUST handle `prevResult` by either passing it through, modifying it, or suppressing it explicitly.
It is a violation of this specification to be unaware of the `prevResult` field.



## Section 5: Examples
We assume the network configuration [shown above](#example-network-configuration-lists) in section 1. For this attachment, the runtime generates `portmap` and `mac` capability args, along with the generic argument "argA=foo".

The container runtime would perform the following steps for the `add` operation.


1) Call the `bridge` plugin with the following JSON, `CNI_COMMAND=ADD`:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "keyA": ["some more", "plugin specific", "configuration"],
  "args": {"argA": "foo"},
  "ipam": {
    "type": "host-local",
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  },
  "dns": {
    "nameservers": [ "10.1.0.1" ]
  }
}
```

The plugin returns the following result:

```json
{
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
}
```

2) Next, call the `tuning` plugin, with `CNI_COMMAND=ADD`. Note that `prevResult` is supplied.

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "runtimeConfig": {
    "mac": "00:11:22:33:44:66"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

The plugin returns the following result. Note that the **mac** has changed.

```json
{
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
}
```

3) Finally, call the `portmap` plugin, with `CNI_COMMAND=ADD`. Note that `prevResult` matches that returned by `tuning`:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "portmap",
  "runtimeConfig": {
    "portMappings"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```


Given the same network configuration JSON list, the container runtime would perform the following steps for the `CHECK` action.

1) first call the `bridge` plugin with the following JSON, including the `prevResult` field containing the JSON response from the `ADD` operation:

```jsonc
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "args": {
    "labels" : {
        "appVersion" : "1.0"
    }
  },
  "ipam": {
    "type": "host-local",
    // ipam specific
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  },
  "dns": {
    "nameservers": [ "10.1.0.1" ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

2) next call the `tuning` plugin with the following JSON, including the `prevResult` field containing the JSON response from the `ADD` operation:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

Given the same network configuration JSON list, the container runtime would perform the following steps for the `DEL` action.
Note that plugins are executed in reverse order from the `ADD` and `CHECK` actions.

1) first call the `tuning` plugin with the following JSON, including the `prevResult` field containing the JSON response from the `ADD` action:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

2) next call the `bridge` plugin with the following JSON, including the `prevResult` field containing the JSON response from the `ADD` action:

```jsonc
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "args": {
    "labels" : {
        "appVersion" : "1.0"
    }
  },
  "ipam": {
    "type": "host-local",
    // ipam specific
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  },
  "dns": {
    "nameservers": [ "10.1.0.1" ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.0.0.5/32",
          "interface": 2
        }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```



Unedited spec below.
---



### IP Allocation

As part of its operation, a CNI plugin is expected to assign (and maintain) an IP address to the interface and install any necessary routes relevant for that interface. This gives the CNI plugin great flexibility but also places a large burden on it. Many CNI plugins would need to have the same code to support several IP management schemes that users may desire (e.g. dhcp, host-local). 

To lessen the burden and make IP management strategy be orthogonal to the type of CNI plugin, we define a second type of plugin -- IP Address Management Plugin (IPAM plugin). It is however the responsibility of the CNI plugin to invoke the IPAM plugin at the proper moment in its execution. The IPAM plugin must determine the interface IP/subnet, Gateway and Routes and return this information to the "main" plugin to apply. The IPAM plugin may obtain the information via a protocol (e.g. dhcp), data stored on a local filesystem, the "ipam" section of the Network Configuration file or a combination of the above.

#### IP Address Management (IPAM) Interface

Like CNI plugins, the IPAM plugins are invoked by running an executable. The executable is searched for in a predefined list of paths, indicated to the CNI plugin via `CNI_PATH`. The IPAM Plugin must receive all the same environment variables that were passed in to the CNI plugin. Just like the CNI plugin, IPAM plugins receive the network configuration via stdin.

Success must be indicated by a zero return code and the following JSON being printed to stdout (in the case of the ADD command):

```
{
  "cniVersion": "1.0.0",
  "ips": [
      {
          "address": "<ip-and-prefix-in-CIDR>",
          "gateway": "<ip-address-of-the-gateway>"  (optional)
      },
      ...
  ],
  "routes": [                                       (optional)
      {
          "dst": "<ip-and-prefix-in-cidr>",
          "gw": "<ip-of-next-hop>"                  (optional)
      },
      ...
  ]
  "dns": {                                          (optional)
    "nameservers": <list-of-nameservers>            (optional)
    "domain": <name-of-local-domain>                (optional)
    "search": <list-of-search-domains>              (optional)
    "options": <list-of-options>                    (optional)
  }
}
```

Note that unlike regular CNI plugins, IPAM plugins should return an abbreviated `Result` structure that does not include the `interfaces` key, since IPAM plugins should be unaware of interfaces configured by their parent plugin except those specifically required for IPAM (eg, like the `dhcp` IPAM plugin).

`cniVersion` specifies a [Semantic Version 2.0](https://semver.org) of CNI specification used by the IPAM plugin. An IPAM plugin may support multiple CNI spec versions (as it reports via the `VERSION` command), here the `cniVersion` returned by the IPAM plugin in the result must be consistent with the `cniVersion` specified in [Network Configuration](#network-configuration). If the `cniVersion` in the network configuration is not supported by the IPAM plugin, the plugin should return an error code 1 (see [Well-known Error Codes](#well-known-error-codes) for details).

The `ips` field is a list of IP configuration information.
See the [IP well-known structure](#ips) section for more information.

The `routes` field is a list of route configuration information.
See the [Routes well-known structure](#routes) section for more information.

The `dns` field contains a dictionary consisting of common DNS information.
See the [DNS well-known structure](#dns) section for more information.

Errors and logs are communicated in the same way as the CNI plugin. See [CNI Plugin Result](#result) section for details.

IPAM plugin examples:
 - **host-local**: Select an unused (by other containers on the same host) IP within the specified range.
 - **dhcp**: Use DHCP protocol to acquire and maintain a lease. The DHCP requests will be sent via the created container interface; therefore, the associated network must support broadcast.

#### Notes
 - Routes are expected to be added with a 0 metric.
 - A default route may be specified via "0.0.0.0/0". Since another network might have already configured the default route, the CNI plugin should be prepared to skip over its default route definition.

### Well-known Structures

#### IPs

```
  "ips": [
      {
          "address": "<ip-and-prefix-in-CIDR>",
          "gateway": "<ip-address-of-the-gateway>",      (optional)
          "interface": <numeric index into 'interfaces' list> (not required for IPAM plugins)
      },
      ...
  ]
```

The `ips` field is a list of IP configuration information determined by the plugin. Each item is a dictionary describing of IP configuration for a network interface.
IP configuration for multiple network interfaces and multiple IP configurations for a single interface may be returned as separate items in the `ips` list.
All properties known to the plugin should be provided, even if not strictly required.
- `address` (string): an IP address in CIDR notation (eg "192.168.1.3/24").
- `gateway` (string): the default gateway for this subnet, if one exists.
   It does not instruct the CNI plugin to add any routes with this gateway: routes to add are specified separately via the `routes` field.
   An example use of this value is for the CNI `bridge` plugin to add this IP address to the Linux bridge to make it a gateway.
- `interface` (uint): the index into the `interfaces` list for a [CNI Plugin Result](#result) indicating which interface this IP configuration should be applied to.
   IPAM plugins should not return this key since they have no information about network interfaces.

#### Routes

```
  "routes": [
      {
          "dst": "<ip-and-prefix-in-cidr>",
          "gw": "<ip-of-next-hop>"               (optional)
      },
      ...
  ]
```

Each `routes` entry is a dictionary with the following fields.  All IP addresses in the `routes` entry must be the same IP version, either 4 or 6.
  - `dst` (string): destination subnet specified in CIDR notation.
  - `gw` (string): IP of the gateway. If omitted, a default gateway is assumed (as determined by the CNI plugin).

Each `routes` entry must be relevant for the sandbox interface specified by CNI_IFNAME.

#### DNS

```
  "dns": {
    "nameservers": <list-of-nameservers>                 (optional)
    "domain": <name-of-local-domain>                     (optional)
    "search": <list-of-additional-search-domains>        (optional)
    "options": <list-of-options>                         (optional)
  }
```

The `dns` field contains a dictionary consisting of common DNS information.
- `nameservers` (list of strings): list of a priority-ordered list of DNS nameservers that this network is aware of. Each entry in the list is a string containing either an IPv4 or an IPv6 address.
- `domain` (string): the local domain used for short hostname lookups.
- `search` (list of strings): list of priority ordered search domains for short hostname lookups. Will be preferred over `domain` by most resolvers.
- `options` (list of strings): list of options that can be passed to the resolver.
  See [CNI Plugin Result](#result) section for more information.
  
## Well-known Error Codes

Error codes 1-99 must not be used other than as specified here.

Error Code|Error Description
---|---
 `1`|Incompatible CNI version
 `2`|Unsupported field in network configuration. The error message must contain the key and value of the unsupported field.
 `3`|Container unknown or does not exist. This error implies the runtime does not need to perform any container network cleanup (for example, calling the `DEL` action on the container).
 `4`|Invalid necessary environment variables, like CNI_COMMAND, CNI_CONTAINERID, etc. The error message must contain the names of invalid variables.
 `5`|I/O failure. For example, failed to read network config bytes from stdin.
 `6`|Failed to decode content. For example, failed to unmarshal network config from bytes or failed to decode version info from string.
 `7`|Invalid network config. If some validations on network configs do not pass, this error will be raised.
 `11`|Try again later. If the plugin detects some transient condition that should clear up, it can use this code to notify the runtime it should re-try the operation later.