---
title: "Structured Logging in Node.js with Winston: A Production Guide"
excerpt: "How I set up a multi-level logging system with Winston that makes debugging production issues actually pleasant — including error classification hierarchies and log rotation."
tags:
  - Node.js
  - Winston
  - DevOps
  - Backend
date: "2025-05-18"
featured: false
coverEmoji: "🪵"
---

Bad logging is worse than no logging. I've debugged production servers where the only output was `console.log('error occurred')` — no context, no stack trace, no timestamp. Here's how to do it right with Winston.

## Why Winston Over console.log

- Multiple transports (file, console, external services) from one call
- Log levels with filtering (`error`, `warn`, `info`, `debug`)
- Structured JSON output for log aggregation tools
- Timestamps, labels, and metadata baked in

## Base Setup

```js
// logger.js
import winston from 'winston';

const { combine, timestamp, json, colorize, simple } = winston.format;

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(timestamp(), json()),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/warn.log',  level: 'warn'  }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: combine(colorize(), simple()),
  }));
}
```

## Custom Error Hierarchy

At Terado Engineering, we implemented a structured error hierarchy so every thrown error carried a type, status code, and operational flag:

```js
export class CustomError extends Error {
  constructor(message, statusCode = 500, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class ValidationError extends CustomError {
  constructor(message) { super(message, 400); }
}

export class NotFoundError extends CustomError {
  constructor(resource) { super(`${resource} not found`, 404); }
}
```

The global error handler then logs operational errors at `warn` level (expected, user-caused) and programming errors at `error` level (unexpected, needs paging).

## Request-Scoped Logging with AsyncLocalStorage

For tracing a single request through multiple service calls, attach a `requestId` using Node's `AsyncLocalStorage` so every log line from that request shares the same ID — invaluable when debugging concurrent traffic.

## Log Rotation

Use `winston-daily-rotate-file` to cap log file size and auto-archive:

```js
new DailyRotateFile({
  filename: 'logs/combined-%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  maxSize: '20m',
  maxFiles: '14d',
})
``` 
