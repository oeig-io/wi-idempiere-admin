---
name: ssh-remote-connection
description: SSH key creation and remote server access patterns for managing iDempiere infrastructure with secure password-less authentication
compatibility: opencode
metadata:
  type: tool
  original_file: ssh-remote-connection-tool.md
  category: infrastructure
  scope: idempiere
---

# SSH Remote Connection Tool

## Summary

The purpose of this document is to describe how to create a local SSH key and connect to ANS infrastructure after an admin has added your public key.

This is important because SSH key authentication provides secure, password-less access to remote systems.

## Prerequisites

- Admin has created your user account on the remote system
- Admin has added your public key to the remote system's authorized_keys

## Create SSH Key Pair

Generate an ed25519 key pair (recommended):

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

Accept the default location (`~/.ssh/id_ed25519`) or specify a custom path.

## Provide Public Key to Admin

Display your public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Send this output to the admin for addition to the remote system.

## Connect to Remote System

After admin confirms your key has been added:

```bash
ssh username@hostname
```

First connection requires accepting the host key:

```bash
ssh -o StrictHostKeyChecking=accept-new username@hostname
```

Subsequent connections work without the option.

## Verify Connection

Test the connection:

```bash
ssh username@hostname "echo 'Connection successful'"
```

## Common Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| Host key verification failed | First connection to host | Use `StrictHostKeyChecking=accept-new` |
| Permission denied (publickey) | Key not added or wrong key | Verify public key with admin |
| Connection refused | SSH service not running | Contact admin |

Tags: #tool-ssh #tool-infrastructure
