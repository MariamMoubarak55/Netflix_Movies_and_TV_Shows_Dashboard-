# Netflix Content Analysis Dashboard — Power BI Build Guide

This guide rebuilds the delivered interactive dashboard as a **real Power BI Desktop report** (`.pbix`)
in roughly 15 minutes, using the exact same BLACK + RED + WHITE visual identity.

**Companion files (same folder):**

| File | Purpose |
|---|---|
| `netflix_clean_for_powerbi.csv` | Cleaned dataset, ready to import (adds `duration_minutes`, `duration_unit`, `primary_country`, `added_year`, and normalizes the 3 corrupt ratings like `"74 min"` → clean values) |
| `Netflix_BlackRed_Theme.json` | Power BI theme: black canvas, red data colors, white text |
| `Netflix_Content_Analysis_Dashboard.html` | Interactive HTML version of the dashboard (works by double-click — use it as the visual target while building) |

---

## 1. Import the data

1. Open **Power BI Desktop** → **Get Data → Text/CSV**.
2. Select `netflix_clean_for_powerbi.csv` → **Transform Data** (opens Power Query).
3. Confirm the table is named **`Netflix`** (rename in Query Settings if needed).
4. Set data types: `release_year` = Whole Number, `duration_minutes` = Whole Number (nullable), `added_year` = Whole Number. → **Close & Apply**.

> Prefer the raw `netflix_titles.csv` instead? In Power Query: promote headers, then fill `rating` and `country` nulls with `Unknown`, and add custom columns:
> - `duration_minutes` = `try Number.From(Text.BeforeDelimiter([duration], " ")) otherwise null`
> - `primary_country` = `Text.Trim(Text.Split([country], ","){0})` (after replacing nulls with `Unknown`)
> - `added_year` = `Date.Year([date_added])`

## 2. Create exploded tables for genres & countries (optional but recommended)

Titles can list several countries/genres. For accurate "Top 10" charts, create two queries:

- **Genres**: reference `Netflix` → keep `show_id` + `listed_in` → remove other columns → **Split Column by Delimiter** (`,` , each occurrence) → Trim → rename column `genre`.
- **Countries**: same for `country` → rename `country_name`.
- Model view: create **one-to-many** relationships `Netflix[show_id]` → `Genres[show_id]` and `Netflix[show_id]` → `Countries[show_id]` (single direction, Netflix on the 1-side). Slicers on `Netflix` will then cross-filter both charts.

## 3. DAX measures (Home → New Measure)

```dax
Total Titles = COUNTROWS ( Netflix )

Total Movies = CALCULATE ( [Total Titles], Netflix[type] = "Movie" )

Total TV Shows = CALCULATE ( [Total Titles], Netflix[type] = "TV Show" )

Avg Movie Duration (Min) =
AVERAGEX (
    FILTER (
        Netflix,
        Netflix[type] = "Movie"
            && NOT ISBLANK ( Netflix[duration_minutes] )
    ),
    Netflix[duration_minutes]
)

Pct Movies = DIVIDE ( [Total Movies], [Total Titles] )
```

For genre/country charts using the exploded tables:

```dax
Genre Titles   = COUNTROWS ( Genres )
Country Titles = COUNTROWS ( Countries )
```

## 4. Apply the Netflix theme

**View → Themes → dropdown arrow → Browse for themes…** → select `Netflix_BlackRed_Theme.json`.
This paints the page black (#0B0B0B), visuals dark (#151515), borders gray and all data colors in red shades.

## 5. KPI cards (4 Card visuals)

Place them side by side under the title. Format → Callout value ≈ 34 pt.

| Card | Field |
|---|---|
| Total Titles | `[Total Titles]` |
| Total Movies | `[Total Movies]` |
| Total TV Shows | `[Total TV Shows]` |
| Avg Movie Duration | `[Avg Movie Duration (Min)]` (add a custom label "min") |

## 6. The 7 visuals

1. **Movies vs TV Shows — Donut chart**: Legend = `type`, Values = `[Total Titles]`. Detail labels → Percent. Colors: Movie `#E50914`, TV Show `#8B0000`.
2. **Titles by Release Year — Line chart**: X = `release_year` (don't summarize), Y = `[Total Titles]`. Optional: add `[Total Movies]` and `[Total TV Shows]` as a second line chart next to it.
3. **Top 10 Countries — Horizontal bar**: Axis = `Countries[country_name]`, Values = `[Country Titles]`. Visual-level filter: `country_name is not Unknown` **and** Top N → Top 10 by Value. Sort descending.
4. **Top 10 Genres — Horizontal bar**: Axis = `Genres[genre]`, Values = `[Genre Titles]`, same Top 10 filter.
5. **Ratings — Column chart**: X = `rating`, Y = `[Total Titles]`, Top 10 filter by value, sorted desc.
6. **Ratings by Type — Stacked bar**: Y = `rating` (top 10 filter), Values = `[Total Movies]` + `[Total TV Shows]` (Movies = `#E50914`, TV = `#8B0000`).
7. **Movie Duration — Histogram**: Column chart, X = `duration_minutes` → right-click the field in the visual → **New group** → Bin size **10**; Y = `[Total Titles]`; visual-level filter `type is Movie`. Analytics pane → **X-axis constant line** = `[Avg Movie Duration (Min)]` (dashed, white) for the "≈100 min" marker.

## 7. Slicers

Add 4 **Dropdown slicers** across the top: `type`, `release_year`, `rating`, and `Countries[country_name]`.
Format → Slicer settings → Style = Dropdown; enable multi-select (Ctrl+Click) if wanted.

## 8. Layout polish (match the HTML version)

- Page: 16:9. Title text box top-left: **NETFLIX** in `#E50914` + "CONTENT ANALYSIS" in white, Segoe UI Bold.
- Snap all visuals to the grid, uniform 12–16 px gaps, rounded corners (theme sets radius 10).
- Add a **Key Findings** text box at the bottom with the six bullets (numbers below).
- Keep red accents for data only — background stays near-black.

## 9. Validate against the notebook (expected values)

| Metric | Expected |
|---|---|
| Total Titles | 8,807 |
| Movies | 6,131 (69.6%) |
| TV Shows | 2,676 (30.4%) |
| Avg movie duration | ≈ 99.6 min (median 98) |
| Top genre | International Movies — 2,752 |
| Top country | United States — ≈ 3,690 credits |
| Top rating | TV-MA — 3,207 (36.4%) |
| Peak release year | 2018 — 1,147 titles |
| TV Shows > Movies | 2021 (315 vs 277) |

## 10. Portfolio / GitHub tips

- Export a high-res screenshot (File → Export → PNG) for the README header.
- Commit: `netflix_titles.csv`, the `.pbix`, `Netflix_BlackRed_Theme.json`, this guide, and the notebook.
- Suggested README structure: objective → data source → cleaning steps (from your notebook) → dashboard screenshot → key findings → how to open the pbix.
