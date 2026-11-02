# Trap list

Things in the data an agent will get wrong. Started 2026-10-25 (Phase 0 exploration).

| Trap | Why an agent will get it wrong |
|---|---|
| Duplicate (q_id, entity) pairs: 135 pairs, mostly NYISO and TVA | It will count MW twice unless told how to handle them |
| Proposed online dates out to 2057 | "Coming online by year" questions will include unrealistic far-future dates |
| Withdrawal dates after the 2025 cutoff | "Withdrawals in 2025" questions may silently include 2026 rows |
| Placeholder IDs like "not assigned" | Uniqueness and project-count logic breaks |
| fips_code lost its leading zeros | Low priority; only matters if you join to other county data |
| Excel serial dates (now converted) | The agent must use the _dt columns, not the raw ones |
| Most hybrid projects (~79% of Solar+Battery) are missing their storage MW | Agent will could undercount storage in the queue if data is missing values |
| mw_N pairs with type_N: mw_2 isnt always battery | Agent could go with the most common pairing and just randoming assign this mw to battery storage |
| One row has mw_3 with no type_3 | So a capacity unassigned to a corresonding power type, seems like a data error |
| "Unknown" status isnt in codebook | Agent could miss these as it wasnt specified by like active, withdrawn, or suspended |
| Six standalone "battery" rows have mw_2 filled in | Agent if looking for capacity for a second type, and there is no second type its just a standalone battery, it may make errors |
| ERCOT active count is higher than the report's (+269, count only; GW matches) | An agent will report 1,796 with no caveat, and it won't match the published figure |
| Negative MW values: four repower or modification projects, like Giga Austin Solar at −53.3 MW, have negative mw_1 | Any sum that includes them comes out too low |
| 30 rows with no queue date or year: they have an agreement date only | Year-based questions will silently drop them |
| About 55 batteries sized at 9.8–9.99 MW: sized just under a 10 MW line | "Small project" questions are sensitive to where you set the cutoff |
| An active project with an online date (Carrizo Springs BESS) | Status and dates can contradict each other |
| Agent GW totals will be lower than Queued Up's published figures | The agent may present them as matching the report. Answers should say "recorded capacity" and caveat |
