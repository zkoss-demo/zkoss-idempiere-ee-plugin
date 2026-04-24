# iDempiere Plugin Version Upgrade Guide

This guide explains how to upgrade an existing iDempiere plugin to a new iDempiere version.
It uses `zkoss-idempiere-ee-plugin` as a concrete reference example (currently on iDempiere 12).

---

## Directory Layout Reference

```
zkoss-idempiere-ee-plugin/
├── parent-repository-pom.xml                      ← top-level parent POM
├── org.idempiere.zkee.comps.fragment/
│   ├── META-INF/MANIFEST.MF
│   ├── build.properties
│   ├── pom.xml
│   └── src/metainfo/zk/zk.xml
└── org.idempiere.zkee.comps.example/
    ├── META-INF/MANIFEST.MF
    ├── build.properties
    ├── pom.xml
    ├── pom-core-parent.xml                        ← alternate parent (direct core reference)
    ├── pom-targetplatform.xml                     ← alternate parent (target platform)
    └── idempiere.core.repository.target           ← Eclipse target platform definition
```

---

## Version Numbers to Track

Before starting, identify the following version numbers in the **new** iDempiere release:

| Variable | Example (v12) | Where to find it |
|---|---|---|
| `IDEMPIERE_VERSION` | `12.0.0` | iDempiere release notes / core MANIFEST.MF files |
| `PLUGIN_VERSION` | `12.0.1` | Your own plugin versioning (usually IDEMPIERE_VERSION + patch) |
| `ZK_CE_VERSION` | `10.0.1` | iDempiere's embedded ZK CE version. Check `org.idempiere.p2.targetplatform/base.target` or `org.adempiere.ui.zk/META-INF/MANIFEST.MF` in the new core |
| `ZK_EE_VERSION` | `10.0.1-Eval` | Compatible ZK EE eval release, usually the same upstream ZK version with `-Eval` appended (see [mavensync.zkoss.org](https://mavensync.zkoss.org)) |
| `TYCHO_VERSION` | `4.0.7` | Eclipse Tycho — only change if iDempiere itself upgrades |
| `JDK_VERSION` | `17` | Java version used by iDempiere |

---

## Step-by-Step Upgrade Procedure

### Step 1 — Update `parent-repository-pom.xml`

This is the single most important file; all child POMs inherit from it.

```xml
<properties>
    <!-- Change to new plugin version -->
    <revision>12.0.1-SNAPSHOT</revision>      <!-- UPDATE: X.Y.Z-SNAPSHOT -->

    <!-- Only change Tycho if iDempiere itself moved to a newer Tycho -->
    <tycho.version>4.0.7</tycho.version>

    <!-- Update if iDempiere now requires a newer JDK -->
    <jdk.version>17</jdk.version>
    <target.version>17</target.version>

    <!-- Path to iDempiere p2 repository (usually stays the same) -->
    <idempiere.core.repository.url>
        file:///${basedir}/../../idempiere/org.idempiere.p2/target/repository
    </idempiere.core.repository.url>
</properties>
```

**Checklist:**
- [ ] `<revision>` → new plugin version (e.g. `13.0.0-SNAPSHOT`)
- [ ] `<tycho.version>` → new Tycho if applicable
- [ ] `<jdk.version>` / `<target.version>` → new Java if applicable
- [ ] `<idempiere.core.repository.url>` → verify path is still correct relative to your workspace

---

### Step 2 — Update `META-INF/MANIFEST.MF` in Every Bundle

Each bundle/fragment has its own MANIFEST.MF. Update these fields:

#### 2a. Bundle version

```
Bundle-Version: 12.0.1.qualifier    →    X.Y.Z.qualifier
```

#### 2b. `Require-Bundle` versions (iDempiere core bundles)

```
Require-Bundle:
  org.adempiere.base;bundle-version="12.0.0"    →    "X.Y.0"
  org.adempiere.plugin.utils;bundle-version="12.0.0"
  org.adempiere.ui.zk;bundle-version="12.0.0"
  org.adempiere.ui;bundle-version="12.0.0"
  org.idempiere.zk.extra;bundle-version="12.0.0"
```

Use the iDempiere **major.minor.0** form unless the target iDempiere bundle declares a different exact version. Do not use the patch version of your plugin for core bundle requirements.

#### 2c. ZK CE bundle versions in `Require-Bundle`

These come bundled with iDempiere. Check the new iDempiere core for the ZK CE version it ships:

```
  zcommon;bundle-version="OLD_ZK_CE_VERSION"    →    "NEW_ZK_CE_VERSION"
  zel;bundle-version="OLD_ZK_CE_VERSION"
  zhtml;bundle-version="OLD_ZK_CE_VERSION"
  zk;bundle-version="OLD_ZK_CE_VERSION"
  zkbind;bundle-version="OLD_ZK_CE_VERSION"
  zkplus;bundle-version="OLD_ZK_CE_VERSION"
  zul;bundle-version="OLD_ZK_CE_VERSION"
  zweb;bundle-version="OLD_ZK_CE_VERSION"
```

#### 2d. Fragment-Host (for fragment bundles only)

```
Fragment-Host: org.adempiere.ui.zk;bundle-version="12.0.0"    →    "X.Y.0"
```

#### 2e. Execution environment (if JDK changed)

```
Bundle-RequiredExecutionEnvironment: JavaSE-17    →    JavaSE-XX
```

**Checklist per MANIFEST.MF:**
- [ ] `Bundle-Version`
- [ ] All `Require-Bundle` iDempiere entries
- [ ] All `Require-Bundle` ZK CE entries
- [ ] `Fragment-Host` (fragment bundles only)
- [ ] `Bundle-RequiredExecutionEnvironment` (if JDK changed)

---

### Step 3 — Update ZK EE Versions in `pom.xml` Files

Every `pom.xml` that references ZK EE artifacts must be updated.

#### In `org.idempiere.zkee.comps.example/pom.xml` — `<dependencies>` section:

```xml
<dependencies>
    <dependency>
        <groupId>org.zkoss.zk</groupId>
        <artifactId>zkbind</artifactId>
        <version>10.0.1-Eval</version>    <!-- UPDATE to new ZK EE version -->
    </dependency>
    <dependency>
        <groupId>org.zkoss.zk</groupId>
        <artifactId>client-bind</artifactId>
        <version>10.0.1-Eval</version>    <!-- UPDATE -->
    </dependency>
    <dependency>
        <groupId>org.zkoss.zk</groupId>
        <artifactId>zkmax</artifactId>
        <version>10.0.1-Eval</version>    <!-- UPDATE -->
    </dependency>
</dependencies>
```

#### In `org.idempiere.zkee.comps.fragment/pom.xml` — `<artifactItems>` section:

```xml
<!-- All org.zkoss.zk artifacts need updating -->
<artifactItem>
    <groupId>org.zkoss.zk</groupId>
    <artifactId>zkex</artifactId>
    <version>10.0.1-Eval</version>        <!-- UPDATE -->
</artifactItem>
<artifactItem>
    <groupId>org.zkoss.zk</groupId>
    <artifactId>zkmax</artifactId>
    <version>10.0.1-Eval</version>        <!-- UPDATE -->
</artifactItem>
<!-- ... zuti, client-bind, za11y — all use same ZK EE version -->
```

**Note on non-ZK dependencies:** `gson`, `javassist`, `jackson-*` versions are independent of iDempiere — only update if you have a specific reason (security patch, API change).

**Checklist:**
- [ ] All `<version>` tags for `org.zkoss.zk` group in every pom.xml

---

### Step 4 — Update Eclipse Target Platform File

**File:** `org.idempiere.zkee.comps.example/idempiere.core.repository.target`

```xml
<target name="idempiere-12-core-repository">    <!-- UPDATE name string for clarity -->
    <locations>
        <location path="../../idempiere/org.idempiere.p2/target/repository" type="Directory"/>
    </locations>
</target>
```

The `path` attribute is a relative path to the compiled iDempiere p2 repository. It typically does **not** change unless your workspace layout changed. However:
- Update the `name` attribute to reflect the new version (cosmetic but helpful).
- If the iDempiere repository folder name changed, update `path` accordingly.

**Checklist:**
- [ ] `name` attribute (cosmetic update)
- [ ] `path` attribute (verify path is still valid)

---

### Step 5 — Update Parent References in Alternate POM Files

The example plugin has two alternate POM files for different build modes.

#### `pom-core-parent.xml`

```xml
<parent>
    <groupId>org.idempiere</groupId>
    <artifactId>org.idempiere.parent</artifactId>
    <version>${revision}</version>
    <!-- revision is inherited from the workspace/command line — usually no change needed -->
    <relativePath>../../idempiere/org.idempiere.parent/pom.xml</relativePath>
</parent>
```

- Verify `<relativePath>` still points to the correct location in your iDempiere workspace.

#### `pom-targetplatform.xml`

```xml
<parent>
    <groupId>org.idempiere</groupId>
    <artifactId>org.idempiere.targetplatform.example.parent</artifactId>
    <version>${revision}</version>
    <relativePath>../parent-targetplatform-pom.xml</relativePath>
</parent>
```

- Verify `<relativePath>` still points correctly.
- If `parent-targetplatform-pom.xml` exists, apply the same `<revision>` update as in Step 1.

---

### Step 6 — Build and Verify

```bash
# From the workspace root, make sure iDempiere core is compiled first.
# This provides idempiere/org.idempiere.p2/target/repository.
cd idempiere
./mvnw clean install -DskipTests

# Then build this plugin
cd ../zkoss-idempiere-ee-plugin
mvn clean verify -f parent-repository-pom.xml
```

**Common build errors and fixes:**

| Error | Cause | Fix |
|---|---|---|
| `Cannot resolve bundle org.adempiere.base_X.Y.Z` | MANIFEST.MF version mismatch | Align Require-Bundle version with actual iDempiere version |
| `Artifact not found: org.zkoss.zk:zkex:NEW_VERSION` | ZK EE version not published yet | Check [mavensync.zkoss.org/zk/ee-eval](https://mavensync.zkoss.org/zk/ee-eval) for available versions |
| `Target platform cannot be resolved` | `.target` file path broken | Verify iDempiere p2 repo was built and path in `.target` is correct |
| `Package org.zkoss.* not accessible` | ZK CE version mismatch in MANIFEST | Align `Require-Bundle` ZK versions with what iDempiere ships |
| `Invalid URI file:${project_loc:...}` | `org.idempiere.p2.targetplatform.target` still contains Eclipse-workspace variables | Replace `${project_loc:org.idempiere.p2.targetplatform}` with the absolute path to your `org.idempiere.p2.targetplatform` directory in both `org.idempiere.p2.targetplatform.target` and `org.idempiere.p2.targetplatform.mirror.target`, then run `mvn install` in that directory (see [STEP_BY_STEP_GUIDE.md §3](STEP_BY_STEP_GUIDE.md)) |

---

## Form Registration Pattern

iDempiere has two form registration patterns. When upgrading, verify which pattern your plugin uses and ensure it still works correctly.

### Pattern A — `@Form` + `IFormController` (preferred)

The activator scans for `@Form`-annotated classes implementing `IFormController`:

```java
// MyActivator.java
Extensions.getMappedFormFactory().scan(context, "com.example.myplugin");

// ChartFormController.java  ← @Form goes HERE, not on CustomForm
@Form
public class ChartFormController implements IFormController {
    private final ChartForm chartForm;

    public ChartFormController() {
        chartForm = new ChartForm();
        Selectors.wireEventListeners(chartForm, this);
    }

    @Override
    public ADForm getForm() {
        return chartForm;
    }
}
```

The `@Form` annotation is from `org.idempiere.ui.zk.annotation.Form`, provided by the `org.idempiere.zk.extra` bundle — ensure it is in `Require-Bundle`.

### Pattern B — `@Form` on `CustomForm` (older style)

Some older plugins put `@Form` directly on the `CustomForm` subclass. This still works but Pattern A is cleaner since it separates the controller lifecycle from the view.

---

## Application Dictionary Registration (2Pack)

Custom forms must be registered in iDempiere's Application Dictionary (`AD_Form` + `AD_Menu`). The recommended approach is to package this as a **2Pack ZIP in `META-INF/`** so it is auto-imported when the plugin loads — no manual SQL required.

### 2Pack structure

```
META-INF/
└── 2Pack_1.0.0.zip
    ├── AD_Menu/
    │   ├── dict/PackOut.xml    ← AD_Form + AD_Menu definitions
    │   └── doc/AD_MenuDoc.xml  ← human-readable summary
```

### 2Pack can be created manually — no running iDempiere needed

The format is plain XML in a ZIP file. Generate two UUIDs (one for `AD_Form_UU`, one for `AD_Menu_UU`) and fill in the template:

**`AD_Menu/dict/PackOut.xml`:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<idempiere Name="AD_Menu" Version="1.0.0"
    idempiereVersion="13.0.0"
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
    <Name>My Plugin Form</Name>
    <Description/>
    <Action>X</Action>
    <AD_Window_ID reference="id"/>
    <AD_Workflow_ID reference="id"/>
    <AD_Task_ID reference="id"/>
    <IsActive>Y</IsActive>
    <IsSummary>N</IsSummary>
    <AD_Process_ID reference="id"/>
    <IsSOTrx>Y</IsSOTrx>
    <AD_Form_ID reference="uuid" reference-key="AD_Form">GENERATED-FORM-UUID</AD_Form_ID>
    <IsReadOnly>N</IsReadOnly>
    <EntityType>U</EntityType>
    <IsCentrallyMaintained>Y</IsCentrallyMaintained>
    <AD_Menu_UU>GENERATED-MENU-UUID</AD_Menu_UU>
    <AD_InfoWindow_ID reference="id"/>
    <PredefinedContextVariables/>
    <AD_Form type="table">
      <AD_Client_ID>0</AD_Client_ID>
      <AD_Org_ID>0</AD_Org_ID>
      <IsActive>Y</IsActive>
      <Name>My Plugin Form</Name>
      <Description>My Plugin Form</Description>
      <Help/>
      <Classname>com.example.myplugin.MyFormController</Classname>
      <AccessLevel>1</AccessLevel>
      <EntityType>U</EntityType>
      <IsBetaFunctionality>N</IsBetaFunctionality>
      <JSPURL/>
      <AD_Form_UU>GENERATED-FORM-UUID</AD_Form_UU>
      <AD_CtxHelp_ID reference="id"/>
      <ImageURL/>
    </AD_Form>
  </AD_Menu>
</idempiere>
```

**`AD_Menu/doc/AD_MenuDoc.xml`** (summary, required but content is informational):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<idempiere Name="AD_Menu">
  <AD_Menu>
    <Name>My Plugin Form</Name>
    <Action>X</Action>
    <AD_Form>
      <Name>My Plugin Form</Name>
      <Classname>com.example.myplugin.MyFormController</Classname>
    </AD_Form>
  </AD_Menu>
</idempiere>
```

**Zip these two files:**
```bash
cd /path/to/plugin/META-INF
mkdir -p 2pack_src/AD_Menu/{dict,doc}
# write PackOut.xml and AD_MenuDoc.xml into the folders above, then:
cd 2pack_src
zip -r ../2Pack_1.0.0.zip AD_Menu/
```

### When upgrading: update `idempiereVersion` in PackOut.xml

When upgrading to a new iDempiere version, update the `idempiereVersion` attribute in `PackOut.xml` to match the new version. The UUIDs stay the same — they identify the same form across versions.

### `AccessLevel` values

| Value | Meaning |
|---|---|
| `1` | Organization |
| `3` | Client |
| `6` | System/Client |
| `7` | All |

---

## Summary Checklist

| # | File | What to update |
|---|---|---|
| 1 | `parent-repository-pom.xml` | `<revision>`, optionally `<tycho.version>`, `<jdk.version>` |
| 2 | Every `META-INF/MANIFEST.MF` | `Bundle-Version`, `Require-Bundle` versions, `Fragment-Host` |
| 3 | Every `pom.xml` | ZK EE `<version>` in `<dependencies>` / `<artifactItems>` |
| 4 | `*.target` file | `name` attribute, verify `path` |
| 5 | Alternate POMs (`pom-core-parent.xml`, `pom-targetplatform.xml`) | Verify `<relativePath>` |
| 6 | `META-INF/2Pack_*.zip` → `PackOut.xml` | Update `idempiereVersion` attribute |
| 7 | Build | Run `mvn clean verify` and fix any resolution errors |

---

## Workspace Layout

See [WORKSPACE_LAYOUT.md](WORKSPACE_LAYOUT.md) for the full directory structure, path resolution details, and idempiere-dev-setup instructions.
