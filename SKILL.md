---
name: recipe-scraper
description: "Scrape recipes from URLs, normalise into structured format, auto-calculate macros per serving using USDA food data."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [recipe, scraper, macros, usda, nutrition, cooking, meal, obsidian]
    related_skills: [meal-planner, fitness-nutrition, ocr-and-documents]
commands:
  - name: scrape
    description: "Scrape a recipe from a URL and return structured data."
    usage: recipe-scraper scrape <url>
    examples:
      - "recipe-scraper scrape https://www.bbcgoodfood.com/recipes/chicken-stir-fry"
      - "scrape this recipe: https://pinchofnom.com/syn-free-chicken-jalfrezi/"
  - name: save
    description: "Save a scraped recipe to Obsidian with macro calculations."
    usage: recipe-scraper save <url> [--servings <n>]
    examples:
      - "recipe-scraper save https://www.bbcgoodfood.com/recipes/chicken-stir-fry --servings 2"
  - name: scale
    description: "Scale a saved recipe to a different number of servings."
    usage: recipe-scraper scale <recipe-name> <new-servings>
    examples:
      - "recipe-scaler scale 'Chicken Stir Fry' 4"
  - name: lookup
    description: "Look up nutritional data for a single ingredient from USDA."
    usage: recipe-scraper lookup <ingredient> [--amount <qty>]
    examples:
      - "recipe-scraper lookup 'chicken breast' --amount '200g'"
  - name: list
    description: "List all saved recipes in Obsidian."
    usage: recipe-scraper list [--tag <tag>]
  - name: search
    description: "Search saved recipes by name, tag, or macro criteria."
    usage: recipe-scraper search <query> [--max-calories <n>] [--min-protein <n>]
---

# Recipe Scraper

Scrape recipes from web URLs, normalise into structured format, auto-calculate macros per serving using USDA FoodData Central, and save to Obsidian with a clean recipe template.

## When to Use This Skill

- User shares a recipe URL and wants it saved
- User asks "how many calories in this recipe?"
- User wants to scale a recipe for more/fewer servings
- User asks to look up macros for a single ingredient
- User wants to find saved recipes under a calorie limit

## Step 1: Scrape Recipe from URL

Use browser tools to navigate to the recipe URL and extract:

1. **Title** — from `<h1>` or structured data (JSON-LD `Recipe` schema)
2. **Ingredients** — list of items with quantities
3. **Instructions** — numbered steps
4. **Servings** — yields or serving count
5. **Prep time** — if available
6. **Cook time** — if available
7. **Tags** — cuisine type, dietary labels (vegetarian, gluten-free, etc.)

### Scraping Strategy

```
1. Navigate to URL with browser_navigate
2. Check for JSON-LD Recipe schema first (most reliable):
   browser_console(expression="JSON.parse(document.querySelector('script[type=\"application/ld+json\"]').textContent)")
   - Extract: name, recipeIngredient, recipeInstructions, recipeYield, nutrition
3. If no JSON-LD, fall back to HTML scraping:
   - Title: document.querySelector('h1')?.textContent
   - Ingredients: look for common patterns (.ingredients li, .recipe-ingredients li)
   - Steps: look for (.method li, .recipe-method li, .instructions ol li)
4. If structured data has nutrition per serving, use it directly
5. If no nutrition data, proceed to Step 2 (macro calculation)
```

### Supported Sites

Tested with structured data on:
- BBC Good Food (bbcgoodfood.com)
- Pinch of Nom (pinchofnom.com)
- AllRecipes (allrecipes.com)
- Jamie Oliver (jamieoliver.com)
- BBC Food (bbc.co.uk/food)
- MyFitnessPal recipes (myfitnesspal.com)

For sites without JSON-LD, the HTML fallback may need site-specific selectors.

## Step 2: Parse Ingredients

Normalize ingredients into a standard format:

| Raw Input | Parsed |
|-----------|--------|
| "2 chicken breasts" | {qty: 2, unit: "breast", item: "chicken", grams: ~300} |
| "1 tbsp olive oil" | {qty: 1, unit: "tbsp", item: "olive oil", grams: 14} |
| "200g spaghetti" | {qty: 200, unit: "g", item: "spaghetti", grams: 200} |
| "a pinch of salt" | {qty: 1, unit: "pinch", item: "salt", grams: 1} |
| "1 can chopped tomatoes (400g)" | {qty: 1, unit: "can", item: "chopped tomatoes", grams: 400} |

### Common Conversions

```python
UNIT_TO_GRAMS = {
    # Liquids (water-based)
    "tsp": 5, "tbsp": 15, "cup": 240, "ml": 1,
    # Weight
    "g": 1, "oz": 28, "lb": 454, "kg": 1000,
    # Approximate
    "can": 400, "clove": 5, "slice": 30, "handful": 30,
    "breast": 150, "thigh": 120, "fillet": 150,
    "stalk": 80, "sprig": 3, "pinch": 1,
    "small": 100, "medium": 150, "large": 200,
}
```

## Step 3: Calculate Macros

For each ingredient, query USDA FoodData Central:

```bash
# Search for ingredient
curl -s "https://api.nal.usda.gov/fdc/v1/foods/search?query=chicken+breast&dataType=Foundation,SR Legacy&pageSize=1&api_key=${USDA_API_KEY}"

# Get detailed nutrition
curl -s "https://api.nal.usda.gov/fdc/v1/food/{fdcId}?nutrients=1008,1003,1005,1099&api_key=${USDA_API_KEY}"
```

### Nutrient IDs

| ID | Nutrient | Unit |
|----|----------|------|
| 1008 | Energy | kcal |
| 1003 | Protein | g |
| 1005 | Carbohydrate | g |
| 1004 | Total fat | g |
| 1079 | Fiber | g |
| 2000 | Sugar | g |
| 1099 | Sodium | mg |

### Fallback Database

If USDA_API_KEY is not set or API is unavailable, use the offline food database from the meal-planner skill at `~/.hermes/skills/health/meal-planner/references/food-database.md`.

### Aggregation

```python
# Sum all ingredients, then divide by servings
total_macros = {
    "calories": sum(ingredient.calories for ingredient in parsed_ingredients),
    "protein": sum(ingredient.protein for ingredient in parsed_ingredients),
    "carbs": sum(ingredient.carbs for ingredient in parsed_ingredients),
    "fat": sum(ingredient.fat for ingredient in parsed_ingredients),
    "fiber": sum(ingredient.fiber for ingredient in parsed_ingredients),
}

per_serving = {k: round(v / servings, 1) for k, v in total_macros.items()}
```

## Step 4: Save to Obsidian

Save recipes to `~/obsidian-vault/3-Resources/Recipes/`:

```markdown
---
type: recipe
title: "{{TITLE}}"
source: "{{URL}}"
servings: {{SERVINGS}}
prep_time: "{{PREP_TIME}}"
cook_time: "{{COOK_TIME}}"
tags: [{{TAGS}}]
date_scraped: {{DATE}}
calories_per_serving: {{CAL}}
protein_per_serving: {{PROTEIN}}
carbs_per_serving: {{CARBS}}
fat_per_serving: {{FAT}}
---

# {{TITLE}}

**Source:** [{{DOMAIN}}]({{URL}})
**Servings:** {{SERVINGS}} | **Prep:** {{PREP_TIME}} | **Cook:** {{COOK_TIME}}

## Macros per Serving

| Nutrient | Amount | % Daily Target* |
|----------|--------|-----------------|
| Calories | {{CAL}} kcal | {{CAL/1930*100}}% |
| Protein | {{PROTEIN}}g | {{PROTEIN/162*100}}% |
| Carbs | {{CARBS}}g | — |
| Fat | {{FAT}}g | — |
| Fiber | {{FIBER}}g | — |

*\*Based on 1,930 kcal, 162g protein daily targets*

## Ingredients

{{INGREDIENT_LIST}}

## Instructions

{{NUMBERED_STEPS}}

## Scaling

| Servings | Calories | Protein | Carbs | Fat |
|----------|----------|---------|-------|-----|
| 1 | {{CAL}} | {{PROTEIN}}g | {{CARBS}}g | {{FAT}}g |
| 2 | {{CAL*2}} | {{PROTEIN*2}}g | {{CARBS*2}}g | {{FAT*2}}g |
| 4 | {{CAL*4}} | {{PROTEIN*4}}g | {{CARBS*4}}g | {{FAT*4}}g |
```

## Step 5: Scale Recipe

When the user asks to scale a saved recipe:

1. Find the recipe in `3-Resources/Recipes/` by name (fuzzy match)
2. Read the frontmatter for `servings`, macros per serving
3. Calculate: `new_macros = per_serving_macros * new_servings`
4. Display scaled ingredient list and updated macro table
5. Optionally update the file with new scaling row

## Step 6: Search and Filter

```bash
# List all recipes
ls ~/obsidian-vault/3-Resources/Recipes/

# Search by tag
grep -rl "tags:.*high-protein" ~/obsidian-vault/3-Resources/Recipes/

# Filter by macros (using frontmatter)
for f in ~/obsidian-vault/3-Resources/Recipes/*.md; do
  cal=$(grep "calories_per_serving" "$f" | cut -d: -f2 | tr -d ' ')
  prot=$(grep "protein_per_serving" "$f" | cut -d: -f2 | tr -d ' ')
  if [ "$cal" -lt 500 ] && [ "${prot%.*}" -gt 30 ]; then
    echo "$f: ${cal}kcal, ${prot}g protein"
  fi
done
```

## Common Pitfalls

1. **USDA API key** — If `USDA_API_KEY` is not in `.env`, macro calculations fall back to the offline database which has ~60 common foods. Not all ingredients will be found.
2. **Unit ambiguity** — "1 can" varies by brand. Prefer scraping recipes that list weights. When ambiguous, note the assumption.
3. **Serving size inconsistency** — Some sites list servings as "serves 4", others as "per portion". Always verify the yield field.
4. **Site-specific scrapers break** — Websites change their HTML structure. JSON-LD is more reliable than CSS selectors. Always try JSON-LD first.
5. **Recipe doubles** — Before saving, check if the recipe already exists in `3-Resources/Recipes/` to avoid duplicates.
6. **Ingredient parsing errors** — Complex instructions like "1 cup carrots, diced" should parse as 1 cup carrots, not "1 cup carrots, diced". Strip preparation instructions after commas.
7. **Browser blocking** — Some recipe sites block headless browsers. Use `browser_vision` to detect CAPTCHAs and inform the user.
8. **Macro rounding** — Always round macros to 1 decimal place. Don't present false precision (342.3333 kcal → 342.3 kcal).

## Verification

After scraping and saving a recipe:
1. Open the saved file: `cat ~/obsidian-vault/3-Resources/Recipes/<recipe>.md`
2. Verify frontmatter has all macro fields populated
3. Verify total macros are reasonable (main dish: 300-800 kcal/serving)
4. Verify the ingredient list matches what was on the website
5. Verify servings number is correct
6. Check that wiki-links work in Obsidian