# canteen-connect: Phase-wise Plan

Follows the build order in [DESIGN.md](DESIGN.md) section 12, with concrete tasks and an exit check for each phase.

## Phase 1: Foundations
- Set up the monorepo: `apps/web`, `services/api`, `services/worker`, `packages/events`, `infra`, `docs`.
- Add TypeScript, lint and test tooling, plus a GitHub Actions workflow for lint, type check and unit tests.
- Add `docker-compose` with a MongoDB replica set and RabbitMQ for local development.
- Build the API skeleton: GraphQL server, JWT login, and role checks (employee, canteen, admin) in every resolver.
- Create the `users`, `menuItems` and `dailyStock` collections, with a versioned index script.
- Build menu and daily-stock management for staff, and a React shell with role-based views.
- **Exit check:** staff can log in, create menu items and set stock, and an employee can see the day's menu.

## Phase 2: Ordering
- Implement `placeOrder` as a single MongoDB transaction: conditional stock decrement, order insert, idempotency key.
- Implement the order state machine as compare-and-set updates, with cancellation rules and stock return.
- Add pickup slots with a cut-off time (configurable).
- Build the staff order queue, order history, and pickup code, using polling for now.
- Add integration tests for oversell races, retries and conflicting state changes.
- **Exit check:** two concurrent orders for the last portion never both succeed, and a retry returns the same order.

## Phase 3: Events and live updates
- Create the shared event types and validation in `packages/events`.
- Write outbox rows in the same transaction as each state change.
- Build the relay that publishes to the `canteen.events` exchange, with dead-letter queues.
- Add GraphQL subscriptions over WebSocket, and replace polling in the UI.
- Make consumers ignore an `eventId` they have already handled.
- **Exit check:** a staff action reaches the employee's screen in under 2 seconds, and the oldest unpublished outbox row stays under 5 seconds.

## Phase 4: Feedback Hub
- Build the feedback widget, the post-collection prompt, `submitFeedback`, and the staff inbox.
- Build the triage worker. It calls the LLM behind a swappable interface, validates the JSON output against a schema, and retries on failure.
- Add the safety rules: a code-side keyword check for food safety and allergen reports, `urgent` priority, and no drafted reply for those.
- Treat employee text as untrusted: delimit it in the prompt, strip names and emails, and give the model no tools.
- Require staff approval before any reply reaches the employee.
- **Exit check:** an allergen report is escalated even if the model misses it, and invalid model output leaves the feedback untriaged in the inbox.

## Phase 5: Observability
- Add OpenTelemetry traces and Prometheus metrics to every service.
- Build the five Grafana dashboards: orders, menu and stock, events, feedback, API.
- Add alerts for outbox age, dead-letter count and LLM errors.
- **Exit check:** every metric in DESIGN.md section 10 is visible, and the two starting targets are measured.

## Phase 6: Cloud and delivery
- Write Bicep templates for Container Apps, Container Registry, Key Vault, and the database and broker choices.
- Extend CI to build images tagged with the commit hash, check the GraphQL schema for breaking changes, and deploy to dev on merge.
- Require manual approval for production, and use managed identity for secrets.
- **Exit check:** a merge to `main` deploys to dev with no manual steps.

## Phase 7: Daily summary and polish
- Add the scheduled job that computes stats with database aggregations. The model writes only the narrative, never the figures.
- Add rate limits and query depth and cost limits.
- Cover accessibility and allergen display on the menu.
- Optional: the reporting projector and PostgreSQL.
- **Exit check:** a manager reads the daily summary and the numbers match the database.

## Phase 8: Real users
- Run a pilot with a small group of colleagues.
- Record the feedback received and the changes made.
- Decide the open questions in DESIGN.md section 13 first, before the pilot: time zone, slot length, uncollected orders, single sign-on, LLM provider and cost ceiling.

## Decisions to make before Phase 1
- Which hosting to use for MongoDB (Atlas or Cosmos DB for MongoDB) and for the broker (self-run RabbitMQ or Azure Service Bus).
- Local accounts or single sign-on.
- Which LLM provider to use.
