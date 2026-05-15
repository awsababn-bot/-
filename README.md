import time
import requests
import ccxt
import pandas as pd
import numpy as np
from scipy.signal import find_peaks

# ==========================================
# ⚙️ إعدادات الحساب والتليجرام
# ==========================================
TELEGRAM_TOKEN = 8967464976:AAHmiIASX6OmtfV6VdfXHQ6lSeS4wytHVRc
CHAT_ID = 1382620861
TIMEFRAME =  4h                              # الإطار الزمني (1h, 4h, 1d)
CHECK_INTERVAL = 900                         # مدة الفحص (900 ثانية = 15 دقيقة)

# قائمة العملات المراد مراقبتها
SYMBOLS = [ BTC/USDT ,  ETH/USDT ,  SOL/USDT ,  BNB/USDT ]

# إعداد الاتصال بالمنصة (بينانس كمثال)
exchange = ccxt.binance({
     enableRateLimit : True,
})

# ==========================================
# ✉️ دالة إرسال التنبيهات لتليجرام
# ==========================================
def send_telegram_message(message):
    url = f"https://api.telegram.com/bot{TELEGRAM_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": message, "parse_mode": "Markdown"}
    try:
        response = requests.post(url, json=payload)
        if response.status_code != 200:
            print(f"❌ خطأ في إرسال التنبيه: {response.text}")
    except Exception as e:
        print(f"❌ حدث خطأ أثناء الاتصال بتليجرام: {e}")

# ==========================================
# 📊 دالة جلب بيانات السوق
# ==========================================
def get_crypto_data(symbol, timeframe=TIMEFRAME, limit=200):
    try:
        bars = exchange.fetch_ohlcv(symbol, timeframe=timeframe, limit=limit)
        df = pd.DataFrame(bars, columns=[ timestamp ,  open ,  high ,  low ,  close ,  volume ])
        return df
    except Exception as e:
        print(f"❌ فشل جلب بيانات {symbol}: {e}")
        return None

# ==========================================
# 🔍 خوارزميات اكتشاف الأنماط الفنية
# ==========================================

def check_double_bottom(df, symbol):
    """اكتشاف نمط القاع المزدوج (انعكاسي صاعد)"""
    lows = df[ low ].values
    closes = df[ close ].values
    
    for i in range(len(lows) - 20, len(lows) - 2):
        p1 = lows[i]
        for j in range(i + 5, len(lows) - 2):
            p2 = lows[j]
            # سماحية بنسبة 0.5% لتساوي القاعين
            if abs(p1 - p2) / p1 < 0.005:
                intermediate_high = max(df[ high ].iloc[i:j])
                current_close = closes[-1]
                
                # شرط الكسر لأعلى
                if current_close > intermediate_high:
                    return f"🟢 إشارة صعود: قاع مزدوج (Double Bottom) 🟢\n\n" \
                           f"الزوج: {symbol}\n" \
                           f"سعر الاختراق الحركي: {current_close}\n" \
                           f"خط العنق المخترق: {intermediate_high}"
    return None

def check_double_top(df, symbol):
    """اكتشاف نمط القمة المزدوجة (انعكاسي هابط)"""
    highs = df[ high ].values
    closes = df[ close ].values
    lows = df[ low ].values
    
    peaks, _ = find_peaks(highs, distance=5)
    if len(peaks) < 2:
        return None
        
    p1_idx, p2_idx = peaks[-2], peaks[-1]
    p1_price, p2_price = highs[p1_idx], highs[p2_idx]
    
    if abs(p1_price - p2_price) / p1_price <= 0.005:
        intermediate_low = min(lows[p1_idx:p2_idx])
        current_close = closes[-1]
        
        # شرط الكسر لأسفل
        if current_close < intermediate_low:
            return f"🔴 إشارة هبوط: قمة مزدوجة (Double Top) 🔴\n\n" \
                   f"الزوج: {symbol}\n" \
                   f"سعر كسر الدعم: {current_close}\n" \
                   f"خط العنق المكسور: {intermediate_low}"
    return None

def check_head_and_shoulders(df, symbol):
    """اكتشاف نمط الرأس والكتفين التقليدي (انعكاسي هابط)"""
    highs = df[ high ].values
    closes = df[ close ].values
    lows = df[ low ].values
    
    peaks, _ = find_peaks(highs, distance=5)
    if len(peaks) < 3:
        return None
        
    ls_idx, h_idx, rs_idx = peaks[-3], peaks[-2], peaks[-1]
    ls_price, h_price, rs_price = highs[ls_idx], highs[h_idx], highs[rs_idx]
    
    # الرأس أعلى من الكتفين، والكتفان متقاربان بنسبة سماحية 1%
    if h_price > ls_price and h_price > rs_price:
        if abs(ls_price - rs_price) / ls_price <= 0.01:
            low1 = min(lows[ls_idx:h_idx])
            low2 = min(lows[h_idx:rs_idx])
            neckline = (low1 + low2) / 2
            current_close = closes[-1]
            
            if current_close < neckline:
                return f"⚠️ إشارة هبوط قوية: رأس وكتفين (Head & Shoulders) ⚠️\n\n" \
                       f"الزوج: {symbol}\n" \
                       f"السعر الحالي: {current_close}\n" \
                       f"متوسط خط العنق المكسور: {neckline}"
    return None

# ==========================================
# 🚀 الحلقة التكرارية لتشغيل البوت
# ==========================================
def monitor_market():
    print("🤖 البوت تم تشغيله بنجاح ويراقب الأسواق الآن...")
    send_telegram_message("🚀 تم تشغيل بوت مراقبة الأنماط الفنية بنجاح وبدء مراقبة الأسواق!")
    
    while True:
        for symbol in SYMBOLS:
            df = get_crypto_data(symbol)
            if df is None or df.empty:
                continue
                
            # 1. فحص القاع المزدوج
            alert = check_double_bottom(df, symbol)
            if alert:
                send_telegram_message(alert)
                time.sleep(2) # حماية من الحظر المؤقت لتليجرام
                
            # 2. فحص القمة المزدوجة
            alert = check_double_top(df, symbol)
            if alert:
                send_telegram_message(alert)
                time.sleep(2)
                
            # 3. فحص الرأس والكتفين
            alert = check_head_and_shoulders(df, symbol)
            if alert:
                send_telegram_message(alert)
                time.sleep(2)
                
        # إيقاف مؤقت قبل الفحص التالي
        print(f"💤 فحص مكتمل. انتظار {CHECK_INTERVAL // 60} دقائق للفحص القادم...")
        time.sleep(CHECK_INTERVAL)

if name == "main":
    monitor_market()
    
