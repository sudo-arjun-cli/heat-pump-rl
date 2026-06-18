# 🔬 Reinforcement Learning Heat Pump Controller: Development History (v4–v16)

This document provides a comprehensive history of the Reinforcement Learning (RL) Heat Pump Controller project, tracing the evolution from the initial takeover at **v4** up to the current **v16** state-of-the-art physics-constrained Contextual RL controller.

---

## 📅 Chronological Version Evolution

### **Version 3 (Baseline / Inherited)**
*   **Goal**: Initial setup for RL control on single-family houses (SFH).
*   **Key Issues**:
    *   The reward function used a scaling factor of `1/1000` which effectively flattened the reward gradients.
    *   The comfort reward ($+0.01$) was dwarfed by the electricity cost ($\approx -0.50$ €), causing the agent to learn a "never heat" policy (shutting off the heat pump to minimize costs, resulting in room temperatures dropping to $6.8^\circ\text{C}$).
    *   Code contained import references to a deleted list of heat pumps.

---

### **Version 4 (Reward Scale & Import Fix)**
*   **What Changed from v3**:
    *   **Reward Scale Fix**: Removed the `1/1000` scaling factor to restore raw signal gradients.
    *   **Asymmetric Underheating Penalty**: Replaced the symmetric quadratic penalty with a harsh underheating penalty ($-5.0 \times (T_{\text{lower}} - T_{\text{room}})^2$) and a mild overheating penalty. Removed the flat $+1.0$ comfort reward.
    *   **Import Fix**: Corrected the heat pump module import in `room_env.py` to use `iDM_AERO_ALM_4_12` directly, bypassing the deleted `HEATPUMP_MODELS` list.
*   **Results**:
    *   Comfort appeared to jump to **96.7%**, but this metric was misleading because `evaluate.py` only checked underheating ($T_{\text{room}} < 20^\circ\text{C}$).
    *   The agent exhibited **massive overheating** (room temperature peaked at $29.7^\circ\text{C}$) because the comfort band upper limit was set to $26.0^\circ\text{C}$ and the overheating penalty was too weak ($-1.0$). The agent chose to blast heat to ensure it never dipped below $20^\circ\text{C}$.

---

### **Version 5 (Comfort Band Correction & Bidirectional Metrics)**
*   **What Changed from v4**:
    *   **Comfort Band Upper Limit**: Corrected $T_{\text{room\_set\_upper}}$ from $26.0^\circ\text{C}$ to $22.0^\circ\text{C}$ in `room_env.py`.
    *   **Overheating Penalty Scaling**: Increased the overheating penalty coefficient from $-1.0$ to $-3.0$ (resulting in a 5:3 asymmetric penalty ratio).
    *   **Bidirectional Comfort Shading**: Refactored `evaluate.py` to count both underheating and overheating violations, shading overheating events orange on the evaluation plots.
*   **Results**:
    *   Comfort dropped to an honest **43.1%** due to the bidirectional evaluation and unavoidable summer overheating (in a heating-only system).
    *   Energy usage dropped from $7,382$ kWh to $5,274$ kWh ($-28.6\%$).
    *   Highest room temperature was successfully reduced from $29.7^\circ\text{C}$ to $27.2^\circ\text{C}$.

---

### **Version 6 (Observation Scaling & Decay Schedules)**
*   **What Changed from v5**:
    *   **Observation Space Additions**: Expanded the observation space from 53 to 58 dimensions by adding temporal contexts (`sin(hour)`, `cos(hour)`, `sin(day_of_week)`, `cos(day_of_week)`) and the normalized `previous_action`.
    *   **Stable-Baselines3 Upgrades**: Switched from `DummyVecEnv` to `SubprocVecEnv` with 4 parallel environments. Wrapped the environment with `VecNormalize` to scale observations to zero-mean and unit-variance.
    *   **Gamma & Learning Rate Tuning**: Increased `gamma` to `0.995` to extend the optimization horizon over a 24-hour cycle. Added a linear learning rate decay schedule ($3\times 10^{-4}$ down to $1\times 10^{-5}$).
    *   **Custom Logging Callback**: Added `TensorboardLoggingCallback` to stream episode costs, energy, comfort %, and cycle counts.
*   **Results**:
    *   The linear decay schedule successfully converged the policy, preventing control oscillations.
    *   Comfort improved significantly to **68.2%**.
    *   Total annual cost dropped to **€1,472.07** (additional $7.5\%$ savings vs. v5). Peak overheating dropped to $25.4^\circ\text{C}$.

---

### **Version 7 (Domain Randomization)**
*   **What Changed from v6**:
    *   **Multi-Building Training**: Enabled domain randomization during training across all 11 buildings in `vonovia_model.py`.
    *   **One-Hot Context**: Added an 11-dimension one-hot encoded building ID vector to the observation space (growing to 69 dimensions) so the policy could distinguish between buildings.
*   **Results**:
    *   **Catastrophic Leaky House Freezing**: Leaky, uninsulated buildings (like Buildings 1, 3, and 4) dropped to temperatures as low as **$6.8^\circ\text{C}$**.
    *   The agent calculated that paying the quadratic comfort penalty was cheaper than paying the massive electric bills required to heat uninsulated buildings. Well-insulated houses, however, performed excellently.

---

### **Version 8 (Environment Sanitization)**
*   **What Changed from v7**:
    *   **Sanitization of Training Pool**: Excluded buildings with a peak heat loss $> 12$ kW at $-10^\circ\text{C}$ ambient (transmission + ventilation losses $H_{\text{ve}} + H_{\text{tr}} > 400$ W/K) from the training set, leaving 6 physically feasible SFH buildings.
*   **Results**:
    *   Catastrophic freezing vanished. Across all 6 feasible buildings, the absolute minimum temperature observed was **$18.5^\circ\text{C}$**.
    *   Comfort scores stabilized between $77.4\%$ and $91.3\%$. The slight underheating was due to the agent "riding the edge" of the 20°C boundary to shave costs.

---

### **Version 9 (Aggressive Asymmetric Penalty)**
*   **What Changed from v8**:
    *   **Stricter Underheating Penalty**: Increased the underheating penalty coefficient from $-5.0$ to $-20.0$ (making underheating 4x more painful).
*   **Results**:
    *   Successfully pushed minimum room temperatures up to $18.9^\circ\text{C} - 19.5^\circ\text{C}$, raising comfort scores to between $87\%$ and $98\%$.
    *   **New Issue: Rapid Cycling**: The agent avoided pre-heating because it was not directly rewarded. Instead, it toggled the compressor on and off rapidly at the $20^\circ\text{C}$ boundary (e.g., $442$ cycles over 90 days).

---

### **Version 10 (Pre-Heating Bonus Loophole)**
*   **What Changed from v9**:
    *   **Pre-heating Reward**: Introduced a flat $+1.0$ comfort reward for maintaining $T_{\text{room}} \ge 20.5^\circ\text{C}$ to incentivize thermal charging and mitigate rapid cycling.
*   **Results**:
    *   **Reward Hacking Loophole**: The agent accumulated massive positive rewards during cheap hours, allowing it to become lazy. It was willing to accept later freezing penalties because its overall episode return remained positive. Room temperatures plummeted to $18.0^\circ\text{C}$ on Building 1.
    *   **Reversion**: The pre-heating bonus was rejected, and the penalty-only structure of **v9** was restored.

---

### **Version 11 (Building-Specific Radiator Capacities)**
*   **What Changed from v10**:
    *   **Radiator Coupling Physics Fix**: Dynamically calculated building-specific radiator coupling coefficients ($H_{\text{rad\_con}}$) based on peak heat loss with a 20% oversizing factor, rather than forcing all buildings to share a single hardcoded radiator capacity.
*   **Results**:
    *   **Vanishing Cycling**: Proper physical coupling allowed the heat pump to transfer heat smoothly. Rapid cycling vanished (Building 2 cycles plummeted from $442$ to $96$).
    *   Comfort scores rose above **$93.7\%$** for most buildings, with minimum temperatures tightening to the boundary.

---

### **Version 13 (Random Initialization Range & 5M Steps)**
*   **What Changed from v11**:
    *   **Random Initialization Range**: Narrowed the random initial temperature ranges on environment reset ($T_{\text{room}} \in [18.0, 24.0]^\circ\text{C}$) to prevent the agent from wasting learning cycles in physically impossible start states.
    *   **5M Steps Run**: Extended training to 5 million steps to reach full policy convergence.
*   **Results**:
    *   **Loophole Re-emergence**: With complete convergence, the agent discovered a mathematical loophole. Because the comfort penalty was purely quadratic, dropping to $19.5^\circ\text{C}$ yielded a tiny penalty. The converged agent chose to ride the $18.5^\circ\text{C}$ boundary to aggressively trade comfort for euros.

---

### **Version 14 (Linear + Quadratic Comfort Penalty)**
*   **What Changed from v13**:
    *   **Linear Comfort Term**: Closed the edge-riding loophole by adding a linear term to the underheating penalty: $-20 \times |T_{\text{lower}} - T_{\text{room}}| - 20 \times (T_{\text{lower}} - T_{\text{room}})^2$. Even a $0.1^\circ\text{C}$ drop was now severely punished.
*   **Results**:
    *   Loophole closed. Minimum temperatures rose back to $19.3^\circ\text{C} - 19.7^\circ\text{C}$, comfort returned to $\ge 95\%$, and compressor starts stabilized at a healthy 2–3 per day.

---

### **Version 15 (Contextual RL & Cascaded Pumps)**
*   **What Changed from v14**:
    *   **Contextual RL Transition**: Replaced the rigid building one-hot IDs with **5 physical context parameters** in the observation space:
        1. $H_{\text{tr}}$: Envelope heat loss coefficient (normalized by $2000.0$)
        2. $H_{\text{ve}}$: Ventilation heat loss coefficient (normalized by $1000.0$)
        3. $c_{\text{bldg}}$: Building thermal mass (normalized by $100.0$)
        4. $\text{area\_floor}$: Floor area (normalized by $600.0$)
        5. $\text{num\_pumps}$: Active pump count (normalized by $5.0$)
    *   **Multi-Family House (MFH) Integration**: Added 4 unrenovated and renovated MFH building models. Fixed a unit bug where MFH specific heat losses ($W/(\text{m}^2K)$) were fed directly into the simulator (now scaled by floor area to yield absolute $W/K$).
    *   **Cascaded Pump Architecture**: Created a cascaded heat pump model where max electrical and thermal capacities scale with the building's required pumps (calculated dynamically in `models/__init__.py`).
    *   **Physics Capping**: Capped the agent's requested supply temperature setpoint to the physical thermodynamic limits of the heat pump cascade in `src/simulator.py` to prevent the ODE from simulating infinite heat transfer.
*   **Results**:
    *   The environment sanitization filter was removed, as the cascaded pumps could heat any leaky or massive building.
    *   The agent learned generalizable physics rather than memorizing one-hot IDs, enabling zero-shot generalization.

---

### **Version 16 (SAC Network Scaling)**
*   **What Changed from v15**:
    *   **Policy Network Scaling**: Scaled up the SAC MLP policy network architecture from default sizes to **`[512, 512, 512]`** to handle the high-dimensional continuous physical inputs.
    *   **Training Run**: Model was trained over 3 million steps.
*   **Results**:
    *   This represents the current state-of-the-art policy, yielding robust winter heat control across both SFH and MFH classes.

---

## 📊 Summary Tables

### **Table 1: Environment & Architecture Progression**

| Version | Obs Dimensions | Action Bounds | Building Classes | Heat Pump Architecture | Train Vector Env |
|:---:|:---:|:---:|:---:|:---:|:---:|
| **v3** | 53 | $[-1, 1] \to [20, 65]^\circ\text{C}$ | 11 SFH Buildings | Single iDM AERO (Fixed) | `DummyVecEnv` |
| **v4** | 53 | $[-1, 1] \to [20, 65]^\circ\text{C}$ | 11 SFH Buildings | Single iDM AERO (Fixed) | `DummyVecEnv` |
| **v5** | 53 | $[-1, 1] \to [20, 65]^\circ\text{C}$ | 11 SFH Buildings | Single iDM AERO (Fixed) | `DummyVecEnv` |
| **v6** | 58 (Temporal + Prev Act) | $[-1, 1] \to [20, 65]^\circ\text{C}$ | 11 SFH Buildings | Single iDM AERO (Fixed) | `SubprocVecEnv` (4 workers) + `VecNormalize` |
| **v7** | 69 (One-Hot Building ID) | $[-1, 1] \to [20, 65]^\circ\text{C}$ | 11 SFH Buildings | Single iDM AERO (Fixed) | `SubprocVecEnv` (4 workers) + `VecNormalize` |
| **v8-v14** | 69 (One-Hot Building ID) | $[-1, 1] \to [20, 65]^\circ\text{C}$ | 6 Sanitized SFHs | Single iDM AERO (Fixed) | `SubprocVecEnv` (4 workers) + `VecNormalize` |
| **v15-v16** | 63 (5 Physics Contexts) | $[-1, 1] \to [20, 65]^\circ\text{C}$ | All SFH + MFH Buildings | Cascaded iDM (1-4 Pumps, Dynamic) | `SubprocVecEnv` (4 workers) + `VecNormalize` |

### **Table 2: Reward Function Progression**

| Version | Comfort Band | Underheating Penalty ($T_{\text{room}} < T_{\text{lower}}$) | Overheating Penalty ($T_{\text{room}} > T_{\text{upper}}$) | In-Band Comfort Reward | Cycle / Action Penalty |
|:---:|:---:|:---:|:---:|:---:|:---:|
| **v3** | $20 - 26^\circ\text{C}$ | $-1.0 \times (21.0 - T_{\text{room}})^2$ (scaled by `1/1000`) | $-1.0 \times (T_{\text{room}} - 21.0)^2$ (scaled by `1/1000`) | $+1.0$ (scaled by `1/1000`) | None |
| **v4** | $20 - 26^\circ\text{C}$ | $-5.0 \times (20.0 - T_{\text{room}})^2$ | $-1.0 \times (T_{\text{room}} - 26.0)^2$ | $0.0$ | None |
| **v5-v8** | $20 - 22^\circ\text{C}$ | $-5.0 \times (20.0 - T_{\text{room}})^2$ | $-3.0 \times (T_{\text{room}} - 22.0)^2$ | $0.0$ | None |
| **v9** | $20 - 22^\circ\text{C}$ | $-20.0 \times (20.0 - T_{\text{room}})^2$ | $-3.0 \times (T_{\text{room}} - 22.0)^2$ | $0.0$ | None |
| **v10** | $20 - 22^\circ\text{C}$ | $-20.0 \times (20.0 - T_{\text{room}})^2$ | $-3.0 \times (T_{\text{room}} - 22.0)^2$ | $+1.0$ for $T_{\text{room}} \ge 20.5^\circ\text{C}$ | None |
| **v11-v13** | $20 - 22^\circ\text{C}$ | $-20.0 \times (20.0 - T_{\text{room}})^2$ | $-3.0 \times (T_{\text{room}} - 22.0)^2$ | $0.0$ | None |
| **v14-v16** | $20 - 22^\circ\text{C}$ | $-20(20 - T) - 20(20 - T)^2$ | $-3.0 \times (T_{\text{room}} - 22.0)^2$ | $0.0$ | None |

### **Table 3: Hyperparameter Progression**

| Parameter | v3 | v4 | v5 | v6 | v7-v11 | v13 | v14 | v15-v16 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Timesteps** | 500k | 500k | 500k | 1M | 1M | 5M | 3M | 3M |
| **Gamma ($\gamma$)** | 0.99 | 0.99 | 0.99 | 0.995 | 0.995 | 0.995 | 0.995 | 0.995 |
| **Learning Rate** | 3e-4 | 3e-4 | 3e-4 | 3e-4 $\to$ 1e-5 | 3e-4 $\to$ 1e-5 | 3e-4 $\to$ 1e-5 | 3e-4 $\to$ 1e-5 | 3e-4 $\to$ 1e-5 |
| **Buffer Size** | 100k | 100k | 100k | 500k | 500k | 500k | 500k | 500k |
| **Learning Starts** | 10k | 10k | 10k | 20k | 20k | 20k | 20k | 20k |
| **Net Architecture** | Default | Default | Default | Default | Default | Default | Default | `[512, 512, 512]` |

---

## 🏆 Key Lessons Learned

1.  **Gradient Scaling Matters**: A bad reward scale (`1/1000`) completely disables policy learning. Always verify that your rewards and gradients are not vanishingly small.
2.  **Evaluate Bidirectionally**: Only evaluating one boundary (underheating) can hide massive policy defects on the other boundary (overheating).
3.  **Physical Feasibility Checks**: Forcing an RL agent to train on impossible physics (e.g., trying to heat a leaky building with an undersized heat pump) degrades the policy gradients across the entire domain.
4.  **Reward Hacking (Lazy Agents)**: Flat bonuses (like the pre-heating comfort bonus) introduce mathematical loopholes. Penalties are generally safer for strict constraint enforcement.
5.  **Contextual Generalization over Memorization**: One-hot IDs result in rigid policies. Contextualizing environments with physical variables ($H_{\text{tr}}, H_{\text{ve}}$, etc.) allows neural networks to learn physical relationships, enabling zero-shot transfer to new buildings.
6.  **Physics Caps Prevent Cheating**: If your environment allows actions to bypass real-world constraints, the agent will exploit them. Hard-clip control variables within actual physical limits *before* running ODE integration.
