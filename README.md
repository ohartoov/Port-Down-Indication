Overview
In LAG (Link Aggregation Group) or multi-plane setups, a VF’s effective bandwidth is the sum of the speeds of all PFs that participate in the aggregation. By default, VFs typically only expose the speed of a single underlying link and cannot report their true aggregated bandwidth.

This feature enables VFs to:

Report the aggregated link speed (sum of all PFs in the LAG / MP-Ethernet group).

Receive asynchronous notifications when their effective bandwidth changes (for example, when a PF goes down or comes back up).

This allows upper-layer tools and applications to make better traffic distribution and capacity decisions.

Interfaces and APIs
New verb: ibv_query_port_speed
c
int ibv_query_port_speed(struct ibv_context *context,
                         uint8_t              port_num,
                         uint64_t            *port_speed);
Description

Returns the effective link speed of the specified port, in 100Mbps, as seen from the given function (PF/VF/SF).

For VFs in LAG / MP-Ethernet configurations, this is the sum of all contributing PF link speeds.

For non-aggregated ports, this returns the normal link speed of the underlying port.

Parameters

context – RDMA device context for the function (PF/VF/SF) whose port you are querying.

port_num – Port number on that context (as in existing ibv_query_port semantics).

port_speed – Output parameter. On success, set to the current effective link speed in 100Mbps.

Return value

0 on success.

Negative errno on failure (for example, if the feature is not supported or the port does not exist).

New async event: IBV_EVENT_DEVICE_SPEED_CHANGE
A new asynchronous event is added to notify applications when the effective link speed of a function changes.

Event type

c
IBV_EVENT_DEVICE_SPEED_CHANGE
When it is generated

Whenever the effective bandwidth for a function changes, for example:

A PF participating in a LAG / MP-Ethernet group goes down or comes back up.

A configuration change affects the aggregated speed seen by the VF.

How to use

Applications should listen on the async event channel using the existing verbs mechanism:

c
struct ibv_async_event event;

while (!done) {
    if (!ibv_get_async_event(context->async_fd, &event)) {
        if (event.event_type == IBV_EVENT_DEVICE_SPEED_CHANGE) {
            /* re-query speed and update application state */
        }
        ibv_ack_async_event(&event);
    }
}
When IBV_EVENT_DEVICE_SPEED_CHANGE is received, the application is expected to call ibv_query_port_speed() to retrieve the new effective speed and adjust its logic accordingly.

BlueField Environment (ECPF / ESW Manager)
On BlueField platforms:

The ECPF acts as the embedded switch (eswitch) manager for all PFs, VFs and SFs on the host.

The ECPF context manages and updates speed information for all functions it owns, similar to a regular host context.

The feature works transparently across PFs, VFs and SFs on supported BlueField and ConnectX generations.

Feature Support and Compatibility
This feature is supported on:

ConnectX‑7 (CX7) and newer devices and corresponding BlueField generations that support VF‑LAG / MP‑Ethernet in hardware and firmware.

If the feature is not supported by a given device:

ibv_query_port_speed() will fail with an appropriate error code.

IBV_EVENT_DEVICE_SPEED_CHANGE events will not be generated.

Port‑Down Behavior
When interpreting the returned speed, note the following:

If all contributing PFs are down but the VF itself is still administratively up, the VF still reports a non‑zero “maximum” speed due to optional internal loopback traffic paths on the device. This represents the theoretical maximum and not usable external bandwidth.

If the VF itself is also down, ibv_query_port_speed() will report 0 for that VF.

Example Usage Scenario
The following scenario demonstrates how to use the new verb and async event in a VF‑LAG / MP‑Ethernet environment.

Initial speed query

Configure a LAG or MP‑Ethernet setup with multiple PFs and one VF per PF.

In the application, call ibv_query_port_speed() on the VF context and port.

Equals the sum of contributing PF speeds in MP‑Ethernet setups.

PF removal / failure

Administratively “remove” (down) one PF that participates in the aggregation.

The application listens for async events with ibv_get_async_event().

When IBV_EVENT_DEVICE_SPEED_CHANGE is received:

Acknowledge the event with ibv_ack_async_event().

Call ibv_query_port_speed() again.

The sum now is of the remaining active PFs (MP‑Ethernet).

All PFs down

Disable all PFs in the aggregation.

IBV_EVENT_DEVICE_SPEED_CHANGE event is received.

Query the speed again:

If the VF remains up, speed may reflect the device’s internal maximum (loopback‑only).

If the VF is also down, the reported speed is 0.

Recovery

Bring PFs back up.

Receive a new IBV_EVENT_DEVICE_SPEED_CHANGE event.

Call ibv_query_port_speed() and the speed returns to the full aggregated value.

Supporting Versions

FW: 40_48_0302
Upstream: 6.19-rc5
rdma-core : 62
