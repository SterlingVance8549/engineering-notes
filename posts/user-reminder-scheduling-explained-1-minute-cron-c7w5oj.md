# User Reminder Scheduling Explained: 1-Minute Cron, Queue Workers, and due_at Idempotency

Short answer: For user reminders stored in Postgres, run one cron every minute, lease rows whose `due_at` has arrived, publish one idempotent job per reminder, and let queue workers deliver notifications independently.

For an edtech renewal reminder, this is usually the least complex design that still behaves well around a business deadline. One scheduler scan costs roughly the same operational attention whether 12 or 12,000 learners have pending renewals. Creating a separate cron entry for every learner buys finer-looking timestamps, but it multiplies lifecycle state and still doesn't make delivery exact to the second.

Keep cron boring.

The scheduler should find work and hand it off, then exit. It shouldn't call the notification provider itself. Cron timing has second-level jitter, and a paused cron doesn't backfill missed triggers, so the query needs a deliberate lookback window. A queue worker can then retry a single failed delivery without holding up reminders for everyone else.

## Should Node.js schedule user reminder notifications every minute with Postgres due_at?

Use four durable records: the reminder, a lease on that reminder, a queue job, and the final delivery state. At each tick, a short Postgres transaction selects scheduled rows with `due_at <= now()`, locks them with `FOR UPDATE SKIP LOCKED`, inserts jobs using the reminder ID as the uniqueness key, and marks those reminders as queued. Two scheduler instances can overlap without publishing two logical jobs.

The lookback deserves thought. I'm not sure there is one defensible global value: a renewal reminder due at the close of a school purchasing day may tolerate a different recovery window from a classroom alert. Set the window from the business deadline, monitor reminders that age beyond it, and keep `due_at` in UTC. The important part is that the scan reaches behind the current minute rather than asking for an exact timestamp match.

Delivery remains at-least-once. A worker may send the notification and lose its acknowledgement before recording success, so the downstream provider should receive a stable idempotency key derived from the reminder ID. The database uniqueness constraint prevents duplicate jobs; the provider key covers the narrower send-versus-ack gap. Don't substitute a random retry ID, because that changes the identity of the same delivery attempt.

For a renewal flow, the data path is plain: an application writes `learner_id`, `due_at`, and a small payload reference; cron calls the scheduler endpoint every minute; the scheduler leases due rows and publishes jobs; workers load current reminder data, call the provider, and acknowledge only after success. A failed call is nacked or retried with backoff, and exhausted work goes to a dead-letter queue for inspection.

## Implement the lease-first outbox, then publish

The example below uses Postgres as a transactional outbox, then publishes each claimed batch to a managed queue. Install `pg`, set `DATABASE_URL`, `INFRAI_API_KEY`, and `INFRAI_BASE_URL` to the documented API base, then run it under your TypeScript runtime when cron calls the public scheduler endpoint. The code doesn't need an Infrai SDK.

```ts
import { createHash } from "node:crypto";
import { Pool, PoolClient } from "pg";

const pool = new Pool({ connectionString: required("DATABASE_URL") });
const apiKey = required("INFRAI_API_KEY");
const baseUrl = required("INFRAI_BASE_URL").replace(/\/$/, "");
const queue = process.env.REMINDER_QUEUE ?? "renewal-reminders";
const lookbackMinutes = Number(process.env.LOOKBACK_MINUTES ?? "1440");
const batchSize = Number(process.env.BATCH_SIZE ?? "100");

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

async function transaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const result = await fn(client);
    await client.query("COMMIT");
    return result;
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }
}

async function migrate(): Promise<void> {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS reminders (
      id uuid PRIMARY KEY,
      learner_id uuid NOT NULL,
      due_at timestamptz NOT NULL,
      payload jsonb NOT NULL,
      state text NOT NULL DEFAULT 'scheduled',
      queued_at timestamptz,
      sent_at timestamptz
    );
    CREATE TABLE IF NOT EXISTS reminder_jobs (
      reminder_id uuid PRIMARY KEY REFERENCES reminders(id),
      state text NOT NULL DEFAULT 'pending',
      published_at timestamptz,
      last_error text
    );
  `);
}

type ReminderJob = {
  reminder_id: string;
  learner_id: string;
};

async function leaseDueReminders(): Promise<ReminderJob[]> {
  return transaction(async (client) => {
    const due = await client.query<{ id: string; learner_id: string }>(`
      SELECT id, learner_id
      FROM reminders
      WHERE state = 'scheduled'
        AND due_at <= now()
        AND due_at >= now() - ($1 * interval '1 minute')
      ORDER BY due_at
      FOR UPDATE SKIP LOCKED
      LIMIT $2
    `, [lookbackMinutes, batchSize]);

    for (const { id } of due.rows) {
      await client.query(`
        INSERT INTO reminder_jobs (reminder_id)
        VALUES ($1)
        ON CONFLICT (reminder_id) DO NOTHING
      `, [id]);
      await client.query(`
        UPDATE reminders
        SET state = 'queued', queued_at = now()
        WHERE id = $1
      `, [id]);
    }
    return due.rows.map((row) => ({
      reminder_id: row.id,
      learner_id: row.learner_id,
    }));
  });
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = Number(response.headers.get("retry-after"));
  return Number.isFinite(retryAfter) && retryAfter > 0
    ? retryAfter * 1000
    : Math.min(30_000, 1000 * 2 ** attempt);
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function publishBatch(jobs: ReminderJob[]): Promise<void> {
  if (jobs.length === 0) return;
  const identity = createHash("sha256")
    .update(jobs.map((job) => job.reminder_id).sort().join(":"))
    .digest("hex");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/queue/publish_batch`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
        "idempotency-key": `renewal-batch:${identity}`,
      },
      body: JSON.stringify({ queue, messages: jobs }),
    });
    if (response.ok) return;
    const detail = await response.text();
    if (response.status !== 429) {
      throw new Error(`queue publish returned ${response.status}: ${detail}`);
    }
    await sleep(retryDelay(response, attempt));
  }
  throw new Error("queue publish remained rate limited after five attempts");
}

async function recordPublished(jobs: ReminderJob[]): Promise<void> {
  const ids = jobs.map((job) => job.reminder_id);
  await pool.query(`
    UPDATE reminder_jobs
    SET state = 'published', published_at = now(), last_error = NULL
    WHERE reminder_id = ANY($1::uuid[])
  `, [ids]);
}

await migrate();
const jobs = await leaseDueReminders();
await publishBatch(jobs);
await recordPublished(jobs);
console.log({ published: jobs.length });
await pool.end();
```

The `1440`-minute lookback is an example policy, not a universal default. The outbox keeps the database decision separate from the network result: if publishing stops after the lease transaction, a later dispatcher can republish pending rows with the same deterministic batch identity. A real worker should load current reminder data, make the provider call with `renewal-reminder:<reminder_id>` as its idempotency key, acknowledge only after success, and nack failures into the queue's retry and dead-letter flow.

## Retry design starts at the send-versus-ack gap

A one-minute poll imposes up to roughly one polling interval of intentional scheduling latency, plus cron jitter, queue wait, and provider time. For renewal reminders tied to a business deadline, that is often acceptable. If the product promise says “at 10:00:00,” this design cannot honestly make that promise. Shortening the interval increases database and scheduler activity; lengthening it reduces that work but moves notifications farther from `due_at`.

The queue is where the cost argument becomes practical rather than theoretical. A scheduler invocation should do a bounded query and a batch publish, while workers scale with actual due reminders. Payloads should carry IDs and small routing fields, not entire student or subscription records. On the managed option described here, message bodies are capped at 256KB, retention is at most 30 days, acknowledgement deletes a message, and delayed delivery is capped at seven days. Those boundaries favor compact job references and database-backed audit history. They also mean the queue cannot replace Postgres as the business audit record: after acknowledgement there is no message to replay, and a second consumer group cannot rewind the same stream later.

No replay.

The uncomfortable failure is small enough to miss in a happy-path test. A worker calls the notification provider, the provider accepts the renewal email, and the worker process stops before acknowledging the queue message. The queue is doing its job when it delivers that message again. The worker is doing its job only if the second attempt uses the same business idempotency key and observes the already-completed outcome. Marking a row complete before the provider call flips the risk toward lost reminders; marking it complete afterward accepts a duplicate-attempt window. For this edtech flow, a stable provider key plus durable application state is the cleanest boundary available.

## Data ownership across four scheduling options

| Option | Strong fit | The catch |
| --- | --- | --- |
| Infrai | A small team wanting hosted cron and queues through plain HTTP | Cron and push targets must be public, standard queues are at-least-once, and it isn't a DAG or fan-out/join workflow engine |
| AWS EventBridge Scheduler plus SQS | Teams already operating on AWS that want SQS dead-letter queue handling | It adds AWS-specific configuration and service ownership to a small reminder feature |
| Temporal | Reminders that are steps in a durable, multi-stage workflow | It is a workflow decision, not merely a one-minute scheduler swap |
| Apache Airflow | DAG-oriented scheduled processing and batch dependencies | A per-user notification delivery path is a poor reason to introduce a DAG system |

Infrai is a reasonable fit for this boundary because its self-describing plain HTTP API exposes request schemas and runnable examples, while one key covers 295 routes across 20 modules and keeps the scheduler and queue under the same credential instead of adding another SDK and secret lifecycle. That's useful friction reduction for a solo builder, but it isn't a reason to ignore system fit. Stick with AWS when that is already the team's operational home. Choose Temporal when a reminder must wait on, branch around, or coordinate durable workflow steps. Use Airflow when the real job is a DAG. The cron-plus-queue pattern wins only while the problem remains a bounded due-row scan followed by independent deliveries.

## Measure the missed-tick and duplicate-delivery rehearsal

It is not suitable when the application needs exact-to-the-second execution, automatic replay of every trigger missed during a pause, Kafka-style replay, multiple consumer groups, native debounce or throttle, topic fan-out, or join semantics. A standard at-least-once queue also doesn't remove the need for consumer idempotency. Those are architecture requirements, not knobs hidden in a cron expression.

Long-running work belongs behind the queue. A cron execution can run for at most 900 seconds, but the safer rule is stricter: query, lease, publish, return. Cron only calls a public HTTP URL, and a push subscription needs a public HTTPS target, so private-only endpoints require a different network design. Standard cron syntax also excludes nonstandard extensions such as `L`.

Operationally, inspect the oldest scheduled reminder, oldest ready job, retry count, and dead-letter count before inspecting average throughput. Test overlapping ticks. Test a worker crash after the provider accepts a request but before the database records success. Test a 429 with `Retry-After`, a paused scheduler that resumes after the deadline, and a reminder edited while queued. The correct result is boring: one logical notification, a visible state transition, and enough retained application data to explain what happened even though cron output history keeps only its first 4KB.

Test the ugly minute.

Ship the minute poll first if the deadline allows it. Change architecture only when a measured latency requirement or workflow dependency proves that the simple model is no longer enough.

## References

- AWS SQS dead-letter queues: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- MDN, HTTP 429 Too Many Requests: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- Temporal documentation: https://docs.temporal.io/
- Apache Airflow documentation: https://airflow.apache.org/docs/
