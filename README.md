import asyncio
from pathlib import Path
import os

from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.types import (
    Message, CallbackQuery,
    InlineKeyboardMarkup, InlineKeyboardButton
)
from aiogram.types.input_file import FSInputFile

# Переменные окружения (Render)
TOKEN = os.getenv("BOT_TOKEN")
CHANNEL_USERNAME = "@loadmax"

if not TOKEN:
    raise RuntimeError("BOT_TOKEN не задан")

bot = Bot(token=TOKEN)
dp = Dispatcher()

# Папки с подарками
GIFT_DIRS = {
    "men": Path("gifts/men"),
    "women": Path("gifts/women"),
    "hot": Path("gifts/hot"),
}


def subscribe_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Подписаться на канал", url="https://t.me/loadmax")],
        [InlineKeyboardButton(text="✅ Проверить подписку", callback_data="check_sub")],
    ])


def gifts_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💪 Мужское здоровье", callback_data="gift_men")],
        [InlineKeyboardButton(text="🌸 Женское здоровье", callback_data="gift_women")],
        [InlineKeyboardButton(text="🔥 Дерзкий календарь (18+)", callback_data="gift_hot")],
    ])


def age_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Мне есть 18 лет", callback_data="age_yes")],
        [InlineKeyboardButton(text="❌ Мне нет 18", callback_data="age_no")],
    ])


async def is_subscribed(user_id: int) -> bool:
    member = await bot.get_chat_member(CHANNEL_USERNAME, user_id)
    return member.status in ("member", "administrator", "creator")


def get_all_files(folder: Path):
    if not folder.exists():
        return []
    return [p for p in sorted(folder.iterdir()) if p.is_file()]


def word_files(n: int) -> str:
    if n % 10 == 1 and n % 100 != 11:
        return "файл"
    if 2 <= n % 10 <= 4 and not (12 <= n % 100 <= 14):
        return "файла"
    return "файлов"


async def send_all_files(chat_id: int, folder: Path, emoji="🎁"):
    files = get_all_files(folder)
    if not files:
        await bot.send_message(chat_id, "⚠️ В этой категории пока нет файлов.")
        return

    n = len(files)
    await bot.send_message(chat_id, f"Готово! Отправляю {n} {word_files(n)} {emoji}")

    for file in files:
        await bot.send_document(chat_id, FSInputFile(str(file)))


@dp.message(Command("start"))
async def start(message: Message):
    await message.answer(
        "🎁 Чтобы получить подарок:\n"
        "1️⃣ Подпишитесь на канал\n"
        "2️⃣ Нажмите «Проверить подписку»",
        reply_markup=subscribe_keyboard()
    )


@dp.callback_query(F.data == "check_sub")
async def check_sub(callback: CallbackQuery):
    if await is_subscribed(callback.from_user.id):
        await callback.message.answer(
            "🎉 Подписка подтверждена!\nВыберите подарок 👇",
            reply_markup=gifts_keyboard()
        )
    else:
        await callback.message.answer(
            "❌ Подписки не вижу.\nПодпишитесь и попробуйте снова 👇",
            reply_markup=subscribe_keyboard()
        )
    await callback.answer()


@dp.callback_query(F.data == "gift_men")
async def gift_men(callback: CallbackQuery):
    await callback.message.answer("💪 Подарок про мужское здоровье 👇")
    await send_all_files(callback.message.chat.id, GIFT_DIRS["men"])
    await callback.answer()


@dp.callback_query(F.data == "gift_women")
async def gift_women(callback: CallbackQuery):
    await callback.message.answer("🌸 Подарок про женское здоровье 👇")
    await send_all_files(callback.message.chat.id, GIFT_DIRS["women"])
    await callback.answer()


@dp.callback_query(F.data == "gift_hot")
async def gift_hot(callback: CallbackQuery):
    await callback.message.answer(
        "🔥 Контент 18+\nПодтвердите возраст:",
        reply_markup=age_keyboard()
    )
    await callback.answer()


@dp.callback_query(F.data == "age_yes")
async def age_yes(callback: CallbackQuery):
    await callback.message.answer("🔥 Дерзкий календарь 👇")
    await send_all_files(callback.message.chat.id, GIFT_DIRS["hot"], emoji="🔥")
    await callback.answer()


@dp.callback_query(F.data == "age_no")
async def age_no(callback: CallbackQuery):
    await callback.message.answer(
        "Хорошо 🙂 Выберите другой подарок 👇",
        reply_markup=gifts_keyboard()
    )
    await callback.answer()


async def main():
    for folder in GIFT_DIRS.values():
        folder.mkdir(parents=True, exist_ok=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
