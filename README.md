# Building the iDempiere ZK EE Components Example Plugin
By default, iDempiere uses ZK CE as its UI framework. To leverage advanced components and features available in ZK EE, an additional ZK EE plugin is required. This document provides a step-by-step guide on how to build the `org.idempiere.zkee.comps.example` plugin from the `zkoss-idempiere-ee-plugin` repository.

For general iDempiere plugin development guidelines, refer to the [iDempiere Wiki](https://wiki.idempiere.org/en/Developing_Plug-Ins_-_Get_your_Plug-In_running).

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

Building iDempiere plugins requires having the iDempiere core libraries available as a local p2 repository. This guide will walk you through the process of setting up the necessary dependencies and building the plugin.

## Prerequisites

Before you begin, ensure you have the following tools installed:

-   **Git:** For cloning the iDempiere repository.
-   **Maven:** For building the projects.
-   **Java Development Kit (JDK):** Version 17 or higher.
-   **iDempiere Runtime**: An active instance (e.g., [Official Docker Image](https://hub.docker.com/r/idempiereofficial/idempiere)).

## Step-by-step Guide

See the [Step-by-step Guide](docs/STEP_BY_STEP_GUIDE.md) for full build and deployment instructions.

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
- [iDempiere Wiki - Make ZK WebApp OSGi](https://wiki.idempiere.org/en/Make_Zk_WebApp_OSGi) - iDempiere OSGi architecture
