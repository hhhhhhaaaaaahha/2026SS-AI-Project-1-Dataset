## 2. Dataset Documentation

As per the project requirements, this dataset was independently collected and constructed rather than sourced from existing public repositories.

### 2.1 Data Collection Methodology
Data was gathered using a custom web scraper (`cpbl_crawler.py`) targeting the [CPBL official schedule page](https://www.cpbl.com.tw/schedule). Collecting this data presented significant technical hurdles, as the modern CPBL website utilizes a Vue.js frontend coupled with a heavily secured ASP.NET backend. 

Standard `GET` or `POST` requests via libraries like `requests` or `BeautifulSoup` were actively blocked by the server due to missing anti-forgery tokens and mismatched cookies. To overcome this, an advanced **XHR Interception and Payload Spoofing** strategy was implemented:
* **Headless Browser Automation:** Selenium WebDriver was deployed to naturally render the DOM, execute the Vue.js lifecycle, and acquire legitimate session cookies.
* **JavaScript Injection:** Custom JavaScript was injected directly into the browser to hijack the native `XMLHttpRequest` prototype. This allowed the crawler to intercept the frontend's API calls before they left the browser.
* **Payload Modification:** The CPBL API exhibited an anomaly where it would aggressively return future schedules (e.g., the year 2026) regardless of the queried month. To bypass this, the crawler dynamically intercepted the outgoing JSON payloads, forcibly rewrote the `calendar` parameter to the target historical month (2023–2025), and captured the server's response.

### 2.2 Data Cleaning and Domain-Specific Filtering
Raw data retrieved from the API (`cpbl_raw_data.csv`) contained significant noise and required rigorous cleaning (`data_cleaning.py`) to prevent data leakage and model hallucination.

1. **Handling "Ghost Games" (Domain Knowledge):** In baseball, games are frequently postponed due to rain. The CPBL database retains these canceled games with a default score of 0:0, creating duplicate entries when the makeup game is played. To prevent the model from learning false 0:0 labels, the `PresentStatus` (or `IsGameStop`) flag was utilized. Only records explicitly marked as completed games were retained.
2. **Future Game Pruning:** The API occasionally returned scheduled but unplayed games with default scores. A temporal filter (`GameDate < pd.Timestamp.today()`) was applied to strictly isolate historical, completed games.
3. **Deduplication:** Due to the API's pagination and bulk-return behavior, identical games were fetched multiple times. Exact duplicates were dropped based on a composite key of `['GameDate', 'VisitingTeamName', 'HomeTeamName']`.

### 2.3 Feature Engineering and Formatting
Machine learning models, particularly Neural Networks, cannot natively process categorical text strings. The dataset was transformed to ensure compatibility:
* **Temporal Features:** The `GameDate` was parsed to extract `Month` and `DayOfWeek`. This is crucial for capturing seasonality effects, as summer months generally favor hitters due to higher temperatures and pitcher fatigue.
* **One-Hot Encoding:** Categorical variables, including `VisitingTeamName`, `HomeTeamName`, and `FieldAbbe` (Ballpark), were transformed using One-Hot Encoding (`pd.get_dummies`). This converts team matchups and park factors into a sparse binary matrix, which is highly readable by tree-based algorithms.
* **Target Variable:** The `VisitingScore` and `HomeScore` were summed to create `TotalScore`, which was then thresholded at 8.5 to generate the final binary target label `IsBigScoringGame`. To prevent data leakage, the original score columns were dropped before training.

### 2.4 Dataset Statistics
After rigorous cleaning and deduplication, the final curated dataset (`cpbl_ml_ready_dataset.csv`) contains **1,128 highly verified, regular-season games** spanning the 2023, 2024, and 2025 seasons.
