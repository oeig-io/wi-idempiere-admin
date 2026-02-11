---
name: idempiere-request
description: Configure and use the Request system in iDempiere including request types, status categories, resolutions, and workflow management
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-request-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Request Tool

The purpose of this document is to describe how to configure and use the Request system in iDempiere.

## Table Naming

Tables prefixed with `r_` are dedicated to supporting requests (e.g., r_request, r_status, r_resolution, r_requesttype).

## Key Concepts

Requests track workflow with two distinct concepts:

| Concept | Purpose | Examples |
|---------|---------|----------|
| **Status** | Lifecycle state | Open, Pending, Closed |
| **Resolution** | Outcome/result | Approved, Rejected |

Do not conflate status and resolution. A request is "Closed" (status) with resolution "Approved" - not status "Approved".

## Navigation

| Window | Menu Path |
|--------|-----------|
| Request Resolution | Request => Request Resolution |
| Request Status Category | Request => Request Status Category |
| Request Type | Request => Request Type |
| Request | Request => Request |
| Request (All) | Request => Request (All) |

## Request vs Request (All)

| Window | WHERE Clause |
|--------|--------------|
| Request | Assigned to current user/role AND status is open |
| Request (All) | No filter - shows all accessible requests |

The Request window filters: `(SalesRep_ID=@#AD_User_ID@ OR AD_Role_ID=@#AD_Role_ID@) AND status is open`

## Updates and History Subtabs

The Request window has two audit subtabs managed by `RequestEventHandler`:

| Subtab | Table | Trigger | Content |
|--------|-------|---------|---------|
| Updates | R_RequestUpdate | Result field has content | Current state snapshot |
| History | R_RequestAction | Any tracked field changes | Previous field values |

**Updates** - `MRequestUpdate.isNewInfo()` returns `getResult() != null`. Only saves when user enters text in Result field.

**History** - `checkChange()` copies old values to R_RequestAction. Saves when any tracked field changes (Status, Resolution, User, Priority, BPartner, linked documents, etc.).

**Update Notification** (R_RequestUpdates) - Manual subscription of users to this request's notifications. The view `RV_RequestUpdates` combines multiple notification sources: direct subscription, group/type/category subscriptions, request contact, assigned user, and role members.

## Query Current Configuration

```sql
-- View request types with their status categories
SELECT rt.name as request_type, sc.name as status_category
FROM r_requesttype rt
JOIN r_statuscategory sc ON rt.r_statuscategory_id = sc.r_statuscategory_id
WHERE rt.ad_client_id = 1000000;

-- View statuses for a category
SELECT name, seqno, isdefault, isclosed, isfinalclose
FROM r_status
WHERE r_statuscategory_id = (SELECT r_statuscategory_id FROM r_statuscategory WHERE name = 'Approval')
ORDER BY seqno;

-- View resolutions
SELECT name, description FROM r_resolution WHERE ad_client_id = 1000000;
```

## Known Issues

### Email notification fails when User field is null

When a request is updated with a null User field, the `RequestEventHandler` attempts to send an email notification and log it to `ad_usermail`. The `ad_usermail.ad_user_id` column has a NOT NULL constraint, causing an error:

```
ERROR: null value in column "ad_user_id" of relation "ad_usermail" violates not-null constraint
```

The request saves successfully, but email notification fails. This does not block normal usage.

Tags: #tool #idempiere #request #workflow
