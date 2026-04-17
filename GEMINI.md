# Project: Bostrom Simulation Substrate (Environment Build)

## 1. System Role & Project Constraints
You are an expert infrastructure and AI systems engineer. Your current task is to build the backend "Environment" (the simulation substrate) using Python (FastAPI) and a relational database (Google Cloud SQL/PostgreSQL). 
**Crucial Constraint:** You are NOT building the agents right now. You are building the physical laws, database schemas, and API endpoints that future agents will inhabit. 

## 2. The State Space (Topology)
The environment is not continuous; it is a discrete mathematical space.
* **Structure:** The world is modeled as a connected graph $G = (V, E)$, where $V$ represents discrete locations (nodes) and $E$ represents traversable paths between them. 
* **Initial Size:** A grid graph of $10 \times 10$ (100 total nodes).
* **Coordinates:** Each node is identifiable by a tuple $(x, y)$.

## 3. The Chronology (Time)
* Time is discrete and progresses in sequential ticks, denoted as $t \in \mathbb{N}$.
* The system must have a central `TickManager` service. An agent's action at time $t$ is resolved before the state transitions to $t+1$.

## 4. The Laws of Physics (Resource Conservation)
The simulation contains a fundamental resource called "Energy" ($E$). The system is closed; energy can be transferred but never created or destroyed.
* Total energy is defined as:
    $$E_{total} = \sum_{i=1}^{|A|} e_i(t) + \sum_{j=1}^{|V|} r_j(t)$$
    Where $A$ is the set of all agents, $e_i(t)$ is the energy of agent $i$ at time $t$, $V$ is the set of all nodes, and $r_j(t)$ is the ambient energy stored at node $j$ at time $t$.
* **Constraint:** $E_{total} = 10000$ at all times $t$. Any API call that attempts an energy transfer resulting in a violation of this equation must return a `400 Bad Request` (Physics Violation).

## 5. Agent Interfaces (The APIs)
The backend must expose these specific endpoints for the future agents to call:
* `GET /observe/{agent_id}`: Returns the current state of the agent's node $(x,y)$ and adjacent nodes.
* `POST /move/{agent_id}`: Requires a target adjacent node. Costs $1$ unit of agent energy $e_i$.
* `POST /harvest/{agent_id}`: Transfers energy from the node $r_j$ to the agent $e_i$. 

## 6. Development Rules
* Write modular code. Separate the `database.py` models from the `routes.py`.
* Include unit tests specifically proving the Conservation of Energy equation holds after 100 random transfers.

## 7. Initial State & Initialization (t=0)
When the database is initialized, the `TickManager` starts at $t=0$ and the environment must be seeded with a perfectly uniform distribution of energy.
* **Nodes:** Every node $j \in V$ begins with exactly $100$ units of ambient energy. $r_j(0) = 100$.
* **Total Check:** Since $|V| = 100$, the sum of all node energy is $100 \times 100 = 10000$. The initial state perfectly satisfies the Conservation of Energy law.
* **Agents:** Agents spawn with $0$ internal energy ($e_i(0) = 0$). To survive or move, their very first action in the simulation must be to call `POST /harvest`.
* **Exhaust Rule:** When an agent successfully calls `POST /move`, their internal energy $e_i$ decreases by $1$, and the ambient energy $r_j$ of the node they *arrived at* increases by $1$ (Kinetic Exhaust).

## 8. The Spawning Mechanic (The Eden Node)
* All agents must be initialized at the exact same coordinate: `(5, 5)`. 
* At $t=0$, Node `(5, 5)` contains exactly 100 units of ambient energy, just like every other node.
* Because agents spawn with 0 internal energy and moving requires 1 unit of energy, agents are initially stranded on `(5, 5)` until they successfully harvest.

## 9. The Harvest Limit (The Throttle)
To prevent immediate resource monopolies, the environment enforces a physical limit on energy extraction per tick, denoted as $h_{max}$.
* **Constraint:** $h_{max} = 10$. 
* Any `POST /harvest` request where the requested amount $> h_{max}$ must be rejected by the API with a `400 Bad Request`.
* If the ambient energy of the node $r_j$ is less than $h_{max}$, the agent may only harvest up to $r_j$.
* **Concurrency Rule:** If multiple agents on the same node call `POST /harvest` in the same tick, the server must process the requests sequentially using database row-level locking to prevent race conditions.