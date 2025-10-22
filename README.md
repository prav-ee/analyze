# Data Processing & CI/CD Pipeline

This project demonstrates a robust and automated data processing workflow. It involves fixing a Python script, preparing data, and setting up a Continuous Integration/Continuous Deployment (CI/CD) pipeline using GitHub Actions to ensure code quality and publish processed results.

## Project Goals

1.  **Fix `execute.py`**: Identify and resolve a non-trivial error within the `execute.py` script to ensure correct and reliable data processing.
2.  **Data Preparation**: Convert the `data.xlsx` spreadsheet into a `data.csv` file for standard text-based data processing.
3.  **CI/CD Automation**: Implement a GitHub Actions workflow (`.github/workflows/ci.yml`) to:
    *   Perform code linting using [Ruff](https://beta.ruff.rs/docs/).
    *   Execute the `execute.py` script.
    *   Capture the script's output (`result.json`).
    *   Publish `result.json` via GitHub Pages.

## Prerequisites

Before you begin, ensure you have:

*   **Python 3.11+**: The project is tested and designed to run with Python 3.11 or newer.
*   **Pandas 2.3**: This specific version of the Pandas library is required for data manipulation.
*   **Ruff**: For code linting.
*   **`execute.py`**: Your Python script for data processing.
*   **`data.xlsx`**: Your initial Excel data file.

## Setup & Local Development

### 1. Data Conversion

First, convert your `data.xlsx` file to `data.csv`. This can be done manually using Excel or programmatically (e.g., using a small Python script):

```bash
# Example Python script to convert data.xlsx to data.csv
python -c "import pandas as pd; df = pd.read_excel('data.xlsx'); df.to_csv('data.csv', index=False)"
```

Commit the generated `data.csv` file to your repository.

### 2. Fixing `execute.py`

Analyze your `execute.py` script for non-trivial errors. Common issues might include:

*   **Incorrect File Path**: The script might be trying to read `data.xlsx` instead of the newly committed `data.csv`. Update `pd.read_excel('data.xlsx')` to `pd.read_csv('data.csv')`.
*   **Data Type Mismatches**: Operations on columns that contain mixed data types (e.g., trying to sum a column with numbers and strings). Use `pd.to_numeric(..., errors='coerce')` to handle this gracefully, followed by `fillna()` if necessary.
*   **Output Format**: Ensure the script's final output to `stdout` is a valid JSON string, as the CI pipeline expects `result.json`. You might need to use `json.dumps()` on your Python dictionary/list.

**Example `execute.py` (after hypothetical fixes):**

```python
import pandas as pd
import json

try:
    # 1. Read from data.csv
    df = pd.read_csv('data.csv')

    # 2. Robust data cleaning and type conversion
    # Assume 'Value' is a column that might have issues and 'Category' is for grouping
    if 'Value' in df.columns:
        df['CleanedValue'] = pd.to_numeric(df['Value'], errors='coerce').fillna(0)
    else:
        # Handle case where 'Value' column might not exist or needs a default
        df['CleanedValue'] = 0 # Or raise an error

    # 3. Perform desired aggregation
    if 'Category' in df.columns:
        # Example: Group by 'Category' and sum 'CleanedValue'
        aggregated_result = df.groupby('Category')['CleanedValue'].sum()
        output_data = aggregated_result.to_dict()
    else:
        # If no category, perhaps sum all values
        output_data = {'total_sum': df['CleanedValue'].sum()}

    # 4. Print valid JSON string to stdout
    print(json.dumps(output_data, indent=2))

except Exception as e:
    print(json.dumps({'error': str(e)}), indent=2)
    # Exit with a non-zero code to indicate failure in CI
    import sys
    sys.exit(1)
```

Commit your fixed `execute.py` to your repository.

### 3. Install Dependencies (Local Testing)

```bash
pip install pandas==2.3 ruff
```

### 4. Run Ruff (Local Testing)

```bash
ruff check .
```

### 5. Run `execute.py` (Local Testing)

```bash
python execute.py > result.json
```

Verify that `result.json` is created and contains the expected JSON output.

## GitHub Actions Workflow (`.github/workflows/ci.yml`)

The project includes a GitHub Actions workflow that automates the following steps on every push to the `main` branch:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch: # Allows manual trigger

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install ruff
      - name: Run Ruff
        run: ruff check .

  build-and-publish:
    needs: lint
    runs-on: ubuntu-latest
    permissions:
      contents: write # Required to publish to GitHub Pages
      pages: write # Required by peaceiris/actions-gh-pages for newer GitHub Pages setups
      id-token: write # Required by peaceiris/actions-gh-pages for newer GitHub Pages setups
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }} # This is a placeholder for the actual URL

    steps:
      - uses: actions/checkout@v4
      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pandas==2.3 ruff

      - name: Run execute.py and generate result.json
        run: python execute.py > result.json

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'result.json' # Uploads the generated JSON file

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

The `lint` job installs Python 3.11 and Ruff, then runs `ruff check .` to ensure code quality and style compliance. The `build-and-publish` job (which depends on `lint` passing) sets up Python 3.11 and installs `pandas==2.3`, executes `python execute.py > result.json` to generate the processed data, and then publishes `result.json` as a static artifact to GitHub Pages.

**Important Note:** The `result.json` file is *not* committed directly to the repository. It is generated during the CI process and published via GitHub Pages.

## Viewing Results on GitHub Pages

Once the GitHub Actions workflow completes successfully on your `main` branch, the `result.json` file will be accessible via GitHub Pages.

The URL will typically be in the format:
`https://YOUR_GITHUB_USERNAME.github.io/YOUR_REPOSITORY_NAME/result.json`

Example: If your username is `octocat` and your repository is `data-project`, the URL would be `https://octocat.github.io/data-project/result.json`.

Ensure GitHub Pages is enabled for your repository (usually set to deploy from the `gh-pages` branch or directly from the `main` branch with a `/docs` folder or GitHub Actions). The provided workflow uses GitHub Actions to build and deploy.

## Technologies Used

*   **Python 3.11+**
*   **Pandas 2.3**
*   **Ruff** (for linting)
*   **GitHub Actions** (for CI/CD)
*   **GitHub Pages** (for publishing results)
*   **Tailwind CSS** (for `index.html`)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.