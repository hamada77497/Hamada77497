import requests
from datetime import datetime, timedelta
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext

# --- إعدادات البوت ---
TELEGRAM_TOKEN = "7910861424:AAFERW23drlQh3cYVkOIWKBuN8NwN4PG2u0"  # التوكن الخاص ببوت Telegram

CONFIG = {
    "gmgn_api_url": "https://api.gmgn.com/v1/tokens",   # رابط API الخاص بـ GMGN
    "gmgn_api_key": "your_gmgn_api_key",               # مفتاح API الخاص بـ GMGN
    "rugcheck_api_url": "https://api.rugcheck.xyz/v1", # رابط API الخاص بـ Rugcheck
    "tweetscout_api_url": "https://api.tweetscout.io/v1",  # رابط API الخاص بـ TweetScout
    "tweetscout_api_key": "your_tweetscout_api_key",   # مفتاح API الخاص بـ TweetScout
    "bubblemaps_api_url": "https://api.bubblemaps.io/v1",  # رابط API الخاص بـ BubbleMaps
    "bubblemaps_api_key": "your_bubblemaps_api_key",   # مفتاح API الخاص بـ BubbleMaps
    "criteria": {
        "liquidity_limit": 100_000,
        "volume_limit": 200_000,
        "age_limit_hours": 24,
        "holders_limit": 400,
    }
}

# --- الوظائف ---
def fetch_tokens():
    """جلب بيانات التوكنز من GMGN."""
    headers = {"Authorization": f"Bearer {CONFIG['gmgn_api_key']}"}
    try:
        response = requests.get(CONFIG["gmgn_api_url"], headers=headers)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching tokens: {e}")
        return []


def filter_tokens(tokens):
    """فلترة التوكنز حسب المعايير."""
    filtered_tokens = []
    current_time = datetime.utcnow()
    for token in tokens:
        try:
            liquidity = token.get("liquidity", 0)
            volume = token.get("volume", 0)
            launch_date = datetime.fromisoformat(token.get("launch_date", ""))
            holders = token.get("holders", 0)

            if (
                liquidity < CONFIG["criteria"]["liquidity_limit"]
                and volume < CONFIG["criteria"]["volume_limit"]
                and current_time - launch_date > timedelta(hours=CONFIG["criteria"]["age_limit_hours"])
                and holders < CONFIG["criteria"]["holders_limit"]
            ):
                filtered_tokens.append(token["contract_address"])
        except Exception as e:
            print(f"Error processing token: {e}")
    return filtered_tokens


def check_contract_safety(contract_address):
    """التحقق من سلامة العقد باستخدام Rugcheck."""
    try:
        response = requests.get(f"{CONFIG['rugcheck_api_url']}/check/{contract_address}")
        response.raise_for_status()
        return response.json().get("safe", False)
    except requests.exceptions.RequestException as e:
        print(f"Error checking contract safety: {e}")
        return False


def analyze_twitter(twitter_handle):
    """تحليل حساب تويتر باستخدام TweetScout."""
    headers = {"Authorization": f"Bearer {CONFIG['tweetscout_api_key']}"}
    try:
        response = requests.get(f"{CONFIG['tweetscout_api_url']}/analyze/{twitter_handle}", headers=headers)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error analyzing Twitter account: {e}")
        return {"error": str(e)}


def analyze_bubblemaps(contract_address, chain="ethereum"):
    """تحليل التوكن باستخدام BubbleMaps."""
    headers = {"Authorization": f"Bearer {CONFIG['bubblemaps_api_key']}"}
    try:
        response = requests.get(f"{CONFIG['bubblemaps_api_url']}/{chain}/tokens/{contract_address}/holders", headers=headers)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error analyzing token holders: {e}")
        return {"error": str(e)}


def process_tokens(update: Update, context: CallbackContext):
    """تنفيذ عملية التحليل الكاملة وإرسال النتائج للمستخدم."""
    update.message.reply_text("Fetching tokens and analyzing...")

    # Step 1: Fetch and filter tokens
    tokens = fetch_tokens()
    filtered_tokens = filter_tokens(tokens)

    if not filtered_tokens:
        update.message.reply_text("No tokens found matching criteria.")
        return

    for contract_address in filtered_tokens:
        # Step 2: Check contract safety
        if not check_contract_safety(contract_address):
            update.message.reply_text(f"Contract {contract_address} is unsafe. Skipping.")
            continue

        # Step 3: Analyze Twitter account (Example)
        twitter_handle = "YourTokenTwitterHandle"  # Replace with dynamic fetching logic
        twitter_analysis = analyze_twitter(twitter_handle)

        # Step 4: Analyze holders
        holder_analysis = analyze_bubblemaps(contract_address)

        # Step 5: Send result
        message = (
            f"Safe Contract Found: {contract_address}\n"
            f"Twitter Analysis: {twitter_analysis}\n"
            f"Holder Analysis: {holder_analysis}\n"
            f"Proceed with sniper bot: t.me/toxi_solana_bo"
        )
        update.message.reply_text(message)


# --- إعداد البوت ---
def main():
    updater = Updater(TELEGRAM_TOKEN)
    dispatcher = updater.dispatcher

    # أوامر البوت
    dispatcher.add_handler(CommandHandler("start", lambda update, _: update.message.reply_text("Welcome! Use /analyze to start.")))
    dispatcher.add_handler(CommandHandler("analyze", process_tokens))

    # تشغيل البوت
    updater.start_polling()
    updater.idle()


if __name__ == "__main__":
    main()
    
