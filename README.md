# Food Delivery & Restaurant Analytics – Power BI & DAX

This project is an exploratory **Power BI dashboard** on restaurant and food delivery data in Belgium.

It combines:

- A rich **DAX model** for:
  - Rating normalization (e.g. adjusted ratings by number of reviews),
  - Category-level price analysis,
  - Detection of niche dishes (e.g. **Kapsalon**, **Hummus/Falafel**, **Vegetarian/Vegan**),
  - Geographic aggregations by **city** and **province**.

- Interactive **Power BI visuals** answering questions such as:
  - Which cuisines are most expensive on average?
  - Which categories (pizza, burgers, sushi, etc.) have the most menu items?
  - Where are restaurants and vegetarian/vegan options most concentrated?
  - Which pizza restaurants or hummus places are top rated?
  - How does **price-to-rating** compare between restaurants?

---

## Contents

- `DAX exported.xlsx` – export of all key **DAX measures and calculated columns**.
- `Visuals_Food_Delivery_Emmanuel.pdf` – screenshots of the main **Power BI visuals**.
- (Optionally) `Food_Delivery.pbix` – the Power BI Desktop file (if you include it in the repo).

---

## 1. Data Model

The model is centered around **restaurants**, their **menu items**, their **categories**, and their **locations**.

### 1.1 Main Tables

- `restaurants`  
  - `restaurant_id`, `primarySlug`, `name`, `city`, `ratings`, `ratingsNumber`

- `menuItems`  
  - `id`, `price`, `primarySlug`, `IsKapsalonCategory` (calculated)

- `categories`  
  - `item_id`, `name`, `IsVegy` (calculated)

- `categories_restaurants`  
  - `restaurant_id`, `category_id`, `Category_short`, `Rating_Hummus`,  
    `AdjustedRating_Hummus`, `AvgAdjRatingHummus`, `HasHummus`

- `locations_to_restaurants`  
  - `restaurant_id`, `location_id`

- `locations`  
  - `ID`, `postalCode`

- `'belgian-cities-geocoded (2)'`  
  - `postal`, `province`

- `CityRestaurantCounts`  
  - `RestaurantCount`

- `PizzaRestaurants`  
  - `ratings` (filtered subset of pizza venues)

These tables allow us to:

- Tie restaurants to **menu items** and **food categories**,
- Map restaurants to **cities** and **provinces** through postal codes,
- Build derived metrics (e.g. “restaurant density”, “avg Kapsalon price per province”).

---

## 2. DAX – Calculated Columns

### 2.1 Restaurant Table

**`AdjustedRating`**

Adjusts the average rating by the **number of ratings**, so popular restaurants get a small boost:

```DAX
AdjustedRating =
IF(
    ISNUMBER([ratingsNumber]) && [ratingsNumber] > 0,
    [Average_Rating] + (LN([ratingsNumber]) / 20),
    BLANK()
)
Province
Maps each restaurant to a Belgian province using location and postal code:
DAX


Province =
VAR locationId =
    MINX(
        FILTER(
            locations_to_restaurants,
            locations_to_restaurants[restaurant_id] = restaurants[primarySlug]
        ),
        locations_to_restaurants[location_id]
    )
VAR postalCode =
    MINX(
        FILTER(
            locations,
            locations[ID] = locationId
        ),
        locations[postalCode]
    )
RETURN
    LOOKUPVALUE(
        'belgian-cities-geocoded (2)'[province],
        'belgian-cities-geocoded (2)'[postal], postalCode
    )
________________________________________
2.2 Menu Items
IsKapsalonCategory
Flags menu items that are a Kapsalon (based on category name):
DAX


IsKapsalonCategory =
IF (
    COUNTROWS(
        FILTER(
            categories,
            categories[item_id] = menuItems[id]
                && CONTAINSSTRING(LOWER(categories[name]), "kapsalon")
        )
    ) > 0,
    TRUE(),
    FALSE()
)
________________________________________
2.3 Categories / Categories_Restaurants
Category_short (in categories_restaurants)
Cleans and normalizes category identifiers into a short, human-readable category:
DAX


Category_short =
VAR catid = [category_id]
VAR NameOnly = LEFT(catid, FIND("_", catid & "_") - 1)
VAR LowerName = LOWER(NameOnly)
RETURN
SWITCH(
    TRUE(),
    // Numbers or known numeric-like codes -> "Other"
    NOT ISERROR(VALUE(NameOnly)), "Other",
    NameOnly = "2600", "Other",
    NameOnly = "701",  "Other",

    // Halal
    CONTAINSSTRING(LowerName, "halal"), "Halal",

    // Turkish (e.g. gyros)
    CONTAINSSTRING(LowerName, "gyros")
        || CONTAINSSTRING(LowerName, "turkish"), "Turkish",

    // Indian
    CONTAINSSTRING(LowerName, "curry"), "Indian",

    // Default
    NameOnly
)
HasHummus (in categories_restaurants)
Identifies restaurants that are very likely to serve hummus/falafel:
DAX


HasHummus =
VAR CategoryShort = LOWER([Category_short])
VAR RestaurantIdText = LOWER([restaurant_id])
RETURN
    (CategoryShort = "falafel" || CategoryShort = "lebanese")
        && (
            CONTAINSSTRING(RestaurantIdText, "humus")
            || CONTAINSSTRING(RestaurantIdText, "hummus")
            || CONTAINSSTRING(RestaurantIdText, "falafel")
        )
IsVegy (in categories)
Flags vegetarian/vegan options:
DAX


IsVegy =
IF(
    CONTAINSSTRING(LOWER(categories[name]), "vegetarian")
        || CONTAINSSTRING(LOWER(categories[name]), "vegan"),
    TRUE,
    FALSE
)
________________________________________
3. DAX – Measures
Below is a selection of key measures used to drive the visuals.
3.1 Hummus Ratings
AvgRatingHummus
DAX


AvgRatingHummus =
AVERAGE(categories_restaurants[Rating_Hummus])
AvgAdjRatingHummus
DAX


AvgAdjRatingHummus =
AVERAGE(categories_restaurants[AdjustedRating_Hummus])
Norm_Avg_Adj_Rating
Caps the adjusted rating at 5.0:
DAX


Norm_Avg_Adj_Rating =
IF(
    categories_restaurants[AvgAdjRatingHummus] > 5,
    5,
    categories_restaurants[AvgAdjRatingHummus]
)
NormalizedRating_Hummus_Measure
Normalizes adjusted hummus ratings to a 0–5 scale relative to the original rating:
DAX


NormalizedRating_Hummus_Measure =
VAR Adjusted =
    CALCULATE(
        MAX(categories_restaurants[AdjustedRating_Hummus])
    )
VAR Rating =
    CALCULATE(
        MAX(categories_restaurants[Rating_Hummus])
    )
RETURN
IF(
    NOT ISBLANK(Adjusted)
        && NOT ISBLANK(Rating)
        && Rating <> 0,
    (Adjusted / Rating) * 5,
    BLANK()
)
________________________________________
3.2 Category Prices & Menu Items
AveragePricePerCategory
Average menu price per restaurant/category combination, using TREATAS:
DAX


AveragePricePerCategory =
CALCULATE(
    AVERAGE(menuItems[price]),
    TREATAS(
        VALUES(categories_restaurants[restaurant_id]),
        restaurants[primarySlug]
    ),
    TREATAS(
        VALUES(categories_restaurants[Category_short]),
        categories_restaurants[Category_short]
    )
)
MenuItemCountPerCategory
Counts how many menu items exist for each category:
DAX


MenuItemCountPerCategory =
CALCULATE(
    COUNT(menuItems[id]),
    TREATAS(
        VALUES(categories_restaurants[restaurant_id]),
        restaurants[primarySlug]
    ),
    TREATAS(
        VALUES(categories_restaurants[Category_short]),
        categories_restaurants[Category_short]
    )
)
________________________________________
3.3 Kapsalon Metrics
KapsalonRestaurantCountByCity
Counts restaurants in a city that have at least one Kapsalon item:
DAX


KapsalonRestaurantCountByCity =
CALCULATE(
    DISTINCTCOUNT(restaurants[restaurant_id]),
    FILTER(
        menuItems,
        COUNTROWS(
            FILTER(
                categories,
                categories[item_id] = menuItems[id]
                    && CONTAINSSTRING(LOWER(categories[name]), "kapsalon")
            )
        ) > 0
    )
)
AvgPriceKapsalon
Rounded average price of Kapsalon items:
DAX


AvgPriceKapsalon =
ROUND(
    CALCULATE(
        AVERAGE(menuItems[price]),
        menuItems[IsKapsalonCategory] = TRUE()
    ),
    1
)
________________________________________
3.4 Vegetarian / Vegan Metrics
VegyRestaurantCount
Distinct restaurants (by primarySlug) offering vegetarian/vegan dishes:
DAX


VegyRestaurantCount =
CALCULATE(
    DISTINCTCOUNT(menuItems[primarySlug]),
    FILTER(
        menuItems,
        LOOKUPVALUE(
            categories[IsVegy],
            categories[item_id], menuItems[id]
        ) = TRUE()
    )
)
VegyRestaurantsByCity
Counts vegetarian/vegan restaurants per city, excluding all Brussels communes:
DAX


VegyRestaurantsByCity =
CALCULATE(
    DISTINCTCOUNT(categories[restaurant_id]),
    FILTER(categories, categories[IsVegy] = TRUE()),
    NOT restaurants[city] IN {
        "Anderlecht", "Auderghem", "Berchem-Sainte-Agathe", "Bruxelles",
        "Etterbeek", "Evere", "Forest", "Ganshoren", "Haren", "Ixelles",
        "Jette", "Koekelberg", "Laeken", "Molenbeek-Saint-Jean",
        "Neder-Over-Heembeek", "Saint-Gilles", "Saint-Josse-Ten-Noode",
        "Schaerbeek", "Uccle", "Watermael-Boitsfort",
        "Woluwe-Saint-Lambert", "Woluwe-Saint-Pierre"
    }
)
________________________________________
3.5 Pizza & Ratings
AvgRating / Average_Rating
DAX


AvgRating =
AVERAGE(restaurants[ratings])

Average_Rating =
AVERAGE(restaurants[ratings])
AvgRatingPizza (on PizzaRestaurants subset)
DAX


AvgRatingPizza =
AVERAGE(PizzaRestaurants[ratings])
RestaurantRating
DAX


RestaurantRating =
MAX(restaurants[ratings])
IsItalianPizza
Flags restaurants that serve Italian pizza:
DAX


IsItalianPizza =
IF (
    COUNTROWS(
        FILTER(
            categories_restaurants,
            categories_restaurants[restaurant_id] = SELECTEDVALUE(restaurants[restaurant_id])
                && categories_restaurants[Category_short] = "italian-pizza"
        )
    ) > 0,
    1,
    0
)
ItalianPizzaRating
DAX


ItalianPizzaRating =
IF(
    [IsItalianPizza] = 1,
    MAX(restaurants[ratings])
)
ItalianPizzaRank
Ranks Italian pizza restaurants by rating:
DAX


ItalianPizzaRank =
IF(
    [IsItalianPizza] = 1,
    RANKX(
        FILTER(ALL(restaurants), CALCULATE([IsItalianPizza]) = 1),
        CALCULATE(MAX(restaurants[ratings])),
        ,
        DESC,
        DENSE
    )
)
TopRating_ItalianPizza
Highest rating among all Italian pizza restaurants:
DAX


TopRating_ItalianPizza =
CALCULATE(
    MAX(restaurants[ratings]),
    FILTER(
        restaurants,
        restaurants[restaurant_id] IN
            SELECTCOLUMNS(
                FILTER(
                    categories_restaurants,
                    categories_restaurants[Category_short] = "italian-pizza"
                ),
                "restaurant_id", categories_restaurants[restaurant_id]
            )
    )
)
TopRating_AmericanPizza
Equivalent measure for American pizza:
DAX


TopRating_AmericanPizza =
CALCULATE(
    MAX(restaurants[ratings]),
    TREATAS(
        VALUES(categories_restaurants[restaurant_id]),
        restaurants[primarySlug]
    ),
    FILTER(
        categories_restaurants,
        categories_restaurants[Category_short] = "american-pizza"
    )
)
________________________________________
3.6 Price vs Rating and Density
PriceToRatingRatio
Compares average price to average rating (excluding Brussels communes):
DAX


PriceToRatingRatio =
VAR ExcludedCities = {
    "Anderlecht", "Auderghem", "Berchem-Sainte-Agathe", "Bruxelles",
    "Etterbeek", "Evere", "Forest", "Ganshoren", "Haren", "Ixelles",
    "Jette", "Koekelberg", "Laeken", "Molenbeek-Saint-Jean",
    "Neder-Over-Heembeek", "Saint-Gilles", "Saint-Josse-Ten-Noode",
    "Schaerbeek", "Uccle", "Watermael-Boitsfort",
    "Woluwe-Saint-Lambert", "Woluwe-Saint-Pierre"
}
RETURN
DIVIDE(
    CALCULATE(
        AVERAGE(menuItems[price]),
        NOT (restaurants[City] IN ExcludedCities)
    ),
    CALCULATE(
        AVERAGE(restaurants[ratings]),
        NOT (restaurants[City] IN ExcludedCities)
    ),
    BLANK()
)
RestaurantCountByProvince
Counts restaurants per province using postal code + province mapping and excluding Brussels communes:
DAX


RestaurantCountByProvince =
VAR ExcludedCities = {
    "Anderlecht", "Auderghem", "Berchem-Sainte-Agathe", "Bruxelles",
    "Etterbeek", "Evere", "Forest", "Ganshoren", "Haren", "Ixelles",
    "Jette", "Koekelberg", "Laeken", "Molenbeek-Saint-Jean",
    "Neder-Over-Heembeek", "Saint-Gilles", "Saint-Josse-Ten-Noode",
    "Schaerbeek", "Uccle", "Watermael-Boitsfort",
    "Woluwe-Saint-Lambert", "Woluwe-Saint-Pierre"
}
RETURN
CALCULATE(
    COUNTROWS(restaurants),
    TREATAS(
        VALUES('belgian-cities-geocoded (2)'[postal]),
        locations[postalCode]
    ),
    FILTER(restaurants, NOT (restaurants[city] IN ExcludedCities))
)
Restaurant Density
Currently implemented as a simple restaurant count (but can be extended to per km² or per 1k inhabitants):
DAX


Restaurant Density =
VAR ExcludedCities = {
    "Anderlecht", "Auderghem", "Berchem-Sainte-Agathe", "Bruxelles",
    "Etterbeek", "Evere", "Forest", "Ganshoren", "Haren", "Ixelles",
    "Jette", "Koekelberg", "Laeken", "Molenbeek-Saint-Jean",
    "Neder-Over-Heembeek", "Saint-Gilles", "Saint-Josse-Ten-Noode",
    "Schaerbeek", "Uccle", "Watermael-Boitsfort",
    "Woluwe-Saint-Lambert", "Woluwe-Saint-Pierre"
}
RETURN
    COUNTROWS(restaurants)
________________________________________
4. Main Visuals & Insights
The following visuals are documented in Visuals_Food_Delivery_Emmanuel.pdf and implemented in the Power BI report.
4.1 Average Price per Category (Cuisine)
Visual: Column chart – Average price per primary cuisine.
Examples (rounded):
•	French: ~15.73
•	Kosher: ~13.83
•	Greek: ~13.67
•	Lebanese: ~13.26
•	Fish: ~13.15
•	Thai: ~13.10
•	Italian: ~12.67
•	…
•	Total average across all categories: ~10.06
This highlights the most expensive cuisines on average (French, Kosher, Greek).
________________________________________
4.2 Menu Item Distribution by Category
Visual: Column chart – No Menu Items per Category.
Top categories by number of items:
•	italian pizza: 132K
•	pita kebab: 106K
•	pasta: 81K
•	burgers: 53K
•	snacks: 47K
•	sushi: 38K
•	fries: 33K
•	american pizza: 33K
•	sandwiches: 32K
•	wok: 28K
•	… and many more down to ~2–3K.
This shows where menu variety is highest.
________________________________________
4.3 Number of Restaurants per City / Province
Visuals:
•	Map: Number of restaurants per city.
•	Bar chart: RestaurantCountByProvince by Province.
Example province counts (approx):
•	Antwerpen: 1080
•	Oost Vlaanderen: 687
•	Vlaams Brabant: 536
•	West Vlaanderen: 419
•	Limburg: 275
This reveals the geographic distribution and concentration of restaurants.
________________________________________
4.4 Top 10 Pizza Restaurants
Visual: Bar chart – Top 10 Pizza Restaurants by Rating.
Examples:
•	Pizza Di Trevi
•	Sim Pizza
•	Pizza Phone
•	Kingslize Pizza
•	Pitza Service
•	Pizza Service
•	Pizza Talia Original
•	Pizza Company Antwerpen
•	Domino’s Pizza
•	Pizza Hut Delivery
Ratings are shown on a 0–5 scale, driven by the pizza-focused measures (AvgRatingPizza, ItalianPizzaRating, etc.).
________________________________________
4.5 Kapsalon – Average Price & Distribution
Visuals:
•	Map: KapsalonRestaurantCountByCity.
•	Table: Average price of restaurants offering Kapsalon by province:
o	Limburg: 12.10
o	Antwerpen: 11.20
o	Vlaams Brabant: 11.10
o	Oost Vlaanderen: 11.00
o	West Vlaanderen: 10.90
o	Total: 11.20
These leverage IsKapsalonCategory, AvgPriceKapsalon, and KapsalonRestaurantCountByCity.
________________________________________
4.6 Best Price-to-Rating Ratio
Visual: Bar chart – Top 10 restaurants with the best value (low PriceToRatingRatio).
Examples:
•	Frituur Amigos
•	Frituur ’t Krokantje
•	QueTacos
•	Frituur op’t Hoekske
•	Frituur 4 You
•	JD Corner
•	Say Pasta
•	Happy Fries
•	Street’Tacos
•	Frituur chef Bruno
Ratios in the range ~0.56 – 0.62 (price per rating point).
________________________________________
4.7 Restaurant Density by City
Visual: Map with categories such as:
•	No restaurant
•	1–2 restaurants
•	3–5 restaurants
•	5+ restaurants
This uses the Restaurant Density and count measures to show concentration of restaurants per city.
________________________________________
4.8 Vegetarian & Vegan Restaurants
Visuals:
•	Map: Vegetarian/Vegan restaurants per city (VegyRestaurantsByCity).
•	Bar chart: Vegetarian/Vegan restaurants per province:
o	Antwerpen
o	Oost Vlaanderen
o	West Vlaanderen
o	Vlaams Brabant
o	Limburg
Powered by IsVegy, VegyRestaurantCount, and VegyRestaurantsByCity.
________________________________________
4.9 Top 3 Hummus / Falafel Restaurants
Visual: Bar chart – Top 3 Best Serving Hummus by rating.
Examples:
•	falafel-top-leuven
•	falafel-king-1
•	falafel-gent
Uses the Hummus-related measures and HasHummus flag.
________________________________________
5. How to Run / Use
1.	Open the .pbix file in Power BI Desktop (if included).
2.	Make sure the data sources are in place (CSV/DB connections as used for the export).
3.	Review and, if needed, adapt:
o	The geocoding / province mapping (table 'belgian-cities-geocoded (2)').
o	The filtering of Brussels communes, which is hard-coded in several measures.
4.	Explore the report pages:
o	Category price & item distribution
o	Geographic views (restaurants, veg/vegan, Kapsalon)
o	Pizza rankings
o	Hummus and vegan insights
o	Price-to-rating analysis
________________________________________
6. Possible Extensions
•	Replace the hard-coded Brussels exclusion list with a dedicated dimension table or flag.
•	Turn Restaurant Density into a true density measure (restaurants per km² or per 1,000 inhabitants).
•	Add time dimension (if data has timestamps) to follow trends.
•	Integrate additional features: 
o	Delivery time,
o	Delivery platforms,
o	User segments.
________________________________________
7. Contact
Author: Emmanuel Goldberg
GitHub: https://github.com/Manu1175
If you’d like a lighter README or separate docs per topic (e.g. Hummus analysis, Pizza rankings, Veg/Vegan map), you can split this file into smaller .md pages.

