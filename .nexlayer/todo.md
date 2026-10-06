# cat-chat-app — before this ships

Two lists, split by who can actually close the item.

## Needs code — the coding agent

- [ ] **Blocker:** Remove the literal database password from the committed nexlayer.yaml (`nexlayer.yaml`)
      _DB_PASSWORD and POSTGRES_PASSWORD are committed as 'postgres', which exposes the credential. Use ${POSTGRES_PASSWORD} as in the planned yaml, and treat the old value as compromised._
- [ ] Drop the hardcoded POSTGRES_PASSWORD ENV from the database image (`database/Dockerfile`)
      _The image bakes in a default password. If the runtime key were ever missing, the database would start with 'postgres' instead of failing._
- [ ] Make the frontend's API and Socket.io URLs same-origin in the production build (`frontend/src/App.js`)
      _The code reads REACT_APP_API_URL and REACT_APP_SOCKET_URL at build time. If they fall back to a localhost address, browsers cannot reach the backend; they should be empty or relative so traffic goes through the Nginx proxy._

## Needs the human

- [ ] Decide: The existing deployment is named 'cat-chat'; this plan deploys as 'cat-chat-app'. Does the old deployment's chat history need to be migrated, or is starting fresh acceptable?
- [ ] Decide: Will the backend ever need more than one instance? That would require a shared Socket.io adapter (e.g. Redis) and moving the in-memory online-users state out of server.js.
- Nothing blocking. Some optional keys are not set; the app runs without them.

## Check after the deploy — the coding agent

- [ ] GET / on the app URL returns 200 with the React index.html
- [ ] GET /api/health on the app URL returns 200 with {"status":"ok"}
- [ ] GET /api/rooms on the app URL returns 200 JSON, not 500, proving the backend reached database.pod and init.sql created the schema
- [ ] GET /socket.io/?EIO=4&transport=polling on the app URL returns 200 with a Socket.io handshake payload
- [ ] Send a message, restart the database service, then GET /api/messages and confirm the message is still returned

---

Machine-readable: `.nexlayer/findings.json`.
