---
description: "Prepare for workouts with pre-workout and post-workout guidance aligned to health goals. Use this skill when the user says 'prepare workout', 'pre-workout', 'post-workout', 'workout prep', 'ready to exercise', 'warm up', 'cool down', 'recovery', 'what workout today', or 'exercise plan'. Provides warm-up, workout plan, cool-down, nutrition, and recovery guidance based on goals and planned exercises."
---

# Workout Preparation Skill

## Context
Pius has specific health and fitness goals that workouts should align to:
- **Health & Fitness Transformation (Q2 2026)** — 5x/week exercise
- **Marathon Preparation** — Full marathon (42.2km) by Sep 2026, 10km in under 55 min
- **Core & Abs** — 3-4x/week core workouts
- **Pelvic Floor** — Targeted exercises 4-5x/week
- **Flexibility** — Daily stretching & mobility
- **Body composition** — Target 18% body fat or less

This skill provides goal-aligned pre-workout preparation, workout execution guidance, and post-workout recovery.

## When to Use
- Before a workout: "prepare my workout", "what should I do today?"
- After a workout: "post-workout", "recovery", "log my workout"
- When planning the week's workouts: "workout plan"

---

## PRE-WORKOUT Workflow

### Step 1: Check Today's Context
Pull data in parallel:

1. **Planned exercises for today:**
   ```
   DailyPlanner-get_exercises(status: "Planned")
   ```
   Filter to today's date.

2. **Recent workout history (avoid overtraining):**
   ```
   DailyPlanner-get_exercise_logs(startDate: "[3 days ago]", endDate: "[today]")
   ```

3. **Today's nutrition so far:**
   ```
   DailyPlanner-get_diet_entries(date: "[today]")
   ```

4. **Water intake:**
   ```
   DailyPlanner-get_water_intake(date: "[today]")
   ```

5. **Goals for alignment:**
   ```
   DailyPlanner-get_goals(tag: "health")
   ```

### Step 2: Determine Workout Focus
Based on planned exercises and recent history, determine today's focus:

| Day Pattern | Focus | Goal Alignment |
|-------------|-------|----------------|
| Upper body last 2 days | Lower body or cardio | General fitness |
| No running in 3+ days | Running session | Marathon prep |
| No core in 2+ days | Core & pelvic | Core strength + pelvic |
| No stretching today | Include mobility | Flexibility |
| Rest day needed | Active recovery / stretching only | Recovery |

If no exercises are planned, suggest based on goal alignment:
```
DailyPlanner-generate_workout_routine(
  focusAreas: "[based on gap analysis]",
  daysPerWeek: 5,
  level: "intermediate",
  location: "both"
)
```

### Step 3: Pre-Workout Checklist
Present readiness assessment:

```markdown
# 🏋️ Pre-Workout Check — [Date]

## 📋 Readiness
| Check | Status | Action |
|-------|--------|--------|
| Hydration | ⚠️ 500ml / 2000ml | Drink 250ml water now |
| Last meal | ✅ 2 hours ago | Good timing |
| Sleep | ❓ Not tracked | — |
| Recent workouts | ✅ Rest day yesterday | Good recovery |
| Muscle soreness | ❓ Ask | Any sore areas? |

## 🎯 Today's Focus: [Focus Area]
**Goal alignment:** [Which health goal this serves]

## 🔥 Warm-Up Routine (10-15 min)
```

### Step 4: Generate Warm-Up
Tailor warm-up to today's workout focus:

**For Running/Cardio:**
```markdown
### 🏃 Running Warm-Up (10 min)
1. **Light walk** — 2 min
2. **Dynamic leg swings** — 10 each side
3. **Hip circles** — 10 each direction
4. **High knees** — 30 sec
5. **Butt kicks** — 30 sec
6. **A-skips** — 30 sec
7. **Light jog** — 3 min (gradually increasing pace)

💡 *Marathon prep tip: Start 30-60 sec/km slower than target pace*
```

**For Strength/Core:**
```markdown
### 💪 Strength Warm-Up (10 min)
1. **Arm circles** — 15 each direction
2. **Cat-cow stretches** — 10 reps
3. **Bodyweight squats** — 15 reps
4. **Glute bridges** — 15 reps
5. **Dead bugs** — 10 each side
6. **Light set of main exercise** — 50% weight, 10 reps

💡 *Core focus: Activate pelvic floor with breathing — exhale on exertion*
```

**For Flexibility/Mobility:**
```markdown
### 🧘 Mobility Warm-Up (5 min)
1. **Neck rolls** — 5 each direction
2. **Shoulder rolls** — 10 each direction
3. **Torso twists** — 10 each side
4. **Ankle circles** — 10 each direction
5. **Deep breathing** — 1 min
```

### Step 5: Workout Plan
Present today's exercises with goal context:

```markdown
## 🏋️ Today's Workout

### Main Workout
| Exercise | Sets | Reps/Duration | Rest | Goal |
|----------|------|---------------|------|------|
| Squats | 4 | 12 | 60s | General fitness |
| Plank | 3 | 45s | 30s | Core strength |
| Kegels | 3 | 15 | 15s | Pelvic floor |
| Running | — | 5 km | — | Marathon prep |

### Pelvic Floor Integration
*Include with every workout:*
- Kegel holds: 3 × 10 sec holds
- Quick flicks: 3 × 15 reps
- Breathing cue: Exhale + lift on exertion

### Intensity Guide
- **Heart rate zone:** [based on goal — easy/tempo/threshold]
- **RPE target:** [6-8 out of 10]
- **Marathon pace guide:** [if running]

### 📱 Quick Log Command
After each exercise, log it:
> "Log [exercise] — [sets]×[reps] at [weight]kg"
```

### Step 6: Nutrition Timing
Check pre-workout nutrition:

```markdown
## 🍽️ Pre-Workout Nutrition
| Timing | Recommendation | Status |
|--------|---------------|--------|
| 2-3 hrs before | Balanced meal (carbs + protein) | ✅ Had lunch at 12:30 |
| 30 min before | Light snack (banana, energy bar) | 💡 Consider a snack |
| Now | 250ml water | ⚠️ Drink now |

### Recommended Pre-Workout Snacks
- 🍌 Banana + tablespoon peanut butter
- 🍞 Toast with honey
- 🥤 Smoothie (banana + oats + milk)
```

---

## POST-WORKOUT Workflow

### Step 7: Cool-Down Routine
Immediately after the workout:

**For Running/Cardio:**
```markdown
### 🧊 Cool-Down (10 min)
1. **Slow jog / walk** — 3 min
2. **Standing quad stretch** — 30 sec each leg
3. **Standing hamstring stretch** — 30 sec each leg
4. **Calf stretch** — 30 sec each leg
5. **Hip flexor stretch** — 30 sec each side
6. **IT band stretch** — 30 sec each side
7. **Deep breathing** — 1 min

💡 *Marathon tip: Foam roll IT band and calves within 30 min*
```

**For Strength:**
```markdown
### 🧊 Cool-Down (10 min)
1. **Light walking** — 2 min
2. **Stretch worked muscles** — 30 sec each
3. **Child's pose** — 30 sec
4. **Cat-cow** — 10 reps
5. **Pelvic floor relaxation breathing** — 1 min
6. **Full body stretch** — 2 min
```

### Step 8: Log the Workout
Help the user log what was done:

```
ask_user: "How did the workout go? Any exercises to adjust?"
```

For each exercise performed:
```
DailyPlanner-create_exercise(
  type: "[exercise type]",
  date: "[today]",
  status: "Completed",
  timeOfDay: "[Morning/Evening]",
  sets: [X],
  reps: [X],
  weight: [X],
  durationMinutes: [X],
  caloriesBurned: [estimated],
  goalId: "[linked health goal]",
  notes: "[user notes]"
)
```

Or log detailed sets:
```
DailyPlanner-create_exercise_log(
  exerciseId: "[exercise_id]",
  date: "[today]",
  setsCount: [X],
  repsPerSet: [X],
  weightPerSet: [X],
  duration: "[HH:MM:SS]",
  calories: [X],
  notes: "[how it felt]"
)
```

### Step 9: Post-Workout Nutrition
```markdown
## 🍽️ Post-Workout Nutrition (within 30-60 min)

### Recovery Window
| Nutrient | Target | Why |
|----------|--------|-----|
| Protein | 20-30g | Muscle repair |
| Carbs | 30-50g | Glycogen replenishment |
| Water | 500ml+ | Rehydration |

### Suggested Post-Workout Meals
| Meal | Protein | Carbs | Calories |
|------|---------|-------|----------|
| Protein shake + banana | 25g | 30g | ~300 |
| Chicken breast + rice | 35g | 45g | ~450 |
| Greek yogurt + granola + berries | 20g | 35g | ~350 |
| Eggs (3) + toast + avocado | 22g | 25g | ~400 |

💡 *Log your post-workout meal:*
> "Log meal: [meal name]"
```

Log water intake:
```
DailyPlanner-create_water_intake(amountMl: 500)
```

### Step 10: Recovery Assessment

```markdown
## 🔄 Recovery Plan

### Immediate (Next 2 hours)
- 💧 Drink 500ml water
- 🍽️ Eat protein-rich meal within 60 min
- 🧊 Ice any sore joints (if needed)

### Today
- 🚶 Light walking throughout the day
- 💧 Hit 2000ml+ total water
- 😴 Aim for 7-8 hours sleep tonight

### Tomorrow's Recommendation
Based on today's workout ([focus area]):
- **Tomorrow's focus:** [complementary muscle group / rest]
- **Intensity:** [lower/same/higher]
- **Pelvic floor:** [include/rest]
- **Flexibility:** ✅ Always include daily stretching
```

### Step 11: Progress Tracking

```markdown
## 📊 Weekly Workout Progress

| Goal | Target | This Week | Status |
|------|--------|-----------|--------|
| Exercise 5x/week | 5 sessions | [X] done | ████░░░░░░ |
| Core 3-4x/week | 3-4 sessions | [X] done | ██████░░░░ |
| Pelvic 4-5x/week | 4-5 sessions | [X] done | ████████░░ |
| Flexibility daily | 7 sessions | [X] done | ██████████ |
| Running (marathon) | 3x/week | [X] done | ██████░░░░ |

### Marathon Prep Metrics
- Weekly mileage: [X] km / [target] km
- Long run this week: [X] km
- Easy pace: [X] min/km
- Tempo pace: [X] min/km
```

---

## Tools & APIs Used
- `DailyPlanner-get_exercises` — Planned and completed exercises
- `DailyPlanner-get_exercise_logs` — Recent workout history
- `DailyPlanner-get_exercise_types` — Available exercises
- `DailyPlanner-create_exercise` — Log exercises
- `DailyPlanner-create_exercise_log` — Detailed set logging
- `DailyPlanner-get_diet_entries` — Nutrition check
- `DailyPlanner-get_water_intake` — Hydration check
- `DailyPlanner-create_water_intake` — Log hydration
- `DailyPlanner-get_goals` — Health goal alignment
- `DailyPlanner-get_body_measurements` — Progress tracking
- `DailyPlanner-generate_workout_routine` — Routine generation
- `ask_user` — Workout feedback and soreness check

## Output Format
Pre-workout: Readiness check → warm-up → workout plan → nutrition timing.
Post-workout: Cool-down → logging → nutrition → recovery plan → progress tracking.

## Goal-Specific Notes

### Marathon (42.2km by Sep 2026)
- Follow 80/20 rule: 80% easy runs, 20% hard
- Weekly long run increases by max 10%
- Include tempo runs 1x/week
- Rest or cross-train the day after long runs
- Track weekly mileage and pace trends

### Core & Pelvic Floor
- Integrate pelvic floor activation into ALL workouts (breathing cue)
- Core work before cardio for better activation
- Progress: bodyweight → light resistance → heavier resistance
- Avoid heavy Valsalva (breath holding) — exhale on exertion

### Body Composition (18% body fat)
- Prioritize protein intake (1.6-2.0g per kg bodyweight)
- Track body measurements monthly
- Strength training preserves muscle during fat loss
- Caloric deficit from diet, not excessive cardio

### Flexibility
- Daily stretching even on rest days (5-10 min minimum)
- Dynamic stretching before workouts
- Static stretching after workouts
- Focus on hip flexors, hamstrings, shoulders (desk worker areas)
