import nest_asyncio
nest_asyncio.apply()

from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes, ConversationHandler

# تعريف الخطوات في المحادثة
PROJECT, QUANTITY, AMOUNT, DATE, DIESEL_AMOUNT, DIESEL_QUANTITY, OPERATION_TYPE, OPERATION_NUMBER = range(8)

# تخزين البيانات
sales_data = []
diesel_purchases = []
workers_wages = 0

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text('برنامج الإدخال الآلي لمشروع البيتين.\nاختر المشروع (البير أو السد):')
    return PROJECT

async def handle_project_choice(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    project = update.message.text
    if project in ['البير', 'السد']:
        context.user_data['current_project'] = project
        await update.message.reply_text('ادخل الكمية:')
        return QUANTITY
    else:
        await update.message.reply_text('يرجى اختيار (البير أو السد).')
        return PROJECT

async def enter_quantity(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        quantity = int(update.message.text)
        context.user_data['quantity'] = quantity
        await update.message.reply_text('ادخل المبلغ:')
        return AMOUNT
    except ValueError:
        await update.message.reply_text('يرجى إدخال الكمية بشكل صحيح.')
        return QUANTITY

async def enter_amount(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        amount = float(update.message.text)
        context.user_data['amount'] = amount
        await update.message.reply_text('ادخل التاريخ (يوم/شهر/سنة):')
        return DATE
    except ValueError:
        await update.message.reply_text('يرجى إدخال المبلغ بشكل صحيح.')
        return AMOUNT

async def enter_date(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    date = update.message.text
    project = context.user_data['current_project']
    quantity = context.user_data['quantity']
    amount = context.user_data['amount']

    sales_data.append((project, quantity, amount, date))
    invoice = f"""
    برنامج الإدخال الآلي لمشروع البيتين
    عملية رقم {len(sales_data)}
    المشروع: {project}
    بيع عدد {quantity} وحدات
    الإجمالي: {amount}
    التاريخ: {date}
    """
    await update.message.reply_text(f'تم إضافة العملية:\n{invoice}')
    return ConversationHandler.END

async def add_diesel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text('ادخل المبلغ:')
    return DIESEL_AMOUNT

async def enter_diesel_amount(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        amount = float(update.message.text)
        context.user_data['diesel_amount'] = amount
        await update.message.reply_text('ادخل الكمية (لتر):')
        return DIESEL_QUANTITY
    except ValueError:
        await update.message.reply_text('يرجى إدخال المبلغ بشكل صحيح.')
        return DIESEL_AMOUNT

async def enter_diesel_quantity(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        quantity = int(update.message.text)
        amount = context.user_data['diesel_amount']
        diesel_purchases.append((quantity, amount))
        diesel_invoice = f"""
        برنامج الإدخال الآلي لمشروع البيتين
        مشتروات الديزل
        الكمية: {quantity} لتر
        المبلغ: {amount}
        """
        await update.message.reply_text(f'تم إضافة مشتروات الديزل:\n{diesel_invoice}')
        return ConversationHandler.END
    except ValueError:
        await update.message.reply_text('يرجى إدخال الكمية بشكل صحيح.')
        return DIESEL_QUANTITY

async def remove_operation(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text('اكتب رقم العملية المراد حذفها:')
    return OPERATION_NUMBER

async def operation_number(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        context.user_data['operation_number'] = int(update.message.text) - 1
        await update.message.reply_text('هل هي بيع، شراء، أو أجور؟')
        return OPERATION_TYPE
    except ValueError:
        await update.message.reply_text('يرجى إدخال رقم عملية صالح.')
        return OPERATION_NUMBER

async def operation_type(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    operation_type = update.message.text.lower()
    index = context.user_data['operation_number']
    
    if operation_type == 'بيع' and 0 <= index < len(sales_data):
        removed = sales_data.pop(index)
        await update.message.reply_text(f"تم حذف عملية البيع رقم {index + 1}: {removed}")
    elif operation_type == 'شراء' and 0 <= index < len(diesel_purchases):
        removed = diesel_purchases.pop(index)
        await update.message.reply_text(f"تم حذف عملية الشراء رقم {index + 1}: {removed}")
    elif operation_type == 'أجور':
        await update.message.reply_text("لا يوجد سجل فردي للأجور لحذفه، يمكنك تعديل إجمالي الأجور بدلاً من ذلك.")
    else:
        await update.message.reply_text("نوع العملية أو رقم العملية غير صالح.")
    
    return ConversationHandler.END

async def total_report(update: Update, context: ContextTypes.DEFAULT_TYPE):
    month = context.args[0] if context.args else "all"
    report = f"برنامج الإدخال الآلي لمشروع البيتين\nتقرير مشروع البيتين للمياة شهر({month}):\n"
    
    report += "\nالمبيعات:\nمبيعات البير:\n"
    total_well_sales = sum(sale[1] for sale in sales_data if sale[0] == 'البير')
    total_well_amount = sum(sale[2] for sale in sales_data if sale[0] == 'البير')
    report += f" {total_well_sales} وحدات\n الإجمالي: {total_well_amount}\n"

    report += "مبيعات السد:\n"
    total_dam_sales = sum(sale[1] for sale in sales_data if sale[0] == 'السد')
    total_dam_amount = sum(sale[2] for sale in sales_data if sale[0] == 'السد')
    report += f" {total_dam_sales} وحدات\n الإجمالي: {total_dam_amount}\n"

    total_sales = total_well_sales + total_dam_sales
    total_amount = total_well_amount + total_dam_amount
    report += f"\nالبيع: {total_sales} وحدات ب{total_amount}\n"

    free_sales = sum(1 for sale in sales_data if sale[2] == 0)
    report += f"المبيع مجاني: {free_sales}\n"

    total_diesel_quantity = sum(diesel[0] for diesel in diesel_purchases)
    total_diesel_amount = sum(diesel[1] for diesel in diesel_purchases)
    report += f"\nالمشتروات:\n{total_diesel_quantity} لتر ديزل الإجمالي: {total_diesel_amount}\n"

    report += f"\nأجور العمال: {workers_wages}\n"

    net_sales = total_amount - total_diesel_amount - workers_wages
    report += f"\nالصافي: {net_sales}\n"

    await update.message.reply_text(report)

async def set_wage(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global workers_wages
    try:
        wage = float(context.args[0])
        workers_wages += wage
        await update.message.reply_text(f"تم إضافة أجور العمال: {wage}. الإجمالي الآن: {workers_wages}.")
    except (IndexError, ValueError):
        await update.message.reply_text("يرجى تقديم المبلغ الصحيح بعد الأمر.")

def main() -> None:
    application = ApplicationBuilder().token("8136003037:AAFAVevARncCcQsZGCL1enhGxhaJ0wh5eSQ").build()

    # إعداد معالج المحادثات للمبيعات
    sale_conversation_handler = ConversationHandler(
        entry_points=[CommandHandler("add", start)],
        states={
            PROJECT: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_project_choice)],
            QUANTITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, enter_quantity)],
            AMOUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, enter_amount)],
            DATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, enter_date)],
        },
        fallbacks=[CommandHandler("start", start)],
    )

    # إعداد معالج المحادثات لمشتروات الديزل
    diesel_conversation_handler = ConversationHandler(
        entry_points=[CommandHandler("diesel", add_diesel)],
        states={
            DIESEL_AMOUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, enter_diesel_amount)],
            DIESEL_QUANTITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, enter_diesel_quantity)],
        },
        fallbacks=[CommandHandler("start", start)],
    )

    # إعداد معالج المحادثات لحذف العمليات
    remove_conversation_handler = ConversationHandler(
        entry_points=[CommandHandler("remove", remove_operation)],
        states={
            OPERATION_NUMBER: [MessageHandler(filters.TEXT & ~filters.COMMAND, operation_number)],
            OPERATION_TYPE: [MessageHandler(filters.TEXT & ~filters.COMMAND, operation_type)],
        },
        fallbacks=[CommandHandler("start", start)],
    )

    application.add_handler(sale_conversation_handler)
    application.add_handler(diesel_conversation_handler)
    application.add_handler(remove_conversation_handler)

    # الأوامر الإضافية
    application.add_handler(CommandHandler('total', total_report))
    application.add_handler(CommandHandler('wage', set_wage))

    application.run_polling()

if __name__ == '__main__':
    main()
