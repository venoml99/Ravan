import sqlite3
import logging
from telegram import Update
from telegram.constants import ChatType
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
from threading import Lock
from html import escape

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

BOT_TOKEN = "8328609591:AAEEOt9iN8f0B-74TrG9Jx2ZYQdGPcmIH4k"
ADMIN_ID = 5401358805

conn = sqlite3.connect("stats.db", check_same_thread=False)
cur = conn.cursor()
cur.execute('''
    CREATE TABLE IF NOT EXISTS stats (
        chat_id     INTEGER,
        user_id     INTEGER,
        user_name   TEXT,
        msg_count   INTEGER DEFAULT 0,
        hours_count INTEGER DEFAULT 0,
        PRIMARY KEY (chat_id, user_id)
    )
''')
conn.commit()

db_lock = Lock()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    chat = update.effective_chat
    user = update.effective_user
    dev_info = (
        "👋 مرحبًا! أنا بوت تتبع النشاط.\n"
        "أقوم بحساب عدد الرسائل وساعات الصوت لكل عضو في المجموعة.\n"
        "تم تطويري بواسطة: @VENOM_L99"
    )
    if chat.type == ChatType.PRIVATE:
        await context.bot.send_message(chat_id=chat.id, text=dev_info)
    else:
        if user:
            try:
                await context.bot.send_message(chat_id=user.id, text=dev_info)
                await update.message.reply_text("✅ تم إرسال معلومات المطور في الخاص.")
            except Exception:
                await update.message.reply_text("تم تطوير هذا البوت بواسطة: @VENOM_L99")

async def count_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    user = update.effective_user
    chat = update.effective_chat
    if user is None or user.is_bot:
        return
    chat_id = chat.id
    user_id = user.id
    user_name = "@" + user.username if user.username else user.full_name
    with db_lock:
        cur.execute("""
            INSERT INTO stats(chat_id, user_id, user_name, msg_count, hours_count)
            VALUES (?, ?, ?, 1, 0)
            ON CONFLICT(chat_id, user_id) DO UPDATE 
            SET msg_count = msg_count + 1,
                user_name = excluded.user_name
        """, (chat_id, user_id, user_name))
        conn.commit()

async def add_hours(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    user = update.effective_user
    chat = update.effective_chat
    if user.id != ADMIN_ID:
        await message.reply_text("⚠️ هذا الأمر متاح للمطور فقط.")
        return
    target = message.reply_to_message.from_user if message.reply_to_message else None
    args = context.args
    if not target or len(args) < 1:
        await message.reply_text("استخدم /add_hours بالرد على رسالة أو مع معرف المستخدم وعدد الساعات.")
        return
    try:
        hours = int(args[0])
    except ValueError:
        await message.reply_text("عدد الساعات يجب أن يكون رقمًا صحيحًا.")
        return
    user_id = target.id
    user_name = "@" + target.username if target.username else target.full_name
    with db_lock:
        cur.execute("""
            INSERT INTO stats(chat_id, user_id, user_name, msg_count, hours_count)
            VALUES (?, ?, ?, 0, ?)
            ON CONFLICT(chat_id, user_id) DO UPDATE 
            SET hours_count = hours_count + ?,
                user_name = excluded.user_name
        """, (chat.id, user_id, user_name, hours, hours))
        conn.commit()
    await message.reply_text(f"✅ تم إضافة {hours} ساعة لـ {user_name}.")

async def reset_user(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    user = update.effective_user
    chat = update.effective_chat
    if user.id != ADMIN_ID:
        await message.reply_text("⚠️ هذا الأمر متاح للمطور فقط.")
        return
    target = message.reply_to_message.from_user if message.reply_to_message else None
    if not target:
        await message.reply_text("يجب الرد على رسالة المستخدم المراد تصفيره.")
        return
    user_id = target.id
    with db_lock:
        cur.execute("UPDATE stats SET msg_count=0, hours_count=0 WHERE chat_id=? AND user_id=?", (chat.id, user_id))
        conn.commit()
    await message.reply_text(f"🔄 تم تصفير بيانات {target.full_name}.")

async def trend_messages(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    chat_id = update.effective_chat.id
    with db_lock:
        cur.execute("""
            SELECT user_id, user_name, msg_count FROM stats
            WHERE chat_id=? AND msg_count > 0
            ORDER BY msg_count DESC LIMIT 10
        """, (chat_id,))
        rows = cur.fetchall()
    if not rows:
        await update.message.reply_text("لا توجد بيانات رسائل بعد.")
        return
    text = "📊 <b>ترتيب الأعضاء حسب عدد الرسائل:</b>\n"
    for i, (uid, name, count) in enumerate(rows, 1):
        mention = f'<a href="tg://user?id={uid}">{escape(name)}</a>'
        text += f"{i}. {mention} - {count} رسالة\n"
    await update.message.reply_text(text, parse_mode='HTML')

async def trend_hours(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    chat_id = update.effective_chat.id
    with db_lock:
        cur.execute("""
            SELECT user_id, user_name, hours_count FROM stats
            WHERE chat_id=? AND hours_count > 0
            ORDER BY hours_count DESC LIMIT 10
        """, (chat_id,))
        rows = cur.fetchall()
    if not rows:
        await update.message.reply_text("لا توجد بيانات ساعات مايك بعد.")
        return
    text = "🎙️ <b>ترتيب الأعضاء حسب ساعات المايك:</b>\n"
    for i, (uid, name, count) in enumerate(rows, 1):
        mention = f'<a href="tg://user?id={uid}">{escape(name)}</a>'
        text += f"{i}. {mention} - {count} ساعة\n"
    await update.message.reply_text(text, parse_mode='HTML')

def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("add_hours", add_hours))
    app.add_handler(CommandHandler("reset_user", reset_user))
    app.add_handler(CommandHandler("trend_messages", trend_messages))
    app.add_handler(CommandHandler("trend_hours", trend_hours))
    app.add_handler(MessageHandler(filters.ALL, count_message))
    app.run_polling()

if __name__ == "__main__":
    main()
