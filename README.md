# Choose My Shirt: A Grounded LLM Experiment

This project demonstrates how an LLM can act as a small, rules-based style recommender using structured data and explicit selection rules.

The experiment asks the model to recommend shirts for particular days while restricting it to the information in the supplied files. The model must explain its choice, identify the rules it applied, and explicitly acknowledge when the files do not contain enough information.

## Experiment goal

The goal is to test whether an LLM can:

- Read shirt attributes from a CSV file.
- Filter shirts by day availability.
- Apply priority and pattern-selection rules in the correct order.
- Explain recommendations step by step.
- Avoid inventing information that is not present in the source files.

## Source files

### `Shirt Colour and Pattern Planner.csv`

This file contains the available shirts and their attributes:

- `Shirt ID`: unique shirt identifier.
- `Color`: shirt color.
- `Pattern`: shirt pattern.
- `Color Priority`: numeric priority for the color.
- `Pattern Priority`: numeric priority for the pattern.
- `Monday` through `Friday`: whether the shirt can be worn on that day (`Yes` or `No`).
- `Notes`: optional notes field.

The available shirts are:

| Shirt ID | Color | Pattern | Color Priority | Pattern Priority |
|---|---|---|---:|---:|
| S01 | White | Checker | 1 | 2 |
| S02 | White | Plain | 1 | 1 |
| S03 | White | Lines | 1 | 1 |
| S04 | Blue | Checker | 2 | 2 |
| S05 | Blue | Plain | 2 | 1 |
| S06 | Gray | Checker | 3 | 2 |
| S07 | Gray | Stripe | 3 | 3 |
| S08 | Maroon | Plain | 4 | 1 |
| S09 | Black | Lines | 5 | 1 |

The day columns contain data only for Monday through Friday. There is no Saturday or Sunday availability data.

### `Rules.txt`

This file contains the shirt-selection rules:

1. If two or more shirts are valid for a day, the shirt with the lowest priority wins.
2. Priority is the sum of color priority and pattern priority.
3. If a white plain shirt is in the valid list, it should be chosen.
4. If a plain shirt is available, it should be chosen over other patterns.

The rule file is treated as selection logic, while the CSV is treated as the factual shirt and availability data.

## How a recommendation is made

For a requested day, the LLM should:

1. Find shirts marked `Yes` for that day.
2. Calculate each valid shirt's total priority:

   `Color Priority + Pattern Priority`

3. Select the lowest-priority valid shirt.
4. Apply the white-plain and plain-pattern preferences when relevant.
5. Explain the selected shirt and cite the rule numbers used.
6. Say that there is not enough information if the requested day or attribute is not covered by the files.

## Sample questions and grounded answers

### Positive case: Friday

**Question:** Which shirt can be chosen on Friday? And why?

**Answer:** Choose `S02 — White Plain`.

- S01 White Checker: `1 + 2 = 3`
- S02 White Plain: `1 + 1 = 2`
- S03 White Lines: `1 + 1 = 2`
- S08 Maroon Plain: `4 + 1 = 5`
- S09 Black Lines: `5 + 1 = 6`

Rules applied:

- Rule 1: the lowest priority wins.
- Rule 2: priority is the sum of color and pattern priorities.
- Rule 3: choose a white plain shirt when available.
- Rule 4: prefer plain over other patterns.

S02 and S03 tie on total priority, but Rule 3 resolves the tie in favor of S02.

### Positive case: Tuesday

**Question:** Which shirt can be chosen on Tuesday? And why?

**Answer:** Choose `S05 — Blue Plain`.

- S04 Blue Checker: `2 + 2 = 4`
- S05 Blue Plain: `2 + 1 = 3`

Rules 1, 2, and 4 apply. Rule 3 does not apply because no valid white plain shirt is available on Tuesday.

### Positive case: pattern query

**Question:** Can I wear any Lines shirt on Friday?

**Answer:** Yes. The valid Friday Lines shirts are:

- `S03 — White Lines`
- `S09 — Black Lines`

This answer uses only the `Pattern` and Friday availability columns. It does not invent any additional Lines shirts.

### Negative case: invalid color/day combination

**Question:** Can I wear a blue shirt on Thursday?

**Answer:** No. The blue shirts, S04 and S05, are both marked `No` for Thursday.

The answer is based on the day-availability data, not on an assumption about what colors are normally suitable for Thursday.

### Negative case: unavailable preferred shirt

**Question:** Why didn't you choose a plain white shirt on Thursday?

**Answer:** A plain white shirt cannot be chosen because S02 is marked `No` for Thursday. The white-plain preference cannot be applied when that shirt is not valid for the requested day.

The valid Thursday shirts are S06 Gray Checker, S07 Gray Stripe, and S09 Black Lines. Rule 1 selects S06 because its total priority is the lowest: `3 + 2 = 5`.

### Missing-data case: Saturday

**Question:** Which color should I choose on Saturday?

**Answer:** There is not enough information. The planner contains availability columns only for Monday through Friday, so the selection rules cannot be applied to Saturday.

This is the key anti-hallucination behavior: the LLM does not guess a Saturday color from weekday patterns, personal style assumptions, or general fashion knowledge.

## Anti-hallucination design

The experiment deliberately constrains the model:

- Use only the attached files as sources.
- Do not use general fashion knowledge.
- Do not infer missing days, colors, patterns, or priorities.
- Distinguish factual data from selection instructions.
- Show calculations instead of presenting an unsupported recommendation.
- Report insufficient information when a requested fact is absent.

The Saturday question is especially useful because it tests whether the model will fabricate a weekend rule. A grounded response must identify that Saturday is outside the CSV's coverage.

## How the model was kept grounded

The prompt gave the model a strict evidence boundary before asking any style question. It explicitly said that the attached CSV and rules file were the only allowed sources. This prevented the model from using general fashion knowledge or silently filling gaps with assumptions.

The grounding approach used four controls:

1. **Source restriction:** Use only the CSV and the rule file.
2. **Instruction separation:** Treat the CSV as factual shirt and availability data, and the text file as selection instructions.
3. **Required calculation:** Filter by day, calculate `Color Priority + Pattern Priority`, and show the relevant values.
4. **Uncertainty instruction:** If the files do not contain the required information, say that there is not enough information and do not guess.

The model was also asked to include rule numbers in recommendations. This made the reasoning auditable and made it easier to detect an unsupported choice.

## Prompt used to establish the experiment

```text
You are a style recommender. The attached CSV contains the list of shirts,
their colors, patterns, priorities, and the days on which each shirt can be
worn. The attached text file contains the rules for selecting a shirt.

Use only these attached documents as your source of truth. Do not use general
fashion knowledge and do not infer facts that are not present. Treat the CSV
as factual data and the text file as selection instructions.

When choosing a shirt for a day:
1. Filter to shirts marked Yes for that day.
2. Calculate total priority as Color Priority + Pattern Priority.
3. Apply the selection rules from the text file.
4. Give a structured, point-by-point answer and identify the rule numbers used.

If the documents do not contain enough information, explicitly say so and do
not hallucinate or make a best guess.
```

## Prompts that make the model helpful

These follow-up prompts produce useful, verifiable responses while preserving the grounding constraint.

### Ask for a day-specific recommendation

```text
Which shirt can be chosen on Friday? And why?
List the valid Friday shirts, show each priority calculation, identify the
recommended shirt, and cite the rule numbers applied.
```

### Ask whether a color or pattern is available

```text
Can I wear a blue shirt on Thursday?
Answer only from the planner. Identify the matching shirts and their Thursday
availability. If none are valid, say so clearly.
```

### Ask for pattern-specific options

```text
Can I wear any Lines shirt on Friday?
List every matching shirt that is valid on Friday. Do not recommend shirts
that are not present in the CSV.
```

### Ask why a preferred shirt was not selected

```text
Why did you not choose a plain white shirt on Thursday?
Check whether a valid white plain shirt exists for Thursday before applying
the preference rules. Explain which rules apply and which cannot be applied.
```

### Test missing information

```text
Which color should I choose on Saturday?
Check whether Saturday is present in the planner. If it is absent, say that
there is not enough information. Do not infer Saturday from another day.
```

### Ask for a concise answer

```text
Give only the recommended shirt, its total priority, and the rule numbers
used. If the information is missing, say "Not enough information" and name
the missing field.
```

## Example prompt

```text
Use only the shirt planner CSV and the shirt-selection rules file.
For a requested day, filter the shirts marked Yes, calculate total priority as
Color Priority + Pattern Priority, and apply the rules in the rules file.
Give a structured answer with the recommended shirt, calculations, and rule
numbers. If the files do not contain enough information, say so and do not guess.
```

## Limitations

- The planner covers Monday through Friday only.
- The source files do not define a rule for resolving every possible tie.
- The files do not provide weather, occasion, fit, fabric, season, or personal-preference information.
- Recommendations are limited to the shirts and rules supplied in this experiment.

## Files

- `Shirt Colour and Pattern Planner.csv` — shirt inventory, priorities, and day availability.
- `Untitled.txt` — selection rules applied to the valid shirts.
- `README.md` — experiment description and sample interactions.
