# Emergency Drone Coordination System (EDCS)

A multi-threaded client-server system implemented in C that coordinates autonomous drone swarms using TCP sockets and thread-safe data structures. The project demonstrates high-concurrency handling through synchronized survivor tracking, real-time mission dispatching, and SDL-based visualization.

---

## Project Overview
This project implements a scalable, multi-threaded drone coordination system designed to simulate and manage search-and-rescue operations. Transitioning from a local simulator to a distributed network architecture, the system enables drones (clients) to communicate with a central server via TCP sockets to receive missions and coordinate aid for survivors.

The implementation mirrors real-world swarm logistics (e.g., Zipline, Amazon Prime Air), focusing on thread synchronization, network latency handling, and concurrent resource management.

---

## Architecture & Roadmap

The project is structured into three distinct development phases, evolving from a single-process simulation to a fault-tolerant distributed system.

### Phase 1: Core Synchronization & Simulation
Focus: Thread safety, Mutex locking, and Local Simulation.

* Thread-Safe Data Structures: Implementation of concurrent linked lists using pthread_mutex_t to handle race conditions during add, remove, and pop operations.
* Memory Management: Rigorous leak prevention in node destruction and buffer overflow protection using snprintf.
* Drone Logic: Drones operate as independent threads, calculating movement vectors towards targets.
* Survivor Generation: A separate thread generates random survivor coordinates to populate the synchronized mission list.
* Visualization: SDL-based rendering of the simulation grid (Red = Survivors, Blue = Drones, Green = Active Missions).

### Phase 2: Networked Communication Layer
Focus: Sockets, TCP/IP, and Client-Server Protocol.

* Server Architecture: A multi-threaded server listens for incoming drone connections and maintains the global state of the simulation.
* Drone Client: Drones transition from threads to standalone processes that connect to the server, sending JSON status updates.
* AI Controller: The server implements a dispatch algorithm to assign the closest idle drone to the oldest unhelped survivor, ensuring fairness and efficiency.
* Protocol: Custom JSON-based protocol for STATUS_UPDATE, ASSIGN_MISSION, and HEARTBEAT messages. 
    * See communication-protocol.md for full specs.

### Phase 3: Scalability & Optimization
Focus: Load Testing, Fault Tolerance, and QoS.

* High Concurrency: Optimized to handle 50+ concurrent drone connections.
* Fault Tolerance: Server logic detects disconnected drones (timeouts) and reassigns their missions to active agents.
* Advanced UI: Remote monitoring via WebSocket dashboard or a networked SDL client view.

---

## Technical Implementation

### 1. Data Structures & Concurrency
The core of the simulation relies on a custom List struct in list.h. Synchronization is granular:
* Locking Strategy: Mutexes are employed only during critical sections (modification of the list or node iteration) to maximize performance.
* Resource Management: A "Free List" mechanism is implemented to reuse nodes, reducing malloc/free overhead during high-frequency updates.

### 2. Simulation Logic (Snippets)

Drone Autonomous Behavior
Each drone runs a behavior loop that handles movement logic and thread locking:
```c
void* drone_behavior(void *arg) {
    Drone *d = (Drone*)arg;
    
    while(1) {
        pthread_mutex_lock(&d->lock);
        
        if(d->status == ON_MISSION) {
            // Logic to move 1 cell closer to target
            if(d->coord.x < d->target.x) d->coord.x++;
            else if(d->coord.x > d->target.x) d->coord.x--;
            
            if(d->coord.y < d->target.y) d->coord.y++;
            else if(d->coord.y > d->target.y) d->coord.y--;

            // Check for arrival
            if(d->coord.x == d->target.x && d->coord.y == d->target.y) {
                d->status = IDLE;
                printf("Drone %d: Mission completed!\n", d->id);
            }
        }
        
        pthread_mutex_unlock(&d->lock);
        sleep(1); // Simulation tick
    }
    return NULL;
}

Server Connection Handler: The server accepts connections and spawns handlers for each drone
```c
while (1) {
    int drone_fd = accept(server_fd, (struct sockaddr*)&client_addr, &addr_len);
    // Spawn a new thread to handle this specific drone connection
    pthread_create(&thread_id, NULL, handle_drone, (void*)&drone_fd);
}
```

### 3. Communication Protocol

Data is exchanged in JSON format for readability and extensibility.

Drone to Server (Status Update):
```
JSON
{ 
    "drone_id": "D1", 
    "status": "idle", 
    "location": [10, 20] 
}
```

Server to Drone (Mission Assignment):
```
JSON
{ 
    "type": "mission", 
    "target": [45, 12] 
}
```

## 🛠 Dependencies & Build
* Language: C (C11 Standard)

* Visualization: SDL2

* Data Parsing: json-c

* System: Linux/Unix (requires pthread library)

### Visualization Key

The SDL view provides real-time feedback on the system state:

* 🔴 Red Cells: Survivors awaiting aid.

* 🔵 Blue Dots: Idle drones patrolling or waiting.

* 🟢 Green Lines: Active mission vectors (Drone → Survivor).

## Key Features
* Thread Synchronization: Robust use of mutexes to prevent race conditions in shared lists.

* Networked AI: Centralized decision-making server managing distributed clients.

* Real-time Visualization: Graphical representation of the swarm logic.

* Fault Tolerance: Handles client disconnections and dynamic re-tasking.
