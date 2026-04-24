# Workspace Layout

The workspace root can be **any directory** of your choice. The only constraint is that the plugin folder must be a **direct sibling** of the `idempiere/` core source folder.

```
<workspace>/
├── idempiere/                        ← iDempiere core source (cloned by dev-setup or manually)
│   ├── org.idempiere.p2/
│   │   └── target/repository/        ← p2 repository produced by Maven build
│   └── org.idempiere.parent/
│       └── pom.xml
├── idempiere-dev-setup/              ← setup scripts (optional sibling)
└── <your-plugin>/                    ← plugin repo (sibling of idempiere/)
    ├── parent-repository-pom.xml
    ├── <fragment-bundle>/            ← fragment bundle
    │   ├── META-INF/MANIFEST.MF
    │   ├── build.properties
    │   ├── pom.xml
    │   └── src/metainfo/zk/zk.xml   ← only needed for ZK EE (Client MVVM)
    └── <plugin-bundle>/              ← plugin bundle
        ├── META-INF/MANIFEST.MF
        ├── build.properties
        ├── pom.xml
        ├── OSGI-INF/
        └── src/
```

## Why fragment + plugin?

OSGi bundles have isolated classloaders. ZK discovers UI components by scanning the classpath of its host bundle (`org.adempiere.ui.zk`). A **fragment** is the only OSGi mechanism that can inject jars and resources into a host bundle's classloader — without it, ZK cannot find the addon widgets. The **plugin** bundle is the actual application code that uses those widgets.

```
org.adempiere.ui.zk (host)
  └── fragment attaches here → ZK sees addon jars + lang-addon.xml
        ← plugin depends on org.adempiere.ui.zk → transitively accesses addon classes
```

Fragments do not become `Active` in the OSGi console. A correctly attached fragment normally shows as `Resolved`; the host bundle and the plugin bundle should be `Active`.

## Why `../../idempiere/`?

In Maven, `${basedir}` evaluates to the **current module's own directory** when building. The `../../idempiere/` path in `parent-repository-pom.xml` is inherited by child bundles, and when a child bundle (e.g., `<your-plugin>/<bundle-a>/`) evaluates it:

```
<workspace>/<your-plugin>/<bundle-a>/   ← basedir during child build
  ../../                                ← go up twice → <workspace>/
  idempiere/                            ← iDempiere core ✓
```

The same logic applies to the `.target` file inside the bundle directory.

## Setting up with idempiere-dev-setup

Run `setup.sh` from the **workspace root** (not from inside `idempiere-dev-setup/`):

```bash
cd <workspace>
./idempiere-dev-setup/setup.sh
# This clones iDempiere to <workspace>/idempiere/
```

Then place (or clone) your plugin at `<workspace>/<your-plugin>/` — as a sibling of `idempiere/`.

## If your layout differs

If the plugin is not a direct sibling of `idempiere/`, update these two locations:

- `parent-repository-pom.xml` → `<idempiere.core.repository.url>` property
- `<bundle>/*.target` → `<location path="...">` attribute

Adjust the number of `../` segments to match your actual directory depth.

Also verify the ZK versions against the iDempiere core you actually built. The safest source is the built core target platform or `org.adempiere.ui.zk/META-INF/MANIFEST.MF`, not an older example value copied from documentation.
