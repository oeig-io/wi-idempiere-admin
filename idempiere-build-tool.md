---
name: idempiere-build
description: hints for building a special instance of idempiere like how iDempiere builds with Jenkins
type: tool
purpose: document how to build iDempiere from source
created: 2026-03-25
---

# iDempiere Build Tool

Resources for building iDempiere from source.

## Jenkins Build References

These Jenkins job configuration pages contain everything needed to build iDempiere from the command line:

- **iDempiere 12:** https://jenkins.idempiere.org/job/iDempiere12/configure
- **iDempiere 13:** https://jenkins.idempiere.org/job/iDempiere13/configure

## Prerequisites

Ensure these are installed (all included in `install-idempiere/idempiere-prerequisites.nix`):

- Java 17 JDK (OpenJDK)
- Maven
- Git

## Build Steps

```bash
# Clone repository
git clone https://github.com/idempiere/idempiere.git
cd idempiere

# Checkout target branch
git checkout release-12    # or release-13

# Clean previous builds (optional, matches Jenkins pre-step)
rm -rf org.idempiere.p2/target/products

# Build
mvn clean verify
```

Optional: pass extra Maven parameters via environment variable:
```bash
export EXTRA_MVN_PRM="--offline"
mvn clean verify $EXTRA_MVN_PRM
```

## Output

- P2 repository: `org.idempiere.p2/target/repository/`
- Linux server package: `org.idempiere.p2/target/products/org.adempiere.server.product/`

## Post-Build Packaging (Linux)

```bash
cd org.idempiere.p2/target/products/org.adempiere.server.product
rm -rf idempiere.gtk.linux.x86_64
mkdir -p idempiere.gtk.linux.x86_64
mv linux/gtk/x86_64 idempiere.gtk.linux.x86_64/idempiere-server
zip -9 -r idempiereServer.gtk.linux.x86_64.zip idempiere.gtk.linux.x86_64/idempiere-server
```
