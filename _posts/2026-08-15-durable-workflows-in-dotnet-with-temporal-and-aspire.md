---
layout: post
title: "Durable Workflows In .NET With Temporal And Aspire"
date: 2026-08-15 14:26:00 +0200
categories: dotnet architecture temporal aspire
excerpt: "A practical introduction to Temporal using an order workflow built with .NET and Aspire."
---

Some operations do not finish in one request.

An order can have several steps. We reserve inventory then charge a payment and update the order status. The application can stop after any of these steps. When it starts again it needs to know what was already completed.

We can build this logic ourselves. We can store every step in a database and create retry jobs. We also need to handle timeouts and application restarts.

This is the problem Temporal tries to solve.

## What Is Temporal?

[Temporal](https://temporal.io/) is a system for running workflows. A workflow is a sequence of steps which can run for seconds days or even months.

Temporal stores the history of every workflow. If a worker process stops then another worker can continue from the same point. The work is not lost with the process.

This is what durable means here. The workflow state survives application restarts and infrastructure failures.

Temporal has a few main parts:

- the Temporal server stores workflow history and creates tasks
- the client starts workflows and reads their state
- workers take tasks from task queues
- workflows describe the order of the steps
- activities perform the real work

Temporal is not a replacement for the application database. It does not store products orders or payments for us. It stores the workflow history needed to continue the process.

It is also more than a message queue. A queue delivers messages. Temporal remembers the complete flow and knows which step should run next.

## Why Use Temporal?

A normal background service works well for many jobs. The problem starts when one job has several connected steps.

Suppose the application reserves inventory and then stops before charging the payment. On restart we need to answer a few questions:

- was inventory already reserved?
- should payment start now?
- did payment start but return no response?
- how many times should we retry?
- what should happen when payment fails?

Temporal keeps the workflow history and uses it to continue the process. We still write the business rules but we do not need to build the workflow engine ourselves.

## When Would I Use It?

Temporal is useful when work:

- has several connected steps
- must survive process restarts
- can take longer than one HTTP request
- needs retries and timeouts
- waits for another service or a human action

Order processing is one example. User onboarding approval flows scheduled jobs and document processing can have the same problem.

I would not use Temporal for every background job. A queue consumer can be enough for short independent work. A database status field can also be enough for a small flow.

Temporal becomes useful when recovery logic starts to take over the application code.

## How Does It Work?

The basic flow looks like this:

1. an application starts a workflow with an ID and some input
2. the Temporal server stores a workflow-started event
3. a worker receives the task from a task queue
4. the workflow decides which activity should run
5. another worker runs the activity
6. Temporal stores the activity result
7. the workflow continues to the next step

If a worker stops then the Temporal server still has the history. A worker can replay that history and rebuild the workflow state.

Workflow code must be deterministic. This means the same history must always produce the same decision. A workflow should not call a database or an HTTP API directly. It should schedule an activity instead.

Activities do the real work. They can update a database or call another service. Activities can fail so Temporal can retry them.

The [.NET SDK documentation](https://github.com/temporalio/sdk-dotnet#workflow-logic-constraints) lists the rules for workflow code. For example a workflow should not use `DateTime.Now` or `Task.Delay`. Temporal has its own safe methods for time and delays.

## The Demo

I built [a small .NET 10 demo](https://github.com/gabisonia/temporal-dotnet-demo-with-aspire) to see how this works with .NET Aspire.

The demo has a Shop API and a Payments API. The Shop API starts the workflow. Temporal then coordinates the order across both services.

The order follows this sequence:

```text
client
  |
  | POST /orders
  v
shop-api
  |
  | start OrderWorkflow on shop-tq
  v
reserve inventory
  |
  | run ChargePayment on payments-tq
  v
payments-api worker
  |
  v
mark order completed
```

The Shop API stores a pending order before it starts the workflow. This gives the client an order ID immediately. The workflow handles the remaining steps.

## Starting An Order

The Shop API creates the order and starts `OrderWorkflow` from the `POST /orders` endpoint:

```csharp
var orderId = Guid.NewGuid().ToString("N");
var amount = request.Quantity * product.Price;
var workflowInput = new OrderWorkflowInput(
    orderId,
    request.Id,
    request.Quantity,
    amount);

await store.CreatePendingOrderAsync(workflowInput, cancellationToken);

await temporalClient.StartWorkflowAsync(
    (IOrderWorkflow wf) => wf.RunAsync(workflowInput),
    new WorkflowOptions(
        id: $"order-{orderId}",
        taskQueue: TemporalTaskQueues.Shop));
```

The workflow ID is based on the order ID. This makes the workflow easy to find in Temporal UI.

The task queue is `shop-tq`. The Shop worker listens to this queue. It knows how to run `OrderWorkflow` and the shop activities.

```csharp
using var worker = new TemporalWorker(
    temporalClient,
    new TemporalWorkerOptions(TemporalTaskQueues.Shop)
        .AddWorkflow<OrderWorkflow>()
        .AddActivity(activities.ReserveInventoryAsync)
        .AddActivity(activities.MarkOrderCompletedAsync)
        .AddActivity(activities.MarkOrderFailedAsync));

await worker.ExecuteAsync(stoppingToken);
```

The API does not wait for the workflow to finish. It returns `202 Accepted` with the new order ID. The client can check the state later with `GET /orders/{orderId}`.

## Shop And Payments

The workflow contains the complete order sequence:

```csharp
[Workflow]
public class OrderWorkflow : IOrderWorkflow
{
    [WorkflowRun]
    public async Task<OrderWorkflowResult> RunAsync(OrderWorkflowInput input)
    {
        try
        {
            await Workflow.ExecuteActivityAsync(
                (IShopActivities act) => act.ReserveInventoryAsync(input),
                new ActivityOptions
                {
                    StartToCloseTimeout = TimeSpan.FromSeconds(15),
                });

            await Workflow.ExecuteActivityAsync(
                PaymentActivityNames.ChargePayment,
                [input.OrderId, input.Amount],
                new ActivityOptions
                {
                    StartToCloseTimeout = TimeSpan.FromSeconds(15),
                    TaskQueue = TemporalTaskQueues.Payments,
                });

            await Workflow.ExecuteActivityAsync(
                (IShopActivities act) => act.MarkOrderCompletedAsync(input.OrderId),
                new ActivityOptions
                {
                    StartToCloseTimeout = TimeSpan.FromSeconds(10),
                });

            return new OrderWorkflowResult(
                input.OrderId,
                "completed",
                "Order completed successfully.");
        }
        catch (Exception ex)
        {
            await Workflow.ExecuteActivityAsync(
                (IShopActivities act) => act.MarkOrderFailedAsync(input.OrderId, ex.Message),
                new ActivityOptions
                {
                    StartToCloseTimeout = TimeSpan.FromSeconds(10),
                });

            return new OrderWorkflowResult(input.OrderId, "failed", ex.Message);
        }
    }
}
```

The first and third activities run on `shop-tq`. The payment activity uses `payments-tq`.

This sends the payment task to the Payments worker. The workflow still controls the order of the steps. The payment code runs inside the Payments API process.

The Payments worker registers the activity by name:

```csharp
using var worker = new TemporalWorker(
    temporalClient,
    new TemporalWorkerOptions(TemporalTaskQueues.Payments)
        .AddActivity(activities.ChargePaymentAsync));
```

The payment database work stays inside the Payments service. The Shop service only knows the activity name and task queue.

## Not Every Failure Should Be Retried

Temporal retries activities by default. This is useful for temporary failures such as a database connection problem or a network timeout.

The demo also has a business rule. Payments above `5000` are declined:

```csharp
if (amount > 5_000m)
{
    payment.Status = "declined";
    await dbContext.SaveChangesAsync(cancellationToken);

    throw new InvalidOperationException(
        $"Payment declined for order '{orderId}' because amount exceeds limit.");
}
```

This is not a temporary failure. Trying the same amount again will give the same result.

The current demo throws a normal `InvalidOperationException`. The payment activity has a timeout but it has no retry limit. It also does not mark the decline as a permanent failure. Temporal treats it as a failure which can be retried. The workflow does not enter the `catch` block after the first decline. Temporal schedules the payment activity again.

Temporal's [retry policy documentation](https://docs.temporal.io/encyclopedia/retry-policies) describes this default. The wait becomes longer after every failed attempt. By default there is no limit on the number of attempts.

This means the README expectation and the current code do not fully match. A high-value order can remain in payment retries instead of moving to the failed state.

There are two ways to model this better:

- throw an `ApplicationFailureException` with `nonRetryable: true` for a permanent rejection
- configure the retry policy so payment declines are not retried

A real payment flow needs both paths. A connection failure should retry. A declined payment should return control to the workflow so it can decide what happens next.

## Activities Can Run More Than Once

Activity code must also be safe to run more than once.

Imagine that inventory is updated successfully but the worker stops before Temporal records the activity result. Temporal cannot know whether the database transaction completed. It runs the activity again.

The current inventory activity subtracts stock every time it runs:

```csharp
if (product.Stock < input.Quantity)
{
    throw new InvalidOperationException(
        $"Not enough inventory for '{input.ProductId}'.");
}

product.Stock -= input.Quantity;
order.Status = "inventory_reserved";

await dbContext.SaveChangesAsync(cancellationToken);
```

The order ID is available but it is not used to protect against repeated work. The activity can subtract the same quantity twice.

The technical word for this is idempotency. An idempotent operation can run more than once but the result stays the same as running it once.

In production I would first check whether this order has already reserved inventory. The database should only subtract stock once for each order ID. The same order ID should also be sent to an external payment provider as an idempotency key.

There is another missing step in the demo. If inventory is reserved and payment later fails then the stock is not restored. A complete workflow needs another activity which releases the inventory. This is called compensation.

Temporal remembers the workflow state. It does not decide our business rules. It also does not make database or API calls happen exactly once. That part still belongs to the application.

## Running Temporal With Aspire

Aspire makes the demo easy to run locally. The AppHost starts:

- PostgreSQL for Temporal
- the Temporal server
- Temporal UI
- PostgreSQL for application data
- the Shop API
- the Payments API

The important part of `AppHost.cs` is the resource wiring:

```csharp
var paymentsApi = builder
    .AddProject<Projects.TemporalDemo_Payments_Api>("payments-api")
    .WaitFor(temporalServer)
    .WaitFor(appDatabase)
    .WithReference(appDatabase)
    .WithEnvironment(
        "Temporal__Address",
        temporalGrpcEndpoint.Property(EndpointProperty.HostAndPort));

var shopApi = builder
    .AddProject<Projects.TemporalDemo_Shop_Api>("shop-api")
    .WaitFor(temporalServer)
    .WaitFor(appDatabase)
    .WithReference(appDatabase)
    .WithEnvironment(
        "Temporal__Address",
        temporalGrpcEndpoint.Property(EndpointProperty.HostAndPort));
```

Temporal uses its own PostgreSQL database for workflow history. The two APIs share another database for business data. Shop tables live in the `shop` schema and payment tables live in the `payments` schema.

Temporal history stores the state needed to run workflows. The application database still owns orders products inventory and payments.

## Running The Demo

The requirements are .NET 10 and Docker.

```bash
git clone https://github.com/gabisonia/temporal-dotnet-demo-with-aspire.git
cd temporal-dotnet-demo-with-aspire
dotnet restore TemporalDemo.slnx
dotnet run --project TemporalDemo.AppHost
```

The Aspire dashboard prints its URL in the terminal. From there you can open the Shop API and Payments API. Temporal UI is also available from the resource links.

First get the seeded products:

```http
GET /products
```

Then create an order for one laptop:

```http
POST /orders
Content-Type: application/json

{
  "id": "11111111-1111-1111-1111-111111111111",
  "quantity": 1
}
```

The laptop costs `1200` so this follows the successful path. The response contains an `orderId`. Use it to check the order and payment:

```http
GET /orders/{orderId}
GET /payments/{orderId}
```

The order should reach `completed` and the payment should reach `approved`.

You can search for `order-{orderId}` in Temporal UI. The event history shows each scheduled activity and its result.

To see the retry limitation create an order for five laptops. The amount becomes `6000`. The payment is stored as declined but the activity continues through the default retry policy. Temporal UI shows the pending activity and its attempt count.

Temporal gives long-running work a place to live outside one application process. It remembers the workflow history and continues after a worker restarts.

The workflow describes the steps. Activities update databases and call other services. Workers connect our code to the Temporal server.

It does not solve every problem. We still need to think about retries repeated activity calls and compensation. Temporal makes these problems easier to see and gives us tools to handle them.

[Source Code](https://github.com/gabisonia/temporal-dotnet-demo-with-aspire)
