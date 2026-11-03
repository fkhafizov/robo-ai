# vla.md 2026.11.03

* https://claude.ai/cowork/cse_01NJoSWKzi5Rg6LETNc1bp9J

# Q

i'd like to revisit the latest vla experiments. Let's say I want to learn how to move a pusher so that a puck gets placed in the right spot. give me simple instructions step by step of the learning process.

# A

Here's how a robot learns to push a puck to a target spot, in nine steps. Each step is one thing you do, why it matters, and a small piece of code. Since you like to plan backwards from the end goal, the first step is to define what success looks like.

---

## Step 1: Define "success"

Before any learning, decide what counts as done:

$$\text{success} \iff \|o_t - g\|_2 < \varepsilon \quad \text{for some } t \le T$$

Here $o_t$ is the puck position, $g$ is the target spot, $\varepsilon$ is the tolerance (we used 9 cm) and $T$ is the time limit (110 steps).

> **Analogy:** before learning to throw darts, agree on what counts as a hit.

## Step 2: Decide what the robot sees and what it controls

- **It sees** (the state): $s_t = (x_p, y_p,\; x_o, y_o,\; x_g, y_g)$, the positions of the pusher, the puck and the goal.
- **It controls** (the action): $a_t = (a_x, a_y) \in [-1,1]^2$, a small move of the pusher.

The robot's "brain" is a function $\pi_\theta$ that maps $s_t \mapsto a_t$. Learning means finding good parameters $\theta$.

## Step 3: Get a teacher

Write an expert $\pi_E$ that already knows how to push. Ours follows a simple rule:

1. Go to the point behind the puck, on the line from the goal through the puck.
2. Push toward the goal.

The teacher doesn't have to be perfect. Ours fails 26% of the time on hard starts.

## Step 4: Record demonstrations

Let the teacher play many episodes and write down every moment:

$$\mathcal{D} = \{(s_i,\; \pi_E(s_i))\}_{i=1}^{N}$$

```python
for episode in range(300):
    s = env.reset()
    while not done:
        a = expert(s)
        data.append((s, a))     # "in this situation, the teacher did this"
        s = env.step(a)
```

> **Analogy:** filming a chess master and noting the move they made in every position.

## Step 5: Train the copycat (behavioral cloning)

Train the network to predict the teacher's move:

$$\min_\theta \; \frac{1}{N}\sum_{i=1}^{N} \big\|\pi_\theta(s_i) - \pi_E(s_i)\big\|_2^2$$

This is plain supervised regression, the same thing you'd do for any prediction model.

## Step 6: Test by letting the copycat drive

A low training loss is **not** proof that the robot works. Run full episodes where the network makes every move, and count successes.

- On familiar starts, our copycat scored **100%**.

## Step 7: Change the exam

Test on start positions the copycat never saw, for example a goal *below* the puck instead of above it.

- Our copycat dropped to **34.8%**.

This is **distribution shift**. One small mistake puts the robot in an unfamiliar situation, where it makes a bigger mistake, and the errors snowball. Collecting six times more ordinary demonstrations barely helped (43.2%).

## Step 8: Fix it with DAgger (the instructor in the passenger seat)

Repeat this loop about 5 times:

1. **The student drives.** Run the current network $\pi_k$, mistakes and all.
2. **The teacher labels.** At every state the student reached, record what the expert *would* do: $\big(s, \pi_E(s)\big)$.
3. **Add to the notebook:** $\mathcal{D} \leftarrow \mathcal{D} \cup \{\text{new pairs}\}$.
4. **Retrain** on the whole notebook.

```python
for k in range(5):
    for episode in range(100):
        s = env.reset(wide=True)
        while not done:
            data.append((s, expert(s)))   # ask the teacher...
            s = env.step(policy(s))       # ...but follow the student
    train(policy, data)
```

The key line is the last one inside the loop: the student's action moves the robot, and the teacher's action is only recorded.

- Result: **81.4%** with your seed, and **92%** averaged over 5 seeds.

## Step 9: Check that the result is real

- **Repeat with several random seeds**, since results varied by about ±7 points.
- **Compare against a fair alternative.** We found that the teacher's demos from the *same hard starts* reached 89.5%, close to DAgger's 92.3%.
- **Try stricter tolerances**, such as 3 cm instead of 9 cm.

---

## The whole process in one line

$$\text{define success} \rightarrow \text{teacher} \rightarrow \text{record} \rightarrow \text{copy} \rightarrow \text{test} \rightarrow \text{harder test} \rightarrow \text{learn from own mistakes} \rightarrow \text{verify}$$

**The main lesson:** *which* situations you learn from matters far more than *how many* examples you collect.

For the VLA version, everything stays the same except Step 2: the robot would *see* a camera image instead of six numbers, and *hear* an instruction like "push the red puck to the left target". Would you like me to sketch that step next?
