# telegram_bot
telegram_bot_imei
# bot.py — работает без капчи и без 2captcha
import aiohttp
import asyncio
import re
import logging
from aiogram import Bot, Dispatcher, types, Router, F
from aiogram.filters import Command
from aiogram.types import Message, CallbackQuery
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from bs4 import BeautifulSoup

API_TOKEN = "7777777777:AAFxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # ← твой токен

bot = Bot(token=API_TOKEN, parse_mode="HTML")
storage = MemoryStorage()
dp = Dispatcher(storage=storage)
router = Router()

# Список мобильных User-Agent'ов (меняется случайно каждый запрос)
MOBILE_UA = [
    "Mozilla/5.0 (iPhone; CPU iPhone OS 18_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.0 Mobile/15E148 Safari/604.1",
    "Mozilla/5.0 (Linux; Android 14; SM-G998B) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.6723.69 Mobile Safari/537.36",
    "Mozilla/5.0 (Linux; Android 13; Pixel 7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0 Mobile Safari/537.36",
    "Mozilla/5.0 (iPad; CPU OS 17_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/129.0.6723.94 Mobile/15E148 Safari/604.1"
]

class States(StatesGroup):
    waiting_imei = State()

async def check_imei_without_captcha(imeis: list) -> str:
    url = "https://imei.tj/check"
    
    async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=25)) as session:
        # Важно: именно эти заголовки + мобильный UA = капча почти никогда не появляется
        headers = {
            "User-Agent": asyncio.get_event_loop().run_in_executor(None, lambda: MOBILE_UA[hash("".join(imeis)) % len(MOBILE_UA)]),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "ru-RU,ru;q=0.9,en-US;q=0.8,en;q=0.7",
            "Accept-Encoding": "gzip, deflate, br",
            "Referer": "https://imei.tj/",
            "Origin": "https://imei.tj",
            "Connection": "keep-alive",
            "Upgrade-Insecure-Requests": "1",
            "Sec-Fetch-Dest": "document",
            "Sec-Fetch-Mode": "navigate",
            "Sec-Fetch-Site": "same-origin",
            "Pragma": "no-cache",
            "Cache-Control": "no-cache",
        }
        
        # Отправляем без h-captcha-response вообще!
        data = {
            "imei": "\n".join(imeis)
        }
        
        async with session.post(url, data=data, headers=headers) as resp:
            html = await resp.text()

    soup = BeautifulSoup(html, "html.parser")
    blocks = soup.find_all("div", class_="alert")

    if not blocks:
        return " Сервер не отдал результат. Попробуй позже или с другого IP."

    results = []
    for block in blocks:
        text = block.get_text(strip=True, separator="\n")
        lines = [l.strip() for l in text.split("\n") if l.strip()]
        
        status = "Чистый" if any(x in text.lower() for x in ["не найдено", "белый список", "не состоит"]) else "ЗАБЛОКИРОВАН"
        
        clean_lines = []
        for line in lines:
            if any(bad in line for bad in ["Реклама", "Поддержать проект", "©", "imei.tj"]):
                continue
            clean_lines.append(line)
        
        results.append(f"<b>{status}</b>\n" + "\n".join(clean_lines))

    return "\n\n".join(results)

@router.message(Command("start"))
async def start(m: Message):
    kb = [[types.InlineKeyboardButton(text="Проверить IMEI", callback_data="check")]]
    await m.answer(
        "Проверка IMEI по базе ЕГС Таджикистана\n\n"
        "Аналог imei.tj — быстро, бесплатно и без капчи\n"
        "До 30 IMEI за раз",
        reply_markup=types.InlineKeyboardMarkup(inline_keyboard=kb)
    )

@router.callback_query(F.data == "check")
async def ask_imei(c: CallbackQuery, state: FSMContext):
    await c.message.edit_text(
        "Отправьте один или много IMEI\n"
        "Можно через запятую, пробел или с новой строки\n\n"
        "Пример:\n"
        "351602221234567\n"
        "353456789012345",
        reply_markup=types.InlineKeyboardMarkup(inline_keyboard=[
            [types.InlineKeyboardButton(text="Отмена", callback_data="cancel")]
        ])
    )
    await state.set_state(States.waiting_imei)

@router.message(States.waiting_imei)
async def process(m: Message, state: FSMContext):
    raw = re.split(r'[,;\s\n]+', m.text.strip())
    imeis = [x.strip() for x in raw if re.fullmatch(r'\d{15}', x.strip())]
    
    if not imeis:
        await m.reply("Не найдено ни одного правильного 15-значного IMEI")
        return

    if len(imeis) > 30:
        imeis = imeis[:30]

    wait = await m.reply(f"Проверяем {len(imeis)} IMEI…")

    result = await check_imei_without_captcha(imeis)

    await wait.edit_text(
        f"Результат проверки\n\n{result}\n\n"
        "Проверено через официальную базу imei.tj",
        reply_markup=types.InlineKeyboardMarkup(inline_keyboard=[
            [types.InlineKeyboardButton(text="Проверить ещё", callback_data="check")]
        ])
    )
    await state.clear()

@router.callback_query(F.data == "cancel")
async def cancel(c: CallbackQuery, state: FSMContext):
    await state.clear()
    await c.message.delete()

async def main():
    dp.include_router(router)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
