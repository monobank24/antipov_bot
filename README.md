import sqlite3

def create_db():
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()

    # Создаем таблицу пользователей
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER UNIQUE,
        start_date TEXT,
        balance REAL DEFAULT 0
    )''')

    # Создаем таблицу для истории транзакций
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS transactions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        amount REAL,
        currency_from TEXT,
        currency_to TEXT,
        timestamp TEXT,
        FOREIGN KEY (user_id) REFERENCES users(user_id)
    )''')

    conn.commit()
    conn.close()

create_db()





import sqlite3
import random
from datetime import datetime
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler, MessageHandler, Filters, CallbackContext
import requests

TOKEN = 'ВАШ_ТОКЕН_БОТА'

# Функция для получения данных о пользователе из базы данных
def get_user(user_id):
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
    user = cursor.fetchone()
    conn.close()
    return user

# Функция для добавления нового пользователя в базу данных
def add_user(user_id):
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    start_date = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    cursor.execute('INSERT INTO users (user_id, start_date) VALUES (?, ?)', (user_id, start_date))
    conn.commit()
    conn.close()

# Функция для получения истории транзакций пользователя
def get_transactions(user_id):
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM transactions WHERE user_id = ?', (user_id,))
    transactions = cursor.fetchall()
    conn.close()
    return transactions

# Функция для добавления транзакции в базу данных
def add_transaction(user_id, amount, currency_from, currency_to):
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    cursor.execute('INSERT INTO transactions (user_id, amount, currency_from, currency_to, timestamp) VALUES (?, ?, ?, ?, ?)', 
                   (user_id, amount, currency_from, currency_to, timestamp))
    conn.commit()
    conn.close()

# Функция для обмена валют
def exchange_currency(amount, from_currency, to_currency):
    # Для упрощения будем делать фейковый обмен: 1 UAH = 0.1 TRX (например, обмен с фиксированным курсом)
    if from_currency == 'UAH' and to_currency == 'TRX':
        return amount * 0.1  # Примерный курс обмена
    return 0

# Основная функция для обработки команды /start
def start(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    user = get_user(user_id)

    if not user:
        add_user(user_id)

    # Кнопки меню
    keyboard = [
        [InlineKeyboardButton("Мой профиль", callback_data='profile')],
        [InlineKeyboardButton("Обменять валюту", callback_data='exchange')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('Привет! Я помогу тебе обменять валюту. Выбери опцию:', reply_markup=reply_markup)

# Обработка кнопки профиля
def profile(update: Update, context: CallbackContext):
    user_id = update.callback_query.from_user.id
    user = get_user(user_id)
    transactions = get_transactions(user_id)

    profile_text = f"ID клиента: {user[1]}\nДата регистрации: {user[2]}\nБаланс: {user[3]} UAH\n\nИстория транзакций:\n"
    
    if transactions:
        for t in transactions:
            profile_text += f"{t[4]}: {t[2]} {t[3]} -> {t[4]} {t[5]}\n"
    else:
        profile_text += "Нет транзакций."

    update.callback_query.answer()
    update.callback_query.edit_message_text(text=profile_text)

# Обработка кнопки обмена валюты
def exchange(update: Update, context: CallbackContext):
    user_id = update.callback_query.from_user.id
    user = get_user(user_id)

    update.callback_query.answer()
    update.callback_query.edit_message_text(text="Введите сумму в UAH, которую хотите обменять на TRX:")

    # Ожидаем ответа от пользователя
    return 'EXCHANGE_AMOUNT'

# Обработка введенной суммы для обмена
def exchange_amount(update: Update, context: CallbackContext):
    try:
        amount = float(update.message.text)
        user_id = update.message.from_user.id
        trx_amount = exchange_currency(amount, 'UAH', 'TRX')

        # Сохраняем транзакцию
        add_transaction(user_id, amount, 'UAH', 'TRX')

        update.message.reply_text(f"Вы обменяли {amount} UAH на {trx_amount} TRX!")

        # Обновляем баланс
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        cursor.execute('UPDATE users SET balance = balance - ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()

    except ValueError:
        update.message.reply_text("Пожалуйста, введите корректную сумму.")

# Главная функция для обработки сообщений
def main():
    updater = Updater(TOKEN, use_context=True)
    dispatcher = updater.dispatcher

    # Обработчики команд
    dispatcher.add_handler(CommandHandler('start', start))
    dispatcher.add_handler(CallbackQueryHandler(profile, pattern='profile'))
    dispatcher.add_handler(CallbackQueryHandler(exchange, pattern='exchange'))

    # Обработка введенной суммы для обмена
    dispatcher.add_handler(MessageHandler(Filters.text & ~Filters.command, exchange_amount))

    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()
