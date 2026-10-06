# Nöbetçi: Duty Scheduling for the ITU IT Department

**Nöbetçi is the duty-scheduling system the IT Department of Istanbul Technical University (BİDB) uses to plan shifts for its student assistants.** I proposed it on my own initiative while working there as a student assistant and built the frontend, the backend and the optimization model. The department uses it officially, and I'm still developing it.

> The code belongs to Istanbul Technical University, so this repository has no source code, screenshots or data. It describes the problem, the model and the architecture.

## The problem

Every period, 35–55 people have to be assigned to day, evening and night shifts on weekdays and weekends. Some shifts have two seat types (desk and operator). The plan has to cover every seat and respect when people are unavailable. It also has to avoid unhealthy patterns such as a night shift followed by a morning shift, and keep the workload fair, both within the period and across past periods.

There are two groups with different rules, so the system has two schedulers:

| | Student assistants | Duty assistants |
|---|---|---|
| Shifts | 6 types: weekday day, evening, night; weekend morning, evening, night | Weekday day shift only, split into morning and afternoon halves |
| Seats | Desk and operator, ratio depends on headcount | Desk and operator, a different ratio |
| Preferences | "likes night shifts", "avoids weekends" | None |

## The model

Both schedulers are constraint programs solved with **Google OR-Tools CP-SAT**.

**Hard constraints** (never violated):

- Every seat is filled.
- No night shift followed by a morning shift.
- At most 2 shifts per person per day.
- Nobody gets more than the period's base load + 2.

**Soft constraints** carry penalties in five tiers, so a lower tier can never outweigh a higher one:

| Tier | Rule | Penalty |
|---|---|---|
| 1 | Assigning someone to a slot they marked unavailable | 200,000 (+25,000 for each further violation on the same person) |
| 2 | Gap between the most and least loaded person | 150,000 |
| 2 | Uneven split of shift types (day, evening, night, weekend) | 150,000 |
| 3 | 3+ consecutive days, two shifts on the same day, back-to-back nights | 5,000 – 20,000 |
| 4 | Weekly clustering, balance with past periods | 1,500 – 2,000 |
| 5 | Personal preferences (night, weekend) | ±5 – 10 |

Some decisions behind it:

- **Unavailability is soft but very expensive.** A hard rule would make the solver return nothing when the month is tight. With a high penalty, the solver always returns a plan, and when a violation is unavoidable it goes to the person who closed the most slots.
- **Fairness carries over between periods.** Each person's target is `base − (total so far − expected total so far)`. Someone who did extra shifts last period gets fewer this time. A new member starts with a difference of zero, so joining late is not a disadvantage.
- **Bounded solve time.** The solver runs with 4 workers and a 120-second limit and reports its status, the load range and any warnings with the plan.

## Architecture

```
Next.js 16 (TypeScript, Tailwind)          FastAPI scheduling service
NextAuth 5, admin and user roles   ──────▶ OR-Tools CP-SAT, two solvers
real-time conflict detection               API-key auth, Docker
          │
          ▼
PostgreSQL
stored functions instead of N+1 queries
```

- Admins open a period, collect unavailability, run the solver and publish the plan. Users mark the slots they can't take and see their own schedule and history.
- The solver service is stateless. The app sends the period, the people with their history and the slots, and receives the assignments with statistics.
- Pytest covers the solver rules, and separate scripts run realistic and stress scenarios (heavy unavailability, clustered absences, fairness over several periods).

## Tech

Python · FastAPI · OR-Tools (CP-SAT) · Pydantic v2 · pytest · Next.js 16 · TypeScript · Tailwind CSS · NextAuth 5 · PostgreSQL · Docker

## Contact

[LinkedIn](https://www.linkedin.com/in/mustafa-k%C3%BC%C3%A7%C3%BCkco%C5%9Fkun/) · [Email](mailto:m.kucukcoskunn@gmail.com)
