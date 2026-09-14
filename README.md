# Btc-bitcoin-bot
Tradingbot
pip install MetaTrader5 requests
python mt5_bridge.py
#!/usr/bin/env python3
"""
Trading Command Center v1 - MetaTrader 5 Local Gateway
Run this script on the Windows machine where MetaTrader 5 terminal is installed.
Prerequisites:
    pip install MetaTrader5 requests
"""

import os
import sys
import time
import argparse
import requests

try:
    import MetaTrader5 as mt5
except ImportError:
    print("Error: MetaTrader5 package is not installed.")
    print("Please install it using: pip install MetaTrader5")
    sys.exit(1)

def run_gateway(api_url, secret, account, password, server, sync_interval=3):
    print("=" * 65)
    print("      TRADING COMMAND CENTER v1 - METATRADER 5 LOCAL GATEWAY      ")
    print("=" * 65)
    print(f" Target Server API: {api_url}")
    print(f" Sync Interval:     {sync_interval}s")
    
    # Initialize MT5
    if not mt5.initialize():
        print(f"âŒ Failed to initialize MT5 terminal: {mt5.last_error()}")
        sys.exit(1)
        
    terminal_info = mt5.terminal_info()
    print(f" âœ“ MT5 Terminal Connected: {terminal_info.name} ({terminal_info.path})")
    
    # Optional login if provided
    if account and password and server:
        authorized = mt5.login(int(account), password=password, server=server)
        if not authorized:
            print(f"âŒ Login failed for account {account} on {server}: {mt5.last_error()}")
            mt5.shutdown()
            sys.exit(1)
        print(f" âœ“ Authenticated account {account} on {server}")

    account_info = mt5.account_info()
    if account_info is None:
        print(f"âŒ Failed to get account info: {mt5.last_error()}")
        mt5.shutdown()
        sys.exit(1)

    print(f" Account:  {account_info.login}")
    print(f" Broker:   {account_info.company} ({account_info.server})")
    print(f" Balance:  USD {account_info.balance:,.2f} {account_info.currency}")
    print(f" Equity:   USD {account_info.equity:,.2f}")
    print(f" Leverage: 1:{account_info.leverage}")
    print("=" * 65)
    print("Streaming live telemetry to Trading Command Center... (Ctrl+C to stop)")

    sync_endpoint = f"{api_url.rstrip('/')}/api/mt5/sync"
    
    try:
        while True:
            acc = mt5.account_info()
            if acc:
                payload = {
                    "accountNumber": str(acc.login),
                    "server": acc.server,
                    "broker": acc.company,
                    "currency": acc.currency,
                    "balance": round(acc.balance, 2),
                    "equity": round(acc.equity, 2),
                    "freeMargin": round(acc.margin_free, 2),
                    "marginLevel": round(acc.margin_level, 2) if acc.margin_level else 0,
                    "leverage": acc.leverage,
                    "openPositionsCount": mt5.positions_total() or 0,
                    "secret": secret,
                }
                headers = {"Content-Type": "application/json"}
                if secret:
                    headers["X-MT5-Secret"] = secret
                
                try:
                    res = requests.post(sync_endpoint, json=payload, headers=headers, timeout=5)
                    if res.status_code == 200:
                        data = res.json()
                        orders = data.get("orders", [])
                        open_pos = payload.get("openPositionsCount", 0)
                        eq_val = acc.equity
                        sys.stdout.write(f"\r[SYNC] Equity: {eq_val:,.2f} | Positions: {open_pos} | Pending Orders: {len(orders)}   ")
                        sys.stdout.flush()
                        
                        # Process pending orders
                        for order in orders:
                            execute_mt5_order(order)
                    else:
                        print(f"\nWarning: Server returned status {res.status_code}: {res.text[:100]}")
                except Exception as e:
                    print(f"\nError communicating with server: {e}")
                    
            time.sleep(sync_interval)
    except KeyboardInterrupt:
        print("\nGateway stopped by user.")
    finally:
        mt5.shutdown()

def execute_mt5_order(order):
    symbol = order.get("symbol", "BTCUSD")
    direction = order.get("direction", "BUY")
    volume = float(order.get("volume", 0.01))
    sl = float(order.get("stopLoss", 0))
    tp = float(order.get("takeProfit", 0))
    
    print(f"\n[EXECUTE] Placing {direction} on {symbol} (Vol: {volume}, SL: {sl}, TP: {tp})...")
    
    order_type = mt5.ORDER_TYPE_BUY if direction == "BUY" else mt5.ORDER_TYPE_SELL
    price = mt5.symbol_info_tick(symbol).ask if direction == "BUY" else mt5.symbol_info_tick(symbol).bid
    
    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": symbol,
        "volume": volume,
        "type": order_type,
        "price": price,
        "sl": sl,
        "tp": tp,
        "deviation": 20,
        "magic": 108821,
        "comment": "TradingCommandCenter v1",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    
    result = mt5.order_send(request)
    if result.retcode != mt5.TRADE_RETCODE_DONE:
        print(f"âŒ Order failed: {result.comment} (code: {result.retcode})")
    else:
        print(f"âœ“ Order executed successfully! Ticket: {result.order}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Trading Command Center MT5 Local Gateway")
    parser.add_argument("--api-url", default="http://localhost:3000", help="Command Center URL")
    parser.add_argument("--secret", default="", help="Webhook Secret")
    parser.add_argument("--account", help="MT5 Account Number")
    parser.add_argument("--password", help="MT5 Password")
    parser.add_argument("--server", help="MT5 Broker Server Name")
    parser.add_argument("--interval", type=int, default=3, help="Sync interval in seconds")
    args = parser.parse_args()
    
    run_gateway(args.api_url, args.secret, args.account, args.password, args.server, args.interval)
