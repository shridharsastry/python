from alpaca.data.historical import StockHistoricalDataClient
from alpaca.data.requests import StockLatestTradeRequest
from alpaca.trading.client import TradingClient
from alpaca.trading.requests import MarketOrderRequest
from alpaca.trading.enums import OrderSide, TimeInForce

import time
import sqlite3
import smtplib
from email.mime.text import MIMEText
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.middleware.cors import CORSMiddleware
import uvicorn
from threading import Thread
from pydantic import BaseModel

# ==================== CONFIG ====================
API_KEY = 'PKUNXIE4A4XWOZV46QUR'
API_SECRET = 'fej0zAwENpeHDbaJ4vSwqEHVnZGmAeq4Pa7uhbjL'
BUDGET = 10000  # USD budget
SYMBOLS = ['AAPL', 'MSFT', 'GOOG']
CHECK_INTERVAL = 60  # seconds
DROP_THRESHOLD = -5  # % drop
GAIN_THRESHOLD = 10  # % gain
EMAIL_SENDER = 'shridharsastry@gmail.com'
EMAIL_PASSWORD = 'Niha246Vmware '
EMAIL_RECEIVER = 'shridharsastry@example.com'

# ==================== INIT ====================
data_client = StockHistoricalDataClient(API_KEY, API_SECRET)
trading_client = TradingClient(API_KEY, API_SECRET, paper=True)

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True, allow_methods=["*"], allow_headers=["*"])

conn = sqlite3.connect('alpaca_trades.db', check_same_thread=False)
cursor = conn.cursor()
cursor.execute('''
    CREATE TABLE IF NOT EXISTS trades (
        id INTEGER PRIMARY KEY,
        symbol TEXT,
        action TEXT,
        price REAL,
        quantity INTEGER,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
    )
''')
conn.commit()

holding = {symbol: {'own': False, 'buy_price': 0.0} for symbol in SYMBOLS}
latest_prices = {}
remaining_budget = BUDGET

# ==================== EMAIL ALERT ====================
def send_email(subject, body):
    msg = MIMEText(body)
    msg['Subject'] = subject
    msg['From'] = EMAIL_SENDER
    msg['To'] = EMAIL_RECEIVER

    with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
        server.login(EMAIL_SENDER, EMAIL_PASSWORD)
        server.sendmail(EMAIL_SENDER, EMAIL_RECEIVER, msg.as_string())

# ==================== LOGGING ====================
def log_trade(symbol, action, price, quantity):
    cursor.execute('INSERT INTO trades (symbol, action, price, quantity) VALUES (?, ?, ?, ?)',
                   (symbol, action, price, quantity))
    conn.commit()

# ==================== DASHBOARD ====================
@app.get('/dashboard', response_class=HTMLResponse)
async def dashboard():
    trades = cursor.execute('SELECT * FROM trades ORDER BY timestamp DESC').fetchall()
    html = "<h2>Trading Dashboard</h2><table border='1'><tr><th>ID</th><th>Symbol</th><th>Action</th><th>Price</th><th>Quantity</th><th>Timestamp</th></tr>"
    for t in trades:
        html += f"<tr><td>{t[0]}</td><td>{t[1]}</td><td>{t[2]}</td><td>{t[3]}</td><td>{t[4]}</td><td>{t[5]}</td></tr>"
    html += "</table>"
    html += f"<h3>Remaining Budget: ${remaining_budget:.2f}</h3>"
    return html

# ==================== TRADING LOGIC ====================
def trading_loop():
    global remaining_budget
    while True:
        try:
            for symbol in SYMBOLS:
                trade_data = data_client.get_stock_latest_trade(StockLatestTradeRequest(symbol_or_symbols=symbol))
                current_price = float(trade_data[symbol].price)
                latest_prices[symbol] = current_price

                if symbol not in holding:
                    holding[symbol] = {'own': False, 'buy_price': 0.0}

                initial_price = holding[symbol]['buy_price'] if holding[symbol]['own'] else current_price
                price_change = ((current_price - initial_price) / initial_price) * 100 if initial_price else 0

                print(f"{symbol}: ${current_price:.2f} | Change: {price_change:.2f}%")

                # Buy logic
                if not holding[symbol]['own'] and price_change <= DROP_THRESHOLD and remaining_budget >= current_price * 10:
                    order = MarketOrderRequest(symbol=symbol, qty=10, side=OrderSide.BUY, time_in_force=TimeInForce.GTC)
                    trading_client.submit_order(order)
                    log_trade(symbol, 'buy', current_price, 10)
                    holding[symbol]['own'] = True
                    holding[symbol]['buy_price'] = current_price
                    remaining_budget -= current_price * 10

                # Sell logic
                if holding[symbol]['own']:
                    gain = ((current_price - holding[symbol]['buy_price']) / holding[symbol]['buy_price']) * 100
                    if gain >= GAIN_THRESHOLD:
                        order = MarketOrderRequest(symbol=symbol, qty=10, side=OrderSide.SELL, time_in_force=TimeInForce.GTC)
                        trading_client.submit_order(order)
                        log_trade(symbol, 'sell', current_price, 10)
                        holding[symbol]['own'] = False
                        remaining_budget += current_price * 10

            # Budget alert
            if remaining_budget < min([latest_prices.get(sym, 0) * 10 for sym in SYMBOLS if latest_prices.get(sym, 0) > 0], default=0):
                send_email("Budget Alert", "Out of budget for new trades!")

        except Exception as e:
            print(f"Error: {e}")

        time.sleep(CHECK_INTERVAL)

# ==================== START ====================
if __name__ == "__main__":
    Thread(target=trading_loop, daemon=True).start()
    uvicorn.run(app, host="localhost", port=8000)
