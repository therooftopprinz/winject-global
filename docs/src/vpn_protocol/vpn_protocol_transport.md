# transport - tunnel and mesh

## node types:
- server node : provides IP and default L3 switching
- gate node : can do L3 switching over gate2gate tunnel, relay foreign node
- foreign node : it connects the mesh over the gate nodes

## connections:
- Gate nodes connects to server node and aquires IP.
- Foreign node connects to gate node, and aquires IP.
- Gate nodes connects to all other gate nodes through hole punching, it maintains link state.
- Server node pick up all gate node link states to build a mesh map
- A gate node can forward tunnel message from one node to other node (non server node).

## messages
```
sequence ipv4_request
{
  u16 request_id;
  u8 client_public_key[];
};

sequence ipv4_response
{
  u16 request_id;
  u8 status;
  optional u32 ipv4;
};

sequence gate_list_request
{
  request_id;
};

sequence gate
{
  u64 gate_id;
  choice
  {
    u32 v4;
    u128 v6;
  } public_access_host;
  u16 public_access_port;
  u32 lan_host_v4;
  u16 lan_port;

  buffer<u8> gate_public_key;
};

sequence gate_list_response
{
  u16 request_id;
  list<gate> gate_list;
};

sequence gate_update
{
  u8 action;
  choice {
    gate gate;
    u64 gate_id;
  } update;
};

sequence gate_list_update_indication
{
  u16 request_id;
  list<gate_update> gate_update;
};

sequence gate_list_update_acknowledge
{
  u16 request_id;
};

sequence hole_punch_request
{
  u16 request_id;
  u64 peer_gate_id;
};

sequence hole_punch_sync_request
{
  u16 request_id;
  u64 peerA_gate_id;
  u64 peerB_gate_id;
};

sequence hole_punch_sync_response
{
  u16 request_id;
};

sequence gate2gate_state
{
  u64 peer_gate_id;
  u64 forward_trip_time_us;
  u64 reverse_trip_time_us;
  u64 forward_drop_rate_pkt_s;
  u64 reverse_drop_rate_pkt_s;
  u64 forward_throughput_byt_s;
  u64 reverse_trhoughput_byt_s;
};

sequence gate2gate_state_indication
{
  u64 source_gate_id;
  list<gate2gate_state> link_list;
};

sequence route_candidates_request
{
  u64 request_id;
};

sequence routes;

sequence route_candidates_response
{

}

sequence l3_tunnel
{
  list<u64> gateway_route;
  buffer<u8> l3_sdu;
}
```

## callflows:

### participants
A,B,C,D - gate<br/>
E,F,G,H - foreign<br/>
Z - server

### gate node initial flow (A:Z), IP Address Acquisition
```mermaid
sequenceDiagram
    participant A as A (gate)
    participant Z as Z (server)
    Note over A,Z: A:Z session creation
    A->>Z: address_request
    Z-->>A: address_response
```

### foreign node initial flow (E:A), IP Address Acquisition
```mermaid
sequenceDiagram
    participant E as E (foreign)
    participant A as A (gate)
    participant Z as Z (server)
    Note over A,Z: A:Z session
    Note over E,A: E:A session creation
    E->>A: address_request
    A->>Z: address_request
    Z-->>A: address_response
    A-->>E: address_response
```

### tunnel flow (A:B, via default), direct path not available
```mermaid
sequenceDiagram
    participant A as A (gate)
    participant Z as Z (server)
    participant B as B (gate)
    Note over A,Z: A:Z session
    Note over Z,B: Z:B session
    A->>Z: tunnel
    Z->>B: tunnel
    B-->>Z: tunnel
    Z-->>A: tunnel
```

### tunnel flow (A:B direct), direct path becomes available
```mermaid
sequenceDiagram
    participant A as A (gate)
    participant B as B (gate)
    Note over A,B: hole-punching via Z
    Note over A,B: A:B session creation
    A->>B: tunnel
    B-->>A: tunnel
```

### full tunnel flow (E:F, via A,B)
```mermaid
sequenceDiagram
    participant E as E (foreign)
    participant A as A (gate)
    participant Z as Z (server)
    participant B as B (gate)
    participant F as F (foreign)
    Note over E,A: E:A session
    Note over A,Z: A:Z session
    Note over Z,B: Z:B session
    Note over B,F: B:F session
    Note over A,B: new best path identified
    par A to B
        A->>B: mesh_update_ind
    and B to A
        B->>A: mesh_update_ind
    end
    par A to B
        A->>B: mesh_update_ack
    and B to A
        B->>A: mesh_update_ack
    end
    Note over A,B: UDP hole punching
    Note over A,B: A:B session creation
    Note over E,A: new best path available
    E->>A: tunnel
    A->>B: tunnel
```
