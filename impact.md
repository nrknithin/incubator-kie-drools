# Quarkus Upgrade Impact Analysis: 3.20.3 -> 3.27 LTS

## Executive Summary
This document analyzes the impact of upgrading the Quarkus version from **3.20.3** to **3.27 LTS** within the `incubator-kie-drools` repository. The upgrade is primarily isolated to the `drools-quarkus-extension` modules and their associated integration tests.

## Current State
- **Current Version:** 3.20.3
- **Definition Location:** `build-parent/pom.xml` (property `<version.io.quarkus>`)
- **Dependency Management:** `drools-quarkus-extension/pom.xml` imports `io.quarkus:quarkus-bom`.

## Impacted Modules

The following modules have direct dependencies on Quarkus and will be affected:

### Core Extensions (Runtime & Deployment)
These modules contain the actual extension logic. Changes in Quarkus SPIs (Build Items, Recorders) between 3.20 and 3.27 could require code changes here.
- `drools-quarkus-extension/drools-quarkus`
- `drools-quarkus-extension/drools-quarkus-deployment`
- `drools-quarkus-extension/drools-quarkus-ruleunits`
- `drools-quarkus-extension/drools-quarkus-ruleunits-deployment`
- `drools-quarkus-extension/drools-quarkus-util-deployment`

### Integration Tests & Examples
These modules consume the extension and will verify the upgrade's success.
- `drools-quarkus-extension/drools-quarkus-integration-test`
- `drools-quarkus-extension/drools-quarkus-integration-test-hotreload`
- `drools-quarkus-extension/drools-quarkus-integration-test-kmodule`
- `drools-quarkus-extension/drools-quarkus-integration-test-multimodule`
- `drools-quarkus-extension/drools-quarkus-quickstart-test`
- `drools-quarkus-extension/drools-quarkus-ruleunit-integration-test`
- `drools-quarkus-extension/drools-quarkus-examples`

## Critical Dependency Updates & Breaking Changes (3.20 -> 3.27)

### 1. Hibernate ORM & Reactive
- **Upgrade:** Quarkus 3.27 includes major Hibernate updates (likely Hibernate ORM 6.6+ or 7.x).
- **Impact:**
    - **Combined Usage:** 3.27 allows combined usage of ORM and Reactive in the same persistence unit.
    - **Configuration Renaming:** Many `quarkus.hibernate-orm.*` properties have been renamed to simplify YAML configuration.
        - Example: `quarkus.hibernate-orm.database.generation` is renamed.
        - Old properties are **deprecated** but might still work. It is cleaner to update them.
    - **Dialect Config:** Offline startup and dialect configuration improvements.

### 2. Property Renaming & Removals
- **Test Configuration:**
    - `quarkus.test.native-image-profile` is **REMOVED**.
    - **Action:** Replace with `quarkus.test.integration-test-profile`.
- **Datasources:**
    - `quarkus.datasource.reactive.url` no longer has an implicit default. It must be explicitly set or the datasource will be deactivated.
- **Transaction Manager:**
    - `quarkus.transaction-manager.object-store-directory` -> `quarkus.transaction-manager.object-store.directory` (occurred in 3.2, but checking for legacy config is good).
- **RESTEasy Reactive -> Quarkus REST:**
    - Fallbacks for old artifact names/config properties (from 3.15 rebranding) were **REMOVED** in 3.20+. Since we are on 3.20.3, we should be safe, but any lingering usage of `quarkus.resteasy.reactive.*` should be `quarkus.rest.*`.

### 3. API Changes
- **Transactions:** `QuarkusTransaction.isActive()` is **deprecated**. Use `getStatus()`.
- **REST Client:** `quarkus-stork` is no longer a transitive dependency of the REST Client. It must be added explicitly if used.

## Upgrade Plan

1.  **Version Update:**
    - Update `<version.io.quarkus>` in `build-parent/pom.xml` to `3.27.0` (or the specific 3.27.x release).

2.  **Configuration Scan & Fix:**
    - **Search:** `quarkus.test.native-image-profile` -> **Replace:** `quarkus.test.integration-test-profile`.
    - **Search:** `quarkus.hibernate-orm.*` -> **Review:** Check for renamed properties (e.g., `database.generation`).
    - **Search:** `quarkus.datasource.reactive.url` -> **Verify:** Ensure it is set if used.
    - **Search:** `quarkus.resteasy.reactive` -> **Verify:** Ensure migrated to `quarkus.rest`.

3.  **Code Adjustments:**
    - **Search:** `QuarkusTransaction.isActive()` -> **Replace:** `getStatus()`.

4.  **Verification:**
    - Build `drools-quarkus-extension`.
    - Run integration tests, paying attention to persistence and native integration tests.

## Risk Assessment
- **High Risk:** Removal of `quarkus.test.native-image-profile` will break native tests immediately if present.
- **Medium Risk:** Hibernate property renaming might cause configuration to be ignored if the deprecation layer is thin.
