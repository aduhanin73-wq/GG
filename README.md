import json
import os
import re
import logging
from datetime import datetime, timedelta, time
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, ContextTypes, filters
from apscheduler.schedulers.asyncio import AsyncIOScheduler
import pytz

# ===== НАСТРОЙКИ =====
BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"  # ← ОБЯЗАТЕЛЬНО ЗАМЕНИ!
TIMEZONE = "Europe/Moscow"  # Измени, если нужно
MORNING_HOUR = 8  # Утреннее уведомление в 8:00 по местному времени

DATA_FILE = "data/users.json"

# ===== МИЛЫЕ ОБРАЩЕНИЯ =====
CUTE_NAMES = [
    "зайка", "солнышко", "пушистик", "светик", "радость моя",
    "мишка", "звёздочка", "котёнок", "ангел", "счастье моё"
]

# --- Загрузка данных ---
def load_data():
    if not os.path.exists(DATA_FILE):
        return {}
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception as e:
        print("Ошибка загрузки:", e)
        return {}

# --- Сохранение данных ---
def save_data(data):
    os.makedirs("data", exist_ok=True)
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

# --- /start ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    first_name = update.effective_user.first_name or "друг"
    data = load_data()
    if user_id not in 
        data[user_id] = {"notes": [], "entries": [], "reminders": []}
        save_data(data)
    await update.message.reply_text(
        f"Привет, {first_name}! 🌞\n"
        "Я буду:\n"
        "• Присылать утренние приветствия с ласковыми словами\n"
        "• Сохранять заметки и благодарности\n"
        "• Напоминать по команде /напомни\n\n"
        "Команды:\n"
        "/note <текст> — заметка\n"
        "/notes — показать заметки\n"
        "/напомни ЧЧ:ММ Текст — напоминание"
    )

# --- /note ---
async def note(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    data = load_data()
    if user_id not in 
        await update.message.reply_text("Сначала напиши /start")
        return
    if not context.args:
        await update.message.reply_text("Используй: /note Текст заметки")
        return
    text = " ".join(context.args)
    data[user_id]["notes"].append(text)
    save_data(data)
    await update.message.reply_text("✅ Заметка сохранена!")

# --- /notes ---
async def notes(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    data = load_data()
    if user_id not in 
        await update.message.reply_text("Сначала напиши /start")
        return
    user_notes = data[user_id].get("notes", [])
    if not user_notes:
        await update.message.reply_text("У тебя пока нет заметок.")
    else:
        msg = "📝 Твои заметки:\n\n" + "\n".join(
            f"{i+1}. {n}" for i, n in enumerate(user_notes[-10:])
        )
        await update.message.reply_text(msg)

# --- /напомни ЧЧ:ММ Текст ---
async def remind(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    data = load_data()
    if user_id not in 
        await update.message.reply_text("Сначала напиши /start")
        return

    if not context.args:
        await update.message.reply_text(
            "📌 Используй так:\n/напомни 15:30 Забрать посылку\n\n"
            "Формат времени: ЧЧ:ММ (24-часовой)"
        )
        return

    full_text = " ".join(context.args)
    time_match = re.match(r'^(\d{1,2}):(\d{2})\s+(.+)$', full_text)
    if not time_match:
        await update.message.reply_text(
            "Неверный формат. Пример:\n/напомни 09:00 Выпить воду"
        )
        return

    hour = int(time_match.group(1))
    minute = int(time_match.group(2))
    reminder_text = time_match.group(3)

    if not (0 <= hour <= 23 and 0 <= minute <= 59):
        await update.message.reply_text("Время должно быть от 00:00 до 23:59")
        return

    now = datetime.now()
    target = now.replace(hour=hour, minute=minute, second=0, microsecond=0)
    if target <= now:
        target += timedelta(days=1)

    job = context.job_queue.run_once(
        send_reminder,
        when=target,
        data={"user_id": user_id, "text": reminder_text}
    )

    if "reminders" not in data[user_id]:
        data[user_id]["reminders"] = []
    data[user_id]["reminders"].append({
        "job_id": job.id,
        "time": target.isoformat(),
        "text": reminder_text
    })
    save_data(data)

    human_time = target.strftime("%d.%m в %H:%M")
    await update.message.reply_text(f"⏰ Напомню: *{reminder_text}*\n{human_time}", parse_mode="Markdown")

# --- Отправка напоминания ---
async def send_reminder(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    user_id = job.data["user_id"]
    text = job.data["text"]
    try:
        await context.bot.send_message(
            chat_id=user_id,
            text=f"🔔 *Напоминаю!*\n\n{text}",
            parse_mode="Markdown"
        )
    except Exception as e:
        print(f"Не удалось отправить напоминание {user_id}: {e}")

# --- Обработка обычных сообщений ---
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    data = load_data()
    if user_id not in 
        await update.message.reply_text("Сначала напиши /start")
        return
    text = update.message.text.strip()
    data[user_id]["entries"].append(text)
    save_data(data)
    await update.message.reply_text("🙏 Спасибо! Это важно.")

# --- УТРЕННЕЕ УВЕДОМЛЕНИЕ ---
async def send_morning_message(context: ContextTypes.DEFAULT_TYPE):
    data = load_data()
    user_ids = list(data.keys())
    import random
    for user_id in user_ids:
        try:
            cute_name = random.choice(CUTE_NAMES)
            message = f"☀️ Доброе утро, {cute_name}! 🌸\n\n" \
                      "Пусть твой день будет таким же тёплым и светлым, как ты!"
            await context.bot.send_message(chat_id=user_id, text=message)
        except Exception as e:
            print(f"Не удалось отправить {user_id}: {e}")

# --- Запуск бота ---
def main():
    logging.basicConfig(level=logging.WARNING)
    app = Application.builder().token(BOT_TOKEN).build()

    # Команды
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("note", note))
    app.add_handler(CommandHandler("notes", notes))
    app.add_handler(CommandHandler("напомни", remind))  # ← Русская команда!

    # Обычные сообщения
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    # Утренние уведомления
    scheduler = AsyncIOScheduler(timezone=pytz.utc)
    tz = pytz.timezone(TIMEZONE)
    now = datetime.now(tz)
    utc_morning = now.replace(hour=MORNING_HOUR, minute=0, second=0, microsecond=0).astimezone(pytz.utc)
    scheduler.add_job(send_morning_message, "cron", hour=utc_morning.hour, minute=0)
    scheduler.start()

    print("✅ Бот запущен! Все функции активны.")
    app.run_polling()

if __name__ == "__main__":
    main()