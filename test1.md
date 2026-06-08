                    +-------------------+
                    |   Client Apps     |
                    | Web / Mobile / PC |
                    +---------+---------+
                              |
                              | HTTPS / WebSocket
                              |
                    +---------v---------+
                    |   API Gateway     |
                    | Auth, Rate Limit  |
                    +---------+---------+
                              |
              +---------------+----------------+
              |                                |
      +-------v--------+              +--------v---------+
      | WebSocket      |              | REST API Service |
      | Gateway Layer  |              | Conversations    |
      +-------+--------+              | Users/Auth       |
              |                       +--------+---------+
              |                                |
              |                                |
      +-------v-------------------------------v-------+
      |              Message Service                  |
      | Validate, AuthZ, Idempotency, Persist Message |
      +--------------------+--------------------------+
                           |
                           v
                    +------+------+
                    |   Kafka     |
                    | conversationId |
                    | partition key  |
                    +------+------+
                           |
            +--------------+---------------+
            |                              |
   +--------v---------+           +--------v---------+
   | Delivery Workers |           | Notification Svc |
   | Online Delivery  |           | Push/Email Later |
   +--------+---------+           +------------------+
            |
            v
   +--------+---------+
   | WebSocket Servers|
   | Send to Clients  |
   +------------------+

Storage:
+------------------+    +------------------+    +------------------+
| Message DB       |    | Redis            |    | User/Conv DB     |
| Cassandra/Dynamo |    | Presence/Sessions|    | Postgres/MySQL   |
+------------------+    +------------------+    +------------------+
