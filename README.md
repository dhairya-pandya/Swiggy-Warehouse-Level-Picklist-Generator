# Warehouse-Level Picklist Optimization for High-Throughput Fulfillment

**Team:** `GETALIFE();`
**Topic:** Warehouse-Level Picklist Optimization for High-Throughput Fulfillment

## 👥 Team Members

| Name | Email | GitHub | LinkedIn |
| :--- | :--- | :--- | :--- |
| **Dhairya Pandya** | [Email](mailto:dhairyapandya2006@gmail.com) | [GitHub](https://github.com/dhairya-pandya) | [LinkedIn](https://linkedin.com/in/dhairya-pandya) |
| **Shantanu Kharwar** | [Email](mailto:shsuasud@gmail.com) | [GitHub](https://github.com/ByteKnight28) | [LinkedIn](https://www.linkedin.com/in/shantanu-kharwar/) |
| **Rounak Agarwal** | [Email](mailto:rounak06.agarwal@gmail.com) | [GitHub](https://github.com/rounak-agl) | [LinkedIn](https://www.linkedin.com/in/rounak-agrwl/) |

---

## 🚀 Performance & Scalability
Our solution is optimized for speed and high-volume data processing. During benchmarking, the engine successfully processed the complete dataset spanning **31 days of orders** and generated all required CSV picklists in **under 25 minutes**. 

This translates to a processing throughput of approximately **1.24 operational days per minute**, demonstrating the system's capability to handle enterprise-level warehouse operations within tight time constraints.

## 📖 Problem Overview
The core objective of this project is to design a **Warehouse Picklist Optimization Engine** that maximizes the number of SKU units picked and loaded before specific cut-off times.

This is not merely a bin-packing problem; it is a complex resource scheduling challenge involving:
* **Hard Constraints:** Strict loading deadlines (cutoffs), picker shift availability, and physical limits (weight/volume).
* **Operational Efficiency:** Minimizing non-productive time (traveling to zones, staging areas) to maximize "picking" time.
* **Fulfillment Value:** Prioritizing total units fulfilled.

---

## 💡 Core Strategy: "The Long Picklist"
Our solution is built on a specific operational insight: **Every picklist incurs a fixed "Setup and Teardown" cost of 4 minutes** (travel to zone + travel to staging).

* **The Strategy:** We maximize throughput by consolidating tasks into the longest possible picklists (up to **140 minutes**) rather than generating many short ones.
* **The Impact:** This saves approximately 4-6 minutes of "dead walking" per cycle, converting that time into productive picking action.

---

## ⚙️ Assumptions & Constraints

| Constraint | Value / Logic | Impact on Throughput |
| :--- | :--- | :--- |
| **Zone Isolation** | Strict | Pickers cannot switch zones within a single list; cross-zone orders are split. |
| **Fragile Items** | Segregated | Fragile items are isolated into specific lists with a reduced **50kg** capacity to prevent damage. |
| **Operational Buffer** | 10 mins/hour | A standard 1-hour slot is calculated as **50 effective minutes** to account for human error/fatigue. |
| **Physical Limits** | 200kg / 2000 Units | Hard caps on weight and unit count per picklist (Normal items). |
| **Look-Ahead** | Active | Pickers can process future Pod Priorities if current queues are empty to prevent idling. |

---

## 🧠 Algorithm Logic

### Step 1: Pre-processing & Hierarchical Grouping
1.  **Data Cleaning:** Physical dimensions (L/W/H) are ignored in favor of Weight and Unit counts.
2.  **Grouping:** Data is grouped by **Pod Priority** (P1 > P9) to respect urgency, then by **Zone**.
3.  **Sorting:** Inside zones, orders are sorted by `order_qty` (descending) to clear high-volume orders first and improve packing density.

### Step 2: Dynamic Time-Window Assignment
We treat picker shifts as dynamic resource windows. The duration of a picklist is determined by the workforce density in that window.

| Priority Window | Time | Available Pickers | Max Duration | Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **P1 - P3** | 9 PM - 4 AM | ~80 | **140 mins** | Maximize duration to clear backlog during peak staff. |
| **P4** | 4 AM - 6 AM | 80 ➝ 35 | **50 mins** | Reduced duration to manage shift drop-off/handover. |
| **P5 - P9** | 6 AM - 11 AM | 35 - 70 | **50 mins** | Standard cycles for steady day-shift flow. |

### Step 3: Proactive Load Balancing
When the queue for the current time window is empty, the algorithm assigns orders from *future* priority pods (e.g., picking P2 orders during the P1 window). This "banks" work and reduces pressure on later shifts with lower availability.

### Step 4: Greedy Construction
Picklists are generated using a greedy approach that fills the list until:
1.  **Time Limit** is reached (calculated via travel + picking time).
2.  **Unit Cap** (2000) is reached.
3.  **Weight Cap** (200kg/50kg) is reached.

---

## 💻 Code Structure: `PickList` Class

The solution is encapsulated in the `PickList` class.

### Class Attributes
* `POD_PRIORITIES`: Defines the processing order (P1 -> P9).
* `PICKER_SHIFTS`: Stores shift start/end times and picker counts.

### Core Methods
* `__init__`: Loads orders and groups them into `self.pod_dict`.
* `build_zone_priority_queue()`: Creates a max-heap of zones based on total SKU demand.
* `get_active_pickers(current_time)`: Returns available picker count based on shift overlaps.
* `generate_picklist(pod_priority)`: **The Main Engine.** Aggregates SKUs, applies bin-packing, checks all constraints, and outputs optimization.
* `calculate_picklist_time()`: Estimates total execution time (Travel + Pick + Unload).
* `export_picklists_to_csv()`: Exports results in the required format: `{date}_{rank}_{picklist_no}.csv`.

---

## 📊 Key Insights & Trade-offs

### 1. Throughput vs. Flexibility
* **Trade-off:** Long picklists (140 mins) lock pickers into a zone, reducing agility.
* **Justification:** Since cutoffs are fixed and known, maximizing volume via reduced setup time is mathematically superior to flexibility.

### 2. The "50-Minute Hour"
* **Insight:** We calculate capacity based on 50 minutes per hour.
* **Result:** This creates a necessary buffer for real-world disruptions (jammed racks, fatigue), ensuring that our theoretical schedule holds up in practice.

### 3. Workforce Re-Engineering
* **Observation:** P4 (6:00 AM) and P5 (7:00 AM) are high-risk failure points due to low staff.
* **Recommendation:** A shift modification is required to ramp up workforce earlier (starting 6 AM instead of 8 AM).
