# Implementation Guide

## Overview

This document explains how to implement the notification system described in **Architecture.md**, step by step. It covers setup, configuration, and integration of **RabbitMQ**, **.NET backend with SignalR**, and a **React frontend**.

---

## 1. RabbitMQ Setup

### Installation Options

* **Local installation** on Windows/Linux server.
* **Docker container** (recommended for development).
* **Managed RabbitMQ service** (e.g., CloudAMQP) for production.

### Enable Management UI

```bash
rabbitmq-plugins enable rabbitmq_management
```

* Access at: `http://localhost:15672` (default user/pass: guest/guest).

### Exchange and Queues

* Create a **topic exchange**: `notifications-exchange`.
* Bind queues (examples):

  * `role_based_queue` ← messages routed by user roles.
  * `priority_queue` ← messages filtered by priority levels.
  * `user_queue` ← messages targeted to specific users.
  * `broadcast_queue` ← messages sent to all subscribers.
* Configure **Dead Letter Exchange (DLX)** for failed messages.

---

## 2. .NET Backend Setup

### Producer (API Layer)

* On event trigger (e.g., new user, alert):

  * Generate a **Notification DTO**.
  * Save it to the **database**.
  * Publish it to `notifications-exchange` with a **routing key**.

```csharp
using var connection = new ConnectionFactory { HostName = "localhost" }.CreateConnection();
using var channel = connection.CreateModel();

channel.ExchangeDeclare("notifications-exchange", ExchangeType.Topic, durable: true);

var body = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(notification));
channel.BasicPublish("notifications-exchange", routingKey, null, body);
```

### Consumer (Worker Service)

* Separate **.NET Worker Service** that:

  * Declares queues and bindings.
  * Subscribes to messages with **manual ack**.
  * Delivers to **SignalR hub**.
  * Updates DB delivery status.

---

## 3. SignalR Hub

* Define a `NotificationHub`:

```csharp
public class NotificationHub : Hub
{
    public async Task Deliver(NotificationDto notification, string targetGroup)
    {
        await Clients.Group(targetGroup).SendAsync("ReceiveNotification", notification);
    }
}
```

* Users join groups on login.

---

## 4. React Frontend

* Connect to the hub on user login:

```javascript
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
    .withUrl("https://localhost:{port}/notifications")
    .build();

connection.on("ReceiveNotification", (notification) => {
  // Show toast (e.g., Sonner)
});

connection.start();
```

* Use a UI library (e.g., Sonner toast) to display notifications.

---

## 5. Database Schema

### `Notifications`

| Column         | Type      | Notes                             |
| -------------- | --------- | --------------------------------- |
| Id             | Guid      | Primary key                       |
| UserId         | Guid      | Target user (nullable)            |
| Group          | string    | Target group (e.g., super\_admin) |
| Title          | string    | Notification title                |
| Message        | string    | Notification body                 |
| Priority       | int       | 0=normal, 2=high                  |
| CreatedAt      | DateTime  |                                   |
| DeliveredAt    | DateTime? |                                   |
| DeliveryStatus | string    | Pending, Delivered, Failed        |

---

## 6. Error Handling

* **Retries**: Failed deliveries retried a limited number of times.
* **DLQ**: Undeliverable messages moved to DLQ.
* **Monitoring**: Use RabbitMQ UI or Prometheus exporters.

---

