# Webhook Wind Tunnel

Webhook Wind Tunnel is a browser-based test bench for developers building webhook integrations. Set up a sequence of deliveries, replay it against a sample checkout app, and see what each event did to the order and shipment records.

**Status:** Build plan for the first release. The first version runs locally with a bundled sample store.

## The problem

A webhook is an automatic message that one service sends to another when something happens. For example, a payment service can tell an online store that a customer has paid. The store's webhook handler receives the message, marks the order as paid, and queues it for shipment.

In practice, the same message can arrive more than once, and related messages can arrive in a different order than expected. Stripe documents both behaviors. A store might acknowledge every message with `200 OK` and still queue two shipments for one order. The developer needs a repeatable way to recreate that sequence and check what the application actually did. [Stripe webhook delivery behavior](https://docs.stripe.com/webhooks#event-ordering)

## The first release

Imagine you changed the checkout code and want to know whether the same payment notification can queue a second shipment. Open the **Test bench**, load the **Duplicate payment** preset, and click **Run comparison**. Wind Tunnel runs the same sequence against two versions of a sample store's webhook handler. Both can return `200 OK`, while one queues two shipments and the other queues one. The comparison shows which delivery caused the difference.

Wind Tunnel builds and runs the test, captures the results, and evaluates the outcome. The **buggy handler** reproduces common mistakes. The **fixed handler** shows the expected behavior for the sample store. Each handler starts with its own empty order and shipment records.

### Test bench

- Open the site directly to the test bench. Start with **Normal checkout** selected and switch between three presets without any setup or sign-in.
- Move a delivery up or down, duplicate a delivery, and inspect the event ID and JSON payload. A duplicate gets its own row and retains the original event ID. **Reset preset** restores the original sequence.
- See the expected result before running the test. The order should be **Paid**, with exactly **one shipment queued**.
- Run both handlers with one click. Show progress while the run is in flight. If the run can't complete, show an error with a retry action and keep the current sequence.

### Comparison results

- Show the expected and actual order status and shipment count for each handler. Give each check a text label for **Pass** or **Fail**.
- Align both handlers' results by delivery. Each step shows which event was sent, the handler's response status, and the order status and shipment count immediately afterward. Stack the two results per step on narrow screens.
- Keep the sequence editor available so users can change a delivery and run the comparison again.

### Run history

- List completed comparisons with their time, preset, and the outcome for each handler. A comparison can finish successfully while exposing a failed check.
- Reopen the exact sequence and results from any completed comparison. History stays available after a page refresh or local server restart.

## Demo scenarios

The fictional store uses two events for one order. `order.created` establishes the order record. `payment.succeeded` marks it paid and queues a simulated shipment.

| Preset | Delivery sequence | Buggy handler | Fixed handler |
| --- | --- | --- | --- |
| **Normal checkout** | Order created → Payment succeeded | Paid, one shipment | Paid, one shipment |
| **Duplicate payment** | Order created → Payment succeeded → Same payment again | Paid, two shipments | Paid, one shipment |
| **Payment arrives first** | Payment succeeded → Order created | Pending, one shipment | Paid, one shipment |

## How it's built

- **Website and API.** Use Next.js and TypeScript for the React screens and server routes. The browser submits the ordered list of sample event IDs. The server validates the sequence and uses its own copies of the sample payloads.
- **Runner.** Invoke the two bundled handlers on the server in the requested order. Record each simulated response and read the actual SQLite order and shipment state after each step. Compute pass or fail from the final database state.
- **Persistence.** Store the exact sequence, both handler results, and each delivery's state in SQLite. Scope the sample order, shipment, and processed-event records to an individual handler run so comparisons and reruns stay independent.
- **Fixed handler.** Record the processed event ID and apply its changes in one database transaction. A payment may establish a paid record before order creation arrives. A later `order.created` fills in the order details while preserving its paid status.

The first release uses simulated deliveries and a single local workspace. Sending tests to a developer's own test endpoint and importing custom payloads are follow-up features.

## Two-day build plan

| Day | Finish line |
| --- | --- |
| **Day 1** | Build the SQLite store, both sample handlers, and the runner. Wire up a basic test bench and compare the normal and duplicate-payment presets end to end. |
| **Day 2** | Finish the out-of-order preset, sequence editor, comparison layout, and history. Run the demo checklist, add verified local setup instructions, and record a short GitHub demo. |

## Demo checklist

1. Open the test bench and run **Normal checkout**. Both handlers should pass both checks.
2. Duplicate the payment in the editor. Verify that both payment rows share an event ID and that the shipment count diverges on the second delivery, even if both responses show `200 OK`.
3. Run **Payment arrives first**. Verify that the late order-created delivery changes the buggy handler's order to **Pending** while the fixed handler's order stays **Paid**.
4. Run the same sequence twice, then restart the local app. Verify that the runs have identical outcomes, their shipments are counted independently, and both comparisons can be reopened from history.
