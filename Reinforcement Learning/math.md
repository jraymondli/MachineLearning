Below is the **same RL example (Q-learning)** explained **purely with math**, step by step, using standard reinforcement-learning notation.

---

## 1. Formal Setup: Markov Decision Process (MDP)

Reinforcement learning is usually modeled as a **Markov Decision Process (MDP)**:

[
\mathcal{M} = (\mathcal{S}, \mathcal{A}, P, R, \gamma)
]

Where:

* (\mathcal{S}): set of states
  [
  \mathcal{S} = {(x,y)\mid x,y \in {0,1,2,3}}
  ]

* (\mathcal{A}): set of actions
  [
  \mathcal{A} = {\text{Up}, \text{Down}, \text{Left}, \text{Right}}
  ]

* (P(s' \mid s,a)): transition probability
  Deterministic in this example:
  [
  P(s' \mid s,a) =
  \begin{cases}
  1 & \text{if } s'=\text{move}(s,a) \
  0 & \text{otherwise}
  \end{cases}
  ]

* (R(s,a)): reward function
  [
  R(s,a)=
  \begin{cases}
  +10 & \text{if } s'=\text{goal} \
  -1 & \text{otherwise}
  \end{cases}
  ]

* (\gamma \in (0,1)): discount factor
  (e.g. (\gamma=0.95))

---

## 2. Objective

We want to learn a **policy** (\pi(a\mid s)) that maximizes the **expected return**:

[
G_t = \sum_{k=0}^{\infty} \gamma^k R_{t+k+1}
]

Formally:

[
\pi^* = \arg\max_\pi \mathbb{E}_\pi \left[ G_0 \right]
]

---

## 3. Value Functions

### State-Value Function

[
V^\pi(s) = \mathbb{E}_\pi \left[ G_t \mid S_t=s \right]
]

### Action-Value Function (Q-function)

[
Q^\pi(s,a) = \mathbb{E}_\pi \left[ G_t \mid S_t=s, A_t=a \right]
]

Q-learning learns **(Q^*(s,a))** directly.

---

## 4. Bellman Optimality Equation

The optimal Q-function satisfies:

[
Q^*(s,a)
========

R(s,a)
+
\gamma \sum_{s'} P(s' \mid s,a)
\max_{a'} Q^*(s',a')
]

In our deterministic grid:

[
Q^*(s,a)
========

r
+
\gamma
\max_{a'} Q^*(s',a')
]

---

## 5. Q-Learning Update Rule

Since (Q^*) is unknown, we use **stochastic approximation**:

[
\boxed{
Q_{t+1}(s,a)
============

Q_t(s,a)
+
\alpha
\left[
r
+
\gamma \max_{a'} Q_t(s',a')
---------------------------

Q_t(s,a)
\right]
}
]

Where:

* (\alpha): learning rate
* (r): observed reward
* (s'): next state

This is exactly what the Python code does.

---

## 6. Temporal Difference (TD) Error

Define the **TD error**:

[
\delta_t
========

r
+
\gamma \max_{a'} Q_t(s',a')
---------------------------

Q_t(s,a)
]

Then:

[
Q_{t+1}(s,a) = Q_t(s,a) + \alpha \delta_t
]

Interpretation:

* (\delta_t > 0): action was better than expected
* (\delta_t < 0): action was worse than expected

---

## 7. Exploration: ε-Greedy Policy

During learning, actions are sampled from:

[
\pi(a \mid s) =
\begin{cases}
\frac{\varepsilon}{|\mathcal{A}|} & \text{random action} \
1-\varepsilon & \text{greedy action}
\end{cases}
]

This ensures:

* **Exploration**: visiting all state–action pairs
* **Convergence** (under standard conditions)

---

## 8. Convergence Guarantee (Intuition)

Q-learning converges to (Q^*) if:

1. All ((s,a)) are visited infinitely often
2. (\sum_t \alpha_t = \infty), (\sum_t \alpha_t^2 < \infty)
3. MDP is stationary

(Proof via stochastic approximation theory.)

---

## 9. Extracting the Policy

After learning:

[
\pi^*(s) = \arg\max_a Q^*(s,a)
]

This greedy policy produces the shortest path to the goal.

---

## 10. One-Line Intuition (Math → Code)

[
\text{New estimate}
===================

\text{Old estimate}
+
\text{learning rate}
\times
(\text{better future estimate} - \text{current guess})
]

---

If you want next, I can:

* Derive Q-learning from **dynamic programming**
* Show **policy gradients mathematically**
* Explain **why deep RL is unstable**
* Walk through a **numerical update by hand**

Just tell me.
