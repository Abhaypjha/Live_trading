import pandas as pd
import pandas_ta as ta
from fyers_apiv3 import fyersModel
from datetime import datetime, timedelta
import time
import os

# ---- Fyers Configuration ----
client_id = "Paste here"
access_token = "Paste Here"
fyers = fyersModel.FyersModel(client_id=client_id, token=access_token, log_path=os.getcwd())

# --------- Parameters ----------
symbols = [
    "NSE:SUNPHARMA-EQ", "NSE:TCS-EQ", "NSE:RELIANCE-EQ", "NSE:INFY-EQ", "NSE:CIPLA-EQ",
    "NSE:MARUTI-EQ", "NSE:DIXON-EQ", "NSE:AXISBANK-EQ", "NSE:BAJAJ-AUTO-EQ", "NSE:DABUR-EQ",
    "NSE:LTIM-EQ", "NSE:BHARTIAIRTEL-EQ", "NSE:VOLTAS-EQ", "NSE:HDFCBANK-EQ",
    "NSE:ICICIBANK-EQ", "NSE:TVSMOTORS-EQ", "NSE:NIFTY50-INDEX", "NSE:BANKNIFTY-INDEX"
]
excel_file = "live_signals2.xlsx"
quantity = 1000  # Number of lots for paper trading

# --------- Excel Loader/Updater ---------
def load_existing_signals():
    if os.path.exists(excel_file):
        df = pd.read_excel(excel_file)
        signals = {}
        for _, row in df.iterrows():
            if row['status'] in ["Open", "Holding"]:
                signals[row['symbol']] = row.to_dict()
        return signals, df
    else:
        columns = ["symbol", "signal", "entry_price", "sl", "tp", "signal_time",
                   "status", "exit_reason", "exit_time", "pnl"]
        return {}, pd.DataFrame(columns=columns)

def update_excel(df, signal_data):
    symbol = signal_data["symbol"]
    signal_time = signal_data["signal_time"]

    existing_row = df[
        (df["symbol"] == symbol) & (df["signal_time"] == signal_time)
    ]

    if not existing_row.empty:
        index = existing_row.index[0]
        for key in signal_data:
            df.at[index, key] = signal_data[key]
    else:
        df = pd.concat([df, pd.DataFrame([signal_data])], ignore_index=True)

    df.to_excel(excel_file, index=False)
    return df

# --------- Fetch OHLC ---------
def fetch_ohlc(symbol):
    data = {
        "symbol": symbol,
        "resolution": "30",
        "date_format": "1",
        "range_from": (datetime.now() - timedelta(days=10)).strftime("%Y-%m-%d"),
        "range_to": datetime.now().strftime("%Y-%m-%d"),
        "cont_flag": "1"
    }
    response = fyers.history(data)
    candles = response.get("candles")
    if candles:
        df = pd.DataFrame(candles, columns=["timestamp", "open", "high", "low", "close", "volume"])
        df["datetime"] = pd.to_datetime(df["timestamp"], unit="s")
        return df
    return pd.DataFrame()

# --------- Signal Checker ---------
def check_signal(symbol, existing_signals):
    df = fetch_ohlc(symbol)
    if df.empty or len(df) < 20:
        return None

    df["EMA_10"] = ta.ema(df["close"], length=10)
    df["EMA_20"] = ta.ema(df["close"], length=20)
    df["RSI"] = ta.rsi(df["close"], length=14)
    adx = ta.adx(df["high"], df["low"], df["close"], length=14)
    df["ADX"] = adx["ADX_14"] if adx is not None else None
    df.dropna(inplace=True)

    df["ha_close"] = (df["open"] + df["high"] + df["low"] + df["close"]) / 4
    df["ha_open"] = df["open"].astype(float)
    for i in range(1, len(df)):
        df.at[df.index[i], "ha_open"] = (df.at[df.index[i - 1], "ha_open"] + df.at[df.index[i - 1], "ha_close"]) / 2

    df["ha_high"] = df[["high", "ha_open", "ha_close"]].max(axis=1)
    df["ha_low"] = df[["low", "ha_open", "ha_close"]].min(axis=1)

    latest = df.iloc[-1]
    prev1 = df.iloc[-2]
    prev2 = df.iloc[-3]

    now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')

    # --------- Exit Logic ---------
    if symbol in existing_signals:
        signal_data = existing_signals[symbol]
        entry = signal_data["entry_price"]
        sl = signal_data["sl"]
        tp = signal_data["tp"]

        if signal_data["signal"] == "Buy":
            if latest["low"] <= sl and latest["high"] >= tp:
                signal_data.update({
                    "status": "Hit TP" if latest["close"] > entry else "Hit SL",
                    "exit_reason": "SL & TP same candle",
                    "exit_time": now,
                    "pnl": round(tp - entry if latest["close"] > entry else sl - entry, 2) * quantity
                })
                return signal_data
            elif latest["low"] <= sl:
                signal_data.update({
                    "status": "Hit SL",
                    "exit_reason": "SL Hit",
                    "exit_time": now,
                    "pnl": round(sl - entry, 2) * quantity
                })
                return signal_data
            elif latest["high"] >= tp:
                signal_data.update({
                    "status": "Hit TP",
                    "exit_reason": "TP Hit",
                    "exit_time": now,
                    "pnl": round(tp - entry, 2) * quantity
                })
                return signal_data

        elif signal_data["signal"] == "Sell":
            if latest["high"] >= sl and latest["low"] <= tp:
                signal_data.update({
                    "status": "Hit TP" if latest["close"] < entry else "Hit SL",
                    "exit_reason": "SL & TP same candle",
                    "exit_time": now,
                    "pnl": round(entry - tp if latest["close"] < entry else entry - sl, 2) * quantity
                })
                return signal_data
            elif latest["high"] >= sl:
                signal_data.update({
                    "status": "Hit SL",
                    "exit_reason": "SL Hit",
                    "exit_time": now,
                    "pnl": round(entry - sl, 2) * quantity
                })
                return signal_data
            elif latest["low"] <= tp:
                signal_data.update({
                    "status": "Hit TP",
                    "exit_reason": "TP Hit",
                    "exit_time": now,
                    "pnl": round(entry - tp, 2) * quantity
                })
                return signal_data

        signal_data.update({
            "status": "Holding",
            "exit_reason": "HOLD",
            "exit_time": "",
            "pnl": round(latest["close"] - entry if signal_data["signal"] == "Buy" else entry - latest["close"], 2) * quantity
        })
        return signal_data

    # --------- Buy Signal ---------
    if (
        latest["EMA_10"] > latest["EMA_20"] and
        prev1["EMA_10"] > prev1["EMA_20"] and
        prev2["EMA_10"] > prev2["EMA_20"] and
        latest["RSI"] > 60 and latest["ADX"] > 25 and
        latest["ha_close"] > latest["ha_open"]
    ):
        entry = round(latest["ha_open"], 2)
        signal_data = {
            "symbol": symbol,
            "signal": "Buy",
            "entry_price": entry,
            "sl": round(entry * 0.995, 2),
            "tp": round(entry * 1.01, 2),
            "signal_time": now,
            "status": "Open",
            "exit_reason": "HOLD",
            "exit_time": "",
            "pnl": "",
        }
        existing_signals[symbol] = signal_data
        return signal_data

    # --------- Sell Signal ---------
    if (
        latest["EMA_10"] < latest["EMA_20"] and
        prev1["EMA_10"] < prev1["EMA_20"] and
        prev2["EMA_10"] < prev2["EMA_20"] and
        latest["RSI"] < 40 and latest["ADX"] > 25 and
        latest["ha_close"] < latest["ha_open"]
    ):
        entry = round(latest["ha_open"], 2)
        signal_data = {
            "symbol": symbol,
            "signal": "Sell",
            "entry_price": entry,
            "sl": round(entry * 1.005, 2),
            "tp": round(entry * 0.99, 2),
            "signal_time": now,
            "status": "Open",
            "exit_reason": "HOLD",
            "exit_time": "",
            "pnl": "",
        }
        existing_signals[symbol] = signal_data
        return signal_data
    return None

# --------- Main Trading Loop ---------
existing_signals, df = load_existing_signals()
while True:
    for symbol in symbols:
        signal_data = check_signal(symbol, existing_signals)
        if signal_data:
            df = update_excel(df, signal_data)
            print(f"Updated signal for {symbol}: {signal_data}")
    time.sleep(60)  # wait for 1 minute before checking again
