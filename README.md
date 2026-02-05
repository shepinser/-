import threading
import telebot
from telebot import types
import sqlite3
from datetime import datetime
import random
from telebot.types import ReplyKeyboardMarkup, KeyboardButton

TOKEN = "8437626033:AAGeXXzGuN26DMMbD2QS0In5MZqCpD5tLjY"
DEVELOPER_ID = 1469419131

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
    score INTEGER DEFAULT 0,
    reg_date TEXT
)
""")
conn.commit()

def is_registered(user_id):
    cursor.execute(
        "SELECT nickname FROM users WHERE tg_id = ?",
        (user_id,)
    )
    user = cursor.fetchone()
    return bool(user and user[0])

# ================== КЛАВИАТУРЫ ==================
def main_keyboard(user_id):
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("📝 Регистрация")
    kb.add("✏️ Редактировать ник")
    kb.add("🎮 Игры")
    kb.add("💳 Подписки")
    kb.add("ℹ️ Помощь")
    if user_id == DEVELOPER_ID:
        kb.add("💻 Админка")
    return kb

def admin_keyboard():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("📋 Все пользователи")
    kb.add("➕ Выдать очки")
    kb.add("❌ Удалить пользователя")
    kb.add("📢 Рассылка")
    kb.add("◀️ Назад")
    return kb

# ================== /START ==================
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

    bot.send_message(
        message.chat.id,
        "👋 Добро пожаловать в *REGENT ONLINE!*",
        parse_mode="Markdown",
        reply_markup=main_keyboard(message.from_user.id)
    )

# ================== ПОМОЩЬ ==================
@bot.message_handler(func=lambda m: m.text == "ℹ️ Помощь")
def help_message(message):
    bot.send_message(
        message.chat.id,
        "🤍 *Помощь*\n\n"
        "📝 Регистрация — создать аккаунт\n"
        "✏️ Редактировать ник — изменить никнейм\n"
        "🎮 Игры — мини-игры\n",
        parse_mode="Markdown"
    )

# ================== РЕГИСТРАЦИЯ ==================
@bot.message_handler(func=lambda m: m.text == "📝 Регистрация")
def register(message):
    if is_registered(message.from_user.id):
        bot.send_message(
            message.chat.id,
            "✅ Вы уже зарегистрированы",
            reply_markup=main_keyboard(message.from_user.id)
        )
        return

    msg = bot.send_message(
        message.chat.id,
        "✍️ Введите никнейм (без пробелов):"
    )
    bot.register_next_step_handler(msg, save_nickname)



def save_nickname(message):
    nickname = message.text.strip()
    cursor.execute(
        "UPDATE users SET nickname = ? WHERE tg_id = ?",
        (nickname, message.from_user.id)
    )
    conn.commit()

    bot.send_message(
        message.chat.id,
        f"🎉 Регистрация завершена!\nВаш ник: *{nickname}*",
        parse_mode="Markdown",
        reply_markup=main_keyboard(message.from_user.id)
    )
jokes = [
    "Почему программисты любят Python? Потому что без него жизнь была бы багована.",
    "— Пап, а Python — это язык программирования или змея?\n— И то, и другое, сынок!",
    "Программист пришёл в магазин, сделал checkout в Git, а не на кассе.",
    "Зачем программисту очки? Чтобы видеть баги в HD.",
    "Почему программисты любят темные темы? Потому что светлая жизнь слишком багованная!",
    "Сколько программистов нужно, чтобы поменять лампочку? Ни одного, это аппаратная проблема.",
    "Если код не работает, добавь ещё один print() — чудо случится!",
    "Девелоперский юмор: «Документируй код или умри!»",
    "— Как прошёл твой день? — 0 ошибок и 1 баг.",
    "Когда программист голоден, он ест bytes.",
    "Счастливый программист — это тот, кто закоммитил без конфликтов.",
    "Debugging — это как быть детективом в криминальном романе своего собственного кода.",
    "Любовь программиста: когда твой код компилируется с первого раза.",
    "Почему Java-разработчики так устают? Потому что они всё время работают с кучей объектов.",
    "Программист пошёл на свидание, но оно зависло на while True:.",
    "Вчера компилятор сказал мне «Hello World!», я расплакался.",
    "Почему программисты не любят природу? Слишком много багов.",
    "Если жизнь выдаёт ошибки, сделай try/except.",
    "Git — это как жизнь: ты не всегда можешь отменить свои действия.",
    "Работает? Не трогай. Не работает? Попробуй sudo.",
    "Сколько Python-разработчиков нужно, чтобы поменять лампочку? Один, и он сделает это красиво.",
    "Реальные программисты называют кофе жидкой мотивацией.",
    "Почему программисты путают Хэллоуин и Рождество? Потому что Oct 31 == Dec 25.",
    "Код без багов — это миф, как единорог.",
    "Когда программист говорит «ещё чуть-чуть» — это значит 5 часов.",
    "Почему программистам нравится зима? Потому что можно заморозить баги.",
    "Если твой код работает — не трогай его, это магия.",
    "— Какой твой любимый язык? — Питон, потому что он обнимает.",
    "Класс без методов — это как кофе без кофеина.",
    "Лучше один раз запустить скрипт, чем тысячу раз дебажить."
]


# ---------- Подписки ----------
@bot.message_handler(func=lambda m: m.text == "💳 Подписки")
def subscriptions_menu(message):
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("📖 Анекдоты")  # рабочая
    kb.add("🔧 Подписка 2 (в разработке)")
    kb.add("🔧 Подписка 3 (в разработке)")
    kb.add("◀️ Назад")
    bot.send_message(message.chat.id, "Выберите подписку:", reply_markup=kb)

# ---------- Анекдоты ----------
@bot.message_handler(func=lambda m: m.text == "📖 Анекдоты")
def setup_jokes(message):
    msg = bot.send_message(message.chat.id, "Введите интервал в минутах для получения анекдотов:")
    bot.register_next_step_handler(msg, save_jokes_interval)

def save_jokes_interval(message):
    try:
        interval = int(message.text.strip())
        if interval < 1:
            raise ValueError
        user_id = message.from_user.id
        user_subscriptions[user_id] = interval
        bot.send_message(message.chat.id, f"✅ Подписка на анекдоты активирована! Интервал: {interval} минут", reply_markup=main_keyboard(user_id))
        start_joke_thread(user_id, interval, message.chat.id)
    except ValueError:
        msg = bot.send_message(message.chat.id, "❌ Пожалуйста, введите число минут (целое число больше 0)")
        bot.register_next_step_handler(msg, save_jokes_interval)

# ---------- Функция отправки анекдотов в отдельном потоке ----------
def start_joke_thread(user_id, interval, chat_id):
    def send_jokes():
        while user_id in user_subscriptions:
            joke = random.choice(jokes)
            try:
                bot.send_message(chat_id, f"📢 Анекдот/Рофл:\n{joke}")
            except:
                pass
            time.sleep(interval * 60)  # интервал в минутах
    thread = threading.Thread(target=send_jokes)
    thread.start()

# ================== СМЕНА НИКА ==================
@bot.message_handler(func=lambda m: m.text == "✏️ Редактировать ник")
def edit_nickname(message):
    msg = bot.send_message(message.chat.id, "Введите новый ник:")
    bot.register_next_step_handler(msg, update_nickname)


def update_nickname(message):
    new_nick = message.text.strip()
    cursor.execute(
        "UPDATE users SET nickname = ? WHERE tg_id = ?",
        (new_nick, message.from_user.id)
    )
    conn.commit()

    bot.send_message(
        message.chat.id,
        f"✅ Ник изменён на *{new_nick}*",
        parse_mode="Markdown",
        reply_markup=main_keyboard(message.

from_user.id)
    )

# ================== ИГРЫ ==================
# активные игры
active_games = {}

# ===== Клавиатура игр =====
def games_keyboard():
    kb = ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add(
        KeyboardButton("🔢 Угадай число"),
        KeyboardButton("🎲 Кубик"),
        KeyboardButton("🎰 Слот")
    )
    kb.add(KeyboardButton("◀️ Назад"))  # кнопка назад
    return kb

# ===== Игры =====
@bot.message_handler(func=lambda m: m.text == "🎮 Игры")
def open_games(message):
    bot.send_message(
        message.chat.id,
        "🎮 Выберите игру:",
        reply_markup=games_keyboard()
    )

@bot.message_handler(func=lambda m: m.text == "🔢 Угадай число")
def guess_start(message):
    number = random.randint(1, 10)
    active_games[message.from_user.id] = number
    bot.send_message(message.chat.id, "🔢 Я загадал число от 1 до 10. Угадай!")

@bot.message_handler(func=lambda m: m.from_user.id in active_games)
def guess_number(message):
    if not message.text.isdigit():
        bot.send_message(message.chat.id, "⚠️ Введите число от 1 до 10")
        return

    if int(message.text) == active_games[message.from_user.id]:
        try:
            cursor.execute(
                "UPDATE users SET score = score + 10 WHERE tg_id = ?",
                (message.from_user.id,)
            )
            conn.commit()
        except:
            pass
        del active_games[message.from_user.id]

        bot.send_message(
            message.chat.id,
            "🎉 Верно! +10 очков 🏆",
            reply_markup=main_keyboard(message.from_user.id)
        )
    else:
        bot.send_message(message.chat.id, "❌ Не угадал, попробуй ещё!")

@bot.message_handler(func=lambda m: m.text == "🎲 Кубик")
def dice(message):
    bot.send_message(
        message.chat.id,
        f"🎲 Выпало: {random.randint(1,6)}",
        reply_markup=main_keyboard(message.from_user.id)
    )

@bot.message_handler(func=lambda m: m.text == "🎰 Слот")
def slot(message):
    emojis = ["🍒", "🍋", "⭐", "🔔", "🍀"]
    result = [random.choice(emojis) for _ in range(3)]

    if len(set(result)) == 1:
        try:
            cursor.execute(
                "UPDATE users SET score = score + 50 WHERE tg_id = ?",
                (message.from_user.id,)
            )
            conn.commit()
        except:
            pass
        text = "🎉 Победа! +50 очков"
    else:
        text = "😢 Не повезло"

    bot.send_message(
        message.chat.id,
        f"🎰 {' | '.join(result)}\n{text}",
        reply_markup=main_keyboard(message.from_user.id)
    )

# ===== Назад =====
@bot.message_handler(func=lambda m: m.text == "◀️ Назад")
def back(message):
    bot.send_message(
        message.chat.id,
        "🏠 Главное меню",
        reply_markup=main_keyboard(message.from_user.id)
    )
# ================== АДМИНКА ==================
@bot.message_handler(func=lambda m: m.text == "💻 Админка" and m.from_user.id == DEVELOPER_ID)
def admin(message):
    bot.send_message(
        message.chat.id,
        "👑 Админ-панель",
        reply_markup=admin_keyboard()
    )


@bot.message_handler(func=lambda m: m.text == "📋 Все пользователи" and m.from_user.id == DEVELOPER_ID)
def all_users(message):
    cursor.execute("SELECT tg_id, nickname, score FROM users")
    users = cursor.fetchall()

    text = "📋 Пользователи:\n\n"
    for u in users:
        text += f"ID: {u[0]} | Ник: {u[1]} | Очки: {u[2]}\n"

    bot.send_message(message.chat.id, text)


@bot.message_handler(func=lambda m: m.text == "➕ Выдать очки" and m.from_user.id == DEVELOPER_ID)
def give_points(message):
    msg = bot.send_message(message.chat.id, "ID и очки через пробел:")
    bot.register_next_step_handler(msg, give_points_step)


def give_points_step(message):
    try:
        tg_id, points = map(int, message.text.split())
        cursor.execute("UPDATE users SET score = score + ? WHERE tg_id = ?", (points, tg_id))
        conn.commit()
        bot.send_message(message.chat.id, "✅ Очки выданы")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка ввода")


@bot.message_handler(func=lambda m: m.text == "❌ Удалить пользователя" and m.from_user.id == DEVELOPER_ID)
def delete_user(message):
    msg = bot.send_message(message.chat.id, "Введите ID пользователя:")
    bot.register_next_step_handler(msg, delete_user_step)


def delete_user_step(message):
    try:
        tg_id = int(message.text)
        cursor.execute("DELETE FROM users WHERE tg_id = ?", (tg_id,))
        conn.commit()
        bot.send_message(message.chat.id, "✅ Пользователь удалён")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка")


@bot.message_handler(func=lambda m: m.text == "📢 Рассылка" and m.from_user.id == DEVELOPER_ID)
def broadcast(message):
    msg = bot.send_message(
        message.chat.id,
        "✉️ Введите сообщение для рассылки всем пользователям:"
    )
    bot.register_next_step_handler(msg, broadcast_step)


def broadcast_step(message):
    cursor.execute("SELECT tg_id FROM users")
    users = cursor.fetchall()

    sent = 0
    failed = 0

    for (tg_id,) in users:
        try:
            bot.send_message(
                tg_id,
                f"📢 Сообщение от администратора:\n\n{message.text}"
            )
            sent += 1
        except Exception as e:
            failed += 1
            print(f"Не отправлено {tg_id}: {e}")

    bot.send_message(
        message.chat.id,
        f"✅ Рассылка завершена\n"
        f"📨 Отправлено: {sent}\n"
        f"❌ Не доставлено: {failed}"
    )

# ================== ЗАПУСК ==================
print("REGENT ONLINE запущен 🚀")
bot.infinity_polling()
