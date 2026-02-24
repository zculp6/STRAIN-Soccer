# ⚽ Soccer Defensive STRAIN Analysis

A Quarto-based analysis pipeline for quantifying defensive pressure in soccer using player tracking data. Built around the **STRAIN** metric — a frame-level measure of how aggressively the closest defender closes down the ball carrier or pass recipient in the attacking third.

---

## 📊 What is STRAIN?

**STRAIN** measures the rate at which a defender closes down the target player, normalized by distance:

```
STRAIN = -closure_rate / distance_to_target
```

Where:
- **closure_rate** = change in distance to target per second (negative = closing in)
- **distance_to_target** = distance from closest defender to the ball carrier or pass recipient
- STRAIN is clamped to **[0, 10]** — retreating defenders contribute 0, not negative pressure

The metric is only computed when the ball is in the **attacking third** and a defending player is physically present in that zone, making it a high-press indicator rather than a general pressure metric.

---

## 📁 Project Structure

```
.
├── soccer_event_animation.qmd   # Main analysis document
├── Sample_Game_1_RawEventsData.csv
├── Sample_Game_1_RawTrackingData_Home_Team.csv
└── Sample_Game_1_RawTrackingData_Away_Team.csv
```

---

## 🗂️ Data

Data is from the [Metrica Sports open sample dataset](https://github.com/metrica-sports/sample-data/tree/master/data) — a fully anonymized match with:

| File | Description |
|------|-------------|
| `RawEventsData.csv` | Event-level data: passes, carries, shots, etc. with frame ranges and player IDs |
| `RawTrackingData_Home_Team.csv` | 25Hz player + ball tracking, Home team |
| `RawTrackingData_Away_Team.csv` | 25Hz player + ball tracking, Away team |

Coordinates are in normalized `[0, 1]` space and are rescaled to metres centred on the pitch midpoint (`x ∈ [-52.5, 52.5]`, `y ∈ [-34, 34]`).

---

## 📦 Dependencies

Install all required R packages:

```r
install.packages(c(
  "tidyverse",
  "gganimate",
  "gifski",
  "sportyR",
  "magick",
  "patchwork"
))
```

| Package | Purpose |
|---------|---------|
| `tidyverse` | Data wrangling and ggplot2 visualizations |
| `gganimate` | Animated pitch and metrics GIFs |
| `gifski` | GIF renderer backend for gganimate |
| `sportyR` | FIFA-spec soccer pitch geometry |
| `magick` | Stitching pitch + metrics GIFs side-by-side |
| `patchwork` | Plot composition |

---

## 🔬 Analysis Pipeline

### 1. Data Ingestion
- Raw tracking CSVs are parsed and column names are normalized (`Player1_x`, `Player1_y`, etc.)
- Player positions are reshaped to long format and rescaled to metres
- Ball position is extracted separately

### 2. Possession Assignment
- Event data is expanded to cover every frame within each event's `Start.Frame`–`End.Frame` range
- Gaps between events are filled using **last-observation-carried-forward (LOCF)**
- For `PASS` events, the **recipient** (`To`) is treated as the target player; all other events use the ball carrier (`From`)

### 3. Attacking Direction Inference
- Each team's mean x-position per period is used to infer which direction they are attacking
- Away team position is used as the anchor signal; Home direction is derived as the inverse
- This correctly handles the **halftime direction flip**

### 4. STRAIN Calculation
- For each frame where the ball is in the defensive third, the closest defending player to the target is identified
- `closure_rate` is computed as the frame-over-frame change in distance (requires consecutive frames ≤ 0.2s apart, same player)
- STRAIN is derived and clamped to `[0, 10]`

### 5. Aggregation
- STRAIN is aggregated into **10-second rolling windows** per defending team
- Player-level summaries compute mean/max STRAIN, mean distance, and mean closure rate

---

## 📈 Visualizations

### Defensive STRAIN Over Time
Line chart of mean STRAIN per 10-second window, split by defending team (Home/Away), with a halftime marker.

### Average STRAIN Per Player
Horizontal bar chart ranking all players by mean STRAIN, with a team average reference line and a blue→orange→red gradient encoding intensity.

### Side-by-Side Animated Analysis
A combined GIF with two panels rendered in sync:

- **Left**: Animated pitch showing all players (red = Home, blue = Away), ball, and a **gold ring** highlighting the closest defender when STRAIN is being tracked
- **Right**: Single animated chart showing Distance, Closure Rate, and STRAIN over time — metrics go `NA` when the Home team is not defending, producing natural gaps in the trace

---

## ⚙️ Configuration

At the top of the animated section, adjust these parameters to explore different passages of play:

```r
ANIM_FPS      <- 10    # playback speed
ANIM_DURATION <- 20    # seconds of match time to render
ANIM_PERIOD   <- 1     # 1 = first half, 2 = second half
TIME_START    <- 0   # match time (seconds) to start from
```

To find high-press sequences worth animating:

```r
possession_data %>%
  filter(Team == "Away", Type %in% c("PASS", "CARRY"),
         Period == 1, Start.X > 0.7) %>%   # high up pitch in period 1
  arrange(Start.Frame) %>%
  mutate(frame_gap  = Start.Frame - lag(End.Frame),
         new_seq    = is.na(frame_gap) | frame_gap > 50) %>%
  group_by(sequence_id = cumsum(new_seq)) %>%
  summarise(n_events = n(), start_time = min(Start.Time..s.),
            duration_s = max(End.Time..s.) - min(Start.Time..s.)) %>%
  filter(n_events >= 3, duration_s >= 3) %>%
  arrange(desc(n_events))
```

> **Note:** For Period 2, flip the x-condition to `Start.X < 0.3` since attacking direction reverses at halftime.

---

## 📝 Notes

- STRAIN is intentionally **one-sided**: defenders backing off contribute 0, not negative values
- The `closure_rate` column in `strain_results` retains its sign (negative = closing in) for downstream use
