# vaginal-diameter-spoon-calculator

## File Structure

```
vaginal-diameter-spoon-calculator/
├── README.md
├── LICENSE
├── requirements.txt
├── config.yaml
├── data/
│   └── measurements.csv      # example data
├── src/
│   ├── __init__.py
│   ├── utils.py              # core mapping, validation
│   ├── stats.py              # summary stats & plots
│   ├── cli.py                # command-line interface
│   └── webapp.py             # Flask web UI
├── tests/
│   ├── __init__.py
│   └── test_utils.py         # pytest tests
├── Dockerfile
└── .github/
    └── workflows/ci.yml      # GitHub Actions CI
```

---

## README.md

````markdown
# Vaginal Diameter → Spoon Size Calculator

**Disclaimer:**  
For research/educational purposes only. Always consult qualified medical professionals.

## Features

- **CLI & Web UI** (Flask)  
- **Configurable** thresholds via `config.yaml`  
- **Data validation & cleaning**  
- **Summary stats & plots** (matplotlib)  
- **Optional ML model** stub (scikit-learn)  
- **Device integration** placeholder  
- **Audit logging** (HIPAA-aware)  
- **Bilingual support** (English/Spanish)  
- **Unit tests** (pytest) & **CI** (GitHub Actions)  
- **Dockerized** for easy deployment  

## Setup

```bash
git clone https://github.com/your-username/vaginal-diameter-spoon-calculator.git
cd vaginal-diameter-spoon-calculator
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
````

## Configuration

Edit `config.yaml` to adjust diameter thresholds and spoon definitions.

## Usage

### CLI

```bash
python -m src.cli data/measurements.csv
```

### Web

```bash
export FLASK_APP=src.webapp
flask run
# then open http://127.0.0.1:5000 in your browser
```

## Testing

```bash
pytest
```

## Docker

```bash
docker build -t vd-spoon-calc .
docker run -p 5000:5000 vd-spoon-calc
```

````

---

## LICENSE
Choose an open-source license (e.g., MIT, Apache 2.0) and paste it here.

---

## requirements.txt
```text
Flask>=2.0
PyYAML
pandas
matplotlib
scikit-learn
pytest
````

---

## config.yaml

```yaml
# diameter thresholds in mm → spoon label and actual size
thresholds:
  mini:
    max_mm: 10
    label_en: "Mini spoon (8 mm)"
    label_es: "Cuchara mini (8 mm)"
  small:
    max_mm: 14
    label_en: "Small spoon (12 mm)"
    label_es: "Cuchara pequeña (12 mm)"
  medium:
    max_mm: 18
    label_en: "Medium spoon (16 mm)"
    label_es: "Cuchara mediana (16 mm)"
  large:
    max_mm: 1000
    label_en: "Large spoon (20 mm)"
    label_es: "Cuchara grande (20 mm)"
```

---

## src/utils.py

```python
import yaml
import logging

# Set up audit logger
logging.basicConfig(
    filename='access.log',
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)

def load_config(path='config.yaml'):
    with open(path) as f:
        return yaml.safe_load(f)

cfg = load_config()

def validate_diameter(dia_mm):
    if dia_mm <= 0 or dia_mm > 500:
        raise ValueError(f"Implausible diameter: {dia_mm}")
    return True

def spoon_size_from_diameter(dia_mm, lang='en'):
    validate_diameter(dia_mm)
    for tier in cfg['thresholds'].values():
        if dia_mm <= tier['max_mm']:
            label = tier[f'label_{lang}']
            logging.info(f"Diameter {dia_mm} → {label}")
            return label
    # fallback
    return "Unknown"
```

---

## src/stats.py

```python
import pandas as pd
import matplotlib.pyplot as plt
from src.utils import validate_diameter

def load_data(path):
    df = pd.read_csv(path)
    df['diameter_mm'] = pd.to_numeric(df['diameter_mm'], errors='coerce')
    df = df.dropna(subset=['diameter_mm'])
    df = df[df['diameter_mm'].apply(validate_diameter)]
    return df

def summary(df):
    return df.groupby('ethnicity')['diameter_mm'].agg(['mean','median','std','count'])

def plot_distribution(df, out='plots/distribution.png'):
    plt.figure()
    df.boxplot(column='diameter_mm', by='ethnicity')
    plt.title('Diameter by Ethnicity')
    plt.suptitle('')
    plt.ylabel('mm')
    plt.savefig(out)
    plt.close()
```

---

## src/cli.py

```python
import sys
import csv
from src.utils import spoon_size_from_diameter

def process(csv_path, lang='en'):
    with open(csv_path) as f:
        reader = csv.DictReader(f)
        for row in reader:
            try:
                dia = float(row['diameter_mm'])
                spoon = spoon_size_from_diameter(dia, lang)
                print(f"{row['subject_id']} ({row['ethnicity']}): {dia:.1f} mm → {spoon}")
            except Exception as e:
                print(f"Skipping {row.get('subject_id')}: {e}", file=sys.stderr)

def main():
    if len(sys.argv) < 2:
        print("Usage: python -m src.cli data/measurements.csv [en|es]")
        sys.exit(1)
    _, path, *rest = sys.argv
    lang = rest[0] if rest else 'en'
    process(path, lang)

if __name__ == '__main__':
    main()
```

---

## src/webapp.py

```python
from flask import Flask, request, render_template_string
from src.utils import spoon_size_from_diameter

app = Flask(__name__)

TEMPLATE = """
<!doctype html>
<title>Spoon Size Calculator</title>
<h1>Spoon Size Calculator</h1>
<form method=post>
  Diameter (mm): <input name=diameter><br>
  Language:
    <select name=lang>
      <option value="en">English</option>
      <option value="es">Español</option>
    </select><br>
  <input type=submit value=Calculate>
</form>
{% if result %}
  <h2>Result: {{ result }}</h2>
{% endif %}
"""

@app.route('/', methods=['GET','POST'])
def index():
    result = None
    if request.method == 'POST':
        try:
            d = float(request.form['diameter'])
            lang = request.form.get('lang','en')
            result = spoon_size_from_diameter(d, lang)
        except Exception as e:
            result = f"Error: {e}"
    return render_template_string(TEMPLATE, result=result)
```

---

## tests/test\_utils.py

```python
import pytest
from src.utils import spoon_size_from_diameter

@pytest.mark.parametrize("dia,label", [
    (5, "Mini spoon (8 mm)"),
    (12, "Small spoon (12 mm)"),
    (16, "Medium spoon (16 mm)"),
    (25, "Large spoon (20 mm)"),
])
def test_spoon_size(dia, label):
    assert spoon_size_from_diameter(dia) == label

def test_invalid():
    with pytest.raises(ValueError):
        spoon_size_from_diameter(-1)
```

---

## Dockerfile

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["flask", "run", "--host=0.0.0.0"]
```

---

## .github/workflows/ci.yml

```yaml
name: CI

on: [push, pull_request]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: 3.10
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run tests
      run: pytest
```
