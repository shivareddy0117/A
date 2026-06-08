# Elevator System — Low Level Design

**Scope (locked):** single building, `N` elevators, `M` floors, directional hall calls, pluggable LOOK/SCAN dispatch, single-threaded core logic.
**Out of scope:** weight/capacity, door timing, emergency/fire override, persistence, UI.

---

## 1. Clarification / Follow-up Questions

These are the questions to ask the interviewer. Each is paired with the default I'd assume (state the assumption, let them correct it — saves time).

| # | Question | Default assumption |
|---|----------|--------------------|
| 1 | How many elevators and floors? | N cars, M floors, single building |
| 2 | Are hall (external) calls directional (separate UP/DOWN buttons) or a single call button? | Directional |
| 3 | Single car or multiple cars needing dispatch? | Multiple → need a scheduler |
| 4 | How smart must dispatch be — wait-time optimal, or a reasonable heuristic? | Heuristic (nearest-suitable car) over a LOOK base |
| 5 | Do we model door states/timing, capacity/weight? | No — treat as extensions |
| 6 | Are priority/express/freight/VIP requests in scope? | No |
| 7 | Real concurrency (thread-safe) or single-threaded simulation? | Single-threaded core; concurrency is an extension |
| 8 | Any special floors (basement, lobby express)? | No; floors are `0..M-1` |

**Key insight to surface:** the single most important modeling decision is distinguishing **hall requests** (a floor + a desired direction, dispatched to *some* car) from **car requests** (a specific car + a target floor). This split drives the entire class design.

---

## 2. Entities / Classes / Interfaces

**Enums**
- `Direction` — `UP`, `DOWN`, `IDLE`
- `ElevatorState` — `IDLE`, `MOVING`, `DOORS_OPEN`, `MAINTENANCE`

**Domain / Command objects** (request hierarchy = Command pattern)
- `Request` *(abstract)* — base, holds `floor`
- `HallRequest` — external call: `floor` + `direction`
- `CarRequest` — internal call: `elevatorId` + target `floor`

**Core**
- `Elevator` — owns its position, direction, state, and its pending stops (split into `upStops` / `downStops` TreeSets for LOOK)
- `ElevatorController` — orchestrator/Singleton; holds the cars and the dispatch strategy; entry point for both request types; drives the simulation `step()`

**Strategy**
- `SchedulingStrategy` *(interface)* — `selectElevator(cars, hallRequest)`
- `NearestElevatorStrategy` — concrete heuristic (idle-distance + same-direction-on-the-way scoring)
- *(alt: `FCFSStrategy`, `LeastBusyStrategy` — swappable)*

**Optional / mention-if-time**
- `Building`, `Floor`, `Button` (`HallButton` / `CarButton`), `Display`, `Door`

---

## 3. UML Class Diagram

```mermaid
classDiagram
    class Direction {
        <<enumeration>>
        UP
        DOWN
        IDLE
    }
    class ElevatorState {
        <<enumeration>>
        IDLE
        MOVING
        DOORS_OPEN
        MAINTENANCE
    }
    class Request {
        <<abstract>>
        #int floor
        +getFloor() int
    }
    class HallRequest {
        -Direction direction
        +getDirection() Direction
    }
    class CarRequest {
        -int elevatorId
        +getElevatorId() int
    }
    class Elevator {
        -int id
        -int currentFloor
        -Direction direction
        -ElevatorState state
        -TreeSet~Integer~ upStops
        -TreeSet~Integer~ downStops
        +addStop(int floor) void
        +step() void
        +isIdle() boolean
        +getCurrentFloor() int
        +getDirection() Direction
    }
    class SchedulingStrategy {
        <<interface>>
        +selectElevator(List~Elevator~, HallRequest) Elevator
    }
    class NearestElevatorStrategy {
        +selectElevator(List~Elevator~, HallRequest) Elevator
    }
    class ElevatorController {
        -List~Elevator~ elevators
        -SchedulingStrategy strategy
        +requestElevator(int floor, Direction dir) void
        +selectFloor(int carId, int target) void
        +setStrategy(SchedulingStrategy) void
        +step() void
    }

    Request <|-- HallRequest
    Request <|-- CarRequest
    SchedulingStrategy <|.. NearestElevatorStrategy
    ElevatorController "1" o-- "N" Elevator
    ElevatorController "1" --> "1" SchedulingStrategy
    Elevator ..> Direction
    Elevator ..> ElevatorState
    ElevatorController ..> HallRequest
    NearestElevatorStrategy ..> Elevator
```

---

## 4. Complete Solution (Java)

```java
// ---------- Enums ----------
public enum Direction { UP, DOWN, IDLE }

public enum ElevatorState { IDLE, MOVING, DOORS_OPEN, MAINTENANCE }

// ---------- Request hierarchy (Command pattern) ----------
public abstract class Request {
    protected final int floor;
    protected Request(int floor) { this.floor = floor; }
    public int getFloor() { return floor; }
}

public class HallRequest extends Request {       // external call
    private final Direction direction;
    public HallRequest(int floor, Direction direction) {
        super(floor);
        this.direction = direction;
    }
    public Direction getDirection() { return direction; }
}

public class CarRequest extends Request {        // internal call
    private final int elevatorId;
    public CarRequest(int elevatorId, int floor) {
        super(floor);
        this.elevatorId = elevatorId;
    }
    public int getElevatorId() { return elevatorId; }
}

// ---------- Elevator (LOOK algorithm) ----------
import java.util.TreeSet;

public class Elevator {
    private final int id;
    private int currentFloor;
    private Direction direction;
    private ElevatorState state;

    // stops split by travel direction => clean LOOK scheduling
    private final TreeSet<Integer> upStops = new TreeSet<>();
    private final TreeSet<Integer> downStops = new TreeSet<>(java.util.Collections.reverseOrder());

    public Elevator(int id) {
        this.id = id;
        this.currentFloor = 0;
        this.direction = Direction.IDLE;
        this.state = ElevatorState.IDLE;
    }

    /** Register a floor this car must stop at (from either a hall or car request). */
    public void addStop(int floor) {
        if (floor == currentFloor) {
            openDoors();
            return;
        }
        if (floor > currentFloor) upStops.add(floor);
        else                      downStops.add(floor);

        if (direction == Direction.IDLE) {
            direction = (floor > currentFloor) ? Direction.UP : Direction.DOWN;
            state = ElevatorState.MOVING;
        }
    }

    /** One simulation tick: advance one floor in the current direction. */
    public void step() {
        if (direction == Direction.UP) {
            currentFloor++;
            if (upStops.remove(currentFloor)) openDoors();
            if (upStops.isEmpty())
                direction = downStops.isEmpty() ? Direction.IDLE : Direction.DOWN;
        } else if (direction == Direction.DOWN) {
            currentFloor--;
            if (downStops.remove(currentFloor)) openDoors();
            if (downStops.isEmpty())
                direction = upStops.isEmpty() ? Direction.IDLE : Direction.UP;
        }
        if (direction == Direction.IDLE) state = ElevatorState.IDLE;
    }

    private void openDoors() {
        state = ElevatorState.DOORS_OPEN;
        System.out.printf("Car %d: doors open at floor %d%n", id, currentFloor);
        // in a real system: timer -> close -> resume MOVING
    }

    public boolean isIdle() {
        return direction == Direction.IDLE && upStops.isEmpty() && downStops.isEmpty();
    }

    public int getId()            { return id; }
    public int getCurrentFloor()  { return currentFloor; }
    public Direction getDirection(){ return direction; }
}

// ---------- Scheduling (Strategy pattern) ----------
import java.util.List;

public interface SchedulingStrategy {
    Elevator selectElevator(List<Elevator> elevators, HallRequest request);
}

public class NearestElevatorStrategy implements SchedulingStrategy {
    private static final int PENALTY = 1000; // de-prioritise cars heading the wrong way

    @Override
    public Elevator selectElevator(List<Elevator> elevators, HallRequest req) {
        Elevator best = null;
        int bestCost = Integer.MAX_VALUE;
        for (Elevator e : elevators) {
            int cost = estimateCost(e, req);
            if (cost < bestCost) { bestCost = cost; best = e; }
        }
        return best;
    }

    private int estimateCost(Elevator e, HallRequest req) {
        int distance = Math.abs(e.getCurrentFloor() - req.getFloor());
        if (e.isIdle()) return distance;

        boolean sameDir = e.getDirection() == req.getDirection();
        boolean onTheWay = sameDir && (
            (req.getDirection() == Direction.UP   && e.getCurrentFloor() <= req.getFloor()) ||
            (req.getDirection() == Direction.DOWN && e.getCurrentFloor() >= req.getFloor()));
        return onTheWay ? distance : distance + PENALTY;
    }
}

// ---------- Controller (Singleton + orchestration) ----------
import java.util.ArrayList;
import java.util.List;

public class ElevatorController {
    private static ElevatorController instance;
    private final List<Elevator> elevators = new ArrayList<>();
    private SchedulingStrategy strategy;

    private ElevatorController(int numElevators) {
        for (int i = 0; i < numElevators; i++) elevators.add(new Elevator(i));
        this.strategy = new NearestElevatorStrategy();
    }

    public static synchronized ElevatorController getInstance(int numElevators) {
        if (instance == null) instance = new ElevatorController(numElevators);
        return instance;
    }

    public void setStrategy(SchedulingStrategy s) { this.strategy = s; }

    /** External hall call: pick a car, then add the stop. */
    public void requestElevator(int floor, Direction dir) {
        Elevator chosen = strategy.selectElevator(elevators, new HallRequest(floor, dir));
        System.out.printf("Hall call floor %d (%s) -> Car %d%n", floor, dir, chosen.getId());
        chosen.addStop(floor);
    }

    /** Internal car call: target a specific car. */
    public void selectFloor(int carId, int targetFloor) {
        elevators.get(carId).addStop(targetFloor);
    }

    /** Advance the whole system one tick. */
    public void step() {
        for (Elevator e : elevators) if (!e.isIdle()) e.step();
    }
}

// ---------- Demo ----------
public class Main {
    public static void main(String[] args) {
        ElevatorController controller = ElevatorController.getInstance(2);

        controller.requestElevator(5, Direction.UP);   // someone on 5 wants up
        controller.requestElevator(2, Direction.UP);   // someone on 2 wants up
        controller.selectFloor(0, 8);                  // rider in car 0 picks floor 8

        for (int tick = 0; tick < 10; tick++) {
            System.out.println("-- tick " + tick + " --");
            controller.step();
        }
    }
}
```

### How a request flows
1. **Hall call** → `ElevatorController.requestElevator(floor, dir)` → `SchedulingStrategy` scores every car and returns the cheapest → that car's `addStop(floor)`.
2. **Car call** → `ElevatorController.selectFloor(carId, target)` → goes straight to that specific car's `addStop`.
3. `addStop` classifies the floor into `upStops` or `downStops` relative to the car's current position and kicks an idle car into motion.
4. Each `step()` advances every active car one floor; arriving at a stop pops it and opens the doors. When a direction's set empties, the car flips direction (LOOK) or goes idle.

**Complexity:** `addStop` is `O(log k)` (TreeSet), dispatch is `O(N)` over cars per hall call, `step` is `O(N)`.

---

## 5. Design Patterns

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** *(primary)* | `SchedulingStrategy` + `NearestElevatorStrategy` | Dispatch policy is the part most likely to change/optimize. Swap in FCFS, least-busy, or wait-time-optimal without touching `Elevator` or `ElevatorController`. |
| **State** *(primary)* | `ElevatorState` | Behavior depends on lifecycle (IDLE/MOVING/DOORS_OPEN/MAINTENANCE). Modeled as an enum here for brevity; promote to a full State pattern (`ElevatorState` interface with `handle()`) if the interviewer pushes on transition logic. |
| **Singleton** | `ElevatorController` | One coordinator owns all cars and is the single entry point for requests. |
| **Command** | `Request` / `HallRequest` / `CarRequest` | Requests are first-class objects — easy to queue, log, prioritize, or replay. |
| **Observer** *(extension)* | `Elevator` → `Display` / `Floor` | Cars notify floor indicators and panels of position/direction changes without coupling. |
| **Factory** *(optional)* | request creation | Centralize building `HallRequest` vs `CarRequest` if construction grows. |

**Talking points if pushed:**
- *Why split `upStops`/`downStops`?* It makes LOOK trivial — serve the current direction's set in sorted order, then flip. One combined set forces re-sorting logic on every direction change.
- *Concurrency extension:* guard each car's stop-sets (or use concurrent structures) and run dispatch under a lock or an actor/event-loop per car.
- *Scaling dispatch:* the Strategy seam is exactly where a smarter algorithm (estimated-time-of-arrival, reinforcement-learned policy) plugs in later.
