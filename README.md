import telebot
import sqlite3
import random
import os

TOKEN = ''

bot = telebot.TeleBot(TOKEN)

def delete_old_db():
    if os.path.exists("users.db"):
        os.remove("users.db")
        print("Старая база удалена!")
    else:
        print("Старая база не найдена — создаём новую.")

def init_db():
    conn = sqlite3.connect("users.db")
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            name TEXT,
            country TEXT
        )
    """)
    conn.commit()
    conn.close()
    print("База успешно создана или уже существует.")

def save_user(user_id, name, country):
    conn = sqlite3.connect("users.db")
    c = conn.cursor()
    c.execute("""
        INSERT INTO users (user_id, name, country)
        VALUES (?, ?, ?)
    """, (user_id, name, country))
    conn.commit()
    conn.close()
    print(f"Сохранён пользователь {name}")

user_data = {}
active_users = set()

@bot.message_handler(commands=['start'])
def handle_start(message):
    user_id = message.from_user.id
    user_data[user_id] = {}
    bot.send_message(message.chat.id, "Привет! Давай зарегистрируемся.\nКак тебя зовут?")
    bot.register_next_step_handler(message, ask_country)

def ask_country(message):
    user_id = message.from_user.id
    user_data[user_id]['name'] = message.text
    bot.send_message(message.chat.id, "Из какой ты страны?")
    bot.register_next_step_handler(message, save_all_data)

def save_all_data(message):
    user_id = message.from_user.id
    user_data[user_id]['country'] = message.text

    save_user(
        user_id,
        user_data[user_id]['name'],
        user_data[user_id]['country']
    )

    bot.send_message(message.chat.id,
        "Спасибо, вы зарегистрировались!\n"
        "Вы теперь можете пользоваться ботом.\n"
        "Нажмите /gamesben если вы не смогли понять, что делает бот.\n"
        "Этот бот помогает в решениях!"
    )

@bot.message_handler(commands=['gamesben'])
def bot_command(message):
    user_id = message.from_user.id
    active_users.add(user_id)
    bot.send_message(
        message.chat.id,
        "Ты что-то должен написать или же сказать мне, и я тебе отвечу.\n"
        "Поддерживаем ли мы такое или нет? Если вы не поняли, задайте вопросы мне, и я отвечу!"
    )

@bot.message_handler(func=lambda message: True)
def answer_question(message):
    user_id = message.from_user.id
    if user_id in active_users:
        answer = random.choice(["Да", "Нет"])
        bot.send_message(message.chat.id, f"Бен: {answer}")

@bot.message_handler(commands=['about'])
def about_command(message):
    bot.send_message(
        message.chat.id,
        "Я бот, который регистрирует имя и страну и случайно отвечает 'Да' или 'Нет' на ваши вопросы."
    )

if __name__ == "__main__":
    delete_old_db()
    init_db()
    print("Мой Бен запущен!")
    bot.infinity_polling()
