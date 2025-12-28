Here’s a **single, self-contained RL example in Python** (classic **Q-learning** on a tiny “gridworld”). No extra libraries needed.

```python
import random
from collections import defaultdict

# ---- Simple 4x4 Gridworld ----
# Start at (0,0). Goal at (3,3) gives +10 and ends.
# Each step costs -1. Agent learns with Q-learning.

W, H = 4, 4
START = (0, 0)
GOAL = (3, 3)
ACTIONS = ["U", "D", "L", "R"]

def step(state, action):
    x, y = state
    if action == "U": y = max(0, y - 1)
    if action == "D": y = min(H - 1, y + 1)
    if action == "L": x = max(0, x - 1)
    if action == "R": x = min(W - 1, x + 1)
    next_state = (x, y)

    if next_state == GOAL:
        return next_state, 10, True
    return next_state, -1, False

def eps_greedy(Q, state, eps):
    if random.random() < eps:
        return random.choice(ACTIONS)
    # Choose action with max Q; break ties randomly
    qs = [(Q[(state, a)], a) for a in ACTIONS]
    max_q = max(qs, key=lambda t: t[0])[0]
    best = [a for (q, a) in qs if q == max_q]
    return random.choice(best)

# ---- Q-learning ----
Q = defaultdict(float)

alpha = 0.2   # learning rate
gamma = 0.95  # discount factor
eps = 0.3     # exploration rate

episodes = 2000
max_steps = 50

for ep in range(episodes):
    s = START
    for _ in range(max_steps):
        a = eps_greedy(Q, s, eps)
        s2, r, done = step(s, a)

        # Q-learning update:
        # Q(s,a) <- Q(s,a) + alpha * (r + gamma*max_a' Q(s',a') - Q(s,a))
        best_next = max(Q[(s2, a2)] for a2 in ACTIONS)
        Q[(s, a)] += alpha * (r + gamma * best_next - Q[(s, a)])

        s = s2
        if done:
            break

# ---- Show the learned greedy path ----
def greedy_policy(Q, state):
    qs = [(Q[(state, a)], a) for a in ACTIONS]
    max_q = max(qs, key=lambda t: t[0])[0]
    best = [a for (q, a) in qs if q == max_q]
    return random.choice(best)

s = START
path = [s]
for _ in range(30):
    if s == GOAL:
        break
    a = greedy_policy(Q, s)
    s, _, _ = step(s, a)
    path.append(s)

print("Learned path:", path)
print("Steps:", len(path) - 1)
```

What you should see after training: a path from `(0,0)` to `(3,3)` in about **6 steps** (the shortest path in a 4x4 grid).

If you want, I can modify this one example to include:

* obstacles / walls,
* a decaying epsilon schedule,
* plotting the policy as arrows in the grid.
