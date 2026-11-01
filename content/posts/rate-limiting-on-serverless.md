+++
title = "Rate Limiting on Serverless - Why In-Memory Counters Don't Work"
date = 2026-11-01
draft = false
tags = ["security", "rate-limiting", "serverless", "postgresql", "nodejs", "backend"]
author = "Harsh"
summary = "How I added rate limiting to a Node.js app on Vercel, why the usual in-memory approach fails on serverless, and how a single PostgreSQL query solved it."
+++

While building an e-book store with user accounts and online payments, I realised the login page had a big problem.

Anyone could try passwords on it **as many times as they wanted.**

That is how accounts get hacked. So I needed rate limiting. And then I found out the usual way of doing it does not work where my app was hosted.

## What is Rate Limiting?

Rate limiting means putting a cap on how many times someone can do something in a fixed amount of time.

**Example:**

- Allow **8 login attempts** per email every **15 minutes**
- The 9th attempt gets: *"Too many attempts. Please try again in 15 minutes."*

It sounds small, but it stops a lot of attacks.

## What Attacks Does It Stop?

**1. Password guessing (brute force)**

A bot tries thousands of common passwords on one account. With a limit of 8 tries every 15 minutes, trying even 10,000 passwords would take more than 10 days. The attack becomes useless.

**2. Email spam**

The "Forgot password" button sends an email. Without a limit, someone can press it 500 times and flood a person's inbox. It also burns through your email service's quota and can get your domain marked as spam.

**3. Fake sign-ups and abuse**

Bots creating hundreds of fake accounts, or hammering the payment API to create orders.

## The Usual Way - and Why It Failed

Most tutorials show something like this:

```js
const attempts = {}; // stored in memory

function isAllowed(key) {
  attempts[key] = (attempts[key] || 0) + 1;
  return attempts[key] <= 8;
}
```

The counter lives in the server's memory. On a normal server that runs all day, this works.

But my app runs on **Vercel**, which is **serverless**.

On serverless, there is no single server running all the time. Instead:

- Many small copies of the app start up when requests come in
- Each copy has **its own memory**
- Copies shut down when they are idle, and their memory is wiped

So one request might go to copy A, the next to copy B. Copy A's counter says 3, copy B's says 1. After a few minutes both are gone and everything starts again from 0.

**The rate limiter would basically never trigger.**

## The Fix - Keep the Counter in the Database

The counter needs to live somewhere **every copy of the app can see**, and that **survives restarts**. I already had a PostgreSQL database, so I used that. No new service, no extra cost.

**The table:**

```sql
CREATE TABLE IF NOT EXISTS rate_limits (
  key          TEXT PRIMARY KEY,
  window_start TIMESTAMPTZ NOT NULL DEFAULT now(),
  count        INTEGER NOT NULL DEFAULT 0
);
```

The `key` is what we are counting, for example `login-email:someone@gmail.com` or `signup-ip:49.36.12.7`.

## One Query Does Everything

This is the part I liked most. A single SQL statement:

- creates the counter if it does not exist,
- adds 1 if we are still inside the time window,
- resets it to 1 if the window has ended,
- and tells us the new count.

```sql
INSERT INTO rate_limits (key, window_start, count)
VALUES ($1, now(), 1)
ON CONFLICT (key) DO UPDATE SET
  count = CASE
    WHEN rate_limits.window_start < now() - make_interval(secs => $2)
    THEN 1
    ELSE rate_limits.count + 1
  END,
  window_start = CASE
    WHEN rate_limits.window_start < now() - make_interval(secs => $2)
    THEN now()
    ELSE rate_limits.window_start
  END
RETURNING count;
```

`$1` is the key and `$2` is the window length in seconds.

Because it is **one statement**, the database handles two requests arriving at the same time correctly. There is no gap where two copies of the app both read "7" and both write "8".

If the returned count is bigger than the limit, the request is refused with HTTP status **429 Too Many Requests**.

## Wrapping It as Express Middleware

```js
function limit(name, max, windowSec, keyFn) {
  return async (req, res, next) => {
    const key = name + ':' + keyFn(req);
    const { ok } = await hitLimit(key, max, windowSec);
    if (ok) return next();
    res.status(429).json({ error: 'Too many attempts. Please try again later.' });
  };
}

app.post('/api/auth/login',
  limit('login-ip', 30, 900, byIp),
  limit('login-email', 8, 900, byEmail),
  loginHandler
);
```

Now adding a limit to any route is one line.

## The Limits I Chose

| Action | Limit | Counted by |
|---|---|---|
| Login | 8 per 15 min | email |
| Login | 30 per 15 min | IP address |
| Sign up | 5 per hour | IP address |
| Forgot password | 3 per hour | email |
| Create payment order | 10 per 10 min | user |

**Why count login by both email and IP?**

- **By email** protects one account from being attacked from many different IPs.
- **By IP** stops one attacker from trying many different accounts.

## Things I Had to Think About

**1. Fail open, not closed**

If the database has a problem, what should the limiter do? I chose to **let the request through** and log the error. Otherwise a small database issue would lock every user out of their account.

**2. Clean up old rows**

Every new email and IP creates a row. To stop the table growing forever, about 1 in 100 requests also deletes rows older than a day.

**3. Getting the real IP**

Behind Vercel's proxy, `req.socket.remoteAddress` is the proxy, not the user. The real address is in the `x-forwarded-for` header.

**4. Don't reveal too much**

For "Forgot password", when the limit is hit I don't show an error. The page says the same thing as always, but simply stops sending more emails. That way nobody can use the page to check which emails have accounts.

## Fixed Window vs Other Methods

What I built is called a **fixed window** limiter. It is the simplest kind.

Its weakness: someone could make 8 attempts at 10:14 and 8 more at 10:15, right after the window resets. For a login page that is fine. If you need something stricter, look at:

- **Sliding window** - looks at the last 15 minutes from right now
- **Token bucket** - tokens refill slowly, and each request uses one

There are also ready-made services like Upstash Redis that do this for serverless apps. I used the database because I already had one and it needed no new account or cost.

## Key Takeaway

On serverless, **anything stored in memory is temporary and not shared.** That breaks rate limiters, sessions, caches and counters.

If something must be remembered across requests, it needs to live in shared storage like a database or Redis.

Rate limiting is one of those features nobody notices until it's missing. It took less than 100 lines and closed some of the most common ways a login page gets abused.
