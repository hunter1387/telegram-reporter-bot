import asyncio
import os
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, ConversationHandler, filters
from telethon.sync import TelegramClient
from telethon.tl.types import (
    InputReportReasonSpam,
    InputReportReasonFake,
    InputReportReasonViolence,
    InputReportReasonPornography,
    InputReportReasonOther,
)

API_ID = int(os.environ.get("API_ID"))
API_HASH = os.environ.get("API_HASH")
PHONE = os.environ.get("PHONE")
BOT_TOKEN = os.environ.get("BOT_TOKEN")

reason_map = {
    "spam": InputReportReasonSpam(),
    "fake": InputReportReasonFake(),
    "violence": InputReportReasonViolence(),
    "porn": InputReportReasonPornography(),
    "other": InputReportReasonOther()
}

# مراحل مکالمه
USERNAME, REASON, COUNT = range(3)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("آیدی هدف (بدون @) را بفرست:")
    return USERNAME

async def username_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['username'] = update.message.text.strip().replace("@", "")
    await update.message.reply_text("دلیل ریپورت (spam, fake, violence, porn, other):")
    return REASON

async def reason_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    reason = update.message.text.strip().lower()
    if reason not in reason_map:
        await update.message.reply_text("❌ دلیل معتبر نیست.")
        return REASON
    context.user_data['reason'] = reason
    await update.message.reply_text("چند پست آخر را ریپورت کنیم؟ (مثلاً 50):")
    return COUNT

async def count_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    count = int(update.message.text.strip())
    username = context.user_data['username']
    reason = context.user_data['reason']

    await update.message.reply_text("⏳ شروع ریپورت...")

    client = TelegramClient("session", API_ID, API_HASH)
    await client.start(phone=PHONE)

    try:
        entity = await client.get_entity(username)
        messages = await client.get_messages(entity, limit=count)
        success = 0
        for i, msg in enumerate(messages, 1):
            try:
                await client.report_messages(entity, [msg.id], reason_map[reason], f"Reported via bot")
                success += 1
            except Exception as e:
                await update.message.reply_text(f"خطا در پست {i}: {e}")
        await update.message.reply_text(f"😽 تمام شد. تعداد موفق: {success}/{count}")
    finally:
        await client.disconnect()
    return ConversationHandler.END

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("❌ لغو شد.")
    return ConversationHandler.END

def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    conv = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            USERNAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, username_handler)],
            REASON: [MessageHandler(filters.TEXT & ~filters.COMMAND, reason_handler)],
            COUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, count_handler)],
        },
        fallbacks=[CommandHandler("cancel", cancel)]
    )

    app.add_handler(conv)
    print("🏳️ Bot is running...")
    app.run_polling()

if __name__ == "__main__":
    main()
