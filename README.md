# Backend Evaluation Submission

A small backend solution for notification handling and maintenance scheduling.

## What is included

- `logging_middleware/`: logging middleware that sends structured events to the evaluation service.
- `notification_app_be/`: Express server that fetches and ranks notifications.
- `vehicle_maintenance_scheduler/`: task scheduler that picks the most valuable maintenance tasks within a time budget.
- `notification_system_design.md`: a simple design summary for the notification service.

## Setup

Install dependencies and run the notification server:

```bash
cd notification_app_be
npm install
EVAL_EMAIL=your.email@example.com \
EVAL_NAME="Your Name" \
EVAL_ROLL_NO=RA2311026011027 \
EVAL_ACCESS_CODE=XXXX \
EVAL_CLIENT_ID=YYYY \
EVAL_CLIENT_SECRET=ZZZZ \
node src/server.js
```

Run the maintenance scheduler:

```bash
node vehicle_maintenance_scheduler/src/scheduler.js \
  vehicle_maintenance_scheduler/tasks.json 8
```

## Notes

- Keep sensitive credentials in environment variables.
- The notification server is a minimal demo and can be extended to cover a full API.
- The scheduler is designed for clarity and works well for moderate input sizes.
