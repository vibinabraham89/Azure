Nice — below is a practical, production-ish mini-implementation you can drop into your soc-triage/src and run locally against the Service Bus emulator (or a real Service Bus namespace). It implements:

FastAPI webhook endpoint that sends a small message (incident_id) to Azure Service Bus.

A parallel worker that receives batches of messages, processes them concurrently (thread pool), and completes the message only after processing succeeds.

Notes on how to scale horizontally (run multiple worker processes) so each worker processes independent incidents in parallel.

I’ll give:

the two code files,

.env example,

exact run commands and explanation of behavior and scaling.

Files to create (place in soc-triage/src)
1) webhook_servicebus.py — FastAPI webhook (producer)
# webhook_servicebus.py
import os
import json
import logging
from datetime import datetime, timezone
from fastapi import FastAPI, Request, HTTPException
from azure.servicebus import ServiceBusClient, ServiceBusMessage
from dotenv import load_dotenv

load_dotenv()
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("webhook_servicebus")

SERVICEBUS_CONN = os.getenv("SERVICEBUS_CONNECTION_STRING")
QUEUE_NAME = os.getenv("SERVICEBUS_QUEUE", "incidents")

if not SERVICEBUS_CONN:
    raise RuntimeError("SERVICEBUS_CONNECTION_STRING not set in env")

app = FastAPI()

def send_incident(payload: dict):
    sb = ServiceBusClient.from_connection_string(SERVICEBUS_CONN)
    with sb:
        sender = sb.get_queue_sender(queue_name=QUEUE_NAME)
        with sender:
            msg = ServiceBusMessage(json.dumps(payload))
            # Optionally set message_id to incident_id for de-duplication
            if payload.get("incident_id"):
                msg.message_id = payload["incident_id"]
            sender.send_messages(msg)
    logger.info("Sent ServiceBus message: %s", payload)

@app.post("/webhook")
async def webhook(request: Request):
    """
    Expect JSON like: {"incident_id": "INC-12345"}
    """
    try:
        payload = await request.json()
    except Exception:
        raise HTTPException(status_code=400, detail="invalid json")

    incident_id = payload.get("incident_id") or (payload.get("incident") or {}).get("id")
    if not incident_id:
        raise HTTPException(status_code=400, detail="missing incident_id")

    message = {
        "incident_id": incident_id,
        "received_at": datetime.now(timezone.utc).isoformat()
    }

    try:
        send_incident(message)
    except Exception as e:
        logger.exception("failed to send servicebus message")
        raise HTTPException(status_code=500, detail="enqueue_failed")

    return {"status": "enqueued", "incident_id": incident_id}


# Demo processor endpoint (optional, worker can call this or you can replace with LangGraph)
@app.post("/process_incident")
async def process_incident(request: Request):
    try:
        payload = await request.json()
    except Exception:
        raise HTTPException(status_code=400, detail="invalid json")
    incident_id = payload.get("incident_id")
    if not incident_id:
        raise HTTPException(status_code=400, detail="missing incident id")
    logger.info("Demo processing incident %s", incident_id)
    # simulate processing: replace with your real logic
    return {"status": "processed", "incident_id": incident_id}

2) worker_servicebus_parallel.py — Parallel worker (consumer)
# worker_servicebus_parallel.py
import os
import json
import logging
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests
from azure.servicebus import ServiceBusClient
from dotenv import load_dotenv

load_dotenv()
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("worker_servicebus_parallel")

SERVICEBUS_CONN = os.getenv("SERVICEBUS_CONNECTION_STRING")
QUEUE_NAME = os.getenv("SERVICEBUS_QUEUE", "incidents")
PROCESSOR_URL = os.getenv("PROCESSOR_URL", "http://localhost:7071/process_incident")

# Tune these:
BATCH_SIZE = int(os.getenv("WORKER_BATCH_SIZE", "5"))       # receive up to this many messages at once
MAX_WORKERS = int(os.getenv("WORKER_MAX_CONCURRENCY", "5")) # number of concurrent processing threads
RECEIVE_WAIT = int(os.getenv("WORKER_RECEIVE_WAIT", "5"))   # seconds to wait for messages
PROCESSOR_TIMEOUT = int(os.getenv("PROCESSOR_TIMEOUT", "300"))

if not SERVICEBUS_CONN:
    raise RuntimeError("SERVICEBUS_CONNECTION_STRING not set")

def call_processor(incident_id: str):
    """Call your processor endpoint synchronously. Raises on failure."""
    payload = {"incident_id": incident_id}
    resp = requests.post(PROCESSOR_URL, json=payload, timeout=PROCESSOR_TIMEOUT)
    resp.raise_for_status()
    return resp.status_code

def process_message_body(body_text: str):
    """
    Put your real processing logic here (LangGraph call, pipeline, etc).
    This function runs in a worker thread.
    """
    try:
        data = json.loads(body_text)
        incident_id = data.get("incident_id")
        if not incident_id:
            raise ValueError("missing incident_id in message body")
    except Exception as e:
        raise

    # Example: call the HTTP processor (could be a local function instead)
    call_processor(incident_id)
    return incident_id

def run_worker_loop():
    sb_client = ServiceBusClient.from_connection_string(SERVICEBUS_CONN)
    with sb_client:
        receiver = sb_client.get_queue_receiver(queue_name=QUEUE_NAME, max_wait_time=RECEIVE_WAIT)
        with receiver:
            logger.info("Worker started, listening to %s (batch=%s, concurrency=%s)", QUEUE_NAME, BATCH_SIZE, MAX_WORKERS)
            executor = ThreadPoolExecutor(max_workers=MAX_WORKERS)
            try:
                while True:
                    messages = receiver.receive_messages(max_message_count=BATCH_SIZE, max_wait_time=RECEIVE_WAIT)
                    messages = list(messages)  # materialize generator
                    if not messages:
                        time.sleep(1)
                        continue

                    # Submit processing tasks for each message body
                    futures = {}
                    for msg in messages:
                        # get body as text (SDK may return list of parts)
                        try:
                            body_obj = msg.body
                            if isinstance(body_obj, (list, tuple)):
                                # join bytes / memoryview pieces
                                joined = b"".join([bytes(x) if not isinstance(x, str) else x.encode() for x in body_obj])
                                body_text = joined.decode("utf-8")
                            else:
                                body_text = str(body_obj)
                        except Exception:
                            body_text = str(msg)

                        fut = executor.submit(process_message_body, body_text)
                        futures[fut] = msg

                    # Wait for tasks to finish; for each successful one, complete the message
                    for fut in as_completed(futures):
                        msg = futures[fut]
                        try:
                            incident_id = fut.result()  # raises if processing failed
                            # complete the message (delete from queue)
                            receiver.complete_message(msg)
                            logger.info("Processed & completed incident=%s msg-id=%s", incident_id, getattr(msg, "message_id", None))
                        except Exception as e:
                            # processing failed: abandon so it becomes visible again / increase delivery count
                            logger.exception("Processing failed for message; abandoning. Error: %s", e)
                            try:
                                receiver.abandon_message(msg)
                            except Exception:
                                logger.exception("Failed to abandon message")
            except KeyboardInterrupt:
                logger.info("Worker stopping by KeyboardInterrupt")
            finally:
                executor.shutdown(wait=True)

if __name__ == "__main__":
    run_worker_loop()

3) .env example

Create soc-triage/src/.env (use real values / emulator values):

# For Service Bus emulator:
SERVICEBUS_CONNECTION_STRING=Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=ANY;UseDevelopmentEmulator=true;
SERVICEBUS_QUEUE=incidents

# Worker settings
WORKER_BATCH_SIZE=5
WORKER_MAX_CONCURRENCY=5
PROCESSOR_URL=http://localhost:7071/process_incident


If you use the official emulator installer, the SharedAccessKey value and exact connection string format may be provided by it; UseDevelopmentEmulator=true is required for some emulator setups. For a real Azure Service Bus namespace, use the real connection string and remove UseDevelopmentEmulator.

4) How this behaves (important details)

Producer (webhook_servicebus.py) sends a tiny message with incident_id. It can also set message_id for duplicate detection.

Worker receives up to BATCH_SIZE messages and uses a thread pool of MAX_WORKERS to process them concurrently. After a message’s processing thread finishes successfully, the worker calls complete_message(msg) to remove it. If processing fails, the code calls abandon_message(msg), causing it to be retried (Service Bus handles delivery count and DLQ).

Parallelism: within one worker process you get MAX_WORKERS concurrent processing. For higher parallelism, run multiple worker processes (or separate Kubernetes pods) pointing to the same queue — Service Bus will distribute messages across consumers.

Exactly-once?: Service Bus gives at-least-once delivery. You must make the processing idempotent (store processed incident ids in DB) to avoid duplicated work on retries.

5) Commands to run locally (emulator)

Start Service Bus emulator (the method depends on the emulator you installed). Example (from the emulator repo):

# run the emulator (instructions from the repo)
./LaunchEmulator.sh
# or docker compose up -d (from the launcher repo)


Open 3 terminals in VS Code:

Terminal A — FastAPI producer & demo processor:

cd C:\path\soc-triage\src
.venv\Scripts\Activate.ps1   # or source .venv/bin/activate
# ensure .env is present, python-dotenv will load it
uvicorn webhook_servicebus:app --host 0.0.0.0 --port 7071 --reload


Terminal B — Worker instance 1:

cd C:\path\soc-triage\src
.venv\Scripts\Activate.ps1
python worker_servicebus_parallel.py


Terminal C — Worker instance 2 (optional, to get more parallel consumers):

cd C:\path\soc-triage\src
.venv\Scripts\Activate.ps1
python worker_servicebus_parallel.py


Running multiple copies of the worker gives you horizontal parallelism — Service Bus will hand out different messages to each consumer.

Send test webhook (use PowerShell Invoke-RestMethod to avoid quoting issues):

$body = @{ incident_id = "INC-1001" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:7071/webhook" -Method POST -Body $body -ContentType "application/json" -Verbose


Watch worker logs — you should see messages being processed concurrently and completed.

6) Production notes & recommendations

Scale: In production run worker as Kubernetes Deployment and scale replicas (use KEDA to autoscale on queue length).

Idempotency: persist incident_id processed marker in a DB (CosmosDB/Postgres) before marking message complete — or use transactional patterns so duplicates are safe.

Ordering: if you must process incidents strictly one-at-a-time globally, run exactly 1 consumer. If you need ordered processing per key, use Service Bus Sessions (set session_id on messages and use session receivers).

DLQ: Service Bus handles DLQ after maxDeliveryCount. Monitor DLQ and implement replay/inspection process.

Authentication: in production use Managed Identity / AAD to authenticate to Service Bus, not connection strings.

Retries: tune maxDeliveryCount and worker backoff. Use exponential backoff for transient failures.

Monitoring: expose metrics for queue length, processing latency and failure counts. Use Azure Monitor.

7) If you prefer simpler horizontal scale (no threads)

An alternative simple approach is to have the worker process one message at a time (no thread pool) and run N worker processes or pods. That often matches K8s scaling semantics better (one message lock per process). Use whichever fits your ops model.

If you’d like I can:

adapt the worker to use Service Bus Sessions (for ordered-per-key processing), or

produce a Kubernetes manifest + KEDA ScaledObject to run this in AKS (production-ready), or

convert the worker to use asyncio and aiohttp for async processing.
