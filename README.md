# Recipe Scraper

> Scrape recipes from URLs, normalise into structured format, auto-calculate macros per serving using USDA food data, and save to Obsidian.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/Marrowleaf/hermes-recipe-scraper)

## Features

- Scrape recipes from major sites (BBC Good Food, Pinch of Nom, AllRecipes, etc.)
- JSON-LD schema detection with HTML fallback
- Automatic ingredient parsing and unit conversion
- USDA FoodData Central macro calculation per serving
- Offline food database fallback for common ingredients
- Scale recipes to different serving counts
- Search and filter saved recipes by macros (calories, protein)
- Save to Obsidian with full recipe template, macros, and scaling tables
- Tag-based recipe organization

## Installation

```bash
hermes skills install productivity/recipe-scraper
```

Or manually clone into `~/.hermes/skills/productivity/recipe-scraper/`.

## Usage

```
recipe-scraper scrape https://www.bbcgoodfood.com/recipes/chicken-stir-fry
recipe-scraper save https://www.bbcgoodfood.com/recipes/chicken-stir-fry --servings 2
recipe-scraper scale "Chicken Stir Fry" 4
recipe-scraper lookup "chicken breast" --amount "200g"
recipe-scraper list --tag high-protein
recipe-scraper search "pasta" --max-calories 500 --min-protein 30
```

## Configuration

- `USDA_API_KEY`: Optional — get from [USDA FoodData Central](https://fdc.nal.usda.gov/api-key-signup/). If not set, uses offline food database (~60 common items)
- Recipes saved to `~/obsidian-vault/3-Resources/Recipes/`
- Daily macro targets: 1,930 kcal, 162g protein (configurable in template)

## Requirements

- Hermes Agent v0.12+
- Browser tools (for scraping recipe sites)
- Obsidian vault (for recipe storage)

## License

MIT