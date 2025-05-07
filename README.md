# trip-planner
## ieor164 final project
## Introduction
This project presents an optimization model that select and schedule events for a 3-day road trip to the Lassen National Park based on time, money, energy, preference, and location. It incorporates a simple uncertainty scenario and allows user to override part of the schedule and resolve the plan. 
## Itinerary
| Time                    | Event                        |     |
| ----------------------- | ---------------------------- | --- |
| Day 1 08:00–Day 1 13:00 | drive_out                    |     |
| Day 1 13:00–Day 1 16:00 | upper_bidwell_park           |     |
| Day 1 16:00–Day 1 18:00 | rest                         |     |
| Day 1 18:00–Day 1 21:00 | north_table_mountain_reserve |     |
| Day 1 21:00–Day 2 00:00 | rest                         |     |
| Day 2 00:00–Day 2 08:00 | lodging3_ontheway            |     |
| Day 2 08:00–Day 2 10:00 | boating_lassen               |     |
| Day 2 10:00–Day 2 21:00 | rest                         |     |
| Day 2 21:00–Day 2 23:00 | stargazing_lassen            |     |
| Day 2 23:00–Day 3 00:00 | rest                         |     |
| Day 3 00:00–Day 3 08:00 | lodging1_campcabin           |     |
| Day 3 08:00–Day 3 10:00 | sulfurworks_lassen           |     |
| Day 3 10:00–Day 3 13:00 | rest                         |     |
| Day 3 13:00–Day 3 18:00 | peaktrail_lassen             |     |
| Day 3 18:00–Day 3 19:00 | rest                         |     |
| Day 3 19:00–Day 4 00:00 | drive_back                   |     |
![Road Trip Schedule](./Pasted%20image%2020250507012329.png)
## Rationale
Trip planning is a scheduling problem that maximizes happiness while making sure the condition is reasonably survivable, for example a human must sleep, eat and cannot teleport. 
The main decision variable is a set of binary variables that denote whether event $i$ happened at hour $j$. Time, money and energy are modeled as constraint since they are requirements to meet. Preference and location are modeled in the objective function to maximize utility and convenience.
To organize events and ease user input, an event is abstracted into a class with the following fields: `name`, `money_cost`, `time_cost`, `preference_point`, `event_type`, `energy_cost` and `location`. Most of the fields are pretty self-explanatory.
### event type
There are four types of event: transportation, lodging, activity, and free. The categorization is to ensure that the plan is logical. For example, there must be one lodging selected every night. 
### energy cost
A human eats and rests. Energy cost enforces a rest block to be inserted every once in a while.
### location
The location field helps find the shortest path to perform the activities. This is especially helpful if you want to visit sceneries on the way and still want the activities be ordered without detour or backtracking. 
## Features
### 1. i did not follow the plan
Some people do not follow plans.  Thus, a `history` dictionary is provided where users can override the schedule by stating "6pm: napped" and resolve the model with that additional constraint which marks 6pm as rest. 
### 2. weather uncertainty
Nature is not deterministic. Certain activities such as stargazing may be less preferred under cloudy weather. This activity is thus modeled with half a chance of having half preference point and half duration.
## the Model
### decision variable
$x_{ij}$ is a binary variable that denotes whether event $j$ is scheduled at hour $i$.
### objective function
$$max\ \sum x_{ij}\cdot preference_j-distancePenalty$$
### constraints
#### distance penalty
$$distPenalty=\lambda\sum_{k=1}\sum_{a\in R}\sum_{b\in R}d_{a,b}v_{a,b,k}$$
This is essentially the sum of traveled distance from this event to the next event, weighted by $\lambda$. However, since rests can be inserted in between events, we cannot simply sum the distance between consecutive events. a helper list of "only events with location" is built and and ordered chronologically.
$d_{a,b}$ denotes the Euclidean distance between event $a$ and event $b$, calculated with the given longitude and latitude of event. 
$v_{a,b,k}$ is a binary variable that denotes whether slot $k$ is event $a$ and slot $k+1$ is event $b$. 
Here are some auxiliary variables to build this constraint: V1, V2, V3: Linearize $v[a,b,k]$ to 1 iff event $a$ is slot $k$ and event $b$ is slot $k+1$. Tdef : Defines each event’s continuous start‐time variable $T[a]$ based on its scheduled start binary flags. ChronoOrder: Uses a big-M constraint to force slot order $u[a,k]$ to match chronological order $T[a]$.

#### energy cost
$$
E_h =
\begin{cases}
E_{\mathrm{initial}} - \Delta_0, & h = 0,\\[6pt]
E_{h-1} - \Delta_h,              & h > 0,
\end{cases}
$$

where $E_h$ denotes energy level at each hour and

$$
\Delta_h \;=\; \sum_{a\in A} \bigl(\text{energy\_cost}[a]\bigr)\,\cdot\text{start}[{a,\,h-\mathrm{duration}[a]+1}]
$$
where $start[a,t]$ denotes whether event $a$ is scheduled and started at hour $t$. This means $\Delta_h$ only updates energy level at the completion of each event.
To prevent exhaustion and unrealistic charging, 
$$0\leq E_h \leq E_{max}$$
This introduce the subtlety that unless you are completely exhausted, you can not lodge because lodging events have an energy cost of $-E_{max}$. To work around this, a special "free" event is introduced with positive energy cost. 
#### other constraints
- EventOnce: Ensures that any activity or transportation event is scheduled at most once.
- DurationLink: Ties the total number of hours an event is active to its block-count variable y[a].
- OneHour: Enforces that exactly one event occupies each hour.
- TotalTime: Caps the total scheduled time across all events to 64 hours.
- Budget: Keeps total spending on lodging and activities within the $1 000 budget.
- StartLink: Matches the number of “start” flags for an event to how many blocks of that event you take.
- XStart: Ensures x[h,a] = 1 exactly when hour h falls inside one of event a’s start-to-end intervals.
- StartDriveOut: Forces the “drive_out” trip to begin at hour 0.
- StartDriveIn: Forces the “drive_back” trip to begin at the final hour slot.
- LodgingTime: Limits lodging to nighttime hours only.
- LodgeCount: Requires exactly two nights of lodging in the plan.
- Stargaze: Restricts stargazing to the 9 PM–5 AM window.
- OneHourScenario: Enforces that exactly one event runs each hour in every weather scenario.
## Code
