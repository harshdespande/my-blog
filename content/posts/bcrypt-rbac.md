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

Why? Because if your database is ever hacked, every user's password is exposed instantly. Even if you encrypt your database, plain text passwords are a ticking time bomb.

BCrypt solves this by converting the password into an unreadable string called a hash.

**Example:**

- Plain text → mypass123
- BCrypt hash → $2a$12$4WZyez9ul2guo9qMGTQxNOyYkhVwJb3yUPnJSfYI2i3sr7CQrZsGW

You can try it yourself here: https://bcrypt-generator.com/

## Key Features of BCrypt

**1. One-way hashing**

Once a password is hashed it cannot be reversed back to plain text. Even the developers of the application cannot see what the original password was. This is fundamentally different from encryption which can be decrypted. Hashing is a one way street.

**2. Salt**

Before hashing BCrypt automatically adds a random string called a salt to the password. This means even if two users have the exact same password their hashes will be completely different. This defeats rainbow table attacks where hackers use precomputed hash databases to crack passwords instantly.

**3. Cost Factor / Work Factor**

The number you see in the hash like $2a$12$ — that 12 is the cost factor. It controls how slow the hashing is. Higher the number more computing power needed to hash. This makes brute force attacks extremely expensive. As hardware gets faster over the years you can increase this number to keep up.

**4. Brute Force Resistance**

Because BCrypt is intentionally slow an attacker trying millions of password combinations per second is slowed down dramatically. MD5 can be cracked at billions of hashes per second. BCrypt at cost factor 12 allows only around 200 to 300 hashes per second. That difference is massive when you are trying to brute force.

**5. Industry Standard**

BCrypt is not something new or experimental. It has been the industry standard since 1999. Spring Security, Node.js, Django, Laravel, Ruby on Rails — almost every major backend framework has BCrypt built in.

**6. Avalanche Effect**

Change even one character in the password and the entire hash changes completely. mypass123 and mypass124 produce totally unrelated hashes. This makes it impossible to guess the original password by studying the hash.

## What Happens if You Do NOT Use BCrypt

- Plain text storage — database leak means instant password exposure for every user
- MD5 or SHA1 — these are fast hashing algorithms not meant for passwords, crackable in seconds with modern GPUs
- Simple encryption — if the encryption key leaks all passwords are exposed at once

Real world breaches like LinkedIn 2012, Adobe 2013, RockYou — all happened because passwords were stored incorrectly. Millions of users affected.

## BCrypt in Spring Boot

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}
```

That one line protects every password in your application automatically.

---

## RBAC — Role Based Access Control

BCrypt answers the question — who are you?

But once you are inside the application the next question is — what are you allowed to do?

That is exactly what RBAC solves.

RBAC stands for Role Based Access Control. Every user is assigned a role and every role has a defined set of permissions. Users can only access what their role allows. Nothing more nothing less.

## Real World Examples

**Bank:**
- Teller → Can process transactions, cannot approve loans
- Branch Manager → Can approve large transactions, can view all accounts
- Auditor → Can only view records, can never modify anything
- IT Admin → Can manage systems, cannot touch financial data

**Hospital:**
- Doctor → Can write prescriptions, can view patient history
- Nurse → Can view prescriptions, cannot write or modify them
- Receptionist → Can book appointments, cannot view medical records
- Lab Technician → Can upload test results, cannot view other departments

## Key Features of RBAC

**1. Principle of Least Privilege**

Every user gets only the minimum access they need to do their job. A junior engineer should never be able to delete production data. RBAC enforces this automatically.

**2. URL Level Protection**

Every endpoint in your application is locked to a specific role. If an unauthorized user tries to access a restricted URL they get a 403 Forbidden error immediately.

**3. Separation of Duties**

Critical operations require different roles. One person cannot both submit and approve the same request. This prevents fraud and errors in business workflows.

**4. Scalability**

When a new employee joins you assign them a role. They instantly get all the right permissions for that role. When they change departments you change their role. No need to manually configure individual permissions every time.

**5. Audit and Accountability**

Since every action is tied to a role and a user you always know who did what and whether they were authorized to do it. Essential for compliance in industries like banking, healthcare and manufacturing.

**6. Easy to Manage**

Instead of managing permissions for hundreds of individual users you manage a handful of roles. Change the role permissions once and it applies to every user with that role instantly.

## RBAC in Spring Boot

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/manager/**").hasRole("MANAGER")
    .requestMatchers("/user/**").hasRole("USER")
    .anyRequest().authenticated()
);
```

Five lines of code lock down your entire application by role.

---

## Why You Need BOTH Together

| | BCrypt | RBAC |
|---|---|---|
| Protects | Passwords in database | Features at runtime |
| Guards against | Database breach | Privilege escalation |
| Question it answers | Who are you? | What can you do? |

**BCrypt without RBAC:**
Passwords are stored safely. But once any user logs in they can access every feature every URL every admin panel. A newly joined intern can approve production requests. A customer can access admin dashboards.

**RBAC without BCrypt:**
Access is perfectly controlled by role. But if your database is ever leaked all passwords are in plain text. Attackers can log in as any user including admins bypassing your RBAC entirely.

**Together they create defense in depth:**
- BCrypt protects your credentials at rest
- RBAC protects your application capabilities at runtime
- One covers the breach scenario the other covers the insider threat scenario

This is not optional. This is the baseline. Any production application without both is incomplete.

Security is not a feature you add later. It is the foundation of every backend application.
