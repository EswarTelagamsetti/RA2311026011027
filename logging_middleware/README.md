# Logging Middleware

This package makes it easy to capture request logs and send them to an evaluation server.
It keeps your Express routes clean while handling authentication, token refresh, and log delivery.

## Installation

Requires Node 18 or newer because it uses the built-in `fetch` API. There are no external
dependencies.

```bash
npm install ./logging_middleware
```

## Usage

Create the logger with your credentials and attach it to your Express app:

```js
const express = require('express');
const LoggingMiddleware = require('logging_middleware');

const app = express();

const logger = new LoggingMiddleware({
  email: process.env.EVAL_EMAIL,
  name: process.env.EVAL_NAME,
  rollNo: process.env.EVAL_ROLL_NO,
  accessCode: process.env.EVAL_ACCESS_CODE,
  clientID: process.env.EVAL_CLIENT_ID,
  clientSecret: process.env.EVAL_CLIENT_SECRET
});

app.use(logger.middleware());

app.get('/ping', async (req, res) => {
  try {
    res.json({ message: 'pong' });
  } catch (error) {
    await logger.log('backend', 'error', 'route', error.message);
    res.status(500).json({ error: 'Something went wrong' });
  }
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

## What it does

- Authenticates with the evaluation server automatically.
- Caches the token in memory and refreshes it before expiry.
- Logs request timing for Express routes.
- Lets you send manual log messages from your application.

## Notes

- The default evaluation API is `http://20.207.122.201/evaluation-service`.
  You can override it with `authUrl` and `logsUrl` when creating `LoggingMiddleware`.
- Network failures are caught and sent to the console without crashing the app.
- Keep credentials in environment variables and out of source control.
