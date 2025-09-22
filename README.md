webhook_servicebus.py

$env:SERVICEBUS_CONNECTION_STRING = "Endpoint=sb://localhost/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=Eby8vdM02xNOcqFlqUwJ7r1dvbjG3rJHB5Bsn4I4k9E=;UseDevelopmentEmulator=true;"
# then start your FastAPI app, for example:
uvicorn webhook_servicebus:app --host 0.0.0.0 --port 8000


SERVICEBUS_CONNECTION_STRING=Endpoint=sb://localhost/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=Eby8vdM02xNOcqFlqUwJ7r1dvbjG3rJHB5Bsn4I4k9E=;UseDevelopmentEmulator=true;
SERVICEBUS_QUEUE=incidents
RESULT_CALLBACK_URL=http://localhost:7071/process_result

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
# If you want workers to POST back results, set this URL (worker will POST to it)
RESULT_CALLBACK_URL = os.getenv("RESULT_CALLBACK_URL", "http://localhost:7071/process_result")

if not SERVICEBUS_CONN:
    raise RuntimeError("SERVICEBUS_CONNECTION_STRING not set in environment")

app = FastAPI()


def send_to_servicebus(payload: dict):
    sb = ServiceBusClient.from_connection_string(SERVICEBUS_CONN)
    with sb:
        sender = sb.get_queue_sender(queue_name=QUEUE_NAME)
        with sender:
            msg_body = json.dumps(payload)
            msg = ServiceBusMessage(msg_body)
            # set message_id for de-duplication if you want:
            if payload.get("incident_id"):
                msg.message_id = payload["incident_id"]
            sender.send_messages(msg)
    logger.info("Enqueued to ServiceBus queue=%s: %s", QUEUE_NAME, payload)


@app.post("/webhook")
async def webhook(request: Request):
    """
    Receives payload from Resilient. Expects JSON with 'incident_id' (or incident.id).
    Enqueues minimal message to Service Bus.
    """
    try:
        payload = await request.json()
    except Exception:
        raise HTTPException(status_code=400, detail="invalid json")

    incident_id = payload.get("incident_id") or (payload.get("incident") or {}).get("id")
    if not incident_id:
        raise HTTPException(status_code=400, detail="missing incident_id")

    msg = {
        "incident_id": incident_id,
        "received_at": datetime.now(timezone.utc).isoformat()
    }

    try:
        send_to_servicebus(msg)
    except Exception as e:
        logger.exception("failed to send message to ServiceBus")
        raise HTTPException(status_code=500, detail="enqueue_failed")

    return {"status": "enqueued", "incident_id": incident_id}


# Optional endpoint: workers can POST back results here.
# You can store results in DB/blob instead -- this is just a demo callback.
@app.post("/process_result")
async def process_result(request: Request):
    try:
        payload = await request.json()
    except Exception:
        raise HTTPException(status_code=400, detail="invalid json")
    logger.info("Received processing result: %s", payload)
    # Persist result to DB / blob in real app. For demo just return.
    return {"status": "ok"}

worker_servicebus_oneper.py

import os
import json
import logging
import time
import requests
from azure.servicebus import ServiceBusClient
from dotenv import load_dotenv

load_dotenv()
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("worker_servicebus_oneper")

SERVICEBUS_CONN = os.getenv("SERVICEBUS_CONNECTION_STRING")
QUEUE_NAME = os.getenv("SERVICEBUS_QUEUE", "incidents")
RESULT_CALLBACK_URL = os.getenv("RESULT_CALLBACK_URL", "http://localhost:7071/process_result")
PROCESSOR_MODULE_TIMEOUT = int(os.getenv("PROCESSOR_TIMEOUT", "300"))

if not SERVICEBUS_CONN:
    raise RuntimeError("SERVICEBUS_CONNECTION_STRING not set in env")

def process_incident_module(incident_id: str):
    """
    Replace this with your real processing logic (LangGraph / agents).
    This function should return a dict with the result.
    For demo, we'll simulate work and return a sample result.
    """
    logger.info("Processing incident in module: %s", incident_id)
    # Simulate work (call external agents, do analysis, etc)
    # For demo, just return a fake result.
    result = {
        "incident_id": incident_id,
        "analysis": {"score": 0.9, "summary": f"Processed {incident_id}"},
        "processed_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
    }
    # Simulate time-consuming work if needed: time.sleep(2)
    return result

def callback_result(result: dict):
    """
    Post back the processing result to an HTTP endpoint (optional).
    Alternatively, write to DB or blob storage.
    """
    try:
        resp = requests.post(RESULT_CALLBACK_URL, json=result, timeout=PROCESSOR_MODULE_TIMEOUT)
        resp.raise_for_status()
        logger.info("Result callback succeeded for %s", result.get("incident_id"))
    except Exception as e:
        logger.exception("Result callback failed for %s: %s", result.get("incident_id"), e)
        # decide whether this should fail processing or be tolerated
        # We will tolerate callback failure (do not requeue message), but in prod consider retrying or storing result for replay.

def run_worker_loop():
    sb_client = ServiceBusClient.from_connection_string(SERVICEBUS_CONN)
    logger.info("Worker started; connecting to Service Bus queue=%s", QUEUE_NAME)
    with sb_client:
        receiver = sb_client.get_queue_receiver(queue_name=QUEUE_NAME, max_wait_time=5)
        with receiver:
            while True:
                try:
                    messages = receiver.receive_messages(max_message_count=1, max_wait_time=10)
                    messages = list(messages)
                    if not messages:
                        # no message => sleep briefly and poll again
                        time.sleep(1)
                        continue

                    for msg in messages:
                        try:
                            # parse message body (SDK may return list/parts)
                            body_obj = msg.body
                            if isinstance(body_obj, (list, tuple)):
                                joined = b"".join([bytes(x) if not isinstance(x, str) else x.encode() for x in body_obj])
                                body_text = joined.decode("utf-8")
                            else:
                                body_text = str(body_obj)

                            payload = json.loads(body_text)
                            incident_id = payload.get("incident_id")
                            if not incident_id:
                                logger.warning("Message missing incident_id; abandoning")
                                receiver.abandon_message(msg)
                                continue

                            # CALL YOUR PROCESSING MODULE
                            result = process_incident_module(incident_id)

                            # OPTIONAL: callback result to FastAPI or persist to storage
                            callback_result(result)

                            # COMPLETE the message only after successful processing
                            receiver.complete_message(msg)
                            logger.info("Completed message for incident=%s", incident_id)

                        except Exception as e:
                            logger.exception("Processing failed for message; abandoning so it can be retried: %s", e)
                            try:
                                receiver.abandon_message(msg)
                            except Exception:
                                logger.exception("Failed to abandon message")
                except KeyboardInterrupt:
                    logger.info("Worker shut down (KeyboardInterrupt)")
                    return
                except Exception:
                    logger.exception("Top-level worker loop error; sleeping briefly then retrying")
                    time.sleep(2)

if __name__ == "__main__":
    run_worker_loop()

soc-triage/src/.env

# Service Bus emulator example (use your emulator docs' exact key if needed)
SERVICEBUS_CONNECTION_STRING=Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=ANY;UseDevelopmentEmulator=true;
SERVICEBUS_QUEUE=incidents

# Where worker posts results (optional)
RESULT_CALLBACK_URL=http://localhost:7071/process_result
PROCESSOR_TIMEOUT=300
