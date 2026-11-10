import asyncio
import json
import logging
from datetime import datetime, timedelta
from pathlib import Path
from zoneinfo import ZoneInfo

from aiogram import Bot, Dispatcher, types
from aiogram.exceptions import TelegramForbiddenError, TelegramRetryAfter
from aiogram.filters import Command, CommandStart
from aiogram.types import URLInputFile
from apscheduler.schedulers.asyncio import AsyncIOScheduler

BOT_TOKEN = "9069015790:AAEdhtHcnVt5HiP95j1Bxqg8Q56RghiVruQ"

TZ = ZoneInfo("Asia/Tashkent")
WARN_BEFORE_MIN = 5  # seans boshlanishidan necha minut oldin ogohlantirish
USERS_FILE = Path(__file__).with_name("subscribers.json")

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
scheduler = AsyncIOScheduler(timezone="Asia/Tashkent")

# Seans vaqtlari (Toshkent vaqti bilan). Yozgi/qishki vaqtga qarab o'zgartiring.
SESSIONS = {
    "london_open": {"name": "London seansi", "emoji": "🇬🇧", "hour": 14, "minute": 0},
    "ny_open": {"name": "New York seansi", "emoji": "🇺🇸", "hour": 19, "minute": 0},
    "overlap": {"name": "London & New York Overlap", "emoji": "🔥", "hour": 21, "minute": 0},
}

# DIQQAT: Bu yerga rasmning TO'G'RIDAN-TO'G'RI havolasini qo'ying
# (.jpg / .png bilan tugaydigan yoki images.unsplash.com/... ko'rinishidagi link).
IMAGES = {
    "london_open": "https://images.unsplash.com/RASM_1.jpg",
    "ny_open": "https://images.unsplash.com/RASM_2.jpg",
    "overlap": "https://images.unsplash.com/RASM_3.jpg",
}

SIGNALS = {
    "london_open": (
        "⚡️ <b>LONDON SEANSI OCHILISHIGA 5 MINUT QOLDI!</b> 🇬🇧\n"
        "🕒 Soat: 14:00 (UZB)\n"
        "• Banklar va yirik institutlar bozorga kirmoqda. Riskni nazorat qiling!"
    ),
    "ny_open": (
        "⚡️ <b>NEW YORK SEANSI OCHILISHIGA 5 MINUT QOLDI!</b> 🇺🇸\n"
        "🕒 Soat: 19:00 (UZB)\n"
        "• AQShning eng yirik iqtisodiy yangiliklari chiqishi mumkin. Ehtiyot bo'ling!"
    ),
    "overlap": (
        "🔥 <b>LONDON &amp; NEW YORK OVERLAP BOSHLANISHIGA 5 MINUT QOLDI!</b> 🔥\n"
        "🕒 Soat: 21:00 (UZB)\n"
        "• Bozor hajmi (Volume) eng yuqori nuqtaga chiqadi!"
    ),
}


# ---------- Obunachilar (foydalanuvchilar ro'yxati) ----------

def load_users() -> set[int]:
    try:
        return set(json.loads(USERS_FILE.read_text(encoding="utf-8")))
    except (FileNotFoundError, ValueError):
        return set()


def save_users(users: set[int]) -> None:
    USERS_FILE.write_text(json.dumps(sorted(users)), encoding="utf-8")


USERS: set[int] = load_users()


# ---------- Vaqt hisoblash ----------

def next_occurrence(hour: int, minute: int) -> datetime:
    """Eng yaqin ish kuni (dushanba-juma) uchun seans vaqtini qaytaradi."""
    now = datetime.now(TZ)
    for add_days in range(0, 8):
        candidate = (now + timedelta(days=add_days)).replace(
            hour=hour, minute=minute, second=0, microsecond=0
        )
        if candidate > now and candidate.weekday() < 5:
            return candidate
    raise RuntimeError("Keyingi seans vaqti topilmadi")


def format_delta(delta: timedelta) -> str:
    total_minutes = int(delta.total_seconds() // 60)
    days, rem = divmod(total_minutes, 60 * 24)
    hours, minutes = divmod(rem, 60)
    parts = []
    if days:
        parts.append(f"{days} kun")
    if hours:
        parts.append(f"{hours} soat")
    parts.append(f"{minutes} minut")
    return " ".join(parts)


def build_countdown_text() -> str:
    now = datetime.now(TZ)
    items = []
    for s in SESSIONS.values():
        items.append((next_occurrence(s["hour"], s["minute"]), s))
    items.sort(key=lambda x: x[0])

    lines = [
        f"🕒 Hozir: <b>{now:%H:%M}</b> (Toshkent vaqti)\n",
        "📊 <b>Keyingi seanslargacha qolgan vaqt:</b>\n",
    ]
    for start, s in items:
        date_part = f" ({start:%d.%m})" if start.date() != now.date() else ""
        lines.append(
            f"{s['emoji']} <b>{s['name']}</b> — {start:%H:%M}{date_part}"
            f"\n   ⏳ {format_delta(start - now)} qoldi\n"
        )
    lines.append(
        f"🔔 Har bir seansdan {WARN_BEFORE_MIN} minut oldin bot sizga o'zi signal yuboradi.\n"
        "To'xtatish: /stop"
    )
    return "\n".join(lines)


# ---------- Buyruqlar ----------

@dp.message(CommandStart())
async def cmd_start(message: types.Message):
    chat_id = message.chat.id
    if chat_id not in USERS:
        USERS.add(chat_id)
        save_users(USERS)
    text = f"👋 Salom, <b>{message.from_user.full_name}</b>!\n✅ Signallar yoqildi.\n\n"
    await message.answer(text + build_countdown_text(), parse_mode="HTML")


@dp.message(Command("vaqt"))
async def cmd_time(message: types.Message):
    await message.answer(build_countdown_text(), parse_mode="HTML")


@dp.message(Command("stop"))
async def cmd_stop(message: types.Message):
    if message.chat.id in USERS:
        USERS.discard(message.chat.id)
        save_users(USERS)
    await message.answer("🔕 Signallar o'chirildi. Qayta yoqish uchun /start bosing.")


# ---------- Signal yuborish (botning o'zi barcha foydalanuvchilarga) ----------

async def send_to_user(chat_id: int, key: str) -> None:
    try:
        await bot.send_photo(
            chat_id=chat_id,
            photo=URLInputFile(IMAGES[key]),
            caption=SIGNALS[key],
            parse_mode="HTML",
        )
    except TelegramForbiddenError:
        # Foydalanuvchi botni bloklagan — ro'yxatdan o'chiramiz
        USERS.discard(chat_id)
        save_users(USERS)
    except TelegramRetryAfter as e:
        await asyncio.sleep(e.retry_after)
        await send_to_user(chat_id, key)
    except Exception as e:
        logging.error("Yuborishda xato (%s): %s", chat_id, e)


async def send_signal(key: str):
    users = list(USERS)
    logging.info("Signal '%s' %d ta foydalanuvchiga yuborilmoqda", key, len(users))
    for chat_id in users:
        await send_to_user(chat_id, key)
        await asyncio.sleep(0.05)  # Telegram limiti (~30 xabar/soniya) uchun


def setup_scheduler():
    """Har bir seansdan 5 minut oldin signal yuborish vazifasini qo'shadi."""
    for key, s in SESSIONS.items():
        session_time = datetime(2000, 1, 1, s["hour"], s["minute"])
        warn_time = session_time - timedelta(minutes=WARN_BEFORE_MIN)
        scheduler.add_job(
            send_signal,
            "cron",
            args=[key],
            day_of_week="mon-fri",
            hour=warn_time.hour,
            minute=warn_time.minute,
            id=key,
        )


async def main():
    logging.basicConfig(level=logging.INFO)
    setup_scheduler()
    scheduler.start()
    print("🚀 Seans eslatma boti ishga tushdi...")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
