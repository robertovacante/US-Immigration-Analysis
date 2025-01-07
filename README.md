## US-Immigration-Analysis
A Python project that uses web scraping techniques to collect U.S. immigration decisions from TRAC data and applies simple data analysis and visualization to reveal key trends and insights.

```s
import requests
from bs4 import BeautifulSoup
import pandas as pd
import re
```

```s
url = "https://trac.syr.edu/immigration/reports/judgereports/"
```


```s
president_party_mapping = {
    (2021, 2024): ("Joe Biden", "Democrat"),
    (2017, 2020): ("Donald Trump", "Republican"),
    (2009, 2016): ("Barack Obama", "Democrat"),
    (2001, 2008): ("George W. Bush", "Republican"),
    (1993, 2000): ("Bill Clinton", "Democrat"),
    (1989, 1992): ("George H. W. Bush", "Republican"),
    (1981, 1988): ("Ronald Reagan", "Republican"),
    (1977, 1980): ("Jimmy Carter", "Democrat"),
    (1974, 1976): ("Gerald Ford", "Republican"),
    (1969, 1974): ("Richard Nixon", "Republican"),
    (1963, 1968): ("Lyndon B. Johnson", "Democrat"),
    (1961, 1963): ("John F. Kennedy", "Democrat"),
}

def get_president_and_party(year):
    year = int(year)
    for years, (president, party) in president_party_mapping.items():
        if years[0] <= year <= years[1]:
            return president, party
    return None, None
```

```s
response = requests.get(url)
response.raise_for_status()
```

```s
soup = BeautifulSoup(response.text, 'html.parser')
```

```s
table = soup.find('table')
```

```s
data = []
current_court = ""
rowspan_counter = 0
```

```s
for row in table.find_all('tr'):
    cols = row.find_all('td')
    
    if len(cols) > 0:
        if 'rowspan' in cols[0].attrs:
            current_court = cols[0].get_text(strip=True)
            rowspan_counter = int(cols[0]['rowspan'])
            
            if len(cols) >= 6:
                judge = cols[1].get_text(strip=True)
                judge_link = cols[1].find('a')['href'] if cols[1].find('a') else None
                total_decisions = cols[2].get_text(strip=True)
                percent_granted_asylum = cols[3].get_text(strip=True)
                percent_granted_other = cols[4].get_text(strip=True)
                percent_denied = cols[5].get_text(strip=True)
            else:
                continue
        else:
            if len(cols) >= 5:
                judge = cols[0].get_text(strip=True)
                judge_link = cols[0].find('a')['href'] if cols[0].find('a') else None
                total_decisions = cols[1].get_text(strip=True)
                percent_granted_asylum = cols[2].get_text(strip=True)
                percent_granted_other = cols[3].get_text(strip=True)
                percent_denied = cols[4].get_text(strip=True)
            else:
                continue
        
        rowspan_counter -= 1
        
        appointment_year = None
        juris_doctor_year = None
        
        if judge_link:
            judge_page_url = url + judge_link
            judge_page = requests.get(judge_page_url)
            judge_soup = BeautifulSoup(judge_page.text, 'html.parser')
            
            bio_paragraph = judge_soup.select_one("div div div p:nth-of-type(2)")
            if bio_paragraph:
                year_match = re.findall(r'\b(\d{4})\b', bio_paragraph.get_text())
                if year_match:
                    appointment_year = year_match[0]
            
            juris_match = re.search(r'Juris.*?(\d{4})', judge_soup.get_text(), re.IGNORECASE)
            if juris_match:
                juris_doctor_year = juris_match.group(1)
        
        president, party = get_president_and_party(appointment_year) if appointment_year else (None, None)
        democrat_appointer = 1 if party == "Democrat" else 0
        
        data.append([current_court, judge, total_decisions, percent_granted_asylum, 
                     percent_granted_other, percent_denied, appointment_year, 
                     juris_doctor_year, president, party, democrat_appointer])
```

```s
columns = ["Immigration Court", "Judge", "Total Decisions", "% Granted Asylum", 
           "% Granted Other Relief", "% Denied", "Appointment Date", "Juris Doctor Year",
           "Appointing President", "Party", "Democrat Appointer"]
df = pd.DataFrame(data, columns=columns)
```

```s
df.to_csv('immigration_judges.csv', index=False)
print("Data saved to 'immigration_judges.csv'")
```

```s
import pandas as pd
import statsmodels.api as sm
df = pd.read_csv('immigration_judges.csv')
```

## Simple Data Analysis (some possible models)
```s
## 1. Impact of Judicial Factors on Denial Rate

X = df[["% Granted Asylum", "% Granted Other Relief", "Democrat Appointer",
        "Juris Doctor Year", "Appointing President"]]
X = pd.get_dummies(X, columns=["Democrat Appointer", "Appointing President", ], drop_first=True)
y = pd.to_numeric(df["% Denied"], errors='coerce')
```

```s
## 2. Effect of Appointment Factors on Asylum Grant Rates

X = df[["Total Decisions", "% Granted Other Relief", "Juris Doctor Year", "Appointing President", "Party", "Democrat Appointer"]]
X = pd.get_dummies(X, columns=["Democrat Appointer", "Appointing President", "Party"], drop_first=True)
X = X.apply(pd.to_numeric, errors='coerce')
y = pd.to_numeric(df["% Granted Asylum"], errors='coerce')

```

```s
## 3. Impact of Judicial and Appointment Factors on Granting Other Relief

X = df[["Total Decisions", "% Granted Asylum", "% Denied", "Juris Doctor Year", "Appointing President", "Party"]]
X = pd.get_dummies(X, columns=["Appointing President", "Party"], drop_first=True)
X = X.apply(pd.to_numeric, errors='coerce')
y = pd.to_numeric(df["% Granted Other Relief"], errors='coerce')
```

```s
X = df[["Total Decisions", "% Granted Asylum", "% Granted Other Relief", "Juris Doctor Year", "Democrat Appointer"]]
X = pd.get_dummies(X, columns=["Democrat Appointer"], drop_first=True)
X = X.apply(pd.to_numeric, errors='coerce')
y = pd.to_numeric(df["Total Decisions"], errors='coerce')
```

```s
X = X.apply(pd.to_numeric, errors='coerce')
X = X.dropna()
y = y.loc[X.index]  # Align y with the cleaned X
X = X.astype(int)
X = sm.add_constant(X)

model = sm.OLS(y, X).fit()
print(model.summary())
```




