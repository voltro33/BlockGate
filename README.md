# Port AGV — multiagent path planning

A one-file classroom demo for a reinforcement-learning course. Three automated guided vehicles (AGVs) share a small container yard. Each one must pick up its own container and drop it on the berth. Open `index.html` in a browser. That is the only step. The page has no libraries, no fonts to download, and no server.

The live counts will not match the table at the top. That table is the original Python project's average over 30 held-out episodes. One browser episode is a single layout. The habits should still line up: the PSO planner is fast and clean, Q-learning usually delivers but can jam in a gap, the greedy rule never crashes but often deadlocks, double Q-learning is slower and finishes less often, and MADDPG and random rarely finish.

## The yard

The map is 12 rows by 14 columns. Row 0, along the top, is the berth (the drop-off). Row numbers increase downward. Column numbers increase to the right.

Three AGVs always start at `(11, 2)`, `(11, 6)`, and `(11, 11)`. Each episode, each AGV is assigned one container, drawn without replacement from this pool:

`(8,1)`, `(9,3)`, `(7,6)`, `(8,8)`, `(9,12)`, `(6,11)`, `(10,1)`, `(7,12)`, `(10,10)`, `(6,2)`.

The opening layout is a fixed draw, so you can restart and compare methods on the same containers. **New layout** shuffles a fresh assignment.

Racks are impassable:

- Column 4, rows 2–9, except row 5 (that cell is a gap).
- Column 9, rows 2–9, except rows 2 and 8 (those two cells are gaps).
- The cells `(10,6)` and `(10,8)`.

The gaps are one cell wide. Two vehicles that both want the same gap are the usual source of trouble.

## How a step works

Each AGV chooses one action: 0 stay, 1 up, 2 down, 3 left, 4 right. All three move at the same time. The rules are applied in this order:

1. **Blocked, not a collision.** A move off the map or into a rack does not happen. The AGV stays where it was. The collision counter does not change.
2. **Same cell.** If two or more AGVs are aiming at the same cell, every one of them stays and is flagged as collided this step.
3. **Swap.** If two AGVs would trade cells, both stay and both are flagged.
4. **Into a vehicle that did not leave.** If an AGV moves into a cell that is still occupied, because the occupant stayed or was forced to stay by a rule above, the mover stays and is flagged. This is repeated so a vehicle that was just forced to stay can block the one behind it.

An AGV that ends the step on its own container picks it up automatically. A carrying AGV that ends the step on any cell in row 0 delivers, parks there, and stops choosing actions. The episode ends when all three containers are delivered, or after 240 steps.

## Reward

The page shows the same scheme the learners were trained with. Each AGV that has not yet delivered receives:

- −0.05 every step.
- +0.05 for each cell closer to its current target, and −0.05 for each cell farther away.
- −2 if it is flagged in a collision.
- +1 when it picks up its container.
- +10 when it delivers. Each teammate also receives +2 on that step.

Before pickup, the target is that AGV's container and distance is ordinary Manhattan distance. After pickup, the target is the berth line, not one particular berth cell, so the distance is just the row number. Moving sideways along a row does not change it. Moving up does.

The team reward on the page is the sum of the three vehicles' rewards.

## The six methods

All six actually choose the actions. None of them is a recorded path.

### 1. Random

Every AGV that is still working picks one of the five actions with equal probability, including stay. It does not know where its container is. A delivery is luck. The episode almost always runs to the step limit. Paths are long because the vehicles keep fidgeting.

### 2. Greedy rule baseline

Each AGV computes a shortest path (A*) around the racks: to its container, or, once it is carrying, to any berth cell. Delivered vehicles are treated as obstacles, because they will never move again.

The vehicles then commit in a fixed order: AGV 1, then AGV 2, then AGV 3. A vehicle takes the next step of its shortest path only if that cell is free of the others and has not already been claimed this step. Otherwise it waits. It will not step into a cell another vehicle is standing on.

It never collides. It also will not take a long way around when the next cell is occupied, so a head-on meeting in a one-cell gap becomes a permanent standoff. Both vehicles wait until step 240. On many layouts the paths do not cross and all three deliver. On many others they do not. It should not finish every layout.

### 3. PSO planner

This is a planner, not a learned policy. It runs once per layout (and again if you switch to it in the middle of an episode, from wherever the vehicles are now).

A particle holds three scores, one per vehicle. Sorting the scores gives a priority order. There are only six orders, and the swarm is seeded with all of them, then nudged toward orders that scored well. Each order is decoded as follows. The first vehicle plans a path in space and time with A*: every cell it will occupy at each future step is reserved, and so is each move, so nobody can swap through it. It goes to its container, then to the nearest berth cell it can park on without using a reserved cell. The next vehicle plans around those reservations, and so on. A vehicle that is already carrying only plans the berth leg. A vehicle that has already delivered keeps its cell for the whole horizon.

The score of an order is

`makespan + 0.05 × total length + 500 × (number of vehicles that could not be scheduled)`.

Makespan is the step on which the last container is delivered. Length counts real moves, not the waiting after a vehicle has parked. The best timetable is then played back action by action.

What to watch: all three containers reach the berth in a short episode, and the collision count stays at 0. If you change method and come back, it builds a new timetable from the current cells rather than replaying the old one.

### 4. Q-learning

Each AGV has its own table of action values. The state is:

`(own row, own column, carrying 0/1, target row, target column, 4-bit occupancy)`.

The four bits say whether another AGV is in the cell above, below, left, or right. They do not see a vehicle two cells away, and they do not see anyone's goal. While a vehicle is carrying, the target stored in the state is the berth line, written as `(0, 0)`, not a chosen column. The reward above is what the tables were trained on: 8,000 episodes, learning rate 0.1, discount 0.96, optimistic initial values, exploration annealed from fully random down to 5%.

In class the page plays the greedy policy: the action with the highest value. Unvisited states, which are rare, fall back to a shortest-path step that avoids an occupied neighbour.

What to watch: on most layouts the vehicles pick up and deliver, and you may see no collision at all. On some layouts two of them commit to the same gap cell on the same step. Both stay, the state does not change, and they try the same action again. The collision counter then climbs until the step limit. That is the narrow-lane failure. The opening layout is one of those jams, so you can see it without hunting. **New layout** usually gives Q-learning a layout it can finish. It does not finish every layout.

### 5. Double Q-learning

Same state and the same reward. Each AGV keeps two tables. One table chooses the best action and the other supplies the value of that action, which stops a single lucky sample from making a risky gap look wonderful. The demo plays the action with the highest sum of the two tables.

Training was shorter and more conservative than Q-learning: 2,500 episodes, learning rate 0.05, discount 0.9, values started at zero, exploration annealed to 10%.

What to watch: more waiting, fewer collision spirals than Q-learning, and more episodes that expire with a container still in the yard. It is slower and less reliable. It should not finish every layout, and it should collide less than Q-learning on average.

### 6. MADDPG

Each AGV has its own small network: 12 local features (position, carrying flag, target, the four occupancy bits, and the direction and distance to the target), one hidden layer of 16 units, and 5 action scores. During training a centralized critic for each AGV saw all three local views and all three actions. Only the actors are stored in the page.

Training was 1,000 episodes. The critic is regressed onto the reward (scaled by a constant so the optimizer can see it; the ratios are the ones above). Each actor ascends the critic's expected value of its own action while the other two vehicles are held at the joint action from the replay buffer. Exploration mixed random moves, the actor's own samples, and some shortest-path moves, and the actor had a light extra pull toward that shortest-path action so the container reward was not invisible. A small entropy bonus kept the distribution from collapsing onto a single move.

That is not enough to coordinate three deliveries. On a check of 80 episodes the sampled actors finished all three containers about 14% of the time, picked up roughly one or two containers per episode, and usually ran long. The page draws each move from the network's distribution, so the same layout will not play the same way twice. This is a real trained network, not a scripted route, and it is not a strong policy.

## Published results

These are the original Python project's measurements, also shown in the page header. Success means all three containers were delivered. Steps, collisions, delivered containers, and path length are averages per episode. Path length counts cells a vehicle actually moved into, not steps where it stayed, was blocked, or collided and stayed.

| Method | Success | Avg steps | Avg collisions | Avg delivered | Avg path |
| --- | --- | --- | --- | --- | --- |
| PSO planner | 1.00 | 19.8 | 0.0 | 3.00 | 16.7 |
| Q-learning | 0.80 | 68.2 | 78.2 | 2.60 | 23.8 |
| Greedy baseline | 0.43 | 144.2 | 0.0 | 1.80 | 10.5 |
| Double Q-learning | 0.27 | 182.0 | 15.6 | 1.67 | 33.6 |
| MADDPG | 0.00 | 240.0 | 16.3 | 0.27 | 115.8 |
| Random | 0.00 | 240.0 | 6.3 | 0.37 | 155.0 |

Q-learning's collision average is high because the episodes it fails are usually collision spirals in a gap, while the episodes it solves are often clean. The average is not "a crash every step on a typical success".

## How to run

Open `index.html` in Chrome, Firefox, Safari, or Edge. Double-click the file, or use File → Open. No install, no network, no build step.

## How the controls work

- **Method buttons (keys 1–6).** Random, greedy, PSO, Q-learning, double Q-learning, MADDPG. Changing method during an episode keeps the containers and the current positions, pickups, deliveries, collision count, and path lengths. The new method chooses the next action from that state. PSO builds a fresh timetable when you select it.
- **Play / Pause (Space).** Play runs until you pause, the yard is clear, or step 240. Pause freezes the vehicles where they are.
- **Step (Right arrow).** Pauses, if needed, and advances exactly one step.
- **Speed.** From 1 to 20 steps per second. The default is 6, which is slow enough to follow a gap. Turn it up to skip through a timeout.
- **Restart (R).** Same containers, vehicles back at their starts, counters cleared. Use this to compare two methods on one layout.
- **New layout (N).** Shuffles three new containers and starts over. If playback was running, the new episode keeps running.

The status line reads Ready, Playing, Paused, All delivered, or Timed out. The event log records pickups, deliveries, and collisions. A move into a rack is not logged, and it is not a collision. Path length under each AGV is how many cells that vehicle has actually entered. The highlighted row in the results table is the method you are watching.
