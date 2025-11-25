cd ~ 
git clone git@github.com:semicap/backend.git 
cd backend 

pip install pandas requests beautifulsoup4 sec-edgar-downloader 

touch parse_sec_data.py 
 
import pandas as pd 
from sec_edgar_downloader import Downloader 
from bs4 import BeautifulSoup 
import os 
import re 


# Initialize downloader 

dl = Downloader("data/sec_filings") 

# Step 1: Load company list 

companies = pd.read_csv("data/companies.csv")  # adjust path if needed 

# Step 2: Download latest 10-K filings for each company 

for symbol in companies["symbol"]: 
    try: 
        dl.get("10-K", symbol, amount=1) 
        print(f"Downloaded latest 10-K for {symbol}") 
    except Exception as e: 
        print(f"Failed to fetch {symbol}: {e}") 

# Step 3: Extract standardized data from downloaded filings 

data_list = [] 
def extract_data_from_file(filepath, symbol): 
    with open(filepath, "r", errors="ignore") as f: 
        soup = BeautifulSoup(f.read(), "html.parser") 
        
    # Example: Extract revenue (which can be tagged differently) 
    patterns = ["Revenue", "Sales", "TotalRevenue", "Total Sales"] 
    text = soup.get_text(" ") 
    for p in patterns: 
        match = re.search(rf"{p}[\s:–-]+\$?([\d,\.]+)", text, re.IGNORECASE) 
        if match: 
            revenue = match.group(1) 
            data_list.append({"symbol": symbol, "item": "Revenue", "value": revenue}) 

            return 
 
# Step 4: Loop through all files and parse 

for root, dirs, files in os.walk("data/sec_filings"): 
    for file in files: 
        if file.endswith(".txt") or file.endswith(".html"): 
            filepath = os.path.join(root, file) 
            symbol = root.split("/")[-1] 
            extract_data_from_file(filepath, symbol) 

# Step 5: Save cleaned data 

df = pd.DataFrame(data_list) 
df.to_csv("data/cleaned_sec_data.csv", index=False) 
