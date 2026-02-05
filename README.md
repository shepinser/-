import telebot
from telebot import types
import sqlite3
from datetime import datetime, date
import random

TOKEN = "ТУТ_ТВОЙ_ТОКЕН"
bot = telebot.TeleBot(TOKEN)

# ================== БАЗА ДАННЫХ ==================
conn = sqlite3.connect("users.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tg_id INTEGER UNIQUE,
    username TEXT,
    first_name TEXT,
    nickname TEXT,
    reg_date TEXT
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS fact_day (
    day TEXT UNIQUE,
    fact TEXT
)
""")

conn.commit()

# ================== ФАКТЫ ==================
FACTS = [
    "Собаки способны понимать до 250 слов и жестов.",
    "У собак уникальный отпечаток носа, как у людей отпечаток пальца.",
    "Собаки чувствуют настроение человека по запаху.",
    "Самая старая собака прожила 29 лет.",
    "Собаки видят сны так же, как люди.",
    "У собак слух в 4 раза лучше человеческого.",
    "Хвост собаки — инструмент общения, а не просто украшение.",
    "Собаки могут запоминать маршруты лучше GPS."
]

# ================== ПРОВЕРКИ ==================
def is_registered(user_id):
    cursor.execute("SELECT nickname FROM users WHERE tg_id = ?", (user_id,))
    data = cursor.fetchone()
    return data and data[0] is not None

def need_registration(message):
    if not is_registered(message.from_user.id):
        bot.send_message(
            message.chat.id,
            "🐶 Для начала нужно зарегистрироваться.\nНажми «Регистрация» 👇",
            reply_markup=register_keyboard()
        )
        return True
    return False

# ================== КЛАВИАТУРЫ ==================
def register_keyboard():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("📝 Регистрация")
    return kb

def main_keyboard():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("📌 Факт дня")
    kb.add("🔁 Случайный факт")
    kb.add("ℹ️ О боте")
    return kb

# ================== START ==================
@bot.message_handler(commands=["start"])
def start(message):
    cursor.execute(
        "INSERT OR IGNORE INTO users (tg_id, username, first_name, reg_date) VALUES (?, ?, ?, ?)",
        (
            message.from_user.id,
            message.from_user.username,
            message.from_user.first_name,
            datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        )
    )
    conn.commit()

    if not is_registered(message.from_user.id):
        bot.send_message(
            message.chat.id,
            "🐶 *Собакен бот*\n\n"
            "Я полезный и залипательный бот с фактами.\n"
            "Но сначала — регистрация 👇",
            parse_mode="Markdown",
            reply_markup=register_keyboard()
        )
    else:
        bot.send_message(
            message.chat.id,
            "🐾 С возвращением!",
            reply_markup=main_keyboard()
        )

# ================== РЕГИСТРАЦИЯ ==================
@bot.message_handler(func=lambda m: m.text == "📝 Регистрация")
def register(message):
    if is_registered(message.from_user.id):
        bot.send_message(message.chat.id, "✅ Ты уже зарегистрирован", reply_markup=main_keyboard())
        return

    msg = bot.send_message(message.chat.id, "✍️ Придумай никнейм (без пробелов):")
    bot.register_next_step_handler(msg, save_nickname)

def save_nickname(message):
    nickname = message.text.strip()

    if " " in nickname or len(nickname) < 3:
        msg = bot.send_message(message.chat.id, "❌ Ник без пробелов, минимум 3 символа")
        bot.register_next_step_handler(msg, save_nickname)
        return

    cursor.execute(
        "UPDATE users SET nickname = ? WHERE tg_id = ?",
        (nickname, message.from_user.id)
    )
    conn.commit()

    bot.send_message(
        message.chat.id,
        f"🎉 Готово! Добро пожаловать, *{nickname}* 🐶",
        parse_mode="Markdown",
        reply_markup=main_keyboard()
    )

# ================== ФАКТ ДНЯ ==================
@bot.message_handler(func=lambda m: m.text == "📌 Факт дня")
def fact_of_day(message):
    if need_registration(message):
        return

    today = date.today().isoformat()

    cursor.execute("SELECT fact FROM fact_day WHERE day = ?", (today,))
    row = cursor.fetchone()

    if row:
        fact = row[0]
    else:
        fact = random.choice(FACTS)
        cursor.execute("INSERT OR IGNORE INTO fact_day (day, fact) VALUES (?, ?)", (today, fact))
        conn.commit()

    bot.send_message(message.chat.id, f"📌 *Факт дня:*\n\n{fact}", parse_mode="Markdown")

# ================== СЛУЧАЙНЫЙ ФАКТ ==================
@bot.message_handler(func=lambda m: m.text == "🔁 Случайный факт")
def random_fact(message):
    if need_registration(message):
        return

    fact = random.choice(FACTS)
    bot.send_message(message.chat.id, f"🐾 {fact}")

# ================== О БОТЕ ==================
@bot.message_handler(func=lambda m: m.text == "ℹ️ О боте")
def about(message):
    if need_registration(message):
        return

    bot.send_message(
        message.chat.id,
        "🐶 *Собакен бот*\n\n"
        "Полезные факты, залипательные знания\n"
        "Заходи каждый день за новым фактом!",
        parse_mode="Markdown"
    )

# ================== ЗАПУСК ==================
print("Собакен бот запущен 🐾")
bot.infinity_polling()
