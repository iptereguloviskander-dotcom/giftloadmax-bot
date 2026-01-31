# giftloadmax-bot
Telegram bot for subscription check and gift distribution
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

# Берём из переменных окружения на сервере (Render)
TOKEN = os.getenv("BOT_TOKEN")
CHANNEL_USERNAME = os.getenv("CHANNEL_USERNAME")  # например: @my_public_channel

if not TOKEN or not CHANNEL_USERNAME:
    raise RuntimeError("BOT_TOKEN или CHANNEL_USERNAME не заданы в переменных окружения")

bot = Bot(token=TOKEN)
dp = Dispatcher()

# Папки с подарками (в каждой может быть любое количество файлов)
GIFT_DIRS = {
    "men": Path("gifts/men"),
    "women": Path("gifts/women"),
    "hot": Path("gifts/hot"),
}


def subscribe_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Подписаться на канал", url=f"https://t.me/{CHANNEL_USERNAME[1:]}")],
        [InlineKeyboardButton(text="✅ Проверить подписку", callback_data="check_sub")],
    ])


def gifts_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💪 Мужское здоровье", callback_data="gift_men")],
        [InlineKeyboardButton(text="🌸 Женское здоровье", callback_data="gift_women")],
        [InlineKeyboardButton(text="🔥 Дерзкий календарь (18+)", callback_data="gift_hot")],
    ])


def age_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Мне есть 18 лет", callback_data="age_yes")],
        [InlineKeyboardButton(text="❌ Мне нет 18", callback_data="age_no")],
    ])


async def is_subscribed(user_id: int) -> bool:
    member = await bot.get_chat_member(CHANNEL_USERNAME, user_id)
    return member.status in ("member", "administrator", "creator")


def get_all_files(folder: Path) -> list[Path]:
    if not folder.exists():
        return []
    return [p for p in sorted(folder.iterdir()) if p.is_file()]


def count_phrase(n: int) -> str:
    # “1 файл”, “2 файла”, “5 файлов”
    if n % 10 == 1 and n % 100 != 11:
        return "файл"
    if 2 <= n % 10 <= 4 and not (12 <= n % 100 <= 14):
        return "файла"
    return "файлов"


async def send_all_files(chat_id: int, folder: Path, emoji: str = "🎁"):
    files = get_all_files(folder)
    if not files:
        await bot.send_message(chat_id, "⚠️ В этой категории пока нет файлов.")
        return

    n = len(files)
    await bot.send_message(chat_id, f"Готово! Отправляю {n} {count_phrase(n)} {emoji}")

    for file in files:
        await bot.send_document(chat_id, FSInputFile(str(file)))


async def require_subscription_or_prompt(callback: CallbackQuery) -> bool:
    if await is_subscribed(callback.from_user.id):
        return True

    await callback.message.answer(
        "❌ Подписки не вижу.\nПодпишитесь и нажмите «Проверить подписку» 👇",
        reply_markup=subscribe_keyboard()
    )
    return False


@dp.message(Command("start"))
async def start(message: Message):
    await message.answer(
        "🎁 Чтобы получить подарок:\n"
        "1️⃣ Подпишитесь на канал\n"
        "2️⃣ Нажмите «Проверить подписку»",
        reply_markup=subscribe_keyboard()
    )


@dp.message(Command("help"))
async def help_cmd(message: Message):
    await message.answer(
        "ℹ️ Как получить подарок:\n"
        "1️⃣ Подпишитесь на канал\n"
        "2️⃣ Нажмите «Проверить подписку»\n"
        "3️⃣ Выберите подарок\n\n"
        "Можно получать подарки сколько угодно раз 🎁"
    )


@dp.message(Command("gifts"))
async def gifts_cmd(message: Message):
    # Если человек вручную вводит /gifts — проверим подписку
    if await is_subscribed(message.from_user.id):
        await message.answer("Выберите подарок 👇", reply_markup=gifts_keyboard())
    else:
        await message.answer(
            "Сначала подтвердите подписку 👇",
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
    ok = await require_subscription_or_prompt(callback)
    await callback.answer()
    if not ok:
        return

    await callback.message.answer("💪 Подарок про мужское здоровье 👇")
    await send_all_files(callback.message.chat.id, GIFT_DIRS["men"], emoji="🎁")


@dp.callback_query(F.data == "gift_women")
async def gift_women(callback: CallbackQuery):
    ok = await require_subscription_or_prompt(callback)
    await callback.answer()
    if not ok:
        return

    await callback.message.answer("🌸 Подарок про женское здоровье 👇")
    await send_all_files(callback.message.chat.id, GIFT_DIRS["women"], emoji="🎁")


@dp.callback_query(F.data == "gift_hot")
async def gift_hot(callback: CallbackQuery):
    ok = await require_subscription_or_prompt(callback)
    await callback.answer()
    if not ok:
        return

    await callback.message.answer(
        "🔥 Этот подарок предназначен для лиц 18+.\n"
        "Подтвердите возраст:",
        reply_markup=age_keyboard()
    )


@dp.callback_query(F.data == "age_yes")
async def age_yes(callback: CallbackQuery):
    ok = await require_subscription_or_prompt(callback)
    await callback.answer()
    if not ok:
        return

    await callback.message.answer("🔥 Дерзкий календарь 👇")
    await send_all_files(callback.message.chat.id, GIFT_DIRS["hot"], emoji="🔥")


@dp.callback_query(F.data == "age_no")
async def age_no(callback: CallbackQuery):
    await callback.message.answer("Ок 🙂 Тогда выберите другой подарок 👇", reply_markup=gifts_keyboard())
    await callback.answer()


async def main():
    # Создаём папки, если их нет
    for folder in GIFT_DIRS.values():
        folder.mkdir(parents=True, exist_ok=True)

    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
