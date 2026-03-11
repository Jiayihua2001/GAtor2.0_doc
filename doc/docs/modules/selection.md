# Selection Modules

Selection modules choose parent structures from the pool for crossover and mutation. The selection pressure balances exploitation (favoring low-energy structures) with exploration (maintaining diversity).

---

## Module Selection

```ini
[modules]
selection_module = Adaptive_tournament_selection
```

---

## Tournament Selection

Selects the best structure from a random subset of the pool.

```ini
[modules]
selection_module = tournament_selection

[selection]
tournament_size = 10
fitness_reversal_probability = 0.0
```

| Option | Type | Description |
|--------|------|-------------|
| `tournament_size` | int | Number of candidates per tournament |
| `fitness_reversal_probability` | float | Probability of selecting the *worst* instead of best (promotes diversity) |

**How it works**: Randomly sample `tournament_size` structures, return the one with the highest fitness. Repeat for the second parent.

---

## Adaptive Tournament Selection

Automatically adjusts the tournament size based on reward feedback.

```ini
[modules]
selection_module = Adaptive_tournament_selection

[selection]
tournament_size = 10
```

**How it works**: Maintains a probability distribution over different tournament sizes. After each GA cycle, the tournament size that produced the best offspring gets a reward, increasing its probability of being selected in future rounds.

---

## Roulette Wheel Selection

Fitness-proportionate selection — structures with higher fitness have a proportionally higher chance of being selected.

```ini
[modules]
selection_module = roulette_selection
```

---

## Adaptive Mixed Selection

Dynamically adjusts the probability of using different selection methods (tournament, roulette, etc.) based on their historical success.

```ini
[modules]
selection_module = AdaptiveMix_selection

[selection]
tournament_size = 10
```

---

## Uniform Selection

Selects parents uniformly at random, regardless of fitness.

```ini
[modules]
selection_module = uniform_selection
```

---

## Elite Selection

Always selects the top structures by fitness.

```ini
[modules]
selection_module = elite_selection
```

---

## Fitness Reversal

All selection modules support fitness reversal — with some probability, the fitness ranking is inverted so that *worse* structures are preferred. This prevents premature convergence:

```ini
[selection]
fitness_reversal_probability = 0.1    # 10% chance of reversed selection
```
