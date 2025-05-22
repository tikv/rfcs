# **RFC: PD Endpoints to Serialize TiKV Maintenance Operations**

- Tracking Issue: https://github.com/tikv/pd/issues/9041

## **Summary**

This proposal introduces new PD endpoints to serialize TiKV maintenance operations, reducing the risk of accidental Raft quorum loss during such processes.

---

## **Motivation**

In modern cloud environments, TiKV clusters require routine maintenance operations—such as node rotations, upgrades, or rolling restarts—to be executed safely. Without proper coordination, multiple operations could overlap and unintentionally break Raft quorum, leading to data inaccessibility or cluster unavailability.

To address this, we propose a unified mechanism to serialize TiDB maintenance operations and prevent concurrent destructive actions of the same type.

---

## **Goals**

* Introduce PD endpoints that allow Kubernetes operators to coordinate and serialize maintenance operations of the same type.  
* Provide a mechanism to:  
  * Mark the start of a task.  
  * Mark the completion of a task.  
  * Query ongoing maintenance task information, including the start time and description.

---

## **Non-Goals**

* This proposal does not prescribe how orchestrators (e.g., TiDB Operator) should handle retries, failures, or other error-handling policies. These will be left to the implementation of the orchestrating system.

---

## **Proposal**

In this proposed design, PD serves as a system of record for tracking ongoing destructive tasks, leveraging embedded etcd transactions to ensure atomic updates. However, the responsibility for enforcing constraints and serializing tasks remains with the orchestrator (e.g., the Kubernetes operator). PD itself does not enforce these policies—by design—allowing for maximum flexibility in task scheduling.

Maintenance tasks can differ significantly in behavior and duration. For instance, a single node restart might require interleaving after running for more than 30 minutes, whereas a full AZ-wide TiKV rolling restart could take several hours. By delegating control to the orchestrator, the PD-side endpoint design remains extensible and versatile, making it compatible with various orchestrators and a wide range of task types.

Please see the proposed new PD endpoints below.

### **New PD Endpoints**

| Function | Endpoint | Method | Description |
| ----- | ----- | ----- | ----- |
| **Func 1** | `/maintenance/{task_type}/{task_id}` | `POST` | Marks the beginning of a maintenance task. Accepts an optional description in the request body. |
| **Func 2** | `/maintenance/{task_type}` | `GET` | Retrieves information about the ongoing maintenance task for the specified type. |
| **Func 3** | `/maintenance/{task_type}/{task_id}` | `DELETE` | Marks the completion of a task if the specified ID matches the current task owner. |

---

## **Use Scenarios**

### **Happy Path**

1. Before performing a maintenance operation, the orchestrator issues a `POST` to register the start of the task.  
2. If a `2XX` response is returned, the operation proceeds.  
3. If a `409` response is returned, another task is already running; retry later.  
4. Use `GET` to inspect ongoing tasks.  
5. Once completed, the orchestrator issues `DELETE` to release the lock.

### **Error Handling**

1. If a task appears to be stuck (e.g., due to Kubernetes operator crash), the current lock can be queried using `GET`.  
2. The Kubernetes operator can then decide to override the lock via `DELETE` (based on policy).

---

## **Implementation Design**

PD will store a key-value entry in etcd to act as a lock. The value will be a JSON object containing the task ID, start timestamp (epoch), and an optional description.

**Example:**

### **Request**

`POST /maintenance/task_tikv/123`

**Body:**

`Upgrade rolling restart for TiKV store-1`

### **Stored in etcd:**

**Key**: `/lock/maintenance/task_tikv`

**Value**:

`{`  
  `"id": "123",`  
  `"start_timestamp": 1712676600,`  
  `"description": "Upgrade rolling restart for TiKV store-1"`  
`}`

### **Endpoint Handler Logic**

#### **Func 1 – `POST /maintenance/{task_type}/{task_id}`**

* If `/lock/maintenance/{task_type}` does **not** exist:  
  * Write the key-value to etcd.  
  * Return `201`.  
* Else:  
  * Return `409 Conflict`.

#### **Func 2 – `GET /maintenance/{task_type}`**

* If the key exists:  
  * Return `200`.  
  * Respond with a JSON body, for example

     `{`  
       `"id": "123",`  
       `"start_timestamp": 1712676600,`  
       `"description": "Upgrade rolling restart for TiKV store-1"`  
     `}`

* Else:  
  * Return `404`.

#### **Func 3 – `DELETE /maintenance/{task_type}/{task_id}`**

* If the key exists:  
  * Parse and verify the `task_id`.  
  * If it matches:  
    * Delete the key.  
    * Return `200`.  
  * Else:  
    * Return `409`.  
* If the key does not exist:  
  * Return `404`.

---

## **Observability**

Define Prometheus metrics to track the state of ongoing tasks:

`prometheus.NewGaugeVec(`  
    `prometheus.GaugeOpts{`  
        `Name: "pd_maintenance_task_info",`  
        `Help: "Current maintenance task info in PD",`  
    `},`  
    `[]string{"task_type", "task_id"},`  
`)`

* When a task starts:

`maintenanceTaskInfo.WithLabelValues("task_tikv", "store-123").Set(1)`

* When a task ends:

`maintenanceTaskInfo.WithLabelValues("task_tikv", "store-123").Set(0)`

---

## **CLI Support**

To complement the HTTP endpoints, we will extend **`pd-ctl`** to support the following subcommands for interacting with maintenance tasks:

* `pd-ctl maintenance set <task_type> <task_id> --desc="<description>"`  
* `pd-ctl maintenance show <task_type>`  
* `pd-ctl maintenance delete <task_type> <task_id>`

This allows users and Kubernetes operators to manually coordinate maintenance tasks or script basic workflows, improving observability and flexibility during cluster operations.

---

## **TTL vs. Explicit Start Timestamp in etcd (Design Decision)**

### **Option 1: TTL-Based Entry**

**Pros:**

* Automatically cleans up stale tasks if Kubernetes operators crash or fail.  
* Prevents indefinite blocking.

**Cons:**

* Less flexible for Kubernetes operator-defined policies.  
* TTL tuning introduces complexity and edge cases.

### **Option 2: Explicit Start Timestamp (Chosen)**

**Pros:**

* Operators retain full control over task lifecycles.  
* Better observability and policy customization.

**Cons:**

* Requires manual intervention for stuck tasks.  
* No auto-recovery if orchestrators crash.

We adopt **explicit start timestamp** for greater flexibility and control.

---

## **Alternative Solution Considered**

As proposed in [\#9401](https://github.com/tikv/pd/issues/9041), a dedicated **store maintenance scheduler** was considered. In this approach, all destructive store-level operations would be gated by the scheduler, which ensures only one store is under operation at any given time.

### **Drawbacks of the Scheduler Approach**

* **Reduced Flexibility**: All operations would have to go through the maintenance scheduler, limiting the ability to define and plug in custom policies.  
* **Code Duplication**: The scheduler’s logic would significantly overlap with the existing TiDB Operator mechanisms for upgrade and rolling restarts, likely leading to duplicated code and reduced maintainability.


