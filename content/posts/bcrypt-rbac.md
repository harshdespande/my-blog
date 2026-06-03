+++
title = "BCrypt and RBAC - Two Pillars of Application Security"
date = 2025-06-03
draft = false
tags = ["security", "bcrypt", "rbac", "spring-boot", "backend"]
author = "Harsh"
+++

When I started learning about application security, two terms kept coming up everywhere — BCrypt and RBAC.

At first they seemed similar — both are about Security, right?

But they solve completely different problems. Let me break it down.

## BCrypt — Password Hashing

When a user creates a password, you should never store it as plain text in your database.

Why? Because if your database is ever hacked, every user's password is exposed instantly.

BCrypt converts the password into an unreadable string called a hash.

Plain text → mypass123

BCrypt hash → $2a$12$4WZyez9ul2guo9qMGTQxNOyYkhVwJb3yUPnJSfYI2i3sr7CQrZsGW

Try it yourself here: https://bcrypt-generator.com/

## Key Features

1. One-way hashing — cannot be reversed
2. Salt — same password gives different hash every time
3. Cost Factor — intentionally slow, hard to brute force
4. Industry Standard — Spring Security, Node.js, Django all use it

## RBAC — Role Based Access Control

BCrypt gets you inside the application. But what can you do once inside?

That is where RBAC comes in.

## Real World Examples

Bank:
- Teller can process transactions
- Branch Manager can approve large transactions
- Auditor can only view never modify

Hospital:
- Doctor can write prescriptions
- Nurse can view prescriptions not write
- Receptionist can only book appointments

## Why You Need BOTH

BCrypt without RBAC — passwords are safe but any user can do anything.

RBAC without BCrypt — access is controlled but database leak exposes all passwords.

Together they are the minimum viable security setup for any backend application.
