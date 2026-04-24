# org.idempiere.zkee.comps.fragment

Fragment that attaches ZK PE/EE jars to `org.adempiere.ui.zk` so PE/EE components and resources are available at runtime.

## Structure
- `META-INF/MANIFEST.MF` — declares the fragment host and bundle classpath (`lib/zkex-osgi.jar`, `lib/zkmax-osgi.jar`, `lib/gson.jar`).
- `build.properties` — packages `META-INF/` and `lib/` jars into the fragment.
- `pom.xml` — eclipse-plugin packaging; uses zk EE eval repo; copies `zkex-osgi`, `zkmax-osgi`, and `gson` into `lib/` during `validate`.
- `src/metainfo/zk/config.xml` — EE-specific zk configuration loaded after ZK EE addons.
- `lib/` — populated at build time with the ZK PE/EE jars.

## How to build
1) Place this project under the main idempiere source tree (sibling of other core modules).
2) Add it to the top-level `pom.xml` modules:
   ```xml
   <module>org.idempiere.zkee.components.fragment</module>
   ```
3) Run the build from the idempiere root (or in Eclipse with M2E):
   ```bash
   mvn -pl org.idempiere.zkee.components.fragment -am -DskipTests verify
   ```
   This copies the EE jars into `lib/` and produces `target/org.idempiere.zkee.components.fragment-<version>.jar` for installation.
