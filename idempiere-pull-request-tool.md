---
name: idempiere-pull-request
description: Checklist and workflow for submitting high-quality pull requests to the iDempiere core
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-pull-request-tool.md
  category: development
  scope: idempiere
---

# iDempiere Pull Request Tool

The purpose of this tool is to provide a checklist and reference for contributing code to the iDempiere core.

This is important because iDempiere is an open source ERP/CRM/MFG/SCM/POS suite where code quality directly affects the entire community. Well-written pull requests are reviewed faster and integrate smoothly.

## TOC

- [Before You Start](#before-you-start)
- [Contribution Checklist](#contribution-checklist)
  - [Code Standards](#code-standards)
  - [Data Access Hierarchy](#data-access-hierarchy)
  - [Testing Requirements](#testing-requirements)
  - [Database Changes](#database-changes)
- [Pre-submission Validation](#pre-submission-validation)
- [Pull Request Creation](#pull-request-creation)
- [External Resources](#external-resources)

## Before You Start

**Every commit must be related to a JIRA ticket.** Do not mix unrelated commits in a single pull request.

If your change includes a refactoring AND a fix, submit as separate commits or separate PRs:
- One commit/PR for the refactoring
- One commit/PR for the actual solution

Reference: [Git Workflow](https://idempiere.github.io/docs/basic-development/contributing-to-core/git-workflow)

## Contribution Checklist

### Code Standards

- [ ] Follow Java standards (meaningful names, camelCase)
- [ ] Use proper indentation and formatting
- [ ] Add **GPLv2 License Header** to each new Java class
- [ ] All code, comments, and identifiers in English
- [ ] PR stays focused on one purpose - only touch classes needed to solve the ticket

**Public Method Changes:**
- Prefer **method overloading** instead of changing existing method signatures (preserves backward compatibility)
- If you change a class or interface signature, regenerate `serialVersionUID`

Reference: [How to Contribute](https://idempiere.github.io/docs/basic-development/contributing-to-core/how-to-contribute)

### Data Access Hierarchy

Use the following methods in order of preference:

1. **Model Classes** - Most `M*` classes have convenient `get` methods
2. **Query class** - Clean and efficient for many use cases
3. **DB class (`DB.get()`)** - Use with care
4. **JDBC (raw access)** - Only if absolutely necessary

**JDBC Usage:** If you must use JDBC, resources must be closed properly:
```groovy
def pstmt = DB.prepareStatement(sql, trxName)
// ... use pstmt ...
DB.close(rs, pstmt)  // Required cleanup
```

### Testing Requirements

- [ ] All existing unit tests pass before submitting
- [ ] Consider adding new unit tests for new logic or bug fixes

### Database Changes

> **⚠️ Warning** - If your contribution includes database changes, follow the [Changing the Database](https://idempiere.github.io/docs/basic-development/contributing-to-core/changing-the-database) instructions **before writing any code**.

- [ ] Migration scripts are present
- [ ] Works in both PostgreSQL AND Oracle
- [ ] Uses data model and query APIs over raw SQL where possible

## Pre-submission Validation

Perform a collateral impact analysis:
- Check where the method/class is referenced elsewhere
- Verify your change does not introduce unexpected behavior
- Test all relevant test cases

Verify common pitfalls:
- Unclosed JDBC resources
- Security risks
- Collateral damage to existing functionality

Reference: [Common Issues](https://idempiere.github.io/docs/basic-development/contributing-to-core/common-issues)

## Pull Request Creation

1. Create a branch from `upstream/master`
2. Include JIRA ticket number in commit message: `IDEMPIERE-1234 Fix XYZ`
3. This enables automatic linking between Git and JIRA

**Expected PR template includes:**
- Description of changes
- Testing performed
- Any database migration scripts
- Related JIRA tickets

## External Resources

| Resource | URL |
|----------|-----|
| CONTRIBUTING.md | https://github.com/idempiere/idempiere/blob/master/CONTRIBUTING.md |
| Pull Request Template | https://github.com/idempiere/idempiere/blob/master/pull_request_template.md |
| How to Contribute (docs) | https://idempiere.github.io/docs/basic-development/contributing-to-core/how-to-contribute |
| Peer Review Checklist | https://wiki.idempiere.org/en/Check_List_Peer_Review |
| Git Workflow | https://idempiere.github.io/docs/basic-development/contributing-to-core/git-workflow |
| Plugin Guidelines | https://idempiere.github.io/docs/basic-development/contributing-to-core/plugin-guidelines.md |

## TL;DR

Before submitting your pull request:

1. ✅ All commits linked to JIRA ticket
2. ✅ Run unit tests - they must pass
3. ✅ Works on PostgreSQL AND Oracle
4. ✅ No JDBC unless absolutely necessary (close resources if used)
5. ✅ Database migrations included
6. ✅ No collateral damage - test related functionality
7. ✅ Commit message includes JIRA ticket (e.g., `IDEMPIERE-1234 Fix XYZ`)
