# npm Best Practices Guide

This document aims to be a one-time stop explaining the security supply-chain
best practices when using npm's package manager. It is one of the workstreams
of the OSSF's [Best Practices for Open Source
Developers](https://github.com/ossf/wg-best-practices-os-developers) working
group. This document provides 1) an overview of security features of npm in the
context of supply-chain, 2) explicit recommendations and 3) details or links to
the official documentation to achieve these recommendations. This document is
intended to complement the official npm documentation, not to be an
alternative.

## TOC

- [npm Best Practices Guide](#npm-best-practices-guide)
  * [TOC](#toc)
  * [Dependency management](#dependency-management)
    + [Intake](#intake)
    + [Declaration](#declaration)
    + [Reproducible installation](#reproducible-installation)
      - [Vendoring dependencies](#vendoring-dependencies)
      - [Hash pinning](#hash-pinning)
        * [Package-lock.json](#package-lockjson)
        * [Shrinkwrap.json](#shrinkwrapjson)
    + [Maintenance](#maintenance)
  * [Release](#release)
    + [Account](#account)
    + [Signing and Verification](#signing-and-verification)
    + [Publishing](#publishing)
  * [Private packages](#private-packages)
    + [Scopes](#scopes)
    + [Private registry configurations](#private-registry-configurations)

## Dependency management

### Intake

The first step to using a dependency is to study its origin, trustworthiness
and security posture. Projects like [Envoy
proxy](https://github.com/envoyproxy/envoy) have [well-documented
criteria](https://github.com/envoyproxy/envoy/blob/main/DEPENDENCY_POLICY.md#new-external-dependencies)
a dependency must meet before it is used.

**Recommendations:**

1. **Be aware of typosquatting attacks**, when an attacker creates an
   official-looking package name to trick users into installing rogue packages
   ([1](https://threatpost.com/discord-stealing-malware-npm-packages/163265/),
   [2](https://blog.sonatype.com/damaging-linux-mac-malware-bundled-within-browserify-npm-brandjack-attempt),
   [3](https://blog.npmjs.org/post/163723642530/crossenv-malware-on-the-npm-registry)). Although
   the npm registry performs scans to detect typosquatting, no system is
   perfect, so stay vigilant. Identify the GitHub repository of the package and
   assess its trustworthiness through, for example, its number of contributors,
   stars, etc.

    1. Additionally, uppercase characters were previously allowed, which is another
       attack vector to be aware of. For example: [JSONStream](https://www.npmjs.com/package/JSONStream),
       [jsonstream](https://www.npmjs.com/package/jsonstream).

    1. Note: Non-ASCII characters are [not supported in npm package
       names](https://docs.npmjs.com/cli/v8/configuring-npm/package-json#name),
       so developers need not worry about homograph attacks, whereby an attacker
       names their package using non-ascii characters that render similar to an
       ascii character.

1. When you identify a GitHub project of interest, follow their documentation
   to identify the corresponding package name.

   Use [OpenSSF Security Scorecards](http://github.com/ossf/scorecard) to
   learn about the current security posture of the dependency.

1. Use [deps.dev](http://deps.dev) to learn about the security posture of
   transitive dependencies. You may list transitive dependencies by using
   [component analysis](https://github.com/microsoft/component-detection/).

1. Use [npm-audit](https://docs.npmjs.com/cli/v8/commands/npm-audit) to learn
   about existing vulnerabilities in the dependencies of the project.

**Warning**: Organization verification does not exist on the npm registry and
it's "first-come first serve." It is possible to challenge the owner of an org
for squatting after the fact using the [dispute
process](https://docs.npmjs.com/policies/disputes#beginning-the-process).

### Declaration

In npm, the
[package.json](https://docs.npmjs.com/cli/v6/configuring-npm/package-json)
describes a project (name, components, dependencies, etc.). Dependencies may be
defined by a package name, url, repo, etc. Additional constraints may be
defined for each dependency, such as which tags, branches, versions or
commitish are allowed.

**Note**: The manifest file ***does not*** list transitive dependencies; it
lists only direct project dependencies.

### Reproducible installation

A reproducible installation is one that guarantees the exact same copy the
dependencies are used each time the package is installed. This has various
benefits, including:

- Ensuring the dependencies installed are the ones declared and reviewed via
  pull requests.

- Helping quickly identify possible compromises of your infrastructure if one
  of your dependencies is found to have vulnerabilities, as you will be able to
  quickly determine the commit range when your repository was at risk.

- Mitigating certain threats such as malicious dependencies. A recent example
  is the [color package's
  sabotage](https://blog.sonatype.com/npm-libraries-colors-and-faker-sabotaged-in-protest-by-their-maintainer-what-to-do-now):
  without a reproducible installation, developers installed and ran a newly
  published (compromised) version of the dependency on their CI/CD system,
  giving an attacker immediate code execution on their infrastructure.

- Detecting package corruptions before installation, for example due to RAM
  corruption.

- Mitigating package mutability introduced by mis-configured registries that
  proxy packages from public registries. Note: Versions are immutable in
  principle, but immutability is enforced by the registry.

There are two ways to reliably achieve a reproducible installation: vendoring
dependencies and pinning by hash.

#### Vendoring dependencies

Vendoring dependencies means keeping a local copy of all the dependencies
(direct and transitive) in the repository. Note that updates to dependencies
may generate large pull requests in the repository's commit history.

#### Hash pinning

Hash pinning informs the package manager the expected hash for each dependency,
without trusting the registries. The package manager then verifies, during each
installation, that the hash of each dependency remains the same.

Npm provides two options to achieve hash pinning.

##### Package-lock.json

The
[package-lock.json](https://docs.npmjs.com/cli/v6/configuring-npm/package-lock-json)
contains ***all*** the dependencies (direct and indirect) along with a
cryptographic hash of their content:

```json
{
  "name": "A",
  "version": "0.1.0",
  ...metadata fields...
  "dependencies": {
    "B": {
      "version": "0.0.1",
      "resolved": "https://registry.npmjs.org/B/-/B-0.0.1.tgz",
      "integrity": "sha512-DeAdb33F+"
      "dependencies": {
        "C": {
          "version": "git://github.com/org/C.git#5c380ae319fc4efe9e7f2d9c78b0faa588fd99b4"
        }
      }
    }
  }
}
```

The package-lock.json file is a ***snapshot of an installation*** that allows
later reproduction of the same installation. As such, the lock file is
generated or updated via the various commands that install packages, e.g., `npm
install`. If some dependencies are missing or not pinned by hash (e.g.,
`integrity` field is not present), the installation will patch the lock file to
reflect the installation.

The lock file ***cannot*** be uploaded to a registry, which means that
consumers who locally install the package via `npm install` may [see different
dependency
versions](https://dev.to/saurabhdaware/but-what-the-hell-is-package-lock-json-b04)
than the repo owners used during testing. Using package-lock.json is akin to
dynamic linking for low-level programming languages: the loader will resolve
dependencies at load time using libraries available on the system. Using this
lock file leaves the task of deciding dependencies' versions to use to the
package consumer.

##### Shrinkwrap.json

[Shrinkwrap.json](https://docs.npmjs.com/cli/v7/configuring-npm/package-lock-json#package-lockjson-vs-npm-shrinkwrapjson)
is another lock file supported by npm. The main difference is that this lock
file ***may*** be uploaded to a registry along with the package. This ensures that
consumers of the dependency will obtain the same dependencies' hashes as the
repo owners intended. With shrinkwrap.json, the package producer takes
responsibility for updating the dependencies on behalf of their consumers. It's
akin to static linking for low-level programming languages: everything is
declared at packaging time.

To generate the shrinkwrap.json, an existing package-lock.json must be present
and the command [`npm
shrinkwrap`](https://docs.npmjs.com/cli/v8/commands/npm-shrinkwrap) must be
run.

**Recommendations:**

1. Developers should declare and commit a manifest file for ***all*** their
   projects. Use the official [Creating a package.json
   file](https://docs.npmjs.com/creating-a-package-json-file) documentation to
   create the manifest file.

1. To add a dependency to a manifest file, ***locally*** run [`npm
   install`](https://docs.npmjs.com/cli/v8/commands/npm-install) or [`npm
   install
   --package-lock-only`](https://docs.npmjs.com/cli/v8/commands/npm-install).

1. In automated environments (e.g., in CI or production) or when building
   artifacts for end-users (e.g., container images, etc):

    1. Always generate a package-lock.json ***locally*** and commit it to the
       repository.

    1. Never run commands that may update the lock files or fetch unpinned
       dependencies:

        1. `npm install` and `npm i`

        1. `npm update`

        1. `npm install-test`

        1. `npm pkg set` and `npm pkg delete`

        1. `npm exec`, `npx`

        1. `npm set-script`

    1. Only run commands that treat the lock file as read-only:

        1. To install a project and its dependencies, use [`npm
           ci`](https://docs.npmjs.com/cli/v8/commands/npm-ci).

        1. To run tests, run [`npm
           install-ci-test`](https://docs.npmjs.com/cli/v8/commands/npm-install-ci-test).

    1. If you need to run a CLI package `tool-cli` from the registry:

        1. Always declare it via a "wrapper" manifest file:
           ```json
           {
             "name": "wrapper-tool-cli",
             "version": "1.0.0",
             "dependencies": {
               "tool-cli": "4.1.1"
             }
           }
           ```

        1. ***Locally*** generate the corresponding lock file via `npm install
           --package-lock-only`

        1. Commit both files to the repository:

1. If a project is a CLI application (`main` entry in the manifest file),
   developers may publish a shrinkwrap.json.

1. If a project is a library (`lib` entry in the manifest file), a
   shrinkwrap.json should ***not*** be published. The reasoning is that version
   resolution should be left to the package consumer. Allow all versions from
   the minimum to the latest you support, e.g., `^m.n.o` to pin to a major
   range; `~m.n.o` to pin to a minor range. Avoid versions with critical
   vulnerabilities as much as possible. Visit the [semver
   calculator](https://semver.npmjs.com/) to help you define the ranges.

### Maintenance

It is important to update dependencies periodically, in particular when new
vulnerabilities are disclosed and patched. Tools that help with dependency
management are easy to use and may implement security checks for you.

**Recommendations:**

1. In order to manage your dependencies, use a tool such as
   [dependabot](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/configuring-dependabot-security-updates)
   or [renovatebot](https://github.com/renovatebot/renovate). These tools
   submit merge requests that you may review and merge into the default branch.

1. Stay up to date about existing vulnerabilities in your dependencies:

    1. If you have installed the tools above, enable security alerts: see
       [dependabot
       config](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-alerts-for-vulnerable-dependencies#dependabot-alerts-for-vulnerable-dependencies)
       and [renovatebot
       config](https://docs.renovatebot.com/configuration-options/#vulnerabilityalerts). Note
       that renovatebot follows config-as-code, so it makes it easier for
       external users to verify it is enabled.

    1. If you prefer a dedicated tool, run
       [npm-audit](https://docs.npmjs.com/cli/v8/commands/npm-audit)
       periodically, e.g., in a GitHub workflow.

1. In order to remove dependencies, periodically run [`npm
   prune`](https://docs.npmjs.com/cli/v8/commands/npm-prune) and submit a merge
   request. Tools above do not support this feature yet, and we are not aware
   of a GitHub action for this feature.

## Release

### Account

Publishing on the npm registry requires creating a user account.

**Recommendations:**

1. Developers should [enable
  2FA](https://docs.npmjs.com/configuring-two-factor-authentication) on their
  account.

### Signing and Verification

npm packages are all signed by a [PGP
key](https://docs.npmjs.com/about-pgp-signatures-for-packages-in-the-public-registry)
owned by the default npm registry.

**Warning**: npm does not support user-level signatures.

### Publishing

Publishing a package uploads it to a registry for others to install and
download.

**Recommendations:**

1. Use an [Automation
   token](https://docs.npmjs.com/creating-and-viewing-access-tokens) to
   authenticate and publish to the default npm registry.

1. Release your package using the commands
   ```
   npm ci
   npm publish
   ```
   through a GitHub workflow as explained in the [official GitHub
   documentation](https://docs.github.com/en/actions/publishing-packages/publishing-nodejs-packages).

1. Consumers of public packages may fall victim to typosquatting attempts. To
   mitigate this problem, create and own your organization on other registries.

**Note**:

1. The default npm registry does not support the default GitHub token for
   authentication, so users need to save it as a [GitHub
   secret](https://docs.github.com/en/actions/publishing-packages/publishing-nodejs-packages#publishing-packages-to-the-npm-registry).

1. There is no official GitHub action to publish an npm package.

1. [CIDR-scoped
   tokens](https://docs.npmjs.com/creating-and-viewing-access-tokens#cidr-restricted-token-errors)
   are scoped by IP range for additional protection, but can only be created
   with the npm CLI.

## Private packages

Dependency confusion attacks, a.k.a substitution attacks, occur when a project
relies on private packages from an internal registry. An attacker may register
the same package name on public registries. By default, npm will fetch the
dependency from the public registry—see details of recent attacks in Alex
Birsan's
[blog](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610). To
alter this behavior so npm fetches the correct dependency, use scopes.

### Scopes

A "scope" is a @-prefixed name that goes at the start of a package name. For
example, `@myorg/foo` is a scoped package. Scoped packages are used like any
other packages in package.json and in Javascript code:

```json
{
  "name": "@myorg/foo",
  "version": "1.2.3",
  "description": "just a scoped package name example",
  "dependencies": {
    "@myorg/bar": "2.x"
  }
}
```

```js
// es modules style
import foo from '@myorg/foo'

// commonjs style
const foo = require('@myorg/foo')
```

Scoped packages on the public npm registry may only be published by the user or
organization associated with it, and packages within that scope may be made
private. Also, scope names can be linked to a given registry.

**Recommendations:**

1. On the public npm registry, [create a free
   organization](https://docs.npmjs.com/creating-an-organization) with the
   “myorg” name. At that point, no one else can publish anything under the
   `@myorg` scope on the public registry, and your builds will fail with `404
   errors` if they’re misconfigured, rather than silently fetching untrusted
   content. This is an important step, because it prevents an attacker from
   hijacking your organization’s name in the public registry, which could
   result in the exact same problems.

1. Locally, use the login command to ensure that all requests for packages
   under the `@myorg` scope will be made to the `https://registry.myorg.local`
   registry. Any other requests that are not bound to a scope will go to the
   default registry.

   ```
   npm login --scope=myorg --registry=https://registry.myorg.local
   ```

   This saves the login information in your `~/.npmrc` file as follows:

   ```js
   @myorg:registry = https://registry.myorg.local/
   //registry.myorg.local:_authToken = xyzabc123-arbitrary-token-value
   ```
1. Do not commit this file `~/.npmrc` with credentials to a repository.

1. In automated environments, provision the secret automatically. Follow a
   similar solution as [GitHub uses in
   workflows](https://docs.github.com/en/actions/publishing-packages/publishing-nodejs-packages). If
   your registry supports ephemeral credentials, use it instead of using
   long-term secrets/tokens.

1. Create a `.npmrc` file in the root of your projects, with a line like this

   ```js
   @myorg:registry = https://registry.myorg.local/
   ```

   You can check the currently configured registry at any time by running this command

   ```
   npm config get registry
   ```

   By doing this, npm will associate the scope with your internal registry when working with that project.

For more information on the topic, see [npm's substitution attack blog
post](https://github.blog/2021-02-12-avoiding-npm-substitution-attacks/).

### Private registry configurations

If you use internal private registries:

1. Only use private registry solutions that support scopes

1. Make your private packages immutable:

    1. Ensure your registry is not configured to “merge” manifests of the same
       name from the upstream public registry. This is sometimes enabled to
       work around resolution collisions, but it is a very bad idea, precisely
       because “work around resolution collisions” is how name hijacking
       exploits work. If possible, use a private registry implementation that
       doesn’t even have this feature.

    1. Once a package is published to the internal proxy registry, it is very
       important that you do not silently fall back to the public registry if
       that package is ever removed.  If the proxy starts serving this package
       name from the public registry, you are back in a situation where an
       attacker can take over the name and gain access to any systems that are
       left behind.

1. Do not ignore build failures. If you configure your projects as above, you
will tend to see a 404 error rather than npm fetching untrusted content. Do not
ignore these errors! Configure your systems to crash as loudly as possible if a
build fails, and fix it right away when this happens.

For more information on the topic, see [npm's substitution attack blog
post](https://github.blog/2021-02-12-avoiding-npm-substitution-attacks/).
