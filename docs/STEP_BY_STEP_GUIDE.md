# Step-by-step Guide

## 1. Clone iDempiere Core

The first step is to clone the main iDempiere project, which provides the core libraries needed to build the plugins.

| Target iDempiere | Branch | Required Java |
|---|---|---|
| 12.x | `release-12` | Java 17 |
| 13.x | `release-13` | Java 21 |

```bash
# Clone iDempiere 13
git clone --branch release-13 https://github.com/idempiere/idempiere.git idempiere
```

This will create a directory named `idempiere` containing the iDempiere source code.

## 2. Build iDempiere Core

This creates a local p2 repository at `idempiere/org.idempiere.p2/target/repository`.

```bash
cd idempiere
./mvnw clean install
```

> **Before building your plugin or fragment:** always make sure your local iDempiere core is up-to-date. Core updates can change the embedded ZK CE version, which means both the plugin manifest ZK `Require-Bundle` entries and the ZK EE jars copied by the fragment must be reviewed. If those versions are inconsistent with the runtime, the fragment can load the wrong classes and you may see widget or classloader errors at runtime.
>
> ```bash
> cd idempiere
> git pull
> ./mvnw clean install
> ```
>
> After pulling, check the ZK CE version in the target platform files or in `org.adempiere.ui.zk/META-INF/MANIFEST.MF`:
>
> ```bash
> rg 'id="zk"' org.idempiere.p2.targetplatform/*.target
> rg 'zk;bundle-version=' org.adempiere.ui.zk/META-INF/MANIFEST.MF
> rg '<tycho.version>' org.idempiere.parent/pom.xml
> ```
>
> If ZK changed, update the ZK bundle versions in `org.idempiere.zkee.comps.example/META-INF/MANIFEST.MF` and update the `org.zkoss.zk` artifact versions in `org.idempiere.zkee.comps.fragment/pom.xml` to the compatible EE eval release (usually the same upstream ZK version with `-Eval` appended). If Tycho changed, update the plugin parent POM.

## 3. Point target platform files to absolute core paths, if needed

Some iDempiere target platform files contain Eclipse workspace variable references. Standalone Maven builds can fail with `Invalid URI file:${project_loc:...}` until those references are converted to absolute paths.

Edit these two files in a text editor:
- `idempiere/org.idempiere.p2.targetplatform/org.idempiere.p2.targetplatform.target`
- `idempiere/org.idempiere.p2.targetplatform/org.idempiere.p2.targetplatform.mirror.target`

Find the line containing:
```
${project_loc:org.idempiere.p2.targetplatform}
```

Replace it with the full absolute path to your directory, for example:
```
/Users/yourname/parent-folder/idempiere/org.idempiere.p2.targetplatform
```

## 4. Clone this Repository

Clone this repository into the same parent folder as iDempiere Core so both directories are siblings:

```bash
cd ..
git clone https://github.com/DevChu/zkoss-idempiere-ee-plugin.git
```

Your directory structure should look like:
```
parent-folder/
├── idempiere/
└── zkoss-idempiere-ee-plugin/
```

## 5. Add a ZK PE/EE fragment

Since we want the web UI to load ZK PE/EE widgets (e.g., from zkex and zkmax), use the fragment project `org.idempiere.zkee.comps.fragment`:

1) Build the fragment:
```bash
cd zkoss-idempiere-ee-plugin/org.idempiere.zkee.comps.fragment
mvn clean -U -DskipTests -am verify
```
   This runs the dependency-copy step and produces `org.idempiere.zkee.comps.fragment/target/org.idempiere.zkee.comps.fragment-<version>.jar`.
   After the validate phase, verify the copied jars match `META-INF/MANIFEST.MF` and `build.properties`:
```bash
find lib -maxdepth 1 -type f -name '*.jar' -printf '%f\n' | sort
rg 'lib/.*\.jar' META-INF/MANIFEST.MF build.properties
```
2) Install the fragment into your OSGi runtime (for example via Felix Web Console, or by placing the jar in the plugins directory) and restart the server so the host bundle (`org.adempiere.ui.zk`) resolves with the fragment on its classpath.
3) Confirm the fragment is **Resolved** and attached to the host; the ZK PE/EE widgets (defined in the embedded `zkex`/`zkmax` lang-addons) should render without "widget class required" errors.
4) If you use Client MVVM (`org.zkoss.clientbind.ClientBindComposer`), the fragment also ships `client-bind`, `zuti`, and `za11y` modules so those runtime classes/resources are available to the host bundle classloader.

### What is in `org.idempiere.zkee.comps.fragment`?

- Purpose: OSGi fragment that attaches ZK PE/EE and supporting jars to `org.adempiere.ui.zk`, exposing lang-addons, widgets, and resources required by `zkex`, `zkmax`, `client-bind`, `zuti`, and `za11y`.
- Key files:
  - `META-INF/MANIFEST.MF`: `Fragment-Host: org.adempiere.ui.zk`, `Bundle-ClassPath` includes `zkex`, `zkmax`, `client-bind`, `zuti`, `za11y`, and supporting jars.
  - `build.properties`: includes `META-INF/` and required `lib/*.jar` entries so they are packaged inside the fragment.
  - `pom.xml`: eclipse-plugin packaging; EE eval repository; dependency-copy execution to fetch required jars into `lib/` (version-stripped).
  - `src/metainfo/zk/zk.xml`: registers `org.zkoss.clientbind.BinderPropertiesRenderer` for Client MVVM setup.
  - `lib/zkex.jar`, `lib/zkmax.jar`, `lib/client-bind.jar`, `lib/zuti.jar`, `lib/za11y.jar`, `lib/gson.jar`, `lib/javassist.jar`, `lib/jackson-*.jar`: runtime binaries and dependencies.
  - `target/`: built outputs (`org.idempiere.zkee.comps.fragment-<version>.jar`, generated manifest, p2 metadata).
 
License Note: 
ZK EE is commercially licensed. This project uses the Evaluation Repository, which allows you to try ZK EE at no cost. When you are ready to use ZK EE in production, please obtain a valid license from ZK Framework and switch to the official ZK EE repository to access the licensed EE components.

## 6. Use ZK EE components in your own plugin (e.g., `org.idempiere.zkee.comps.example`)
 
1) Ensure the ZK EE fragment (`org.idempiere.zkee.comps.fragment`) is installed and resolved in the runtime; restart the server so `org.adempiere.ui.zk` resolves with the fragment on its classpath.
2) If your build cannot see the EE jar, add a dependency-copy step similar to the fragment (pulling the EE jar into `lib/`) or add the EE bundle to your target platform so Tycho can resolve it.
3) In ZUL, once the fragment is attached, you can directly use EE components (e.g., `<timepicker .../>`) because the lang-addon from the fragment registers them. For Client MVVM examples (`ClientBindComposer`), see `org.idempiere.zkee.comps.example/src/web/mvvm-example.zul`.
4) Build your plugin.
```bash
cd zkoss-idempiere-ee-plugin/org.idempiere.zkee.comps.example
mvn clean verify
```
Artifacts are written to `target/`.

## 7. Deploy it to iDempiere

1) Start the iDempiere runtime.
2) In the Apache Felix Web Console (`https://localhost:8443/osgi/system/console/`), open the **Bundles** page and use **Install/Start** to deploy the plugin and fragment.
3) Restart iDempiere Runtime to reload the fragment.
4) In **Bundles**, confirm the fragment is **Resolved** and the example plugin is **Active**. Fragments attach to a host bundle and do not become Active.
5) Log in with the SuperUser account.
6) Type "ZK EE" in the top-left search box and click "ZK EE Components Example" to open the plugin.
7) Confirm the timepicker component renders.
