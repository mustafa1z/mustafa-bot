from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes
import ccxt
import pandas as pd

API_TOKEN = '7594857915:AAEVsa9ehi3mEzEaKkW2pc7V3SKBach7Kfw'  # استبدله إذا غيّرت التوكن

def compute_rsi(data, period=14):
    delta = data.diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

def compute_macd(data, fast=12, slow=26, signal=9):
    ema_fast = data.ewm(span=fast, adjust=False).mean()
    ema_slow = data.ewm(span=slow, adjust=False).mean()
    macd = ema_fast - ema_slow
    signal_line = macd.ewm(span=signal, adjust=False).mean()
    return macd, signal_line

def get_trade_signal():
    exchange = ccxt.binance()
    bars = exchange.fetch_ohlcv('BTC/USDT', timeframe='1h', limit=100)

    df = pd.DataFrame(bars, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    df['RSI'] = compute_rsi(df['close'])
    df['MACD'], df['Signal'] = compute_macd(df['close'])

    rsi = df['RSI'].iloc[-1]
    macd_current = df['MACD'].iloc[-1]
    signal_current = df['Signal'].iloc[-1]

    advice = "محايد"
    if rsi < 30 and macd_current > signal_current:
        advice = "شراء (RSI منخفض و MACD إيجابي)"
    elif rsi > 70 and macd_current < signal_current:
        advice = "بيع (RSI مرتفع و MACD سلبي)"

    return {
        'price': df['close'].iloc[-1],
        'rsi': round(rsi, 2),
        'macd': round(macd_current, 4),
        'signal_line': round(signal_current, 4),
        'advice': advice
    }

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("مرحبًا! أرسل /signal للحصول على تحليل BTC/USDT.")

async def signal(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        data = get_trade_signal()
        msg = (
            f"**تحليل BTC/USDT:**\n"
            f"السعر الحالي: {data['price']}\n"
            f"RSI: {data['rsi']}\n"
            f"MACD: {data['macd']}\n"
            f"خط الإشارة: {data['signal_line']}\n\n"
            f"**التوصية: {data['advice']}**"
        )
        await update.message.reply_text(msg, parse_mode="Markdown")
    except Exception as e:
        await update.message.reply_text(f"حدث خطأ أثناء التحليل: {str(e)}")

app = ApplicationBuilder().token(API_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("signal", signal))
app.run_polling()
