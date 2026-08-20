# iDempiere ZK EE Components Plugin
By default, iDempiere uses ZK CE as its UI framework. To leverage advanced components and features available in ZK EE, an additional ZK EE plugin is required. This repository provides that plugin, as a fragment plus an example form, and documents the pattern so you can build your own.

For general iDempiere plugin development guidelines, refer to the official [Developing Plug-Ins](https://docs.idempiere.org/docs/category/developing-plug-ins) documentation.

## Introduction

This repository demonstrates how to create an iDempiere plugin that uses ZK EE components.

This branch has been tested with **iDempiere 13**.

If you want to build the plugin for **iDempiere 12**, please check out the [`v12` branch](https://github.com/zkoss-demo/zkoss-idempiere-ee-plugin/tree/v12).

We assume readers:
- Know the iDempiere basics
- Know the ZK framework basics

With this project you can:
- Build a ZK plugin with ZK EE components and install it into iDempiere
- Follow the example project to create your own plugin with ZK EE components

## iDempiere 13 Highlights

- Added runtime modules to the fragment: `client-bind`, `zuti`, and `za11y`.
- Enabled Client MVVM setup through fragment-level ZK configuration (`BinderPropertiesRenderer`).
- Disabled ZK EE's inaccessible widget block service in the fragment configuration for compatibility with the iDempiere login flow.

## Installing

You need a running iDempiere 13 instance - for example the
[official Docker image](https://hub.docker.com/r/idempiereofficial/idempiere).

Prebuilt jars are attached to each [release](https://github.com/zkoss-demo/zkoss-idempiere-ee-plugin/releases), so nothing has to be built to try the
plugin:

| File | Purpose |
|---|---|
| `org.idempiere.zkee.comps.fragment-13.0.0.jar` | The ZK EE fragment. **Required.** |
| `org.idempiere.zkee.comps.example-13.0.0.jar` | The example form. Optional. |

**The order matters.** A fragment only takes effect once its host bundle is re-resolved, which
means a restart:

1. Install the fragment through the Felix Web Console (`/osgi/system/console/bundles`).
2. **Restart the iDempiere runtime.** Besides re-resolving the host, ZK reads the fragment's
   `zk.xml` library properties during web application initialization, so hot-deploying alone will
   not enable Client MVVM.
3. Install the example plugin if you want it.
4. On the **Bundles** page, confirm the fragment shows as *Fragment* and the example plugin as
   *Active* - not merely *Resolved*.
5. Log out and back in, so the menu tree is rebuilt with the entry the plugin adds.

For the other ways to get a bundle into a runtime - p2 repository, Gogo shell, dropping it into
the plugins directory - see iDempiere's [Distributing and Installing Plug-ins](https://docs.idempiere.org/docs/basic-development/plugin-development/distributing-plugins).

To build the jars yourself instead, see [Building from source](#building-from-source) at the end
of this file.

---

## Appendix: Why Fragment is Needed

### The Technical Reason

| Constraint | Explanation |
|------------|-------------|
| **OSGi classloaders** | Each OSGi bundle has its own classloader - bundles are isolated |
| **ZK's lang-addon.xml** | ZK discovers components via `metainfo/zk/lang-addon.xml` using the **host bundle's classloader** |
| **Fragment behavior** | A fragment shares the **same classloader** as its host bundle |

**Result**: To make `org.adempiere.ui.zk` "see" the ZK EE widgets (`zkex.jar`, `zkmax.jar`), those JARs must be on its classloader. A **fragment** is the only OSGi-compliant way to inject resources into another bundle's classloader without modifying the host.

### Architecture Flow

```
ZK EE widgets need to be discovered by ZK's classloader
    ↓
ZK runs inside org.adempiere.ui.zk bundle
    ↓
OSGi bundles have isolated classloaders
    ↓
Only a FRAGMENT can share the host's classloader
    ↓
Therefore: Fragment is required
```

### References

- [OSGi vogella blog](https://vogella.com/blog/osgi-bundles-fragments-dependencies/) - "A fragment is loaded in the same classloader as the host"
- [bnd Fragment-Host docs](https://bnd.bndtools.org/heads/fragment_host.html) - "A fragment is a bundle that is attached to a host bundle"
- [iDempiere docs - OSGi and MANIFEST.MF Background](https://docs.idempiere.org/docs/basic-development/plugin-development/plugin-development-background) - how iDempiere uses OSGi and what
  each `MANIFEST.MF` attribute does
- [iDempiere wiki - Make ZK WebApp OSGi](https://wiki.idempiere.org/en/Make_Zk_WebApp_OSGi) -
  how ZK is wired into iDempiere as an OSGi webapp. Still only on the legacy wiki; the new
  documentation site has no equivalent page yet

## Prerequisites for building

Before you begin, ensure you have the following tools installed:

-   **Git:** For cloning the iDempiere repository.
-   **Maven:** For building the projects.
-   **Java Development Kit (JDK):** Version 17 or higher.
-   **iDempiere Runtime**: An active instance (e.g., [Official Docker Image](https://hub.docker.com/r/idempiereofficial/idempiere)).

## Building from source

Only needed if you want to change the code. To install and try the plugin, download the jars
attached to a release, as described under [Installing](#installing).

Building iDempiere plugins requires the iDempiere core libraries as a local p2 repository.

See the [Step-by-step Guide](docs/STEP_BY_STEP_GUIDE.md) for full build instructions, and the
[plugin creation guide](docs/IDEMPIERE_NEW_PLUGIN_GUIDE.md) for building your own plugin around
this pattern.

### Publishing a release

`./make-release.sh` collects every built jar into a local `release/` folder, dropping the
`-SNAPSHOT` from the file name. That folder is git-ignored: upload its contents as attachments
on the GitHub release page rather than committing them.

The file name comes from the Maven version, which stays `13.0.0-SNAPSHOT`; the version OSGi
actually uses is the `Bundle-Version` inside the jar, where `.qualifier` has been expanded to
a build timestamp such as `13.0.0.202608190118`. Every build therefore gets a distinct OSGi
version - Felix sees a rebuild as newer and updates it - while the published file stays
`...-13.0.0.jar`.

## License

**The plugin source code** in this repository is licensed under the
[GNU General Public License v2.0](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html) or
later, the same license iDempiere itself uses. See [LICENSE.md](LICENSE.md).

**The source tree contains no ZK binaries.** The fragment declares the ZK EE jars as Maven
dependencies; the build downloads them from ZK's Evaluation repository into a git-ignored `lib/`.

**The released jars are a different matter.** A fragment's whole job is to put those jars on the
host bundle's class loader, so the built fragment embeds them:

| Released jar | Contains | Redistributable under the GPL? |
|---|---|---|
| `org.idempiere.zkee.comps.example` | Only this project's own compiled code | Yes - GPLv2 or later |
| `org.idempiere.zkee.comps.fragment` | 11 jars, ~7.5 MB: **ZK EE Evaluation** (`zkex`, `zkmax`, `client-bind`, `zuti`, `za11y`) plus `gson`, `javassist` and Jackson | **No** - the ZK jars are governed by ZK's license, not the GPL; the rest keep their own open-source licenses |

| Component | License |
|---|---|
| This project's source and its own compiled bundle | GPLv2 or later |
| ZK EE (`zkex`, `zkmax`, `client-bind`, `zuti`, `za11y`) | Commercial - Evaluation builds are used here |
| `gson`, `javassist`, `jackson-*` | Their own open-source licenses, redistributed unmodified |
| ZK CE (in iDempiere) | LGPL - already part of iDempiere, unchanged by this plugin |

**What you may do with a downloaded fragment.** Install it and evaluate it. The Evaluation
binaries inside are provided for evaluation only: they are not yours to redistribute, and a valid
ZK EE license or subscription is required before use in production. Contact <info@zkoss.org>.

**One caveat when backing out.** Removing the fragment returns the runtime to plain ZK CE and
leaves iDempiere core untouched - but any form *you* wrote against an EE component stops working
with it, so plan that boundary deliberately.

---

## See also

Three repositories bring ZK commercial products into iDempiere. All three use the same
OSGi **fragment + plugin** pattern described in the appendix above; they differ only in
which jars the fragment carries.

| Repository | Brings in | Start there when you want |
|---|---|---|
| **This repository** | ZK EE (`zkex`, `zkmax`, `client-bind`, `zuti`, `za11y`) | The general ZK EE component set. Its [new-plugin guide](docs/IDEMPIERE_NEW_PLUGIN_GUIDE.md) is the reference the other two repositories follow |
| [zkoss-idempiere-zkcharts-plugin](https://github.com/zkoss-demo/zkoss-idempiere-zkcharts-plugin) | ZK Charts, ZK Pivottable | Charts and pivot tables - including replacing iDempiere's built-in chart rendering globally |
| [zkoss-idempiere-keikai-plugin](https://github.com/zkoss-demo/zkoss-idempiere-keikai-plugin) | Keikai Spreadsheet (`keikai`, `keikai-ex`, `keikai-pdf`) | An Excel-compatible spreadsheet inside an iDempiere form |

The three fragments target the same host bundle, `org.adempiere.ui.zk`. OSGi allows a host
any number of fragments, so they can be installed side by side - but check for overlap
first: the Keikai fragment carries its own `zkex` and `zkcharts`, and not necessarily at
the same versions as the other two fragments ship. Each fragment needs a restart to attach.
