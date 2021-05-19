# XMLRPC API Versioning

The primary purpose of this RFC is to improve the versioning of the
(XMLRPC) API in Uyuni and SUSE Manager (SUMA).

When the consumers use the API, there must be a way for them to make
sure they call the endpoints correctly (they need to call an existing
method with correct parameters and they also need to make some
asspumptions on the return value structure).

An optional goal of this RFC is to provide better guidelines for
writing scripts that consume the API in the `API > FAQ` section of the
Uyuni/SUMA Web UI.


## Exposed API information

The current Uyuni/SUMA exposes the API information via the
`api` namespace methods:

- `getVersion` - the API version, e.g. `25`. This number grows with
  time, but there are no strict rules about it.
- `systemVersion` - the Uyuni server version (e.g. `4.2.0 Beta1` for
  SUMA)

Furthermore, the namespace also exposes a basic API introspection
calls describing the structure of the call:
- `getApiNamespaces` - all API namespaces
- `getApiNamespaceCallList` - API methods of given namespace
- `getApiCallList` - all API methods grouped by namespace


## Problems

We must provide a sane way for users to write robust scripts that
target various Uyuni/SUMA versions.

In the `spacecmd` tool, this is done by API version check endpoint
(`api.getVersion` above). The logic simply checks the API version and
reacts accordingly (e.g. warns the user, that given operation is not
supported by the API version).

A similar approach is taken by the `errata-import.pl` script by Steve
Meier [1]. The script contains a list of supported versions and checks
in runtime, whether the API version of the Uyuni/SUMA server is
contained in this list.

This works well, until the user needs to target both Uyuni and SUSE
Manager in their scripts - the API version numbers are not consistent
(API version `25` in Uyuni is currently not the same as version `25`
in SUMA).

Second problem of this approach emerges, when the product developers
forget to bump the API version on breaking changes. In this case, even
a robust client application that thorougly checks the API version can
use an API method in a wrong way.

Tackling these problems has various solutions, described in the
[Solutions](#solutions) section below.


## API incompatibility handling on the client side

When the API consumers call a method in a wrong way (incorrect
parameters, missing method or namespace), they get back an XMLRPC
fault with code `-1`.
Note: This fault code is not reserved solely to report the API calling
mismatch cases, but is used in other cases (see
the`CustomInfoHandler.java` file in Uyuni).

```python
# Non-existing method/wrong method signature
<Fault -1: 'redstone.xmlrpc.XmlRpcFault: Could not find method: test in class: com.redhat.rhn.frontend.xmlrpc.ansible.AnsibleHandler with params: []'>

# Non-existing handler 
<Fault -1: 'The specified handler cannot be found'>
```

They can use this fault in a basic recovery mechanism from errors
caused by backwards-incompatible changes.


Together: api version, api list introspection and recovery.


## Breaking and non-breaking changes

The API mutates over time, in general there are 3 types of changes:

1. not breaking - growing: adding a new namespace, introducing a new
method in an existing namespace
2. breaking - shrinking: removing a namespace, removing a method from
an existing namespace
3. potentially breaking - modifying:
  3.1 breaking: changing a signature of an existing method
  3.2 breaking: changing a behavior of an existing method
  3.3 non breaking: adding a new field in a structure accepted /
  returned by a method
  3.4 breaking: removing a field in a structure accepted by a method /
  returned by a method


## [Solutions]

The following section contains a list of solutions to the problems
mentioned above.


### Solution 1

- Consider the API versions independent in Uyuni and SUSE
  Manager. These are 2 different projects and version `X` has a
  different contents and compatibility in Uyuni/SUMA.
- Introduce an API "flavor" under a new `api.getFlavor` method, which
  returns the project name (e.g. `uyuni`/`suse-manager`) read from a
  config file.
- Only "bump" the API versions on breaking changes.
- Increasing API versions within a minor SUMA release is forbidden
  (e.g. SUMA 4.2 API is always backwards-compatible, otherwise it's
  a bug)!
- We must write a CI job for watching introducing of breaking changes
  (it detects and report back any changes in the `xmlrpc` java
  package, unless a "API changes in this PR are non-breaking" checkbox
  is checked by the PR author)
- Minor note: Uyuni would typically have higher version number than
  SUMA as the changes there are more frequent, but in some cases (a
  SUMA maintenance update gets released before a new Uyuni release),
  this doesn't need to be true.
- API Consumers would need to introduce a check for the flavor in
  their scripts.

#### Pros
- Trivial implementation on server side

#### Cons
- Tracking non-breaking changes (to differentiate between various
  maintenance updates) is a bit more complex (need to use the API
  introspection calls, e.g. `getApiCallList`).
- Brings complexity to the API consumers and `spacecmd`. The scripts
  would need to check the flavor and the API version.
  
#### Usage instructions




### Solution 2: No flavor, use `systemVersion` instead
In this case, the API must be stable within a Uyuni/SUMA minor
version. (TODO @hustodemon: ask @moio/@mc i think this should be the
case already).

The user scripts could then use the `api.systemVersion` instead of
checking the flavor and API version.

#### Pros
- No implementation needed on the server side

#### Cons
- Version format: the scripts need to consider various corner cases
  (`4.2.0 RC2`, `2021.05`) and would need to make sure they are
  handled correctly by implementing various, possibly error-prone
  comparators in their code.
- Same problem with tracking non-breaking changes like in the
  Solution 1.


### Solution 3: Bump API version on any change in the API
- Same as Solution 1, but the version gets bumped on any (even
non-breaking) change within a product release or a maintenance update.
- Brekaing API within a SUMA maintenance update is still forbidden.

#### Pros & Cons
- Trivial implementation on server side
- Does not suffer from the first "Cons" of the Solution 1

#### Cons
- Brings complexity to the API consumers and `spacecmd`. The scripts
  would need to check the flavor and the API version.


### Solution 4: Use a `major.minor` versioning scheme
Bump the major version on breaking changes, break the minor one on
non-breaking changes.

TODO: enhance according to mc's comment!

Sol 1 + introspection has the same advantages.

#### Uyuni vs. SUMA


#### Pros
- Consumers able to track non-breaking changes too

#### Cons
- frequent changes in minor version
- does it solve the problem?


### Enhance the introspection calls
They would provide complete info about the parameters and the structure.
For each Map or Serializer in params.

## Enhance the reported exceptions

### Guidelines for writing API consuming applications
- TODO: write a guide for external ppl about the recommended ways to use the api?
  - super safe mode: use introspection + version check
    - should be case for spacecmd -> TODO: guide for devels?
  - pretty safe mode: use introspection
  - cowboy mode: don't use anything, just call it and catch exception


## Process for deprecation and removal API methods
- also breaking changes?

# Unresolved questions
- [ ] decide which solution to take
  - [ ] decide if to implement the CI job
- [ ] decide if to write the guidelines in th API > FAQ page

[1]: https://github.com/stevemeier/cefs/blob/master/errata-import.pl