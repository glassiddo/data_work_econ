# Targeted R Checks

Use small diagnostics near risky code. Adapt object names and syntax to the project. Do not paste every check into every review; choose the check that matches the risk and report the result in researcher-facing language.

## Unit and Duplicate Keys

```r
key_check <- cleaned |>
  summarise(n_rows = n(), .by = c(iso3, year)) |>
  filter(n_rows > 1) |>
  arrange(desc(n_rows), iso3, year)

key_check
```

Use this whenever the intended unit should be unique, changing `iso3, year` to the project key.

## Join Row Counts, Duplicates, and Unmatched Keys

```r
n_before <- nrow(master)

joined <- master |>
  left_join(add_on, join_by(iso3, year), relationship = "many-to-one")

tibble::tibble(
  rows_before = n_before,
  rows_after = nrow(joined),
  row_change = nrow(joined) - n_before
)

dupe_keys <- master |>
  summarise(master_rows = n(), .by = c(iso3, year)) |>
  full_join(
    add_on |> summarise(add_on_rows = n(), .by = c(iso3, year)),
    join_by(iso3, year)
  ) |>
  filter(coalesce(master_rows, 0) > 1 | coalesce(add_on_rows, 0) > 1)

unmatched_master <- master |>
  anti_join(add_on, join_by(iso3, year)) |>
  distinct(iso3, country_name, year) |>
  arrange(country_name, year) |>
  slice_head(n = 30)

master_keys <- master |>
  distinct(iso3, year) |>
  mutate(in_master = TRUE)

add_on_keys <- add_on |>
  distinct(iso3, year) |>
  mutate(in_add_on = TRUE)

master_keys |>
  full_join(add_on_keys, join_by(iso3, year)) |>
  mutate(
    in_master = replace_na(in_master, FALSE),
    in_add_on = replace_na(in_add_on, FALSE)
  ) |>
  count(in_master, in_add_on)
```

If the join is intentionally many-to-many, explain the row grain it creates. If not, collapse or filter the added data before joining.

## Identifier Conversion and Label Drift

```r
conversion_failures <- country_panel |>
  mutate(iso2_new = countrycode::countrycode(iso3, "iso3c", "iso2c", warn = FALSE)) |>
  filter(is.na(iso2_new)) |>
  distinct(iso3, country_name) |>
  arrange(country_name)

label_drift <- joined |>
  filter(!is.na(country_name.x), !is.na(country_name.y), country_name.x != country_name.y) |>
  distinct(iso3, country_name.x, country_name.y) |>
  arrange(iso3)
```

Use this pattern for country codes, admin IDs, survey codes, commodity codes, and any crosswalk. Inspect failures before adding manual fixes.

## Reshape Keys

```r
long_data |>
  summarise(n = n(), .by = c(iso3, year, variable_name)) |>
  filter(n > 1) |>
  arrange(desc(n), iso3, year)
```

Run before `pivot_wider()`. Also inspect selected columns for `pivot_longer()`, especially year patterns and columns that encode source, unit, scenario, or statistic.

## Missingness Introduced by Cleaning

```r
bind_rows(
  raw |> summarise(stage = "raw", missing = sum(is.na(treatment_raw))),
  cleaned |> summarise(stage = "cleaned", missing = sum(is.na(treatment)))
)

joined |>
  summarise(
    rows = n(),
    missing_added = sum(is.na(gdp_b)),
    missing_added_share = mean(is.na(gdp_b))
  )
```

Use separate checks for missingness caused by joins, recodes, parsing, and denominators when those mechanisms differ.

## Recode Coverage and Examples

```r
raw_values <- raw |>
  distinct(treatment_raw, treatment_label)

raw_values |>
  anti_join(recode_table, join_by(treatment_raw == source_code)) |>
  arrange(treatment_raw)

cleaned |>
  arrange(treatment, id) |>
  slice_head(n = 3, by = treatment) |>
  select(id, treatment_raw, treatment_label, treatment)
```

If no codebook exists, say this checks observed coverage only, not source-defined validity.

## Aggregation With Observed Input Counts

```r
panel |>
  summarise(
    total = if (all(is.na(value))) NA_real_ else sum(value, na.rm = TRUE),
    n_input = n(),
    n_observed = sum(!is.na(value)),
    .by = c(iso3, year)
  )
```

This follows a conservative "NA only if all inputs are missing" rule: use observed values when at least one exists, but keep fully missing groups as missing. Change the rule only when the source meaning or research design supports it.

## Aggregation Choice and Components

```r
micro |>
  summarise(
    unweighted_rate = mean(rate, na.rm = TRUE),
    weighted_rate = weighted.mean(rate, population, na.rm = TRUE),
    total_population = sum(population, na.rm = TRUE),
    .by = region
  )

cleaned |>
  arrange(desc(gdp_pc)) |>
  select(iso3, country_name, year, gdp_b, pop_m, gdp_pc, source_file) |>
  slice_head(n = 15)
```

For constructed variables, inspect the components, not only the final value. Large or impossible values often come from units, denominators, duplicate rows, failed joins, or real edge cases.

## Spatial Match Quality

```r
matched_points <- points |>
  st_join(polygons, join = st_within)

matched_points |>
  st_drop_geometry() |>
  summarise(
    n_points = n(),
    n_unmatched = sum(is.na(unit_id)),
    share_unmatched = mean(is.na(unit_id)),
    .by = source_file
  )
```

```r
matched_points |>
  filter(is.na(unit_id)) |>
  mutate(
    x = st_coordinates(geometry)[, 1],
    y = st_coordinates(geometry)[, 2]
  ) |>
  st_drop_geometry() |>
  select(point_id, x, y, expected_country, source_file) |>
  slice_head(n = 20)
```

Also inspect examples that matched the wrong nearby unit when expected geography labels are available; unmatched counts alone are not enough.

## Spatial Overlaps and Area Shares

```r
overlap <- units |>
  st_intersection(polygons) |>
  mutate(overlap_area_km2 = as.numeric(st_area(st_transform(geometry, 6933))) / 1e6)

overlap |>
  st_drop_geometry() |>
  summarise(
    n_pieces = n(),
    overlap_area_km2 = sum(overlap_area_km2),
    .by = unit_id
  )
```

```r
area_by_unit |>
  left_join(units |> st_drop_geometry(), join_by(unit_id)) |>
  mutate(raw_share = matched_area_km2 / unit_area_km2) |>
  filter(raw_share > 1) |>
  arrange(desc(raw_share))
```

If shares exceed one, possible causes include overlapping source polygons, duplicated intersections, CRS/area mistakes, or denominator mismatch.

## Raster Extraction

Before accepting a raster extraction, confirm what cell values mean. Counts, averages, categories, probabilities, and indexes require different summaries. Do not copy a summary function from another raster task until the variable meaning is clear.

Useful checks include:

- Compare exact, weighted, and default extraction only after choosing the correct summary statistic.
- Inspect units with the largest differences across extraction methods.
- Check whether partial cells are handled in a way that matches the intended measure.
- For counts, verify whether cell values should be summed, area-weighted, or normalized before aggregation.
- For categories, verify whether the intended polygon value is majority, any presence, modal class, shares by class, or something else.
