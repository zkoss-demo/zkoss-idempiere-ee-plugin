# iDempiere Plugin Creation Guide — ZK EE / ZK Addon Jar

This guide enables any AI Agent to create a new iDempiere plugin from scratch.
The plugin uses the **fragment + plugin** OSGi pattern to embed ZK EE or ZK addon jars.

---

## Required Inputs (collect from user before starting)

| Input | Example | Notes |
|---|---|---|
| `PLUGIN_ID` | `com.mycompany.myplugin` | Reverse-domain Java package style |
| `PLUGIN_NAME` | `My Plugin` | Human-readable display name |
| `IDEMPIERE_VERSION` | `13.0.0` | Target iDempiere version (major.minor.patch) |
| `PLUGIN_VERSION` | `13.0.0` | Plugin version, usually IDEMPIERE_VERSION + patch |
| `ZK_JAR_TYPE` | `ee-eval` \| `addon` | ZK EE evaluation jars or a custom addon jar |
| `ZK_EE_VERSION` | `10.0.1-Eval` | Required if ZK_JAR_TYPE = ee-eval; choose a version compatible with the ZK CE version bundled by iDempiere |
| `ZK_ADDON_ARTIFACT` | `zkcharts` | Required if ZK_JAR_TYPE = addon (Maven artifactId) |
| `ZK_ADDON_VERSION` | `10.0.1` | Required if ZK_JAR_TYPE = addon |
| `ZK_ADDON_GROUP` | `org.zkoss.zk` | Maven groupId of the addon jar |
| `ZK_ADDON_REPO_URL` | `https://mavensync.zkoss.org/maven2` | Maven repo URL for the addon |
| `WORKSPACE` | `/home/dev/idempiere-ws` | Absolute path to workspace root |
| `JAVA_VERSION` | `21` | JDK version. Use Java 17 for iDempiere 12 and Java 21 for iDempiere 13 |
| `VENDOR` | `My Company` | Bundle-Vendor field |

### Derived values (computed from inputs — do not ask user)

```
FRAGMENT_ID   = ${PLUGIN_ID}.fragment
FRAGMENT_DIR  = ${WORKSPACE}/${PLUGIN_ID}/${FRAGMENT_ID}
PLUGIN_DIR    = ${WORKSPACE}/${PLUGIN_ID}/${PLUGIN_ID}
PLUGIN_ROOT   = ${WORKSPACE}/${PLUGIN_ID}
IDEMPIERE_MAJOR_MINOR = first two segments of IDEMPIERE_VERSION (e.g., 12.0)
ZK_CE_VERSION = read from the built iDempiere core target platform or org.adempiere.ui.zk manifest
TYCHO_VERSION = read from ${WORKSPACE}/idempiere/org.idempiere.parent/pom.xml
JAVA_EXEC_ENV = JavaSE-${JAVA_VERSION}
```

### Version matrix

| Target iDempiere | Core branch | Required Java |
|---|---|---|
| 12.x | `release-12` | Java 17 |
| 13.x | `release-13` | Java 21 |

Do not mix Java levels across major iDempiere versions. Keep `JAVA_VERSION`, `Bundle-RequiredExecutionEnvironment`, Tycho compiler settings, and the JDK used to run Maven aligned with this table.

### Finding the ZK CE version

Do not hardcode the ZK CE version from memory. Read it from the iDempiere core you built:

```bash
rg 'id="zk"' ${WORKSPACE}/idempiere/org.idempiere.p2.targetplatform/*.target
rg 'zk;bundle-version=' ${WORKSPACE}/idempiere/org.adempiere.ui.zk/META-INF/MANIFEST.MF
```

Use that value for the plugin bundle's ZK `Require-Bundle` entries. For ZK EE eval artifacts, use the compatible EE eval version, usually the same upstream ZK version with `-Eval` appended.

### Finding the Tycho version

Do not hardcode the Tycho version if you are generating a plugin for a different iDempiere release. Read it from the iDempiere parent POM:

```bash
rg '<tycho.version>' ${WORKSPACE}/idempiere/org.idempiere.parent/pom.xml
```

Use the discovered value for `TYCHO_VERSION` in `parent-repository-pom.xml`.

---

## Workspace Layout

See [WORKSPACE_LAYOUT.md](WORKSPACE_LAYOUT.md) for the full directory structure, OSGi fragment rationale, and path resolution details.

---

## Step 0 — Set Up iDempiere Core with idempiere-dev-setup

Before creating any plugin, you need a built iDempiere core. The `idempiere-dev-setup` repository provides scripts that automate cloning and building iDempiere.

### 0a. Clone idempiere-dev-setup

```bash
cd ${WORKSPACE}
git clone https://github.com/idempiere/idempiere-dev-setup.git
```

### 0b. Clone and build iDempiere core

Run `setup.sh` **from the workspace root** (not from inside `idempiere-dev-setup/`). This clones iDempiere as `${WORKSPACE}/idempiere/` and runs the Maven build.

```bash
cd ${WORKSPACE}
# Skip DB setup and Eclipse workspace — only clone + build the core
./idempiere-dev-setup/setup.sh --skip-setup-db --skip-setup-ws \
    --branch release-${IDEMPIERE_MAJOR_MINOR}
```

Available `--branch` values:

| iDempiere version | Branch |
|---|---|
| 12.x | `release-12` |
| 13.x | `release-13` |
| latest (master) | omit `--branch` |

If you prefer a manual shallow clone instead of `idempiere-dev-setup`:
```bash
cd ${WORKSPACE}
git clone --branch release-13 --depth 1 https://github.com/idempiere/idempiere.git idempiere
cd idempiere && ./mvnw clean install -DskipTests
```

After the build completes, verify the p2 repository was produced:
```bash
ls ${WORKSPACE}/idempiere/org.idempiere.p2/target/repository/
# Should show: artifacts.jar  binary  content.jar  features  plugins
```

### 0c. Prerequisites check

```bash
java -version   # Java 17 for iDempiere 12, Java 21 for iDempiere 13
mvn -version    # must be >= 3.8.6
git --version
```

The `setup.sh` script has additional options — run `./idempiere-dev-setup/setup.sh --help` for the full list. Notable options:

| Option | Purpose |
|---|---|
| `--skip-setup-db` | Skip DB import (only needed when running iDempiere server) |
| `--skip-setup-ws` | Skip Eclipse workspace setup |
| `--branch <name>` | Checkout a specific branch (e.g. `release-13`) |
| `--repository-url <url>` | Use a different git remote (fork, mirror) |
| `--source <folder>` | Clone to a different folder name (default: `idempiere`) |

---

## Step 1 — Prerequisites

### 1a. iDempiere core must be built

```bash
cd ${WORKSPACE}/idempiere
./mvnw clean install -DskipTests
# Produces: ${WORKSPACE}/idempiere/org.idempiere.p2/target/repository/
```

If this directory does not exist, the plugin build will fail with "target platform cannot be resolved."

### 1b. Java version check

```bash
java -version   # Java 17 for iDempiere 12, Java 21 for iDempiere 13
mvn -version    # must be >= 3.8.6
```

---

## Step 2 — Create Directory Structure

```bash
mkdir -p ${PLUGIN_ROOT}
mkdir -p ${FRAGMENT_DIR}/META-INF
mkdir -p ${FRAGMENT_DIR}/lib
mkdir -p ${FRAGMENT_DIR}/src/metainfo/zk
mkdir -p ${PLUGIN_DIR}/META-INF
mkdir -p ${PLUGIN_DIR}/OSGI-INF
mkdir -p ${PLUGIN_DIR}/src/$(echo ${PLUGIN_ID} | tr '.' '/')
mkdir -p ${PLUGIN_DIR}/src/web
```

---

## Step 3 — Create `parent-repository-pom.xml`

**File:** `${PLUGIN_ROOT}/parent-repository-pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.idempiere</groupId>
  <artifactId>${PLUGIN_ID}.parent</artifactId>
  <version>${revision}</version>
  <packaging>pom</packaging>

  <properties>
    <revision>${PLUGIN_VERSION}-SNAPSHOT</revision>
    <tycho.version>${TYCHO_VERSION}</tycho.version>
    <jdk.version>${JAVA_VERSION}</jdk.version>
    <target.version>${JAVA_VERSION}</target.version>
    <idempiere.core.repository.url>file:///${basedir}/../../idempiere/org.idempiere.p2/target/repository</idempiere.core.repository.url>
  </properties>

  <repositories>
    <repository>
      <id>idempiere-core</id>
      <url>${idempiere.core.repository.url}</url>
      <layout>p2</layout>
    </repository>
    <repository>
      <id>Central</id>
      <url>https://repo1.maven.org/maven2</url>
    </repository>
  </repositories>

  <build>
    <plugins>
      <plugin>
        <groupId>org.eclipse.tycho</groupId>
        <artifactId>tycho-maven-plugin</artifactId>
        <version>${tycho.version}</version>
        <extensions>true</extensions>
      </plugin>
      <plugin>
        <groupId>org.eclipse.tycho</groupId>
        <artifactId>tycho-compiler-plugin</artifactId>
        <version>${tycho.version}</version>
        <configuration>
          <source>${jdk.version}</source>
          <target>${target.version}</target>
          <release>${jdk.version}</release>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.eclipse.tycho</groupId>
        <artifactId>target-platform-configuration</artifactId>
        <version>${tycho.version}</version>
        <configuration>
          <executionEnvironment>JavaSE-${JAVA_VERSION}</executionEnvironment>
          <environments>
            <environment>
              <os>linux</os><ws>gtk</ws><arch>x86_64</arch>
            </environment>
            <environment>
              <os>win32</os><ws>win32</ws><arch>x86_64</arch>
            </environment>
          </environments>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

**Replace all template variables** (`${PLUGIN_ID}`, `${PLUGIN_VERSION}`, `${JAVA_VERSION}`) with actual values.

---

## Step 4 — Create the Fragment Bundle

### 4a. `${FRAGMENT_DIR}/META-INF/MANIFEST.MF`

#### For ZK EE (`ZK_JAR_TYPE = ee-eval`)

```
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-Name: ${PLUGIN_NAME} Fragment
Bundle-SymbolicName: ${FRAGMENT_ID}
Bundle-Version: ${IDEMPIERE_VERSION}.qualifier
Bundle-Vendor: ${VENDOR}
Bundle-RequiredExecutionEnvironment: JavaSE-${JAVA_VERSION}
Bundle-ClassPath: .,
 lib/zkex.jar,
 lib/gson.jar,
 lib/zkmax.jar,
 lib/client-bind.jar,
 lib/jackson-databind.jar,
 lib/jackson-core.jar,
 lib/jackson-annotations.jar,
 lib/javassist.jar,
 lib/zuti.jar,
 lib/za11y.jar
Fragment-Host: org.adempiere.ui.zk;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0"
Eclipse-BundleShape: dir
Automatic-Module-Name: ${FRAGMENT_ID}
```

#### For ZK Addon (`ZK_JAR_TYPE = addon`)

```
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-Name: ${PLUGIN_NAME} Fragment
Bundle-SymbolicName: ${FRAGMENT_ID}
Bundle-Version: ${IDEMPIERE_VERSION}.qualifier
Bundle-Vendor: ${VENDOR}
Bundle-RequiredExecutionEnvironment: JavaSE-${JAVA_VERSION}
Bundle-ClassPath: .,
 lib/${ZK_ADDON_ARTIFACT}.jar
Fragment-Host: org.adempiere.ui.zk;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0"
Eclipse-BundleShape: dir
Automatic-Module-Name: ${FRAGMENT_ID}
```

**Note:** For addons that have transitive dependencies, add each jar to `Bundle-ClassPath` and list them in `pom.xml` `<artifactItems>`. Use `stripVersion=true` so jar names are stable (no version suffix).

### 4b. `${FRAGMENT_DIR}/build.properties`

#### For ZK EE

```properties
source.. = src/
output.. = target/classes/
bin.includes = META-INF/,\
               .,\
               lib/zkex.jar,\
               lib/gson.jar,\
               lib/zkmax.jar,\
               lib/client-bind.jar,\
               lib/jackson-databind.jar,\
               lib/jackson-core.jar,\
               lib/jackson-annotations.jar,\
               lib/javassist.jar,\
               lib/zuti.jar,\
               lib/za11y.jar
jre.compilation.profile = JavaSE-${JAVA_VERSION}
```

#### For ZK Addon

```properties
source.. = src/
output.. = target/classes/
bin.includes = META-INF/,\
               .,\
               lib/${ZK_ADDON_ARTIFACT}.jar
jre.compilation.profile = JavaSE-${JAVA_VERSION}
```

### 4c. `${FRAGMENT_DIR}/pom.xml`

> **Important (iDempiere 13+):** Before using `org.idempiere.parent` as the fragment parent, you must first convert the Eclipse-workspace variables in the target platform files to absolute paths (see [STEP_BY_STEP_GUIDE.md §3](STEP_BY_STEP_GUIDE.md)). The files `org.idempiere.p2.targetplatform.target` and `org.idempiere.p2.targetplatform.mirror.target` contain `${project_loc:org.idempiere.p2.targetplatform}` which causes standalone Maven builds to fail with `Invalid URI` errors. Replace these with the absolute path to your `org.idempiere.p2.targetplatform` directory, then reinstall with `mvn install` in that directory. After that, `org.idempiere.parent` works correctly as the fragment parent.

#### For ZK EE

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.idempiere</groupId>
    <artifactId>${PLUGIN_ID}.parent</artifactId>
    <version>${revision}</version>
    <relativePath>../parent-repository-pom.xml</relativePath>
  </parent>

  <artifactId>${FRAGMENT_ID}</artifactId>
  <packaging>eclipse-plugin</packaging>

  <repositories>
    <repository>
      <id>zkoss-ee-eval</id>
      <url>https://mavensync.zkoss.org/zk/ee-eval</url>
    </repository>
  </repositories>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <executions>
          <execution>
            <id>copy-zkee-jars</id>
            <phase>validate</phase>
            <goals><goal>copy</goal></goals>
            <configuration>
              <artifactItems>
                <artifactItem>
                  <groupId>org.zkoss.zk</groupId><artifactId>zkex</artifactId>
                  <version>${ZK_EE_VERSION}</version>
                </artifactItem>
                <artifactItem>
                  <groupId>org.zkoss.zk</groupId><artifactId>zkmax</artifactId>
                  <version>${ZK_EE_VERSION}</version>
                </artifactItem>
                <artifactItem>
                  <groupId>org.zkoss.zk</groupId><artifactId>client-bind</artifactId>
                  <version>${ZK_EE_VERSION}</version>
                </artifactItem>
                <artifactItem>
                  <groupId>org.zkoss.zk</groupId><artifactId>zuti</artifactId>
                  <version>${ZK_EE_VERSION}</version>
                </artifactItem>
                <artifactItem>
                  <groupId>org.zkoss.zk</groupId><artifactId>za11y</artifactId>
                  <version>${ZK_EE_VERSION}</version>
                </artifactItem>
                <artifactItem>
                  <groupId>com.google.code.gson</groupId><artifactId>gson</artifactId>
                  <version>2.10.1</version>
                </artifactItem>
                <artifactItem>
                  <groupId>org.javassist</groupId><artifactId>javassist</artifactId>
                  <version>3.29.2-GA</version>
                </artifactItem>
                <artifactItem>
                  <groupId>com.fasterxml.jackson.core</groupId><artifactId>jackson-databind</artifactId>
                  <version>2.15.2</version>
                </artifactItem>
                <artifactItem>
                  <groupId>com.fasterxml.jackson.core</groupId><artifactId>jackson-core</artifactId>
                  <version>2.15.2</version>
                </artifactItem>
                <artifactItem>
                  <groupId>com.fasterxml.jackson.core</groupId><artifactId>jackson-annotations</artifactId>
                  <version>2.15.2</version>
                </artifactItem>
              </artifactItems>
              <outputDirectory>lib</outputDirectory>
              <stripVersion>true</stripVersion>
              <overWriteReleases>true</overWriteReleases>
              <overWriteSnapshots>true</overWriteSnapshots>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

#### For ZK Addon

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.idempiere</groupId>
    <artifactId>${PLUGIN_ID}.parent</artifactId>
    <version>${revision}</version>
    <relativePath>../parent-repository-pom.xml</relativePath>
  </parent>

  <artifactId>${FRAGMENT_ID}</artifactId>
  <packaging>eclipse-plugin</packaging>

  <repositories>
    <repository>
      <id>zk-addon-repo</id>
      <url>${ZK_ADDON_REPO_URL}</url>
    </repository>
  </repositories>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <executions>
          <execution>
            <id>copy-addon-jar</id>
            <phase>validate</phase>
            <goals><goal>copy</goal></goals>
            <configuration>
              <artifactItems>
                <artifactItem>
                  <groupId>${ZK_ADDON_GROUP}</groupId>
                  <artifactId>${ZK_ADDON_ARTIFACT}</artifactId>
                  <version>${ZK_ADDON_VERSION}</version>
                </artifactItem>
              </artifactItems>
              <outputDirectory>lib</outputDirectory>
              <stripVersion>true</stripVersion>
              <overWriteReleases>true</overWriteReleases>
              <overWriteSnapshots>true</overWriteSnapshots>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### 4d. `${FRAGMENT_DIR}/src/metainfo/zk/zk.xml` (ZK EE only)

Only create this file if `ZK_JAR_TYPE = ee-eval`. It enables ZK Client MVVM binding.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<zk>
  <config-name>${PLUGIN_ID}-fragment</config-name>
  <depends>client-bind</depends>
  <listener>
    <listener-class>org.zkoss.clientbind.BinderPropertiesRenderer</listener-class>
  </listener>
  <!-- Required for iDempiere 13 with ZK Max 10.
       iDempiere sends login echo events to an initially invisible AdempiereWebUI window. -->
  <library-property>
    <name>org.zkoss.zkmax.au.IWBS.disable</name>
    <value>true</value>
  </library-property>
</zk>
```

---

## Step 5 — Create the Plugin Bundle

### 5a. `${PLUGIN_DIR}/META-INF/MANIFEST.MF`

```
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-Name: ${PLUGIN_NAME}
Bundle-SymbolicName: ${PLUGIN_ID}
Bundle-Version: ${PLUGIN_VERSION}.qualifier
Bundle-Vendor: ${VENDOR}
Bundle-RequiredExecutionEnvironment: JavaSE-${JAVA_VERSION}
Bundle-ActivationPolicy: lazy
Bundle-Activator: ${PLUGIN_ID}.MyActivator
Import-Package: org.osgi.framework;version="[1.10.0,2.0.0]",
 org.osgi.service.component.annotations;version="1.5.1"
Require-Bundle: org.adempiere.base;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0",
 org.adempiere.plugin.utils;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0",
 org.adempiere.ui.zk;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0",
 org.adempiere.ui;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0",
 org.idempiere.zk.extra;bundle-version="${IDEMPIERE_MAJOR_MINOR}.0",
 zcommon;bundle-version="${ZK_CE_VERSION}",
 zel;bundle-version="${ZK_CE_VERSION}",
 zhtml;bundle-version="${ZK_CE_VERSION}",
 zk;bundle-version="${ZK_CE_VERSION}",
 zkbind;bundle-version="${ZK_CE_VERSION}",
 zkplus;bundle-version="${ZK_CE_VERSION}",
 zul;bundle-version="${ZK_CE_VERSION}",
 zweb;bundle-version="${ZK_CE_VERSION}"
Service-Component: OSGI-INF/${PLUGIN_ID}.MyActivator.xml
Automatic-Module-Name: ${PLUGIN_ID}
```

### 5b. `${PLUGIN_DIR}/build.properties`

```properties
source.. = src/
output.. = target/classes/
bin.includes = META-INF/,\
               .,\
               OSGI-INF/
```

### 5c. `${PLUGIN_DIR}/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.idempiere</groupId>
    <artifactId>${PLUGIN_ID}.parent</artifactId>
    <version>${revision}</version>
    <relativePath>../parent-repository-pom.xml</relativePath>
  </parent>

  <artifactId>${PLUGIN_ID}</artifactId>
  <packaging>eclipse-plugin</packaging>

  <!-- Add ZK EE repo only if ZK_JAR_TYPE = ee-eval -->
  <repositories>
    <repository>
      <id>ZK Evaluation</id>
      <url>https://mavensync.zkoss.org/eval/</url>
    </repository>
    <repository>
      <id>ZK EE Evaluation</id>
      <url>https://mavensync.zkoss.org/zk/ee-eval</url>
    </repository>
  </repositories>

  <!-- Add ZK EE compile-time dependencies only if ZK_JAR_TYPE = ee-eval -->
  <dependencies>
    <dependency>
      <groupId>org.zkoss.zk</groupId>
      <artifactId>zkbind</artifactId>
      <version>${ZK_EE_VERSION}</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.zkoss.zk</groupId>
      <artifactId>client-bind</artifactId>
      <version>${ZK_EE_VERSION}</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.zkoss.zk</groupId>
      <artifactId>zkmax</artifactId>
      <version>${ZK_EE_VERSION}</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
</project>
```

For `ZK_JAR_TYPE = addon`: remove the ZK EE repositories and dependencies blocks. The addon classes are available at runtime via the fragment on the `org.adempiere.ui.zk` classloader.

### 5d. `${PLUGIN_DIR}/OSGI-INF/${PLUGIN_ID}.MyActivator.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<scr:component xmlns:scr="http://www.osgi.org/xmlns/scr/v1.1.0"
               immediate="true"
               name="${PLUGIN_ID}.MyActivator">
  <implementation class="${PLUGIN_ID}.MyActivator"/>
</scr:component>
```

### 5e. `${PLUGIN_DIR}/src/<package>/MyActivator.java`

Replace `.` in `${PLUGIN_ID}` with `/` for the file path.

```java
package ${PLUGIN_ID};

import org.adempiere.plugin.utils.Incremental2PackActivator;
import org.adempiere.webui.Extensions;
import org.osgi.framework.BundleContext;
import org.osgi.service.component.annotations.Component;

@Component(immediate = true)
public class MyActivator extends Incremental2PackActivator {

    public MyActivator() {}

    @Override
    public void start(BundleContext context) throws Exception {
        super.start(context);
        Extensions.getMappedFormFactory().scan(context, "${PLUGIN_ID}");
    }
}
```

`Incremental2PackActivator` imports any `META-INF/2Pack_*.zip` files packaged in the plugin bundle. This is the recommended way to register Application Dictionary records such as `AD_Form` and `AD_Menu`; avoid manual SQL for distributable plugins.

### 5f. `${PLUGIN_DIR}/META-INF/2Pack_1.0.0.zip` (recommended form registration)

Custom forms must be registered in iDempiere's Application Dictionary (`AD_Form` + `AD_Menu`). Package the registration as a 2Pack ZIP in `META-INF/` so the activator imports it when the plugin starts.

```
META-INF/
└── 2Pack_1.0.0.zip
    ├── AD_Menu/
    │   ├── dict/PackOut.xml
    │   └── doc/AD_MenuDoc.xml
```

Generate two UUIDs before filling the template:

```bash
uuidgen # AD_Form_UU
uuidgen # AD_Menu_UU
```

Create `AD_Menu/dict/PackOut.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<idempiere Name="AD_Menu" Version="1.0.0"
    idempiereVersion="${IDEMPIERE_VERSION}"
    DataBaseVersion="2025-01-01"
    Description="" Author="SuperUser"
    AuthorEmail="superuser @ idempiere.com"
    CreatedDate="2025-01-01" UpdatedDate="2025-01-01"
    PackOutVersion="100" UpdateDictionary="false"
    Client="0-SYSTEM-System"
    AD_Client_UU="11237b53-9592-4af1-b3c5-afd216514b5d">
  <AD_Menu type="table">
    <SeqNo>999</SeqNo>
    <AD_Client_ID>0</AD_Client_ID>
    <AD_Org_ID>0</AD_Org_ID>
    <Name>${PLUGIN_NAME}</Name>
    <Action>X</Action>
    <IsActive>Y</IsActive>
    <IsSummary>N</IsSummary>
    <AD_Form_ID reference="uuid" reference-key="AD_Form">GENERATED-FORM-UUID</AD_Form_ID>
    <EntityType>U</EntityType>
    <AD_Menu_UU>GENERATED-MENU-UUID</AD_Menu_UU>
    <AD_Form type="table">
      <AD_Client_ID>0</AD_Client_ID>
      <AD_Org_ID>0</AD_Org_ID>
      <IsActive>Y</IsActive>
      <Name>${PLUGIN_NAME}</Name>
      <Description>${PLUGIN_NAME}</Description>
      <Classname>${PLUGIN_ID}.MyFormController</Classname>
      <AccessLevel>7</AccessLevel>
      <EntityType>U</EntityType>
      <IsBetaFunctionality>N</IsBetaFunctionality>
      <AD_Form_UU>GENERATED-FORM-UUID</AD_Form_UU>
    </AD_Form>
  </AD_Menu>
</idempiere>
```

Create `AD_Menu/doc/AD_MenuDoc.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<idempiere Name="AD_Menu">
  <AD_Menu>
    <Name>${PLUGIN_NAME}</Name>
    <Action>X</Action>
    <AD_Form>
      <Name>${PLUGIN_NAME}</Name>
      <Classname>${PLUGIN_ID}.MyFormController</Classname>
    </AD_Form>
  </AD_Menu>
</idempiere>
```

Zip the 2Pack files:

```bash
cd ${PLUGIN_DIR}/META-INF
mkdir -p 2pack_src/AD_Menu/{dict,doc}
# Write PackOut.xml and AD_MenuDoc.xml into 2pack_src/AD_Menu/{dict,doc}, then:
cd 2pack_src
zip -r ../2Pack_1.0.0.zip AD_Menu/
cd ..
rm -rf 2pack_src
```

### 5g. `${PLUGIN_DIR}/src/<package>/MyFormController.java`

```java
package ${PLUGIN_ID};

import org.adempiere.webui.panel.ADForm;
import org.adempiere.webui.panel.IFormController;
import org.idempiere.ui.zk.annotation.Form;
import org.zkoss.zk.ui.select.Selectors;

@Form
public class MyFormController implements IFormController {
	private final MyForm myForm;

	public MyFormController() {
		myForm = new MyForm();
		Selectors.wireEventListeners(myForm, this);
	}

	@Override
	public ADForm getForm() {
		return myForm;
	}
}
```

### 5h. `${PLUGIN_DIR}/src/<package>/MyForm.java`

```java
package ${PLUGIN_ID};

import org.adempiere.webui.panel.CustomForm;
import org.zkoss.zk.ui.Component;
import org.zkoss.zk.ui.Executions;
import org.zkoss.zk.ui.select.Selectors;

public class MyForm extends CustomForm {
    private static final long serialVersionUID = 1L;

    public MyForm() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        try {
            // CRITICAL: DO NOT REMOVE THIS BLOCK.
            // ZK resolves "~./" resources through the thread context classloader.
            // Without this swap, iDempiere/ZK may look in org.adempiere.ui.zk
            // instead of this plugin bundle and fail to load the ZUL.
            Thread.currentThread().setContextClassLoader(getClass().getClassLoader());
            Component form = Executions.createComponents("~./my-form.zul", this, null);
            Selectors.wireComponents(form, this, false);
        } finally {
            Thread.currentThread().setContextClassLoader(cl);
        }
    }
}
```

**Important:** Always swap the context classloader before `Executions.createComponents()` and restore it in `finally`. This ensures ZK resolves `~./` paths from the plugin's classloader, not the caller's.

### 5i. `${PLUGIN_DIR}/src/web/my-form.zul`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<zk>
  <window title="${PLUGIN_NAME}" border="normal" width="600px">
    <label value="Hello from ${PLUGIN_NAME}!"/>
  </window>
</zk>
```

---

## Step 6 — Build

### 6a. Build fragment first

```bash
cd ${FRAGMENT_DIR}
mvn clean -U -DskipTests verify
# Output: target/${FRAGMENT_ID}-${IDEMPIERE_VERSION}.qualifier.jar
```

After `mvn validate` or `mvn verify`, verify that the downloaded jars match the fragment manifest and `build.properties`:

```bash
cd ${FRAGMENT_DIR}
find lib -maxdepth 1 -type f -name '*.jar' -printf '%f\n' | sort
rg 'lib/.*\.jar' META-INF/MANIFEST.MF build.properties
```

Every jar listed in `Bundle-ClassPath` and `bin.includes` must exist in `lib/`. If Maven downloaded an additional transitive jar that is required at runtime, add it to both files. If a listed jar is missing, fix the dependency-copy artifact list or the stable jar name produced by `stripVersion=true`.

### 6b. Build plugin

```bash
cd ${PLUGIN_DIR}
mvn clean -U -DskipTests verify
# Output: target/${PLUGIN_ID}-${PLUGIN_VERSION}.qualifier.jar
```

### 6c. Locate built JARs

```bash
FRAGMENT_JAR=$(ls ${FRAGMENT_DIR}/target/*.jar | grep -v sources)
PLUGIN_JAR=$(ls ${PLUGIN_DIR}/target/*.jar | grep -v sources)
echo "Fragment: $FRAGMENT_JAR"
echo "Plugin:   $PLUGIN_JAR"
```

---

## Step 7 — Deployment

### Option A — Auto-deploy script (recommended)

Create `${PLUGIN_ROOT}/deploy.sh`:

```bash
#!/bin/bash
# Usage: ./deploy.sh <idempiere-home>
# Example: ./deploy.sh /opt/idempiere

IDEMPIERE_HOME=${1:-/opt/idempiere}
PLUGINS_DIR="$IDEMPIERE_HOME/plugins"

if [ ! -d "$PLUGINS_DIR" ]; then
    echo "ERROR: plugins directory not found at $PLUGINS_DIR"
    echo "Usage: ./deploy.sh <idempiere-home>"
    exit 1
fi

FRAGMENT_JAR=$(ls ${FRAGMENT_DIR}/target/*.jar 2>/dev/null | grep -v sources | head -1)
PLUGIN_JAR=$(ls ${PLUGIN_DIR}/target/*.jar 2>/dev/null | grep -v sources | head -1)

if [ -z "$FRAGMENT_JAR" ] || [ -z "$PLUGIN_JAR" ]; then
    echo "ERROR: JARs not found. Run 'mvn clean verify' in fragment and plugin directories first."
    exit 1
fi

echo "Deploying to: $PLUGINS_DIR"
echo "  Fragment: $(basename $FRAGMENT_JAR)"
echo "  Plugin:   $(basename $PLUGIN_JAR)"

cp "$FRAGMENT_JAR" "$PLUGINS_DIR/"
cp "$PLUGIN_JAR" "$PLUGINS_DIR/"

echo "Done. Restart iDempiere or use Felix Console to install bundles."
```

```bash
chmod +x ${PLUGIN_ROOT}/deploy.sh
```

Run:
```bash
${PLUGIN_ROOT}/deploy.sh /path/to/idempiere-server
```

### Option B — Felix OSGi Console (hot-deploy, no restart)

1. Open `http://localhost:8080/osgi/system/console/bundles` (or port 8443 for HTTPS)
2. Click **Install/Update** button
3. Upload `${FRAGMENT_ID}-*.jar` → click **Install**
4. Upload `${PLUGIN_ID}-*.jar` → click **Install**
5. Start the plugin bundle (fragment starts automatically)

### Option C — Drop into plugins directory

```bash
# Copy both JARs to the iDempiere plugins folder
cp ${FRAGMENT_DIR}/target/${FRAGMENT_ID}-*.jar  $IDEMPIERE_HOME/plugins/
cp ${PLUGIN_DIR}/target/${PLUGIN_ID}-*.jar      $IDEMPIERE_HOME/plugins/
# Restart iDempiere server
```

---

## Step 8 — Verification

### 8a. Bundle state check (Felix Console)

Open `http://localhost:8080/osgi/system/console/bundles` and search for your plugin ID.

| Bundle | Expected state | Notes |
|---|---|---|
| `${FRAGMENT_ID}` | **Resolved** | Fragments never reach Active — this is correct |
| `${PLUGIN_ID}` | **Active** | Must be Active, not just Installed/Resolved |

If the plugin is **Resolved** but not **Active**, click the Start button.
If the plugin is **Installed**, there is a dependency resolution error — check the logs.

### 8b. Verify custom form registration

Use the 2Pack ZIP from Step 5f as the primary registration path. After the plugin starts, check the server log for the 2Pack import and then search for `${PLUGIN_NAME}` in the iDempiere menu/window search.

Manual SQL registration is not recommended for distributable plugins because it is harder to repeat across environments and upgrades.

### 8c. Server log check

```bash
# Look for bundle activation message
grep "${PLUGIN_ID}" $IDEMPIERE_HOME/log/console.log

# Look for errors
grep -i "error\|exception" $IDEMPIERE_HOME/log/console.log | grep "${PLUGIN_ID}"
```

### 8d. Common errors and fixes

| Error | Cause | Fix |
|---|---|---|
| Fragment stays **Resolved** | Expected — fragments don't activate | No action needed |
| Plugin shows **Installed** | Missing dependency | Check Felix console bundle details for unresolved imports |
| `ClassNotFoundException: org.zkoss.zk...` | Fragment not resolved to host | Verify `Fragment-Host` bundle-version matches deployed iDempiere |
| `Cannot resolve bundle org.adempiere.base_X` | `Require-Bundle` version mismatch | Align version in MANIFEST.MF with deployed iDempiere version |
| `Target platform cannot be resolved` | p2 repo not built | Run `./mvnw clean install` in `${WORKSPACE}/idempiere/` |
| ZK widget not found | lang-addon.xml missing from fragment | Add `src/metainfo/zk/lang-addon.xml` or check addon jar contains it |
| Form not visible in UI | 2Pack was not packaged or imported | Verify `META-INF/2Pack_*.zip` is inside the plugin jar and the activator extends `Incremental2PackActivator` |

---

## Reference: File Template Checklist

Every file to create, with what to substitute:

| File | Variables to substitute |
|---|---|
| `parent-repository-pom.xml` | PLUGIN_ID, PLUGIN_VERSION, JAVA_VERSION |
| `${FRAGMENT_ID}/META-INF/MANIFEST.MF` | FRAGMENT_ID, PLUGIN_NAME, IDEMPIERE_VERSION, IDEMPIERE_MAJOR_MINOR, VENDOR, JAVA_VERSION, jar list |
| `${FRAGMENT_ID}/build.properties` | JAVA_VERSION, jar list |
| `${FRAGMENT_ID}/pom.xml` | FRAGMENT_ID, ZK_EE_VERSION or ZK_ADDON_* |
| `${FRAGMENT_ID}/src/metainfo/zk/zk.xml` | PLUGIN_ID (ZK EE only) |
| `${PLUGIN_ID}/META-INF/MANIFEST.MF` | PLUGIN_ID, PLUGIN_NAME, PLUGIN_VERSION, IDEMPIERE_MAJOR_MINOR, VENDOR, JAVA_VERSION, ZK_CE_VERSION |
| `${PLUGIN_ID}/build.properties` | (no variables) |
| `${PLUGIN_ID}/pom.xml` | PLUGIN_ID, ZK_EE_VERSION (ZK EE only) |
| `${PLUGIN_ID}/OSGI-INF/*.xml` | PLUGIN_ID |
| `${PLUGIN_ID}/src/.../MyActivator.java` | PLUGIN_ID |
| `${PLUGIN_ID}/META-INF/2Pack_1.0.0.zip` | IDEMPIERE_VERSION, PLUGIN_ID, PLUGIN_NAME, generated UUIDs |
| `${PLUGIN_ID}/src/.../MyFormController.java` | PLUGIN_ID |
| `${PLUGIN_ID}/src/.../MyForm.java` | PLUGIN_ID |
| `${PLUGIN_ID}/src/web/my-form.zul` | PLUGIN_NAME |
| `deploy.sh` | FRAGMENT_DIR, PLUGIN_DIR, FRAGMENT_ID, PLUGIN_ID |
