ربات تلگرامی برای فروشگاه 
این ربات با زبان پاییتون نوشته  شده و برای فرشگاه ها کاربرد دارد این ربات قابلیت افزودن محصول،لیست محصولات،قیمت دهی و مقایسه چند محصول باهم هم از نظر قیمت و فاکتور های دیگ است.
[unishop.py](https://github.com/user-attachments/files/22254603/unishop.py)
from aiogram import Bot, Dispatcher, types, F
from aiogram.client.default import DefaultBotProperties
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.enums import ParseMode
from aiogram.filters import Command
import mysql.connector
import asyncio
from dotenv import load_dotenv
import os

# بارگذاری متغیرهای محیطی
load_dotenv()
TOKEN = os.getenv('BOT_TOKEN',)

bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
storage = MemoryStorage()
dp = Dispatcher(storage=storage)

# اتصال به MySQL
def get_db_connection():
    try:
        conn = mysql.connector.connect(
            host="localhost",
            user="root",
            password="",
            database="unishop_db",
            auth_plugin='mysql_native_password',
            connect_timeout=5
        )
        return conn
    except Exception as e:
        print(f"خطا در اتصال به دیتابیس: {e}")
        raise

# دستور /start با منوی اینلاین
@dp.message(Command('start'))
async def start(message: types.Message):
    keyboard = types.InlineKeyboardMarkup(
        inline_keyboard=[
            [types.InlineKeyboardButton(text="➕ افزودن محصول", callback_data="add_help")],
            [types.InlineKeyboardButton(text="🔍 جستجوی محصول", callback_data="search_help")],
            [types.InlineKeyboardButton(text="🔄 مقایسه محصولات", callback_data="compare_help")],
            [types.InlineKeyboardButton(text="🗑️ حذف محصول", callback_data="delete_help")],
            [types.InlineKeyboardButton(text="📋 لیست محصولات", callback_data="list_products")],
        ]
    )
    await message.answer(
        "🛒 به ربات فروشگاهی یونی بات خوش آمدید!\n\n"
        "برای استفاده از هر قابلیت، دکمه مورد نظر را انتخاب کنید:",
        reply_markup=keyboard
    )

# نمایش راهنمای /add وقتی کاربر دکمه را میزند
@dp.callback_query(F.data == "add_help")
async def add_help(callback: types.CallbackQuery):
    await callback.message.edit_text(
        "📝 برای افزودن محصول، از فرمت زیر استفاده کنید:\n\n"
        "<code>/add نام محصول|قیمت|ویژگی‌ها (اختیاری)</code>\n\n"
        "مثال:\n"
        "<code>/add لپ‌تاپ ایسوس|15000000|پردازنده Core i7, 16GB RAM</code>",
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

# نمایش راهنمای /search
@dp.callback_query(F.data == "search_help")
async def search_help(callback: types.CallbackQuery):
    await callback.message.edit_text(
        "🔍 برای جستجوی محصول، از فرمت زیر استفاده کنید:\n\n"
        "<code>/search نام محصول</code>\n\n"
        "مثال:\n"
        "<code>/search لپ‌تاپ</code>",
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

# نمایش راهنمای /compare
@dp.callback_query(F.data == "compare_help")
async def compare_help(callback: types.CallbackQuery):
    await callback.message.edit_text(
        "🔄 برای مقایسه محصولات، کافیست دستور زیر را بزنید:\n\n"
        "<code>/compare</code>\n\n"
        "ربات به صورت خودکار ارزان‌ترین و گران‌ترین محصول را مقایسه می‌کند.",
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

# نمایش راهنمای /delete
@dp.callback_query(F.data == "delete_help")
async def delete_help(callback: types.CallbackQuery):
    await callback.message.edit_text(
        "🗑️ برای حذف محصول، یکی از روش‌های زیر را استفاده کنید:\n\n"
        "• حذف با ID: <code>/delete 1</code>\n"
        "• حذف با نام: <code>/delete نام محصول</code>\n\n"
        "مثال:\n"
        "<code>/delete لپ‌تاپ ایسوس</code>",
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

# نمایش لیست محصولات
@dp.callback_query(F.data == "list_products")
async def list_products_callback(callback: types.CallbackQuery):
    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute('SELECT * FROM products ORDER BY price')
        products = cursor.fetchall()

        if not products:
            await callback.message.edit_text("❌ هیچ محصولی در فروشگاه موجود نیست")
        else:
            response = "📋 لیست محصولات:\n\n"
            for product in products:
                response += (
                    f"📦 {product['name']}\n"
                    f"💰 {product['price']:,} تومان\n"
                    f"📌 {product['features']}\n"
                    f"🆔 شناسه: {product['id']}\n\n"
                )
            await callback.message.edit_text(response)
    except Exception as e:
        await callback.message.edit_text(f"❌ خطا در دریافت لیست محصولات: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            cursor.close()
            conn.close()
    await callback.answer()

# دستور /add
@dp.message(Command('add'))
async def add_product(message: types.Message):
    text = message.text.replace('/add ', '').strip()
    args = text.split('|')

    if len(args) < 2:
        await message.answer(
            "📝 برای افزودن محصول، از فرمت زیر استفاده کنید:\n\n"
            "<code>/add نام محصول|قیمت|ویژگی‌ها (اختیاری)</code>\n\n"
            "مثال:\n"
            "<code>/add لپ‌تاپ ایسوس|15000000|پردازنده Core i7, 16GB RAM</code>",
            parse_mode=ParseMode.HTML
        )
        return

    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        name = args[0].strip()
        price = float(args[1].strip())
        features = args[2].strip() if len(args) > 2 else "بدون ویژگی"

        cursor.execute(
            'INSERT INTO products (name, price, features) VALUES (%s, %s, %s)',
            (name, price, features)
        )
        conn.commit()
        await message.answer(
            f"✅ محصول «{name}» با قیمت {price:,} تومان اضافه شد!")
    except ValueError:
        await message.answer("❌ قیمت باید عدد باشد!")
    except Exception as e:
        await message.answer(f"❌ خطا در افزودن محصول: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            cursor.close()
            conn.close()

# دستور /search
@dp.message(Command('search'))
async def search_product(message: types.Message):
    keyword = message.text.replace('/search ', '').strip()

    if not keyword:
        await message.answer("❌ لطفا نام محصول را وارد کنید\nمثال: /search لپ‌تاپ")
        return

    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        
        cursor.execute('SELECT * FROM products WHERE name LIKE %s',
                      (f'%{keyword}%',))
        products = cursor.fetchall()

        if not products:
            await message.answer(f"❌ محصولی با نام «{keyword}» یافت نشد")
        else:
            response = f"🔍 نتایج جستجو برای «{keyword}»:\n\n"
            for product in products:
                response += (
                    f"📦 {product['name']}\n"
                    f"💰 {product['price']:,} تومان\n"
                    f"📌 {product['features']}\n"
                    f"🆔 شناسه: {product['id']}\n\n"
                )
            await message.answer(response)
    except Exception as e:
        await message.answer(f"❌ خطا در جستجو: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            cursor.close()
            conn.close()

# دستور /compare
@dp.message(Command('compare'))
async def compare_products(message: types.Message):
    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        
        cursor.execute('SELECT id, name, price, features FROM products ORDER BY price')
        products = cursor.fetchall()

        if len(products) < 2:
            await message.answer(
                "❌ حداقل به ۲ محصول برای مقایسه نیاز دارید\n"
                "ابتدا محصولات را با دستور /add اضافه کنید"
            )
            return

        cheapest = products[0]
        most_expensive = products[-1]

        if cheapest['id'] == most_expensive['id']:
            p1, p2 = products[0], products[1] if len(products) > 1 else products[0]
        else:
            p1, p2 = cheapest, most_expensive

        diff = abs(p1['price'] - p2['price'])

        response = (
            "🔄 مقایسه محصولات:\n\n"
            f"🟢 ارزان‌تر: {p1['name']}\n"
            f"   💰 {p1['price']:,} تومان\n"
            f"   📌 {p1['features']}\n\n"
            f"🔴 گران‌تر: {p2['name']}\n"
            f"   💰 {p2['price']:,} تومان\n"
            f"   📌 {p2['features']}\n\n"
            f"💡 تفاوت قیمت: {diff:,} تومان"
        )
        await message.answer(response)
    except Exception as e:
        await message.answer(f"❌ خطا در مقایسه محصولات: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            cursor.close()
            conn.close()

# دستور /list
@dp.message(Command('list'))
async def list_products(message: types.Message):
    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        
        cursor.execute('SELECT * FROM products ORDER BY price')
        products = cursor.fetchall()

        if not products:
            await message.answer("❌ هیچ محصولی در فروشگاه موجود نیست")
            return

        response = "📋 لیست محصولات:\n\n"
        for product in products:
            response += (
                f"📦 {product['name']}\n"
                f"💰 {product['price']:,} تومان\n"
                f"📌 {product['features']}\n"
                f"🆔 {product['id']}\n\n"
            )
        await message.answer(response)
    except Exception as e:
        await message.answer(f"❌ خطا در دریافت لیست محصولات: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            cursor.close()
            conn.close()

# دستور /delete
@dp.message(Command("delete"))
async def delete_product(message: types.Message):
    args = message.text.split()

    if len(args) < 2:
        await message.answer(
            "🗑️ برای حذف محصول، یکی از روش‌های زیر را استفاده کنید:\n\n"
            "• حذف با ID: <code>/delete 1</code>\n"
            "• حذف با نام: <code>/delete نام محصول</code>\n\n"
            "مثال:\n"
            "<code>/delete لپ‌تاپ ایسوس</code>",
            parse_mode=ParseMode.HTML
        )
        return

    target = ' '.join(args[1:])

    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        try:
            product_id = int(target)
            cursor.execute('DELETE FROM products WHERE id = %s', (product_id,))
        except ValueError:
            cursor.execute('DELETE FROM products WHERE name LIKE %s',
                          (f'%{target}%',))

        if cursor.rowcount > 0:
            conn.commit()
            await message.answer("✅ محصول با موفقیت حذف شد")
        else:
            await message.answer("❌ محصولی یافت نشد")
    except Exception as e:
        await message.answer(f"❌ خطا در حذف محصول: {str(e)}")
    finally:
        if 'conn' in locals() and conn.is_connected():
            cursor.close()
            conn.close()

async def main():
    print("🤖 ربات فروشگاه شروع شد...")
    await dp.start_polling(bot, skip_updates=True)

if __name__ == '__main__':
    asyncio.run(main())
