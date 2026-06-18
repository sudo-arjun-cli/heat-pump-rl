# 🔬 Reinforcement Learning Heat Pump Controller: Development History (v4–v16)

This document provides a comprehensive history of the Reinforcement Learning (RL) Heat Pump Controller project, tracing the evolution from the initial takeover at **v4** up to the current **v16** state-of-the-art physics-constrained Contextual RL controller. All evaluation results are obtained over a winter 2026 test period (90 days, 2160 hours).

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
*   **Evaluation (Winter 2026 - vonovia_model)**:
    *   **T_room range**: $19.3 - 23.9^\circ\text{C}$
    *   **Energy / Cost**: $2,814.0\text{ kWh} \ / \ \text{€}942.42$
    *   **Comfort / HP Cycles**: $65.9\% \ / \ 169\text{ cycles}$
    *   *Note: This policy successfully converged and avoided major underheating, but had some overheating and boundary oscillations.*

---

### **Version 7 (Domain Randomization)**
*   **What Changed from v6**:
    *   **Multi-Building Training**: Enabled domain randomization during training across all 11 buildings in `vonovia_model.py`.
    *   **One-Hot Context**: Added an 11-dimension one-hot encoded building ID vector to the observation space (growing to 69 dimensions) so the policy could distinguish between buildings.
*   **Evaluation (Winter 2026)**:
    *   **vonovia_model**: Comfort $69.5\%$, Cost €915.77, Energy $2,757.6$ kWh, HP cycles 45.
    *   **sfh_1919_1948_0_soc** (Leaky House): Comfort **$1.3\%$**, Cost **€6,725.48**, Energy $20,218.3$ kWh. Room temperature peaked at **$34.7^\circ\text{C}$**.
    *   **sfh_2016_now_0_soc** (Insulated): Comfort $49.1\%$, Cost €857.63, Energy $2,592.8$ kWh.
    *   *Note: High-loss buildings (like sfh_1919) suffered from severe underheating and wild overheating because the heat pump (max 12 kW) was physically undersized for their peak demand (up to 33 kW), polluting training gradients.*

---

### **Version 8 (Environment Sanitization)**
*   **What Changed from v7**:
    *   **Sanitization of Training Pool**: Excluded buildings with a peak heat loss $> 12$ kW at $-10^\circ\text{C}$ ambient (transmission + ventilation losses $H_{\text{ve}} + H_{\text{tr}} > 400$ W/K) from the training set, leaving 6 physically feasible SFH buildings.
*   **Evaluation (Winter 2026 - 6 Feasible Buildings)**:
    *   **vonovia_model**: Comfort $81.7\%$, Cost €903.90, Energy $2,711.3$ kWh, T_room $18.4 - 25.3^\circ\text{C}$, HP cycles 78.
    *   **sfh_1995_2001_0_soc**: Comfort $85.6\%$, Cost €994.09, Energy $2,986.1$ kWh, T_room $19.1 - 23.4^\circ\text{C}$.
    *   **sfh_2016_now_0_soc**: Comfort $39.4\%$, Cost €828.81, Energy $2,495.8$ kWh, T_room $17.8 - 23.2^\circ\text{C}$.
    *   *Note: Freezing was eliminated (absolute minimum rose to 17.4°C), but comfort scores were still constrained because the underheating penalty wasn't harsh enough to prevent the agent from "riding the edge".*

---

### **Version 9 (Aggressive Asymmetric Penalty)**
*   **What Changed from v8**:
    *   **Stricter Underheating Penalty**: Increased the underheating penalty coefficient from $-5.0$ to $-20.0$ (making underheating 4x more painful).
*   **Evaluation (Winter 2026 - 6 Feasible Buildings)**:
    *   **vonovia_model**: Comfort $89.9\%$, Cost €896.35, Energy $2,700.8$ kWh, HP cycles 121.
    *   **sfh_2010_2015_0_soc**: Comfort $61.1\%$, Cost €1,048.50, Energy $3,159.6$ kWh, HP cycles 186.
    *   **sfh_2002_2009_0_soc**: Comfort $11.7\%$, Cost €735.82, Energy $2,216.6$ kWh, HP cycles 154.
    *   *Note: Pushed minimum room temperatures up, but comfort fell on well-insulated houses because of strict 20-22°C checks and rapid cycling at the boundaries.*

---

### **Version 10 (Pre-Heating Bonus Loophole)**
*   **What Changed from v9**:
    *   **Pre-heating Reward**: Introduced a flat $+1.0$ comfort reward for maintaining $T_{\text{room}} \ge 20.5^\circ\text{C}$ to incentivize thermal charging and mitigate rapid cycling.
*   **Evaluation (Winter 2026 - 6 Feasible Buildings)**:
    *   **vonovia_model**: Comfort $85.3\%$, Cost €875.95, Energy $2,631.3$ kWh.
    *   **sfh_1995_2001_0_soc**: Comfort $19.8\%$, Cost €878.66, Energy $2,640.7$ kWh, HP cycles 333.
    *   **sfh_2016_now_0_soc**: Comfort $19.9\%$, Cost €801.27, Energy $2,410.5$ kWh, HP cycles 346.
    *   *Note: Reward Hacking Loophole. The agent accumulated massive preheating rewards during cheap hours and tolerated later freezing. Reverted to the penalty-only structure of v9.*

---

### **Version 11 (Building-Specific Radiator Capacities)**
*   **What Changed from v10**:
    *   **Radiator Coupling Physics Fix**: Dynamically calculated building-specific radiator coupling coefficients ($H_{\text{rad\_con}}$) based on peak heat loss with a 20% oversizing factor, rather than forcing all buildings to share a single hardcoded radiator capacity.
*   **Evaluation (Winter 2026 - 6 Feasible Buildings)**:
    *   **vonovia_model**: Comfort **$95.7\%$**, Cost €931.39, Energy $2,801.9$ kWh, HP cycles 166.
    *   **sfh_1984_1994_0_soc**: Comfort **$96.1\%$**, Cost €1,603.26, Energy $4,827.3$ kWh, HP cycles 29.
    *   **sfh_2016_now_0_soc**: Comfort **$93.9\%$**, Cost €896.10, Energy $2,697.7$ kWh, HP cycles 136.
    *   *Note: Physics fix was highly successful. Giving buildings sized radiators eliminated rapid boundary cycling. Comfort skyrocketed to 86–96% across all test targets.*

---

### **Version 13 (Random Initialization Range & 5M Steps)**
*   **What Changed from v11**:
    *   **Random Initialization Range**: Narrowed the random initial temperature ranges on environment reset ($T_{\text{room}} \in [18.0, 24.0]^\circ\text{C}$).
    *   **5M Steps Run**: Extended training to 5 million steps to reach full policy convergence.
*   **Evaluation (Winter 2026 - 6 Feasible Buildings)**:
    *   **vonovia_model**: Comfort $90.4\%$, Cost €877.77, Energy $2,638.4$ kWh, HP cycles 251.
    *   **sfh_2002_2009_0_soc**: Comfort $92.2\%$, Cost €774.40, Energy $2,326.7$ kWh, HP cycles 489.
    *   *Note: The agent fully converged on the comfort-cost trade-off. It learned to ride the bottom boundary (letting room temperature dip to 18.9°C) during high-price hours to save energy costs.*

---

### **Version 14 (Linear + Quadratic Comfort Penalty)**
*   **What Changed from v13**:
    *   **Linear Comfort Term**: Closed the edge-riding loophole by adding a linear term to the underheating penalty: $-20 \times |T_{\text{lower}} - T_{\text{room}}| - 20 \times (T_{\text{lower}} - T_{\text{room}})^2$.
*   **Evaluation (Winter 2026 - 6 Feasible Buildings)**:
    *   **vonovia_model**: Comfort **$96.2\%$**, Cost €883.33, Energy $2,657.8$ kWh, HP cycles 123.
    *   **sfh_1984_1994_0_soc**: Comfort **$94.9\%$**, Cost €1,610.15, Energy $4,839.0$ kWh, HP cycles 115.
    *   **sfh_2016_now_0_soc**: Comfort **$95.8\%$**, Cost €874.49, Energy $2,631.6$ kWh, HP cycles 381.
    *   *Note: Loophole closed. Minimum temperatures rose back, comfort returned to >90% averages, and costs remained low.*

---

### **Version 15 (Contextual RL & Cascaded Pumps)**
*   **What Changed from v14**:
    *   **Contextual RL Transition**: Replaced the building one-hot IDs with **5 physical context parameters** in the observation space ($H_{\text{tr}}, H_{\text{ve}}, c_{\text{bldg}}, \text{area\_floor}, \text{num\_pumps}$).
    *   **Multi-Family House (MFH) Integration**: Added 4 MFH models. Fixed a unit bug where MFH specific losses ($W/(\text{m}^2K)$) were fed directly instead of scaling by area.
    *   **Cascaded Pump Architecture**: Created a cascaded heat pump model where max electrical/thermal capacities scale dynamically with the building's required pumps.
    *   **Physics Capping**: Capped the agent's requested supply temperature to the physical limits of the heat pump cascade in `src/simulator.py`.
*   **Evaluation (Winter 2026)**:
    *   **vonovia_model**: Comfort $59.9\%$, Cost €967.35, Energy $2,909.4$ kWh, HP cycles 107.
    *   **sfh_2002_2009_0_soc**: Comfort $80.6\%$, Cost €840.40, Energy $2,522.1$ kWh.
    *   **mfh_1919_1948_0_soc** (3 Pumps, Leaky MFH): Comfort $60.7\%$, Cost €5,379.55, Energy $16,161.2$ kWh.
    *   *Note: Generalization was successful; the agent could heat uninslated buildings and MFHs without crashing. However, comfort fell due to the complexity of learning continuous physical relationships.*

---

### **Version 16 (SAC Network Scaling)**
*   **What Changed from v15**:
    *   **Policy Network Scaling**: Scaled up the SAC MLP policy network architecture from default sizes to **`[512, 512, 512]`** to handle physical features. Trained for 3 million steps.
*   **Evaluation (Winter 2026)**:
    *   **vonovia_model**: Comfort **$80.8\%$**, Cost €916.28, Energy $2,755.9$ kWh, HP cycles 248.
    *   **mfh_1919_1948_2_kfw** (1 Pump, Renovated MFH): Comfort **$92.3\%$**, Cost €1,259.04, Energy $3,784.3$ kWh.
    *   **mfh_1919_1948_0_soc** (3 Pumps, Leaky MFH): Comfort $27.6\%$, Cost €5,703.07, Energy $17,123.7$ kWh.
    *   *Note: Scaled network improved generalization significantly, yielding high comfort (92.3%) on renovated MFH targets, though leaky structures remain difficult.*

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

### **Table 3: Winter 2026 Evaluation Performance Comparison (`vonovia_model` Only)**
*Evaluating the exact same building across models highlights the relative impact of our reward and physics upgrades:*

| Model Version | T_room Range | Energy (kWh) | Cost (€) | Comfort % | HP Cycles | Main Upgrade / Characteristics |
|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| **v6** | $19.3 - 23.9^\circ\text{C}$ | $2814.0$ | €942.42 | $65.9\%$ | 169 | Parallel workers, LR schedule, normalization |
| **v7** | $17.7 - 23.6^\circ\text{C}$ | $2757.6$ | €915.77 | $69.5\%$ | 45 | Domain randomization (11 buildings, 11-hot ID) |
| **v8** | $18.4 - 25.3^\circ\text{C}$ | $2711.3$ | €903.90 | $81.7\%$ | 78 | Sanitized environments (eliminated leaky targets) |
| **v9** | $19.2 - 22.7^\circ\text{C}$ | $2700.8$ | €896.35 | $89.9\%$ | 121 | Aggressive asymmetric penalty ($-20.0$) |
| **v10** | $19.4 - 22.8^\circ\text{C}$ | $2631.3$ | €875.95 | $85.3\%$ | 110 | Pre-heating comfort bonus $+1.0$ (reward hacking) |
| **v11** | $19.8 - 22.9^\circ\text{C}$ | $2801.9$ | €931.39 | **$95.7\%$** | 166 | Dynamic radiator capacities physical sizing |
| **v13** | $18.9 - 22.5^\circ\text{C}$ | $2638.4$ | €877.77 | $90.4\%$ | 251 | Convergence on 5M steps (riding bottom edge) |
| **v14** | $19.1 - 22.1^\circ\text{C}$ | $2657.8$ | €883.33 | **$96.2\%$** | 123 | Linear + Quadratic comfort penalty (closed loophole) |
| **v15** | $18.9 - 24.3^\circ\text{C}$ | $2909.4$ | €967.35 | $59.9\%$ | 107 | Contextual RL (5 physical parameter inputs) |
| **v16** | $18.9 - 24.6^\circ\text{C}$ | $2755.9$ | €916.28 | $80.8\%$ | 248 | Contextual RL + Scaled network `[512, 512, 512]` |

---

## 🏆 Key Lessons Learned

1.  **Gradient Scaling Matters**: A bad reward scale (`1/1000`) completely disables policy learning. Always verify that your rewards and gradients are not vanishingly small.
2.  **Evaluate Bidirectionally**: Only evaluating one boundary (underheating) can hide massive policy defects on the other boundary (overheating).
3.  **Physical Feasibility Checks**: Forcing an RL agent to train on impossible physics (e.g., trying to heat a leaky building with an undersized heat pump) degrades the policy gradients across the entire domain.
4.  **Reward Hacking (Lazy Agents)**: Flat bonuses (like the pre-heating comfort bonus) introduce mathematical loopholes. Penalties are generally safer for strict constraint enforcement.
5.  **Contextual Generalization over Memorization**: One-hot IDs result in rigid policies. Contextualizing environments with physical variables ($H_{\text{tr}}, H_{\text{ve}}$, etc.) allows neural networks to learn physical relationships, enabling zero-shot transfer to new buildings.
6.  **Physics Caps Prevent Cheating**: If your environment allows actions to bypass real-world constraints, the agent will exploit them. Hard-clip control variables within actual physical limits *before* running ODE integration.
