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

BCrypt solves this by converting the password into an unreadable string called a hash.

Plain text → mypass123

BCrypt hash → $2a$12$4WZyez9ul2guo9qMGTQxNOyYkhVwJb3yUPnJSfYI2i3sr7CQrZsGW

Try it yourself here: https://bcrypt-generator.com/

## Key Features

1. One-way hashing — cannot be reversed
2. Salt — random data added before hashing, so the same password gives a different hash every time
3. Cost Factor — intentionally slow, you can control it
4. Brute Force Resistance — slow hashing = hard to crack
5. Industry Standard — used in Spring Security, Node.js, Django, BCrypt.js

## How It Works

User enters password → BCrypt hashes it → Compares with stored hash → Match = Login Success

## RBAC — Role Based Access Control

BCrypt gets you inside the application. But what can you do once you are inside?

That is where RBAC comes in. RBAC controls what each user is allowed to access based on their role.

## Real World Examples

Bank:
- Teller → Can process transactions
- Branch Manager → Can approve large transactions
- Auditor → Can only view, never modify

Hospital:
- Doctor → Can write prescriptions
- Nurse → Can view prescriptions, not write
- Receptionist → Can only book appointments

## Key Features

1. Each URL and feature is locked to specific roles
2. Unauthorized access is blocked automatically
3. Users only see what they are supposed to see

## How It Works

User logs in → Role is identified → User requests a page → Role checked → Allowed or Blocked

## Why You Need BOTH

BCrypt without RBAC — passwords are safe but any logged-in user can do anything.

RBAC without BCrypt — access is controlled but if the database leaks, all passwords are exposed.

Together they are the minimum viable security setup for any backend application.

Currently implementing both in a Spring Boot R&D portal at Gabriel India Limited — BCrypt + RBAC were the first two things wired in, before any business feature was written.