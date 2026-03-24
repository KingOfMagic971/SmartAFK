# SmartAFK
Умный режим AFK.Сортирует входящие сообщения по важности и пересылает их в выделеный чат Smart AFK Logs
from pyrogram import Client, filters, enums
from pyrogram.types import InlineKeyboardMarkup, InlineKeyboardButton, CallbackQuery
import time

# --- НАСТРОЙКИ ---
AFK_STATUS = {"is_afk": False, "reason": "", "log_chat_id": None}

# 1. Функция для создания лог-канала (Auto-Setup)
async def get_log_chat(client):
    if not AFK_STATUS["log_chat_id"]:
        # Поиск существующего канала или создание нового
        async for dialog in client.get_dialogs():
            if dialog.chat.title == "Smart AFK Logs":
                AFK_STATUS["log_chat_id"] = dialog.chat.id
                return dialog.chat.id
        
        chat = await client.create_channel("Smart AFK Logs", "Логи твоего Smart AFK модуля")
        AFK_STATUS["log_chat_id"] = chat.id
    return AFK_STATUS["log_chat_id"]

# 2. Команда включения AFK
@Client.on_message(filters.me & filters.command("afk", prefixes="."))
async def go_afk(client, message):
    reason = " ".join(message.command[1:]) if len(message.command) > 1 else "Не указана"
    AFK_STATUS["is_afk"] = True
    AFK_STATUS["reason"] = reason
    AFK_STATUS["start_time"] = time.strftime("%H:%M:%S")
    
    await get_log_chat(client) # Проверка канала
    await message.edit(f"💤 **Smart AFK включен**\nПричина: `{reason}`")

# 3. Обработка входящих сообщений
@Client.on_message(filters.private & ~filters.me)
async def handle_afk(client, message):
    if AFK_STATUS["is_afk"]:
        # Кнопки для выбора
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🔥 Срочно", callback_data=f"afk_prio|high|{message.id}")],
            [InlineKeyboardButton("⏳ Подожду", callback_data=f"afk_prio|low|{message.id}")]
        ])
        await message.reply(
            f"Привет! Хозяина сейчас нет.\n**Причина:** `{AFK_STATUS['reason']}`\n\nВыбери важность сообщения:",
            reply_markup=kb
        )

# 4. Логика кнопок и отправка в Smart AFK Logs
@Client.on_callback_query(filters.regex(r"^afk_prio\|"))
async def afk_callback(client, callback_query: CallbackQuery):
    data = callback_query.data.split("|")
    prio = "ВЫСОКИЙ (СРОЧНО) 🔥" if data[1] == "high" else "Обычный ⏳"
    msg_id = int(data[2])
    
    # Получаем само сообщение пользователя
    user_msg = await client.get_messages(callback_query.message.chat.id, msg_id)
    text = user_msg.text or "[Медиа-сообщение]"
    
    # Формируем отчет в спец. канал
    report = (
        f"👤 **User:** {callback_query.from_user.mention}\n"
        f"🕒 **Время:** `{time.strftime('%H:%M:%S')}`\n"
        f"⚡️ **Приоритет:** `{prio}`\n\n"
        f"📝 **Message:**\n`{text}`"
    )
    
    log_id = await get_log_chat(client)
    await client.send_message(
        log_id, 
        report, 
        reply_markup=InlineKeyboardMarkup([
            [InlineKeyboardButton("🔗 Перейти к чату", url=f"tg://user?id={callback_query.from_user.id}")]
        ])
    )
    
    await callback_query.answer("Твой запрос передан!", show_alert=True)
    await callback_query.message.delete() # Удаляем кнопки, чтобы не спамить
