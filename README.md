# Masirat_Assessment_Task_5
System design task

1. Data Synchronization (Hosted Instance to MERN): 
   - I Use a hybrid Ingestion Strategy i.e, From modern instances, the Load Balancer receives incoming POST requests with data (push model). This is preferred for low latency.
   For legacy instances or those without webhook support, we can use a backend cron service that acts as a polling agent to ensure no data is missed.
   By routing both flows into a Message Queue, we ensure that the backend isn't overwhelmed by spikes in data, preventing "cascading failures"
  
2. Handling Real-time Updates: 
   - Instead of the backend manually sending a notification after every save (which can fail), the system listens to the database's own oplog.
   Once a change is committed to MongoDB, the Change Stream triggers a notification service. This ensures that the Frontend only updates when the data is officially persisted.

3. Data Consistency During Migrations:
   - We use the Message Queue to prevent data loss.If we need to move data or change the database, the Queue holds all incoming requests in a line. We can "replay" or retry messages if a save fails during the       move.
   - Version Headers: Imagine your Hosted Instance sends a piece of data.
      Old Format: { "name": "John Doe" }
      New Format: { "first_name": "John", "last_name": "Doe" }
      If an "Old Worker" pulls a "New Format" message from the queue, it will look for name, find nothing, and potentially crash or save a blank record to your database.
      Solution is  we add a small piece of metadata to the message in the queue (eg: Header: schema_version: 2)
      our Backend code uses a simple "If/Else" or "Switch" statement, read the header and process accordingly (i.e, if the old worker can only process the version 1 schema, then it wont try to process the schema         version 2). The data is saved correctly regardless of which version of the backend picked it up.
   - Idempotency: Ensure that the "Save" operation in MongoDB is idempotent (e.g., using upsert). If a migration causes a message to be processed twice, the data remains consistent.


BRIEF SUMMARY OF COMPONENTS AND DATA FLOW

 - Load Balancer: The entry point for all external traffic. It routes incoming Webhooks (Push model) to the Message Queue and handles the initial traffic distribution.
 - Backend Cron Service: A scheduled worker that polls legacy hosted instances (Pull model) to fetch data.
 - Message Queue: It buffers incoming data, decouples the producers from the consumers, and enables retries/replays for reliability.
 - Backend Instances: Stateless workers that consume messages from the queue. They handle business logic, validate Version Headers, and perform Idempotent saves to the database.
 - MongoDB: The primary database.
 - WebSocket/SSE Service: A dedicated real-time gateway.

Data Flow: 

 - Ingestion: Data arrives from a Hosted Instance either via a Push (Webhook to Load Balancer) or a Pull (Cron Service fetch).
 - Queueing: The data is pushed into the Message Queue. At this stage, it is assigned a version Header to ensure the workers know how to handle the specific data format.
 - Persistence: A Backend instance pulls the message from the Message Queue. It checks the version Header for compatibility. If the MongoDB instance is undergoing a migration or is temporarily unavailable, the    backend leaves the message in the Queue. This ensures the data stays in the Queue safely until the database is ready to accept the save.
 - Broadcast: The Change Stream event triggers the WebSocket Service, which identifies the relevant connected users and "pushes" the update to the Frontend UI immediately.
