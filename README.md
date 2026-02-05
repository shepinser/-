import asyncio
import sqlite3
import logging
from datetime import datetime, date, timedelta
from typing import List, Dict, Optional, Tuple
from contextlib import closing

from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command, CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardMarkup, KeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder, ReplyKeyboardBuilder

import os
from dotenv import load_dotenv

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Загрузка переменных окружения
load_dotenv()
BOT_TOKEN = os.getenv('BOT_TOKEN')

# ID админов (через запятую в .env файле)
ADMIN_IDS = list(map(int, os.getenv('ADMIN_IDS', '').split(','))) if os.getenv('ADMIN_IDS') else []

# Константы
FACTS_PER_DAY = 3  # Сколько фактов получает пользователь в день
MAX_FACTS_PER_MESSAGE = 50  # Максимум фактов за одну вставку
DAILY_FACT_TIME = "09:00"  # Время отправки ежедневного факта

# Emoji для кнопок
EMOJI = {
    'fact': '📚', 'story': '📖', 'daily': '📅', 'random': '🎲',
    'profile': '👤', 'top': '🏆', 'admin': '⚙️', 'back': '🔙',
    'stats': '📊', 'broadcast': '📢', 'add': '➕', 'list': '📋',
    'delete': '❌', 'next': '➡️', 'prev': '⬅️', 'fun': '😄',
    'science': '🔬', 'history': '🏛️', 'nature': '🌿', 'tech': '💻',
    'mystery': '🔮', 'all': '🌐', 'like': '❤️', 'dislike': '👎',
    'settings': '⚙️', 'achievements': '🏅', 'streak': '🔥'
}

# Категории фактов
CATEGORIES = {
    'fun': f"{EMOJI['fun']} Веселые",
    'science': f"{EMOJI['science']} Наука",
    'history': f"{EMOJI['history']} История",
    'nature': f"{EMOJI['nature']} Природа",
    'tech': f"{EMOJI['tech']} Технологии",
    'mystery': f"{EMOJI['mystery']} Тайны",
    'all': f"{EMOJI['all']} Все категории"
}

# Класс базы данных
class Database:
    def __init__(self, db_name='facts.db'):
        self.conn = sqlite3.connect(db_name, check_same_thread=False)
        self.conn.row_factory = sqlite3.Row
        self.create_tables()
    
    def create_tables(self):
        cursor = self.conn.cursor()
        
        # Пользователи
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                first_name TEXT,
                last_name TEXT,
                registered_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                last_active TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                daily_facts_count INTEGER DEFAULT 0,
                last_daily_date DATE,
                total_facts_read INTEGER DEFAULT 0,
                favorite_category TEXT,
                daily_streak INTEGER DEFAULT 0,
                last_streak_date DATE,
                level INTEGER DEFAULT 1,
                experience INTEGER DEFAULT 0
            )
        ''')
        
        # Факты/Истории
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS facts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                content TEXT NOT NULL,
                category TEXT DEFAULT 'general',
                added_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                added_by INTEGER,
                likes INTEGER DEFAULT 0,
                views INTEGER DEFAULT 0,
                is_daily_fact BOOLEAN DEFAULT 0,
                fact_date DATE,
                difficulty TEXT DEFAULT 'easy',
                FOREIGN KEY (added_by) REFERENCES users (user_id)
            )
        ''')
        
        # Прочитанные факты
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS user_facts (
                user_id INTEGER,
                fact_id INTEGER,
                read_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                liked BOOLEAN DEFAULT 0,
                time_spent INTEGER DEFAULT 0,
                PRIMARY KEY (user_id, fact_id),
                FOREIGN KEY (user_id) REFERENCES users (user_id),
                FOREIGN KEY (fact_id) REFERENCES facts (id)
            )
        ''')
        
        # Достижения
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS achievements (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                achievement_type TEXT,
                achieved_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users (user_id)
            )
        ''')
        
        self.conn.commit()
    
    # Методы для пользователей
    def add_user(self, user_id: int, username: str, first_name: str, last_name: str = ""):
        cursor = self.conn.cursor()
        try:
            cursor.execute('''
                INSERT OR IGNORE INTO users (user_id, username, first_name, last_name)
                VALUES (?, ?, ?, ?)
            ''', (user_id, username, first_name, last_name))
            self.conn.commit()
            return True
        except Exception as e:
            logger.error(f"Error adding user: {e}")
            return False
    
    def get_user(self, user_id: int):
        cursor = self.conn.cursor()
        cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
        return cursor.fetchone()
    
    def update_user_activity(self, user_id: int):
        cursor = self.conn.cursor()
        cursor.execute('''
            UPDATE users 
            SET last_active = CURRENT_TIMESTAMP 
            WHERE user_id = ?
        ''', (user_id,))
        self.conn.commit()
    
    def update_daily_stats(self, user_id: int):
        cursor = self.conn.cursor()
        user = self.get_user(user_id)
        
        if user:
            today = date.today()
            last_date = datetime.strptime(user['last_daily_date'], '%Y-%m-%d').date() if user['last_daily_date'] else None
            
            # Обновляем серию
            if last_date == today - timedelta(days=1):
                new_streak = user['daily_streak'] + 1
            elif last_date == today:
                new_streak = user['daily_streak']
            else:
                new_streak = 1
            
            cursor.execute('''
                UPDATE users 
                SET daily_facts_count = daily_facts_count + 1,
                    last_daily_date = CURRENT_DATE,
                    daily_streak = ?,
                    experience = experience + 10,
                    level = CASE 
                        WHEN experience >= level * 100 THEN level + 1 
                        ELSE level 
                    END
                WHERE user_id = ?
            ''', (new_streak, user_id))
            self.conn.commit()
    
    # Методы для фактов
    def add_facts(self, facts_text: str, category: str = "general", added_by: int = None):
        cursor = self.conn.cursor()
        facts = [fact.strip() for fact in facts_text.split(';') if fact.strip()]
        
        added_count = 0
        for fact in facts[:MAX_FACTS_PER_MESSAGE]:
            try:
                cursor.execute('''
                    INSERT INTO facts (content, category, added_by)
                    VALUES (?, ?, ?)
                ''', (fact, category, added_by))
                added_count += 1
            except Exception as e:
                logger.error(f"Error adding fact: {e}")
        
        self.conn.commit()
        return added_count
    
    def get_random_fact(self, user_id: int = None, category: str = "all"):
        cursor = self.conn.cursor()
        
        if category == "all":
            cursor.execute('''
                SELECT f.* FROM facts f
                LEFT JOIN user_facts uf ON f.id = uf.fact_id AND uf.user_id = ?
                WHERE uf.fact_id IS NULL OR ? IS NULL
                ORDER BY RANDOM()
                LIMIT 1
            ''', (user_id, user_id))
        else:
            cursor.execute('''
                SELECT f.* FROM facts f
                LEFT JOIN user_facts uf ON f.id = uf.fact_id AND uf.user_id = ?
                WHERE (uf.fact_id IS NULL OR ? IS NULL) AND f.category = ?
                ORDER BY RANDOM()
                LIMIT 1
            ''', (user_id, user_id, category))
        
        fact = cursor.fetchone()
        
        if fact:
            # Увеличиваем просмотры
            cursor.execute('UPDATE facts SET views = views + 1 WHERE id = ?', (fact['id'],))
            self.conn.commit()
            
            # Отмечаем как прочитанный
            if user_id:
                try:
                    cursor.execute('''
                        INSERT OR IGNORE INTO user_facts (user_id, fact_id)
                        VALUES (?, ?)
                    ''', (user_id, fact['id']))
                    
                    # Обновляем статистику пользователя
                    cursor.execute('''
                        UPDATE users 
                        SET total_facts_read = total_facts_read + 1
                        WHERE user_id = ?
                    ''', (user_id,))
                    
                    # Проверяем достижения
                    self.check_achievements(user_id)
                    
                    self.conn.commit()
                except:
                    pass
        
        return fact
    
    def get_fact_by_id(self, fact_id: int):
        cursor = self.conn.cursor()
        cursor.execute('SELECT * FROM facts WHERE id = ?', (fact_id,))
        return cursor.fetchone()
    
    def like_fact(self, user_id: int, fact_id: int):
        cursor = self.conn.cursor()
        
        # Проверяем, читал ли пользователь факт
        cursor.execute('SELECT * FROM user_facts WHERE user_id = ? AND fact_id = ?', (user_id, fact_id))
        if cursor.fetchone():
            cursor.execute('''
                UPDATE user_facts 
                SET liked = CASE WHEN liked = 0 THEN 1 ELSE 0 END
                WHERE user_id = ? AND fact_id = ?
            ''', (user_id, fact_id))
            
            # Обновляем счетчик лайков
            cursor.execute('''
                UPDATE facts 
                SET likes = likes + CASE 
                    WHEN (SELECT liked FROM user_facts WHERE user_id = ? AND fact_id = ?) = 1 THEN 1 
                    ELSE -1 
                END
                WHERE id = ?
            ''', (user_id, fact_id, fact_id))
            
            self.conn.commit()
            return True
        
        return False
    
    def get_user_stats(self, user_id: int):
        cursor = self.conn.cursor()
        
        cursor.execute('''
            SELECT 
                u.*,
                COUNT(DISTINCT uf.fact_id) as facts_read,
                COUNT(CASE WHEN uf.liked = 1 THEN 1 END) as liked_count,
                (SELECT COUNT(*) FROM facts) as total_facts
            FROM users u
            LEFT JOIN user_facts uf ON u.user_id = uf.user_id
            WHERE u.user_id = ?
            GROUP BY u.user_id
        ''', (user_id,))
        
        return cursor.fetchone()
    
    def check_achievements(self, user_id: int):
        cursor = self.conn.cursor()
        
        # Получаем статистику
        cursor.execute('SELECT total_facts_read FROM users WHERE user_id = ?', (user_id,))
        result = cursor.fetchone()
        
        if result:
            facts_read = result['total_facts_read']
            achievements = []
            
            # Проверяем достижения
            if facts_read >= 1:
                achievements.append(('first_fact', 'Первый факт!'))
            if facts_read >= 10:
                achievements.append(('fact_collector_10', 'Коллекционер (10 фактов)'))
            if facts_read >= 50:
                achievements.append(('fact_master_50', 'Мастер фактов (50 фактов)'))
            if facts_read >= 100:
                achievements.append(('fact_guru_100', 'Гуру фактов (100 фактов)'))
            
            # Добавляем новые достижения
            for achievement_type, _ in achievements:
                cursor.execute('''
                    INSERT OR IGNORE INTO achievements (user_id, achievement_type)
                    VALUES (?, ?)
                ''', (user_id, achievement_type))
        
        self.conn.commit()
    
    def get_achievements(self, user_id: int):
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT achievement_type, achieved_date 
            FROM achievements 
            WHERE user_id = ?
            ORDER BY achieved_date DESC
        ''', (user_id,))
        return cursor.fetchall()
    
    def get_top_users(self, limit: int = 10):
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT user_id, username, first_name, total_facts_read, level, daily_streak
            FROM users
            ORDER BY total_facts_read DESC, level DESC, daily_streak DESC
            LIMIT ?
        ''', (limit,))
        return cursor.fetchall()
    
    def get_stats(self):
        cursor = self.conn.cursor()
        cursor.execute('SELECT COUNT(*) as total_users FROM users')
        total_users = cursor.fetchone()['total_users']
        
        cursor.execute('SELECT COUNT(*) as total_facts FROM facts')
        total_facts = cursor.fetchone()['total_facts']
        
        cursor.execute('SELECT COUNT(*) as total_reads FROM user_facts')
        total_reads = cursor.fetchone()['total_reads']
        
        cursor.execute('SELECT SUM(likes) as total_likes FROM facts')
        total_likes = cursor.fetchone()['total_likes'] or 0
        
        return {
            'total_users': total_users,
            'total_facts': total_facts,
            'total_reads': total_reads,
            'total_likes': total_likes
        }
    
    def get_all_users(self):
        cursor = self.conn.cursor()
        cursor.execute('SELECT user_id FROM users')
        return [row['user_id'] for row in cursor.fetchall()]

# Состояния FSM
class AdminStates(StatesGroup):
    waiting_for_facts = State()
    waiting_for_category = State()
    waiting_for_broadcast = State()

class UserStates(StatesGroup):
    reading_fact = State()

# Инициализация бота
bot = Bot(token=BOT_TOKEN)
storage = MemoryStorage()
dp = Dispatcher(storage=storage)
db = Database()

# Клавиатуры
def get_main_keyboard(user_id: int = None) -> ReplyKeyboardMarkup:
    """Главная клавиатура"""
    builder = ReplyKeyboardBuilder()
    
    builder.add(KeyboardButton(text=f"{EMOJI['random']} Случайный факт"))
    builder.add(KeyboardButton(text=f"{EMOJI['daily']} Факт дня"))
    builder.add(KeyboardButton(text=f"{EMOJI['story']} Интересная история"))
    builder.add(KeyboardButton(text=f"{EMOJI['profile']} Мой профиль"))
    builder.add(KeyboardButton(text=f"{EMOJI['top']} Топ читателей"))
    builder.add(KeyboardButton(text=f"{EMOJI['fun']} Категории"))
    
    # Кнопка админа
    if user_id in ADMIN_IDS:
        builder.add(KeyboardButton(text=f"{EMOJI['admin']} Админ-панель"))
    
    builder.adjust(2, 2, 1, 1)
    return builder.as_markup(resize_keyboard=True)

def get_categories_keyboard() -> InlineKeyboardMarkup:
    """Клавиатура категорий"""
    builder = InlineKeyboardBuilder()
    
    for key, value in CATEGORIES.items():
        if key != 'all':
            builder.add(InlineKeyboardButton(text=value, callback_data=f"category_{key}"))
    
    builder.add(InlineKeyboardButton(text=f"{EMOJI['random']} Любая категория", callback_data="category_all"))
    builder.add(InlineKeyboardButton(text=f"{EMOJI['back']} Назад", callback_data="back_to_main"))
    builder.adjust(2, 2, 2, 1)
    return builder.as_markup()

def get_fact_keyboard(fact_id: int, liked: bool = False) -> InlineKeyboardMarkup:
    """Клавиатура под фактом"""
    builder = InlineKeyboardBuilder()
    
    like_emoji = EMOJI['like'] if liked else "🤍"
    builder.add(InlineKeyboardButton(text=f"{like_emoji} Нравится", callback_data=f"like_{fact_id}"))
    builder.add(InlineKeyboardButton(text=f"{EMOJI['next']} Еще факт", callback_data="next_fact"))
    builder.add(InlineKeyboardButton(text=f"{EMOJI['fun']} Категории", callback_data="show_categories"))
    builder.add(InlineKeyboardButton(text=f"{EMOJI['profile']} Профиль", callback_data="show_profile"))
    
    builder.adjust(2, 2)
    return builder.as_markup()

def get_admin_keyboard() -> ReplyKeyboardMarkup:
    """Клавиатура админ-панели"""
    builder = ReplyKeyboardBuilder()
    
    builder.add(KeyboardButton(text=f"{EMOJI['add']} Добавить факты"))
    builder.add(KeyboardButton(text=f"{EMOJI['stats']} Статистика"))
    builder.add(KeyboardButton(text=f"{EMOJI['broadcast']} Рассылка"))
    builder.add(KeyboardButton(text=f"{EMOJI['list']} Список фактов"))
    builder.add(KeyboardButton(text=f"{EMOJI['back']} Назад"))
    
    builder.adjust(2, 2, 1)
    return builder.as_markup(resize_keyboard=True)

def get_back_keyboard() -> InlineKeyboardMarkup:
    """Кнопка назад"""
    builder = InlineKeyboardBuilder()
    builder.add(InlineKeyboardButton(text=f"{EMOJI['back']} Назад", callback_data="back_to_main"))
    return builder.as_markup()

# Обработчики команд
@dp.message(CommandStart())
async def cmd_start(message: types.Message, state: FSMContext):
    """Обработчик команды /start"""
    await state.clear()
    
    # Регистрация пользователя
    user_id = message.from_user.id
    username = message.from_user.username or ""
    first_name = message.from_user.first_name or ""
    last_name = message.from_user.last_name or ""
    
    db.add_user(user_id, username, first_name, last_name)
    db.update_user_activity(user_id)
    
    # Приветственное сообщение
    await message.answer(
        f"👋 Привет, {first_name}!\n\n"
        f"Я — бот с интересными фактами и историями!\n"
        f"Каждый день ты можешь читать новые факты, "
        f"собирать достижения и соревноваться с другими.\n\n"
        f"✨ <b>Что умеет бот:</b>\n"
        f"• {EMOJI['random']} Случайные факты из разных категорий\n"
        f"• {EMOJI['daily']} Ежедневные факты для поддержания серии\n"
        f"• {EMOJI['story']} Длинные интересные истории\n"
        f"• {EMOJI['profile']} Прокачка профиля и уровней\n"
        f"• {EMOJI['top']} Рейтинг самых активных читателей\n"
        f"• {EMOJI['achievements']} Достижения и награды\n\n"
        f"<i>Просто начни читать — это затягивает!</i>",
        reply_markup=get_main_keyboard(user_id),
        parse_mode='HTML'
    )

@dp.message(Command("help"))
async def cmd_help(message: types.Message):
    """Обработчик команды /help"""
    help_text = (
        f"📚 <b>Помощь по боту</b>\n\n"
        f"{EMOJI['random']} <b>Случайный факт</b> - получай случайный факт из любой категории\n"
        f"{EMOJI['daily']} <b>Факт дня</b> - специальный факт для поддержания ежедневной серии\n"
        f"{EMOJI['story']} <b>Интересная история</b> - длинные захватывающие истории\n"
        f"{EMOJI['profile']} <b>Мой профиль</b> - твоя статистика, уровень и достижения\n"
        f"{EMOJI['top']} <b>Топ читателей</b> - рейтинг самых активных пользователей\n"
        f"{EMOJI['fun']} <b>Категории</b> - выбор тематики фактов\n\n"
        f"<i>Просто нажимай на кнопки и погружайся в мир интересных фактов!</i>"
    )
    await message.answer(help_text, parse_mode='HTML')

@dp.message(F.text == f"{EMOJI['random']} Случайный факт")
async def random_fact(message: types.Message, state: FSMContext):
    """Случайный факт"""
    user_id = message.from_user.id
    db.update_user_activity(user_id)
    
    fact = db.get_random_fact(user_id)
    
    if fact:
        await state.set_state(UserStates.reading_fact)
        await state.update_data(current_fact_id=fact['id'])
        
        # Проверяем, лайкал ли пользователь этот факт
        cursor = db.conn.cursor()
        cursor.execute('SELECT liked FROM user_facts WHERE user_id = ? AND fact_id = ?', (user_id, fact['id']))
        liked_row = cursor.fetchone()
        liked = liked_row['liked'] if liked_row else False
        
        await message.answer(
            f"<b>📚 Факт #{fact['id']}</b>\n"
            f"<i>Категория: {CATEGORIES.get(fact['category'], 'Общая')}</i>\n\n"
            f"{fact['content']}\n\n"
            f"<i>👁 Просмотров: {fact['views']} | ❤ Лайков: {fact['likes']}</i>",
            reply_markup=get_fact_keyboard(fact['id'], liked),
            parse_mode='HTML'
        )
    else:
        await message.answer(
            "😔 Пока нет доступных фактов. Попробуйте позже или выберите другую категорию.",
            reply_markup=get_back_keyboard()
        )

@dp.message(F.text == f"{EMOJI['daily']} Факт дня")
async def daily_fact(message: types.Message):
    """Факт дня"""
    user_id = message.from_user.id
    db.update_user_activity(user_id)
    
    # Проверяем лимит на сегодня
    user = db.get_user(user_id)
    today = date.today()
    last_date = datetime.strptime(user['last_daily_date'], '%Y-%m-%d').date() if user['last_daily_date'] else None
    
    if last_date == today and user['daily_facts_count'] >= FACTS_PER_DAY:
        await message.answer(
            f"📊 <b>Лимит на сегодня исчерпан!</b>\n\n"
            f"Вы уже прочитали {user['daily_facts_count']} из {FACTS_PER_DAY} доступных сегодня фактов.\n"
            f"Новый лимит будет доступен завтра!\n\n"
            f"{EMOJI['random']} Вы можете продолжать читать случайные факты без ограничений!",
            parse_mode='HTML'
        )
        return
    
    fact = db.get_random_fact(user_id)
    
    if fact:
        db.update_daily_stats(user_id)
        
        # Обновляем информацию о пользователе
        user = db.get_user(user_id)
        
        await message.answer(
            f"<b>📅 Факт дня #{fact['id']}</b>\n"
            f"<i>Категория: {CATEGORIES.get(fact['category'], 'Общая')}</i>\n\n"
            f"{fact['content']}\n\n"
            f"<b>📊 Ваша статистика сегодня:</b>\n"
            f"• Прочитано сегодня: {user['daily_facts_count']}/{FACTS_PER_DAY}\n"
            f"• Серия дней: {user['daily_streak']} {EMOJI['streak']}\n"
            f"• Уровень: {user['level']}\n"
            f"• Опыт: {user['experience']}/{user['level'] * 100}\n\n"
            f"<i>👁 Просмотров: {fact['views']} | ❤ Лайков: {fact['likes']}</i>",
            parse_mode='HTML'
        )
    else:
        await message.answer(
            "😔 На сегодня фактов больше нет. Попробуйте завтра!",
            reply_markup=get_back_keyboard()
        )

@dp.message(F.text == f"{EMOJI['profile']} Мой профиль")
async def my_profile(message: types.Message):
    """Профиль пользователя"""
    user_id = message.from_user.id
    db.update_user_activity(user_id)
    
    stats = db.get_user_stats(user_id)
    
    if stats:
        # Получаем достижения
        achievements = db.get_achievements(user_id)
        achievements_list = "\n".join([
            f"• {ach['achievement_type'].replace('_', ' ').title()} - {ach['achieved_date'][:10]}"
            for ach in achievements[:5]  # Показываем 5 последних
        ]) if achievements else "Пока нет достижений"
        
        await message.answer(
            f"<b>{EMOJI['profile']} Ваш профиль</b>\n\n"
            f"<b>👤 Основная информация:</b>\n"
            f"• Имя: {stats['first_name']}\n"
            f"• Уровень: {stats['level']}\n"
            f"• Опыт: {stats['experience']}/{stats['level'] * 100}\n\n"
            f"<b>📊 Статистика:</b>\n"
            f"• Всего прочитано: {stats['total_facts_read']} фактов\n"
            f"• Понравилось: {stats.get('liked_count', 0)} фактов\n"
            f"• Серия дней: {stats['daily_streak']} {EMOJI['streak']}\n"
            f"• Прочитано сегодня: {stats['daily_facts_count']}/{FACTS_PER_DAY}\n"
            f"• Дата регистрации: {stats['registered_date'][:10]}\n\n"
            f"<b>🏅 Последние достижения:</b>\n{achievements_list}\n\n"
            f"<i>Продолжайте читать факты для получения новых достижений!</i>",
            parse_mode='HTML'
        )
    else:
        await message.answer("Профиль не найден. Попробуйте снова.")

@dp.message(F.text == f"{EMOJI['top']} Топ читателей")
async def top_readers(message: types.Message):
    """Топ читателей"""
    top_users = db.get_top_users(10)
    
    if top_users:
        top_text = "<b>🏆 Топ 10 читателей</b>\n\n"
        
        for i, user in enumerate(top_users, 1):
            medal = ""
            if i == 1:
                medal = "🥇"
            elif i == 2:
                medal = "🥈"
            elif i == 3:
                medal = "🥉"
            else:
                medal = f"{i}."
            
            username = user['username'] or f"{user['first_name']}"
            top_text += (
                f"{medal} <b>{username}</b>\n"
                f"   📚 Фактов: {user['total_facts_read']} | "
                f"⭐ Уровень: {user['level']} | "
                f"🔥 Серия: {user['daily_streak']}\n\n"
            )
        
        await message.answer(top_text, parse_mode='HTML')
    else:
        await message.answer("Пока нет данных для топа.")

@dp.message(F.text == f"{EMOJI['fun']} Категории")
async def show_categories(message: types.Message):
    """Показать категории"""
    await message.answer(
        "📚 <b>Выберите категорию:</b>\n\n"
        "Вы можете читать факты по конкретной тематике или выбрать случайную категорию.",
        reply_markup=get_categories_keyboard(),
        parse_mode='HTML'
    )

# Обработчики callback для категорий
@dp.callback_query(F.data.startswith("category_"))
async def process_category(callback: types.CallbackQuery, state: FSMContext):
    """Обработка выбора категории"""
    category = callback.data.split("_")[1]
    user_id = callback.from_user.id
    
    fact = db.get_random_fact(user_id, category if category != 'all' else None)
    
    if fact:
        await state.set_state(UserStates.reading_fact)
        await state.update_data(current_fact_id=fact['id'])
        
        # Проверяем, лайкал ли пользователь этот факт
        cursor = db.conn.cursor()
        cursor.execute('SELECT liked FROM user_facts WHERE user_id = ? AND fact_id = ?', (user_id, fact['id']))
        liked_row = cursor.fetchone()
        liked = liked_row['liked'] if liked_row else False
        
        await callback.message.edit_text(
            f"<b>📚 Факт #{fact['id']}</b>\n"
            f"<i>Категория: {CATEGORIES.get(fact['category'], 'Общая')}</i>\n\n"
            f"{fact['content']}\n\n"
            f"<i>👁 Просмотров: {fact['views']} | ❤ Лайков: {fact['likes']}</i>",
            reply_markup=get_fact_keyboard(fact['id'], liked),
            parse_mode='HTML'
        )
    else:
        await callback.answer("😔 В этой категории пока нет фактов", show_alert=True)

# Обработчики для лайков и навигации
@dp.callback_query(F.data.startswith("like_"))
async def process_like(callback: types.CallbackQuery):
    """Обработка лайка"""
    fact_id = int(callback.data.split("_")[1])
    user_id = callback.from_user.id
    
    success = db.like_fact(user_id, fact_id)
    
    if success:
        # Получаем обновленные данные
        fact = db.get_fact_by_id(fact_id)
        cursor = db.conn.cursor()
        cursor.execute('SELECT liked FROM user_facts WHERE user_id = ? AND fact_id = ?', (user_id, fact_id))
        liked_row = cursor.fetchone()
        liked = liked_row['liked'] if liked_row else False
        
        await callback.message.edit_reply_markup(
            reply_markup=get_fact_keyboard(fact_id, liked)
        )
        await callback.answer("❤️ Ваш голос учтен!" if liked else "🤍 Лайк убран")
    else:
        await callback.answer("Сначала прочитайте этот факт!", show_alert=True)

@dp.callback_query(F.data == "next_fact")
async def next_fact(callback: types.CallbackQuery, state: FSMContext):
    """Следующий факт"""
    user_id = callback.from_user.id
    
    # Получаем категорию из предыдущего факта
    state_data = await state.get_data()
    current_fact_id = state_data.get('current_fact_id')
    
    if current_fact_id:
        current_fact = db.get_fact_by_id(current_fact_id)
        category = current_fact['category'] if current_fact else None
    else:
        category = None
    
    fact = db.get_random_fact(user_id, category)
    
    if fact:
        await state.update_data(current_fact_id=fact['id'])
        
        # Проверяем, лайкал ли пользователь этот факт
        cursor = db.conn.cursor()
        cursor.execute('SELECT liked FROM user_facts WHERE user_id = ? AND fact_id = ?', (user_id, fact['id']))
        liked_row = cursor.fetchone()
        liked = liked_row['liked'] if liked_row else False
        
        await callback.message.edit_text(
            f"<b>📚 Факт #{fact['id']}</b>\n"
            f"<i>Категория: {CATEGORIES.get(fact['category'], 'Общая')}</i>\n\n"
            f"{fact['content']}\n\n"
            f"<i>👁 Просмотров: {fact['views']} | ❤ Лайков: {fact['likes']}</i>",
            reply_markup=get_fact_keyboard(fact['id'], liked),
            parse_mode='HTML'
        )
        await callback.answer()
    else:
        await callback.answer("😔 Больше фактов в этой категории нет", show_alert=True)

@dp.callback_query(F.data == "show_categories")
async def show_categories_callback(callback: types.CallbackQuery):
    """Показать категории из callback"""
    await callback.message.edit_text(
        "📚 <b>Выберите категорию:</b>\n\n"
        "Вы можете читать факты по конкретной тематике или выбрать случайную категорию.",
        reply_markup=get_categories_keyboard(),
        parse_mode='HTML'
    )
    await callback.answer()

@dp.callback_query(F.data == "show_profile")
async def show_profile_callback(callback: types.CallbackQuery):
    """Показать профиль из callback"""
    user_id = callback.from_user.id
    stats = db.get_user_stats(user_id)
    
    if stats:
        achievements = db.get_achievements(user_id)
        achievements_list = "\n".join([
            f"• {ach['achievement_type'].replace('_', ' ').title()} - {ach['achieved_date'][:10]}"
            for ach in achievements[:5]
        ]) if achievements else "Пока нет достижений"
        
        await callback.message.edit_text(
            f"<b>{EMOJI['profile']} Ваш профиль</b>\n\n"
            f"<b>👤 Основная информация:</b>\n"
            f"• Имя: {stats['first_name']}\n"
            f"• Уровень: {stats['level']}\n"
            f"• Опыт: {stats['experience']}/{stats['level'] * 100}\n\n"
            f"<b>📊 Статистика:</b>\n"
            f"• Всего прочитано: {stats['total_facts_read']} фактов\n"
            f"• Понравилось: {stats.get('liked_count', 0)} фактов\n"
            f"• Серия дней: {stats['daily_streak']} {EMOJI['streak']}\n"
            f"• Прочитано сегодня: {stats['daily_facts_count']}/{FACTS_PER_DAY}\n\n"
            f"<b>🏅 Последние достижения:</b>\n{achievements_list}",
            parse_mode='HTML',
            reply_markup=get_back_keyboard()
        )
    await callback.answer()

@dp.callback_query(F.data == "back_to_main")
async def back_to_main(callback: types.CallbackQuery, state: FSMContext):
    """Вернуться в главное меню"""
    await state.clear()
    await callback.message.delete()
    await callback.message.answer(
        "Главное меню:",
        reply_markup=get_main_keyboard(callback.from_user.id)
    )
    await callback.answer()

# Админ-панель
@dp.message(F.text == f"{EMOJI['admin']} Админ-панель")
async def admin_panel(message: types.Message, state: FSMContext):
    """Админ-панель"""
    user_id = message.from_user.id
    
    if user_id in ADMIN_IDS:
        await message.answer(
            f"⚙️ <b>Админ-панель</b>\n\n"
            f"Выберите действие:",
            reply_markup=get_admin_keyboard(),
            parse_mode='HTML'
        )
    else:
        await message.answer("У вас нет доступа к админ-панели.")

@dp.message(F.text == f"{EMOJI['add']} Добавить факты")
async def add_facts_start(message: types.Message, state: FSMContext):
    """Начало добавления фактов"""
    user_id = message.from_user.id
    
    if user_id in ADMIN_IDS:
        await message.answer(
            f"➕ <b>Добавление фактов</b>\n\n"
            f"Отправьте факты в формате:\n"
            f"<code>Факт 1;Факт 2;Факт 3</code>\n\n"
            f"<i>Каждый факт должен заканчиваться точкой с запятой (;)\n"
            f"Максимум {MAX_FACTS_PER_MESSAGE} фактов за раз</i>\n\n"
            f"Сначала выберите категорию:",
            parse_mode='HTML'
        )
        
        # Создаем клавиатуру с категориями для админа
        builder = InlineKeyboardBuilder()
        for key, value in CATEGORIES.items():
            if key != 'all':
                builder.add(InlineKeyboardButton(text=value, callback_data=f"admin_category_{key}"))
        builder.adjust(2, 2, 2)
        
        await message.answer("Выберите категорию:", reply_markup=builder.as_markup())
        
        await state.set_state(AdminStates.waiting_for_category)
    else:
        await message.answer("У вас нет доступа.")

@dp.callback_query(F.data.startswith("admin_category_"), AdminStates.waiting_for_category)
async def select_category_for_facts(callback: types.CallbackQuery, state: FSMContext):
    """Выбор категории для добавления фактов"""
    category = callback.data.split("_")[2]
    await state.update_data(category=category)
    await state.set_state(AdminStates.waiting_for_facts)
    
    await callback.message.edit_text(
        f"✅ Категория выбрана: {CATEGORIES[category]}\n\n"
        f"Теперь отправьте факты в формате:\n"
        f"<code>Факт 1;Факт 2;Факт 3</code>\n\n"
        f"<i>Каждый факт должен заканчиваться точкой с запятой (;)</i>",
        parse_mode='HTML'
    )
    await callback.answer()

@dp.message(AdminStates.waiting_for_facts)
async def add_facts_process(message: types.Message, state: FSMContext):
    """Обработка добавленных фактов"""
    state_data = await state.get_data()
    category = state_data.get('category', 'general')
    
    added_count = db.add_facts(message.text, category, message.from_user.id)
    
    if added_count > 0:
        stats = db.get_stats()
        await message.answer(
            f"✅ Успешно добавлено {added_count} фактов!\n\n"
            f"📊 Общая статистика:\n"
            f"• Всего фактов: {stats['total_facts']}\n"
            f"• Всего пользователей: {stats['total_users']}\n"
            f"• Всего прочтений: {stats['total_reads']}\n"
            f"• Всего лайков: {stats['total_likes']}",
            reply_markup=get_admin_keyboard()
        )
    else:
        await message.answer(
            "❌ Не удалось добавить факты. Проверьте формат.",
            reply_markup=get_admin_keyboard()
        )
    
    await state.clear()

@dp.message(F.text == f"{EMOJI['stats']} Статистика")
async def admin_stats(message: types.Message):
    """Статистика бота"""
    user_id = message.from_user.id
    
    if user_id in ADMIN_IDS:
        stats = db.get_stats()
        top_users = db.get_top_users(5)
        
        top_text = ""
        for i, user in enumerate(top_users, 1):
            username = user['username'] or f"{user['first_name']}"
            top_text += f"{i}. {username}: {user['total_facts_read']} фактов\n"
        
        await message.answer(
            f"📊 <b>Статистика бота</b>\n\n"
            f"<b>Основные метрики:</b>\n"
            f"• Всего пользователей: {stats['total_users']}\n"
            f"• Всего фактов: {stats['total_facts']}\n"
            f"• Всего прочтений: {stats['total_reads']}\n"
            f"• Всего лайков: {stats['total_likes']}\n\n"
            f"<b>Топ-5 читателей:</b>\n{top_text}",
            parse_mode='HTML'
        )
    else:
        await message.answer("У вас нет доступа.")

@dp.message(F.text == f"{EMOJI['broadcast']} Рассылка")
async def broadcast_start(message: types.Message, state: FSMContext):
    """Начало рассылки"""
    user_id = message.from_user.id
    
    if user_id in ADMIN_IDS:
        await message.answer(
            "📢 <b>Рассылка сообщения</b>\n\n"
            "Отправьте сообщение, которое будет разослано всем пользователям.\n"
            "Можно использовать HTML разметку.",
            parse_mode='HTML'
        )
        await state.set_state(AdminStates.waiting_for_broadcast)
    else:
        await message.answer("У вас нет доступа.")

@dp.message(AdminStates.waiting_for_broadcast)
async def broadcast_process(message: types.Message, state: FSMContext):
    """Обработка рассылки"""
    user_ids = db.get_all_users()
    success_count = 0
    fail_count = 0
    
    await message.answer(f"📤 Начинаю рассылку для {len(user_ids)} пользователей...")
    
    for user_id in user_ids:
        try:
            await bot.send_message(
                user_id,
                f"📢 <b>Важное сообщение от администратора:</b>\n\n{message.text}",
                parse_mode='HTML'
            )
            success_count += 1
            await asyncio.sleep(0.05)  # Чтобы не превысить лимиты Telegram
        except Exception as e:
            fail_count += 1
            logger.error(f"Failed to send to {user_id}: {e}")
    
    await message.answer(
        f"✅ Рассылка завершена!\n\n"
        f"• Успешно отправлено: {success_count}\n"
        f"• Не удалось отправить: {fail_count}",
        reply_markup=get_admin_keyboard()
    )
    await state.clear()

@dp.message(F.text == f"{EMOJI['list']} Список фактов")
async def list_facts(message: types.Message):
    """Список фактов"""
    user_id = message.from_user.id
    
    if user_id in ADMIN_IDS:
        cursor = db.conn.cursor()
        cursor.execute('SELECT COUNT(*) as count FROM facts')
        total = cursor.fetchone()['count']
        
        cursor.execute('SELECT * FROM facts ORDER BY id DESC LIMIT 5')
        recent_facts = cursor.fetchall()
        
        facts_list = ""
        for fact in recent_facts:
            short_content = fact['content'][:50] + "..." if len(fact['content']) > 50 else fact['content']
            facts_list += f"#{fact['id']} [{fact['category']}]: {short_content}\n\n"
        
        await message.answer(
            f"📋 <b>Статистика фактов</b>\n\n"
            f"Всего фактов: {total}\n\n"
            f"<b>Последние 5 фактов:</b>\n{facts_list}",
            parse_mode='HTML'
        )
    else:
        await message.answer("У вас нет доступа.")

@dp.message(F.text == f"{EMOJI['back']} Назад")
async def back_to_main_from_admin(message: types.Message, state: FSMContext):
    """Вернуться в главное меню из админ-панели"""
    await state.clear()
    await message.answer(
        "Главное меню:",
        reply_markup=get_main_keyboard(message.from_user.id)
    )

@dp.message(F.text == f"{EMOJI['story']} Интересная история")
async def interesting_story(message: types.Message):
    """Интересная история (длинный факт)"""
    user_id = message.from_user.id
    db.update_user_activity(user_id)
    
    # Ищем факт подлиннее (более 200 символов)
    cursor = db.conn.cursor()
    cursor.execute('''
        SELECT * FROM facts 
        WHERE LENGTH(content) > 200
        ORDER BY RANDOM() 
        LIMIT 1
    ''')
    
    story = cursor.fetchone()
    
    if story:
        # Отмечаем как прочитанный
        try:
            cursor.execute('''
                INSERT OR IGNORE INTO user_facts (user_id, fact_id)
                VALUES (?, ?)
            ''', (user_id, story['id']))
            
            cursor.execute('''
                UPDATE users 
                SET total_facts_read = total_facts_read + 1
                WHERE user_id = ?
            ''', (user_id,))
            
            cursor.execute('UPDATE facts SET views = views + 1 WHERE id = ?', (story['id'],))
            db.conn.commit()
        except:
            pass
        
        await message.answer(
            f"📖 <b>Интересная история #{story['id']}</b>\n"
            f"<i>Категория: {CATEGORIES.get(story['category'], 'Общая')}</i>\n\n"
            f"{story['content']}\n\n"
            f"<i>👁 Просмотров: {story['views']} | ❤ Лайков: {story['likes']}</i>",
            parse_mode='HTML'
        )
    else:
        await message.answer(
            "📖 <b>Интересная история</b>\n\n"
            "Пока длинных историй нет в базе.\n"
            "Администратор добавит их в ближайшее время!\n\n"
            "А пока можете почитать случайные факты!",
            parse_mode='HTML'
        )

# Фоновая задача для ежедневной рассылки фактов
async def scheduled_daily_facts():
    """Ежедневная рассылка фактов дня"""
    while True:
        now = datetime.now().strftime("%H:%M")
        if now == DAILY_FACT_TIME:
            logger.info("Starting daily facts distribution...")
            
            user_ids = db.get_all_users()
            for user_id in user_ids:
                try:
                    user = db.get_user(user_id)
                    if user:
                        today = date.today()
                        last_date = datetime.strptime(user['last_daily_date'], '%Y-%m-%d').date() if user['last_daily_date'] else None
                        
                        # Отправляем только если сегодня еще не отправляли
                        if last_date != today:
                            fact = db.get_random_fact(user_id)
                            if fact:
                                db.update_daily_stats(user_id)
                                
                                await bot.send_message(
                                    user_id,
                                    f"🌅 <b>Доброе утро!</b>\n\n"
                                    f"📅 <b>Ваш факт дня:</b>\n\n"
                                    f"{fact['content']}\n\n"
                                    f"<i>Не забудьте продолжить свою серию чтения!</i>",
                                    parse_mode='HTML'
                                )
                                await asyncio.sleep(0.05)
                except Exception as e:
                    logger.error(f"Failed to send daily fact to {user_id}: {e}")
            
            logger.info("Daily facts distribution completed")
            await asyncio.sleep(60)  # Ждем минуту, чтобы не запускать снова в эту же минуту
        await asyncio.sleep(30)  # Проверяем каждые 30 секунд

# Основная функция
async def main():
    """Основная функция запуска бота"""
    logger.info("Starting bot...")
    
    # Запускаем фоновую задачу
    asyncio.create_task(scheduled_daily_facts())
    
    # Запускаем бота
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
