## Overview

In **LAG (Link Aggregation Group)** or **multi-plane** setups, a Virtual Function (VF)'s effective bandwidth equals the sum of all **Physical Functions (PFs)** participating in the aggregation.
By default, VFs only expose a single underlying link speed and cannot report their true aggregated bandwidth.

This feature introduces support for:

- **Aggregated speed reporting** — VFs report the *sum* of all PF speeds in the group.
- **Asynchronous bandwidth notifications** — VFs receive events when their effective bandwidth changes (e.g., when a PF link goes down or comes back up).

This enhancement allows upper-layer tools and applications to make better **traffic distribution** and **capacity decisions**.

***

## Interfaces and APIs

### New Verb: `ibv_query_port_speed`

```c
int ibv_query_port_speed(struct ibv_context *context, uint8_t port_num, uint64_t *port_speed);
```

**Description**
Returns the effective link speed of the specified port (in units of 100 Mbps) as seen from the given function (PF/VF/SF).

- For VFs in LAG or MP‑Ethernet: returns aggregated speed (sum of PF speeds).
- For non-aggregated ports: returns underlying port link speed.

**Parameters**


| Name | Description |
| :-- | :-- |
| `context` | RDMA device context for the function being queried. |
| `port_num` | Port number within that context (same as existing `ibv_query_port` semantics). |
| `port_speed` | Output pointer to store the current effective link speed. |

**Return Value**

- `0` on success.
- Negative `errno` on failure (e.g., unsupported feature or invalid port).

***

### New Async Event: `IBV_EVENT_DEVICE_SPEED_CHANGE`

This asynchronous event is triggered when a function’s effective bandwidth changes.

**When generated**

- A PF in a LAG / MP‑Ethernet group goes down or comes back up.
- A configuration change affects the aggregated VF speed.

**Usage Example**

```c
struct ibv_async_event event;

while (!done) {
    if (!ibv_get_async_event(context->async_fd, &event)) {
        if (event.event_type == IBV_EVENT_DEVICE_SPEED_CHANGE) {
            /* re-query speed and update application state */
        }
        ibv_ack_async_event(&event);
    }
}
```

Upon receiving `IBV_EVENT_DEVICE_SPEED_CHANGE`, the application should re-query the speed via `ibv_query_port_speed()` and update its internal state accordingly.

***

## BlueField Environment

On **BlueField** platforms:

- The **ECPF** acts as the embedded switch (eswitch) manager for all PFs, VFs, and SFs.
- The ECPF context manages and updates speed info for all functions under its control.
- Works transparently across PFs, VFs, and SFs on supported **BlueField** and **ConnectX** devices.

***

## Feature Support and Compatibility

| Device | Support Status |
| :-- | :-- |
| ConnectX‑7 (CX7) | ✅ Supported |
| Newer generations | ✅ Supported |
| Older devices | ❌ Not supported |

When unsupported:

- `ibv_query_port_speed()` fails with an appropriate error.
- No `IBV_EVENT_DEVICE_SPEED_CHANGE` events are generated.

***

## Port-Down Behavior

- If **all PFs are down** but the VF is still administratively up, the VF reports a non-zero *maximum* (loopback-only) speed.
- If the **VF itself is down**, `ibv_query_port_speed()` reports `0`.

***

## Example Usage Scenario

1. **Setup**: Create a LAG or MP‑Ethernet with multiple PFs and one VF per PF.
2. **Initial Query**:
Call `ibv_query_port_speed()` → returns sum of contributing PF link speeds.
3. **PF Failure**:
    - When a PF goes down, `IBV_EVENT_DEVICE_SPEED_CHANGE` is emitted.
    - Requery aggregated speed → reflects remaining active PFs.
4. **All PFs Down**:
    - VF still up → reports theoretical loopback speed.
    - VF down → reports 0.
5. **Recovery**:
    - PFs go back up → event triggered again.
    - Requery → full aggregated bandwidth restored.

***

## Supporting Versions

| Component | Version |
| :-- | :-- |
| **Firmware (FW)** | 40_48_0302 |
| **Upstream Kernel** | 6.19‑rc5 |
| **rdma-core** | 62 |


***

