# First_project
import requests
import time
import json
import csv
import os
from datetime import datetime

# ------------- CONFIG -------------
INTERVAL_MINUTES = 15  # How often to check (minutes)
AMOUNT_PER_TRADE = 10  # $10 per simulated trade
PORTFOLIO_FILE = "portfolio.json"
TRANSACTION_LOG = "transactions.csv"
API_KEY = "YOUR_ALPHA_VANTAGE_API_KEY"
ALPHA_VANTAGE_URL = "https://www.alphavantage.co/query"
# ----------------------------------

def get_sp500_tickers():
    """
    Download S&P 500 tickers from Wikipedia.
    Returns a list of tickers.
    """
    wiki_url = "https://en.wikipedia.org/wiki/List_of_S%26P_500_companies"
    resp = requests.get(wiki_url)
    tickers = []
    if resp.ok:
        from bs4 import BeautifulSoup
        soup = BeautifulSoup(resp.text, "html.parser")
        table = soup.find("table", {"id": "constituents"})
        for row in table.find_all("tr")[1:]:
            ticker = row.find_all("td")[0].text.strip().replace('.', '-')
            tickers.append(ticker)
    return tickers

def get_stock_price(ticker):
    """
    Fetch daily prices for ticker from Alpha Vantage.
    Returns (current_price, previous_close) or (None, None) if error.
    """
    params = {
        "function": "TIME_SERIES_INTRADAY",
        "symbol": ticker,
        "interval": "5min",
        "apikey": API_KEY
    }
    try:
        resp = requests.get(ALPHA_VANTAGE_URL, params=params)
        data = resp.json()
        ts_name = [k for k in data if k.startswith("Time Series")][0]
        times = sorted(data[ts_name].keys(), reverse=True)
        latest = float(data[ts_name][times[0]]["4. close"])
        if len(times) > 1:
            previous = float(data[ts_name][times[1]]["4. close"])
        else:
            previous = latest
        return latest, previous
    except Exception:
        return None, None

def load_portfolio(filename):
    if not os.path.exists(filename):
        return {}
    with open(filename, "r") as f:
        return json.load(f)

def save_portfolio(portfolio, filename):
    with open(filename, "w") as f:
        json.dump(portfolio, f, indent=2)

def log_transaction(row, filename):
    file_exists = os.path.exists(filename)
    with open(filename, "a", newline="") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["ticker", "datetime", "action", "amount", "price", "pct_change"])
        writer.writerow(row)

def run_tracker():
    tickers = get_sp500_tickers()
    portfolio = load_portfolio(PORTFOLIO_FILE)
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    for ticker in tickers:
        price, prev = get_stock_price(ticker)
        if price is None or prev is None or prev == 0:
            continue
        pct_change = (price - prev) / prev * 100

        action = None
        if pct_change <= -5:
            action = "buy"
            portfolio[ticker] = portfolio.get(ticker, 0) + AMOUNT_PER_TRADE / price
        elif pct_change >= 5:
            if portfolio.get(ticker, 0) * price >= AMOUNT_PER_TRADE:
                action = "sell"
                portfolio[ticker] = portfolio.get(ticker, 0) - AMOUNT_PER_TRADE / price

        if action:
            log_transaction([
                ticker,
                now,
                action,
                AMOUNT_PER_TRADE,
                round(price, 2),
                round(pct_change, 2)
            ], TRANSACTION_LOG)

    save_portfolio(portfolio, PORTFOLIO_FILE)

if __name__ == "__main__":
    while True:
        run_tracker()
        print(f"Checked stocks at {datetime.now()}")
        time.sleep(INTERVAL_MINUTES * 60)