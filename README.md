# Notification System Architecture

## Overview

This notification system is designed to bdecouples event producers from notification delivery using **RabbitMQ** as the message broker and delivers updates to clients in real-time via **SignalR**.

## High-Level Flow

1. **Event Trigger**

   * Any business or system event (e.g., user created, data updated, system alert).

2. **Producer (Backend Service or API Layer)**

   * Receives the event and generates a notification payload.
   * Persists the notification in the **database** for history and tracking.
   * Publishes the notification to a **RabbitMQ exchange** with a routing key.

3. **Message Broker (RabbitMQ Exchange + Queues)**

   * The RabbitMQ exchange routes notifications into queues based on rules (e.g., user, group, priority).
   * Example bindings could include:

     * Role-based queues.
     * Priority queues (e.g., high-priority vs normal).
     * User-specific or broadcast queues.
   * **Dead Letter Queue (DLQ)** stores messages that cannot be delivered after retries for further inspection and troubleshooting.

4. **Consumer Service (Worker)**

   * Subscribes to the relevant RabbitMQ queues.
   * Uses acknowledgment strategies to ensure reliable delivery.
   * Applies retry logic, moving failed messages to the DLQ when necessary.
   * Delivers notifications to the **SignalR hub**.
   * Updates the **database delivery status** (e.g., DeliveredAt, DeliveryAttempts, Status).

5. **Real-Time Hub (SignalR)**

   * Manages client connections.
   * Routes messages to connected users or groups.
   * Avoids persistence to prevent duplication (persistence handled by the database).

6. **Frontend Application (React Example)**

   * Connects to the SignalR hub after authentication.
   * Listens for notification events.
   * Displays notifications in real-time using UI components (e.g., toasts, banners, alerts).

---
<img width="1348" height="818" alt="image" src="https://github.com/user-attachments/assets/c57ca508-0086-4c67-a37d-09ec5e26a23a" />



