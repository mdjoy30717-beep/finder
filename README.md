import sqlite3
import logging
from telegram import Update, ReplyKeyboardMarkup, InlineKeyboardMarkup, InlineKeyboardButton
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler, filters, 
    ContextTypes, ConversationHandler, CallbackQueryHandler
)

# Logging Setup
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# Config - আপনার দেওয়া তথ্য এখানে যুক্ত করা হয়েছে
BOT_TOKEN = "8151927272:AAGfDSR2jKVffDm8ptVTg9VijT_DDQO6kAM"
ADMIN_ID = 8151927272  # Admin: arivan, joy (@bd_joy_trader)

# Database Setup
DB_NAME = 'elaka_sheba.db'

def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    # Users & Providers Table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            name TEXT,
            phone TEXT,
            is_provider INTEGER DEFAULT 0,
            profession TEXT,
            address TEXT,
            is_available INTEGER DEFAULT 1,
            is_verified INTEGER DEFAULT 0,
            is_banned INTEGER DEFAULT 0
        )
    ''')
    # Favorites Table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS favorites (
            user_id INTEGER,
            provider_id INTEGER,
            PRIMARY KEY (user_id, provider_id)
        )
    ''')
    conn.commit()
    conn.close()

init_db()

# States
REG_NAME, REG_PROFESSION, REG_PHONE, REG_ADDRESS = range(4)

def is_banned(user_id):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT is_banned FROM users WHERE user_id = ?', (user_id,))
    res = cursor.fetchone()
    conn.close()
    return res and res[0] == 1

def get_main_keyboard():
    return ReplyKeyboardMarkup([
        ["🔍 সেবা খুঁজুন", "📝 সেবা দিতে নিবন্ধন"],
        ["👤 আমার প্রোফাইল", "⭐ ফেভারিট সার্ভিস"],
        ["📦 আমার অর্ডারসমূহ", "📢 হেল্প ও সাপোর্ট"],
        ["⚙️ অ্যাকাউন্ট সেটিংস", "ℹ️ আমাদের সম্পর্কে"]
    ], resize_keyboard=True)

# /start Command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if is_banned(user_id):
        await update.message.reply_text("⛔ আপনার অ্যাকাউন্টটি সাময়িকভাবে স্থগিত করা হয়েছে।")
        return

    await update.message.reply_text(
        "👋 **'এলাকা সেবা'** বটে আপনাকে স্বাগতম!\n\n"
        "আপনার প্রয়োজনীয় সেবা পেতে বা সেবা প্রদানকারী হিসেবে কাজ পেতে নিচের বাটন বেছে নিন।",
        reply_markup=get_main_keyboard(),
        parse_mode='Markdown'
    )

# Profile Handler
async def show_profile(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if is_banned(user_id): return

    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT name, phone, is_provider, profession, address, is_available, is_verified FROM users WHERE user_id = ?', (user_id,))
    user = cursor.fetchone()
    conn.close()

    if not user or user[2] == 0:
        msg = (
            "👤 **আপনার প্রোফাইল (গ্রাহক)**\n\n"
            f"🆔 ID: `{user_id}`\n"
            "অ্যাকাউন্ট টাইপ: 👤 সাধারণ গ্রাহক\n"
            "স্ট্যাটাস: 🟢 সক্রিয়\n\n"
            "💡 *আপনি কি সেবা দিয়ে আয় করতে চান? '📝 সেবা দিতে নিবন্ধন' বাটনে চাপ দিন।*"
        )
        reply_markup = get_main_keyboard()
    else:
        name, phone, _, profession, address, is_available, is_verified = user
        badge = "✅ (Verified)" if is_verified else "⚪ (Unverified)"
        status_text = "🟢 এভেলেবল (কাজ নিচ্ছি)" if is_available else "🔴 ব্যস্ত (কাজ নিচ্ছি না)"
        
        msg = (
            f"👤 **আপনার প্রোফাইল (সেবাদাতা)** {badge}\n\n"
            f"👤 **নাম:** {name}\n"
            f"📞 **ফোন:** {phone}\n"
            f"💼 **পেশা:** {profession}\n"
            f"📍 **ঠিকানা:** {address}\n"
            f"📊 **কাজের স্ট্যাটাস:** {status_text}\n"
        )
        inline_btns = [
            [InlineKeyboardButton("🔄 স্ট্যাটাস পরিবর্তন (এভেলেবল/ব্যস্ত)", callback_data="toggle_status")]
        ]
        reply_markup = InlineKeyboardMarkup(inline_btns)

    await update.message.reply_text(msg, parse_mode='Markdown', reply_markup=reply_markup)

# Callback Query
async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id

    if query.data == "toggle_status":
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute('UPDATE users SET is_available = CASE WHEN is_available = 1 THEN 0 ELSE 1 END WHERE user_id = ?', (user_id,))
        conn.commit()
        conn.close()
        await query.edit_message_text("✅ আপনার কাজের স্ট্যাটাস পরিবর্তন করা হয়েছে!")

    elif query.data.startswith("fav_"):
        provider_id = int(query.data.split("_")[1])
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute('INSERT OR IGNORE INTO favorites VALUES (?, ?)', (user_id, provider_id))
        conn.commit()
        conn.close()
        await query.message.reply_text("⭐ ফেভারিট লিস্টে যোগ করা হয়েছে!")

# Registration Flow
async def reg_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if is_banned(user_id): return ConversationHandler.END

    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT is_provider FROM users WHERE user_id = ?', (user_id,))
    res = cursor.fetchone()
    conn.close()

    if res and res[0] == 1:
        await update.message.reply_text("আপনি ইতোমধ্যে একজন নিবন্ধিত সেবাদাতা!")
        return ConversationHandler.END

    await update.message.reply_text("আপনার পূর্ণ নাম লিখুন:")
    return REG_NAME

async def reg_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['name'] = update.message.text
    keyboard = [["🏗️ রাজমিস্ত্রি", "⚡ ইলেকট্রিশিয়ান"], ["🚰 প্লাম্বার", "🎨 রং মিস্ত্রি"]]
    await update.message.reply_text("আপনার পেশা নির্বাচন করুন:", reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True, one_time_keyboard=True))
    return REG_PROFESSION

async def reg_prof(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['profession'] = update.message.text.replace("🏗️ ", "").replace("⚡ ", "").replace("🚰 ", "").replace("🎨 ", "")
    await update.message.reply_text("আপনার মোবাইল নম্বর লিখুন:")
    return REG_PHONE

async def reg_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['phone'] = update.message.text
    await update.message.reply_text("আপনার এলাকার নাম/ঠিকানা লিখুন:")
    return REG_ADDRESS

async def reg_address(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    address = update.message.text
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO users (user_id, name, phone, is_provider, profession, address)
        VALUES (?, ?, ?, 1, ?, ?)
        ON CONFLICT(user_id) DO UPDATE SET
        name=excluded.name, phone=excluded.phone, is_provider=1, profession=excluded.profession, address=excluded.address
    ''', (user_id, context.user_data['name'], context.user_data['phone'], context.user_data['profession'], address))
    conn.commit()
    conn.close()

    await update.message.reply_text("✅ আপনার রেজিস্ট্রেশন সম্পন্ন হয়েছে!", reply_markup=get_main_keyboard())
    return ConversationHandler.END

# Search Logic
async def search_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [["🏗️ রাজমিস্ত্রি", "⚡ ইলেকট্রিশিয়ান"], ["🚰 প্লাম্বার", "🎨 রং মিস্ত্রি"]]
    await update.message.reply_text("কোন পেশার সেবা প্রয়োজন বেছে নিন:", reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True))

async def search_results(update: Update, context: ContextTypes.DEFAULT_TYPE):
    prof = update.message.text.replace("🏗️ ", "").replace("⚡ ", "").replace("🚰 ", "").replace("🎨 ", "")
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT user_id, name, phone, address, is_available, is_verified FROM users WHERE profession LIKE ? AND is_provider = 1 AND is_banned = 0', (f'%{prof}%',))
    results = cursor.fetchall()
    conn.close()

    if not results:
        await update.message.reply_text("দুঃখিত, এই পেশার কোনো সেবাদাতা পাওয়া যায়নি।")
        return

    for p in results:
        p_id, name, phone, addr, avail, ver = p
        badge = "✅ (Verified)" if ver else "⚪ (Unverified)"
        st = "🟢 এভেলেবল" if avail else "🔴 ব্যস্ত"
        
        card = (
            f"👷‍♂️ **{name}** {badge}\n"
            f"💼 পেশা: {prof}\n"
            f"📍 এলাকা: {addr}\n"
            f"📊 স্ট্যাটাস: {st}\n"
            f"📞 ফোন: `{phone}`"
        )
        btn = InlineKeyboardMarkup([[InlineKeyboardButton("⭐ ফেভারিটে রাখুন", callback_data=f"fav_{p_id}")]])
        await update.message.reply_text(card, parse_mode='Markdown', reply_markup=btn)

# Other Buttons
async def show_favorites(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT u.name, u.profession, u.phone FROM users u JOIN favorites f ON u.user_id = f.provider_id WHERE f.user_id = ?', (user_id,))
    favs = cursor.fetchall()
    conn.close()

    if not favs:
        await update.message.reply_text("আপনার ফেভারিট লিস্টে কেউ নেই।")
        return

    res = "⭐ **আপনার সেভ করা সেবাদাতাগণ:**\n\n"
    for f in favs:
        res += f"👤 {f[0]} ({f[1]})\n📞 `{f[2]}`\n-------------------\n"
    await update.message.reply_text(res, parse_mode='Markdown')

async def show_help(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = "📢 **হেল্প সেন্টার**\n\nযেকোনো সহায়তায় সরাসরি এডমিনকে নক দিন।"
    btn = InlineKeyboardMarkup([[InlineKeyboardButton("💬 এডমিনের সাথে কথা বলুন", url="https://t.me/bd_joy_trader")]])
    await update.message.reply_text(text, reply_markup=btn, parse_mode='Markdown')

async def show_orders(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("📦 আপনার সাম্প্রতিক অ্যাক্টিভিটি এখানে প্রদর্শিত হবে।")

async def show_settings(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("⚙️ তথ্য আপডেট করতে '📝 সেবা দিতে নিবন্ধন' এ গিয়ে পুনরায় ফর্ম পূরণ করুন।")

async def show_about(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("ℹ️ **এলাকা সেবা**\nলোকাল সেবাদাতা এবং গ্রাহকদের সরাসরি সংযোগের প্ল্যাটফর্ম।")

# Admin Controls
async def admin_ban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    try:
        target_id = int(context.args[0])
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute('UPDATE users SET is_banned = 1 WHERE user_id = ?', (target_id,))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ User `{target_id}` banned.")
    except:
        await update.message.reply_text("ব্যবহার: `/ban USER_ID`", parse_mode='Markdown')

async def admin_verify(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    try:
        target_id = int(context.args[0])
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute('UPDATE users SET is_verified = 1 WHERE user_id = ?', (target_id,))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ User `{target_id}` verified with Blue Tick!")
    except:
        await update.message.reply_text("ব্যবহার: `/verify USER_ID`", parse_mode='Markdown')

# Main Runner
if __name__ == '__main__':
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    reg_handler = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex('^📝 সেবা দিতে নিবন্ধন$'), reg_start)],
        states={
            REG_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, reg_name)],
            REG_PROFESSION: [MessageHandler(filters.TEXT & ~filters.COMMAND, reg_prof)],
            REG_PHONE: [MessageHandler(filters.TEXT & ~filters.COMMAND, reg_phone)],
            REG_ADDRESS: [MessageHandler(filters.TEXT & ~filters.COMMAND, reg_address)],
        },
        fallbacks=[]
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("ban", admin_ban))
    app.add_handler(CommandHandler("verify", admin_verify))
    app.add_handler(reg_handler)
    
    app.add_handler(MessageHandler(filters.Regex('^👤 আমার প্রোফাইল$'), show_profile))
    app.add_handler(MessageHandler(filters.Regex('^🔍 সেবা খুঁজুন$'), search_start))
    app.add_handler(MessageHandler(filters.Regex('^(🏗️ রাজমিস্ত্রি|⚡ ইলেকট্রিশিয়ান|🚰 প্লাম্বার|🎨 রং মিস্ত্রি)$'), search_results))
    app.add_handler(MessageHandler(filters.Regex('^⭐ ফেভারিট সার্ভিস$'), show_favorites))
    app.add_handler(MessageHandler(filters.Regex('^📢 হেল্প ও সাপোর্ট$'), show_help))
    app.add_handler(MessageHandler(filters.Regex('^📦 আমার অর্ডারসমূহ$'), show_orders))
    app.add_handler(MessageHandler(filters.Regex('^⚙️ অ্যাকাউন্ট সেটিংস$'), show_settings))
    app.add_handler(MessageHandler(filters.Regex('^ℹ️ আমাদের সম্পর্কে$'), show_about))
    
    app.add_handler(CallbackQueryHandler(button_callback))

    print("Bot is running...")
    app.run_polling()
