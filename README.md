import nest_asyncio
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes

nest_asyncio.apply()

# نسبة الربح الشهرية
PROFIT_RATE = 0.24

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text("مرحبًا! أدخل المبلغ المراد استثماره:")

async def calculate_profit(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    try:
        # قراءة المبلغ المدخل
        initial_amount = float(update.message.text)
        current_amount = initial_amount
        total_profit = 0

        # بناء الرسالة للرد
        message = f"المبلغ المراد استثماره: {initial_amount}\n\n"

        # حساب الأرباح لكل شهر
        for month in range(1, 13):
            profit = current_amount * PROFIT_RATE
            total_profit += profit
            current_amount += profit

            message += f"نهاية الشهر {month}: {current_amount:.2f} (الربح + رأس المال)\n"
            message += f"صافي الربح = {profit:.2f} دولار\n\n"

        message += f"بعد 12 شهر:\n"
        message += f"إجمالي رأس المال = {current_amount:.2f}\n"
        message += f"الأرباح المقدرة منه = {total_profit:.2f}\n"

        await update.message.reply_text(message)

    except ValueError:
        await update.message.reply_text("يرجى إدخال مبلغ صحيح.")

def main() -> None:
    # إعداد البوت بالتوكن الخاص بك
    application = ApplicationBuilder().token("170f738efea039931334926bca4c7a6f341a74ad").build()

    # إعداد الأوامر
    application.add_handler(CommandHandler("start", start))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, calculate_profit))

    # بدء التشغيل
    application.run_polling()

if __name__ == '__main__':
    main()
