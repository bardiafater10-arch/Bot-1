# -*- coding: utf-8 -*-
import os
import json
import random
import string
import telebot
from telebot import types

# =======================
# تنظیمات اولیه — این سه مقدار رو تنظیم کن
# =======================
TOKEN = "8537825137:AAED0TFXKCLxnZPrLGhtbWG2H4qIRRFiQoU"
ADMINS = [6708555455]              # لیست ایدی‌های عددی ادمین(ها) — مثال: [6708555455, 987654321]
CHANNELS = ["@bardia_Faster"]      # لیست کانال‌ها با @ — می‌تونی چندتا بذاری
# =======================

FILES_PATH = "Robot_files_downloader"
USERS_FILE = "uduudsers.json"
FILES_FILE = "filduuduees.json"

if not os.path.exists(FILES_PATH):
    os.makedirs(FILES_PATH)

bot = telebot.TeleBot(TOKEN)

# =======================
# بارگذاری و ذخیره‌سازی JSON
# =======================
def load_json(path):
    if os.path.exists(path):
        try:
            with open(path, "r", encoding="utf-8") as f:
                return json.load(f)
        except:
            return {}
    return {}

def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=4)

users = load_json(USERS_FILE)   # user_id (str) -> {points, bought_files:list, ref_code, ref_asked:bool, used_refs:list}
files = load_json(FILES_FILE)   # file_code -> {name, caption, price, file_path}

# =======================
# توابع کمکی
# =======================
def is_admin(user_id):
    return int(user_id) in [int(x) for x in ADMINS]

def generate_ref_code():
    while True:
        code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=7))
        existing = [u.get("ref_code") for u in users.values()]
        if code not in existing:
            return code

def ensure_user(uid):
    uid = str(uid)
    if uid not in users:
        users[uid] = {
            "points": 0,
            "bought_files": [],
            "ref_code": generate_ref_code(),
            "ref_asked": False,
            "used_refs": []
        }
        save_json(USERS_FILE, users)
    return users[uid]

def is_member_all(user_id):
    """بررسی عضویت در تمام CHANNELS (با username یا id کار می‌کند)"""
    for ch in CHANNELS:
        try:
            member = bot.get_chat_member(ch, int(user_id))
            if member.status in ["left", "kicked"]:
                return False
        except Exception:
            # تو اینجا سعی کن با username چک کنی (در صورتی که int() اشکال داشت)
            try:
                member = bot.get_chat_member(ch, user_id)
                if member.status in ["left", "kicked"]:
                    return False
            except Exception:
                return False
    return True

def send_user_menu(uid):
    uid = str(uid)
    if is_admin(uid):
        return  # ادمین منوی کاربر را نمی‌بیند
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("کد دعوت و اطلاعات اکانت")
    markup.add("جستجوی فایل")
    bot.send_message(uid, "منوی کاربری:", reply_markup=markup)

def send_admin_menu(uid):
    uid = str(uid)
    if not is_admin(uid):
        return
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("افزودن فایل", "افزودن امتیاز")
    markup.add("لیست فایل‌ها")
    bot.send_message(uid, "پنل ادمین:", reply_markup=markup)

# =======================
# START
# =======================
@bot.message_handler(commands=['start'])
def cmd_start(msg):
    user_id = str(msg.from_user.id)
    ensure_user(user_id)

    # ادمین => منوی ادمین
    if is_admin(user_id):
        send_admin_menu(user_id)
        return

    # اگر عضو کانال نیست، اول عضویت اجباری نشان بده
    if not is_member_all(user_id):
        markup = types.InlineKeyboardMarkup()
        for ch in CHANNELS:
            markup.add(types.InlineKeyboardButton(f"عضویت در {ch}", url=f"https://t.me/{ch[1:]}"))
        markup.add(types.InlineKeyboardButton("✅ بررسی عضویت", callback_data="check_membership"))
        bot.send_message(user_id, "برای استفاده از ربات باید ابتدا در کانال(ها) عضو شوید 👇", reply_markup=markup)
        return

    # اگر عضو است و هنوز پرسش ریفرال انجام نشده، یک‌بار بپرس
    if not users[user_id].get("ref_asked", False):
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("دارم", callback_data="ref_have"))
        markup.add(types.InlineKeyboardButton("ندارم", callback_data="ref_no"))
        bot.send_message(user_id, "آیا کد دعوت دارید؟", reply_markup=markup)
        return

    # در غیر این صورت منو را نمایش بده
    send_user_menu(user_id)

# =======================
# CALLBACK: بررسی عضویت
# =======================
@bot.callback_query_handler(func=lambda c: c.data == "check_membership")
def cb_check_membership(call):
    uid = str(call.from_user.id)
    # حذف پیام دکمه‌ها (که کاربر روش کلیک کرده)
    try:
        bot.delete_message(call.message.chat.id, call.message.message_id)
    except:
        pass

    if is_member_all(uid):
        bot.answer_callback_query(call.id, "✅ عضویت تایید شد")
        # بعد از تایید، اگر پرسش ریفرال انجام نشده از کاربر بپرس
        if not users[uid].get("ref_asked", False):
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("دارم", callback_data="ref_have"))
            markup.add(types.InlineKeyboardButton("ندارم", callback_data="ref_no"))
            bot.send_message(uid, "آیا کد دعوت دارید؟", reply_markup=markup)
        else:
            send_user_menu(uid)
    else:
        bot.answer_callback_query(call.id, "❌ هنوز عضو نشده‌اید")

# =======================
# CALLBACK: ریفرال (دارم / ندارم)
# =======================
@bot.callback_query_handler(func=lambda c: c.data in ["ref_have", "ref_no"])
def cb_ref_choice(call):
    uid = str(call.from_user.id)
    # حذف پیام انتخاب
    try:
        bot.delete_message(call.message.chat.id, call.message.message_id)
    except:
        pass

    if call.data == "ref_no":
        users[uid]["ref_asked"] = True
        save_json(USERS_FILE, users)
        bot.answer_callback_query(call.id, "خوب، بدون کد ادامه می‌دهیم")
        send_user_menu(uid)
        return

    # اگر دارم => از کاربر بخواه کد را وارد کند (و پیام ورودی حذف شود)
    bot.answer_callback_query(call.id, "لطفاً کد دعوت را وارد کنید (حروف و عدد)")
    msg = bot.send_message(uid, "لطفاً کد دعوت را وارد کنید:")
    bot.register_next_step_handler(msg, process_ref_input)

def process_ref_input(message):
    uid = str(message.from_user.id)
    code = message.text.strip().upper()
    # حذف پیام ورودی کاربر برای تمیزی
    try:
        bot.delete_message(message.chat.id, message.message_id)
    except:
        pass

    # اگر خودشان کد خودشون را زدند یا قبلا از یک کد استفاده کرده‌اند => رد کن
    if code == users[uid].get("ref_code"):
        bot.send_message(uid, "❌ نمی‌توانید از کد خود استفاده کنید")
        users[uid]["ref_asked"] = True
        save_json(USERS_FILE, users)
        send_user_menu(uid)
        return

    # پیدا کردن صاحب کد
    owner = None
    for other_uid, info in users.items():
        if info.get("ref_code") == code:
            owner = other_uid
            break

    if owner is None:
        bot.send_message(uid, "❌ کد دعوت معتبر نیست")
        users[uid]["ref_asked"] = True
        save_json(USERS_FILE, users)
        send_user_menu(uid)
        return

    # جلوگیری از استفاده‌ی مجدد از همان کد توسط همان کاربر
    if code in users[uid].get("used_refs", []):
        bot.send_message(uid, "❌ قبلا از این کد استفاده کرده‌اید")
        users[uid]["ref_asked"] = True
        save_json(USERS_FILE, users)
        send_user_menu(uid)
        return

    # اعمال امتیاز به صاحب کد و ذخیره
    users[owner]["points"] = users[owner].get("points", 0) + 1
    users[uid].setdefault("used_refs", []).append(code)
    users[uid]["ref_asked"] = True
    save_json(USERS_FILE, users)

    # اطلاع به هر دو طرف
    bot.send_message(uid, "✅ کد دعوت پذیرفته شد — ممنون!")
    try:
        bot.send_message(owner, f"✅ یک امتیاز جدید از کد دعوت دریافت کردید! امتیاز فعلی: {users[owner]['points']}")
    except:
        pass
    send_user_menu(uid)

# =======================
# منوی کاربر: دیدن کد و جستجوی فایل
# =======================
@bot.message_handler(func=lambda m: m.text == "کد دعوت و اطلاعات اکانت")
def cmd_account_info(msg):
    uid = str(msg.from_user.id)
    ensure_user(uid)
    bot.send_message(uid, f"امتیاز شما: {users[uid].get('points',0)}\nکد دعوت شما: {users[uid].get('ref_code')}")

@bot.message_handler(func=lambda m: m.text == "جستجوی فایل")
def cmd_search_file(msg):
    uid = str(msg.from_user.id)
    if not is_member_all(uid):
        bot.send_message(uid, "❌ لطفاً ابتدا در کانال(ها) عضو شوید")
        return
    q = bot.send_message(uid, "کد فایل را وارد کنید:")
    bot.register_next_step_handler(q, process_file_search)

def process_file_search(message):
    uid = str(message.from_user.id)
    code = message.text.strip()
    # حذف پیام ورودی برای تمیزی
    try:
        bot.delete_message(message.chat.id, message.message_id)
    except:
        pass

    if code not in files:
        bot.send_message(uid, "فایل پیدا نشد ❌")
        return

    # اگر قبلاً خریداری کرده باشه:
    if code in users[uid].get("bought_files", []):
        f = files[code]
        bot.send_document(uid, open(f["file_path"], "rb"), caption=f.get("caption",""))
        bot.send_message(uid, "✅ شما قبلاً این فایل را خریده‌اید — ارسال رایگان شد")
        return

    price = files[code]["price"]
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton(f"خرید فایل ({price} امتیاز)", callback_data=f"buy_{code}"))
    bot.send_message(uid, f"فایل پیدا شد: {files[code]['name']}\nقیمت: {price} امتیاز\nآیا مطمئن هستید؟", reply_markup=markup)

# =======================
# خرید فایل (callback)
# =======================
@bot.callback_query_handler(func=lambda c: c.data and c.data.startswith("buy_"))
def cb_buy_file(call):
    uid = str(call.from_user.id)
    code = call.data[4:]
    # حذف پیام دکمه‌ها تا شلوغی نشه
    try:
        bot.edit_message_reply_markup(call.message.chat.id, call.message.message_id, reply_markup=None)
    except:
        pass

    if code not in files:
        bot.answer_callback_query(call.id, "فایل موجود نیست ❌")
        return

    price = files[code]["price"]
    if users[uid].get("points",0) < price:
        bot.answer_callback_query(call.id, "❌ امتیاز کافی ندارید")
        return

    users[uid]["points"] = users[uid].get("points",0) - price
    users[uid].setdefault("bought_files", []).append(code)
    save_json(USERS_FILE, users)

    f = files[code]
    bot.send_document(uid, open(f["file_path"], "rb"), caption=f.get("caption",""))
    bot.answer_callback_query(call.id, "✅ خرید انجام شد — فایل ارسال شد")

# =======================
# پنل ادمین — همه پیام‌های ادمین اینجا مدیریت می‌شه
# =======================
@bot.message_handler(func=lambda m: is_admin(m.from_user.id))
def admin_panel_handler(message):
    uid = str(message.from_user.id)
    text = (message.text or "").strip()

    # همیشه اول منوی ادمین رو نشان بده (اگر پیام متن نیست هم نشان بده)
    if text == "" or text not in ["افزودن فایل", "افزودن امتیاز", "لیست فایل‌ها"]:
        send_admin_menu(uid)
        return

    # ---------- افزودن فایل (مرحله اول) ----------
    if text == "افزودن فایل":
        msg = bot.send_message(uid, "✅ لطفاً فایل را ارسال کنید (هر نوع فایلی).")
        bot.register_next_step_handler(msg, admin_receive_file)
        return

    # ---------- افزودن امتیاز ----------
    if text == "افزودن امتیاز":
        msg = bot.send_message(uid, "لطفاً ایدی عددی کاربر را وارد کنید:")
        bot.register_next_step_handler(msg, admin_receive_points_userid)
        return

    # ---------- لیست فایل‌ها ----------
    if text == "لیست فایل‌ها":
        if not files:
            bot.send_message(uid, "📂 هیچ فایلی ثبت نشده است.")
            return
        out = "📂 لیست فایل‌ها:\n\n"
        for code, d in files.items():
            out += f"🔸 کد: `{code}`\n📄 نام: {d['name']}\n💰 قیمت: {d['price']}\n———————\n"
        bot.send_message(uid, out, parse_mode="Markdown")
        return

# =======================
# افزودن فایل — مراحل
# =======================
def admin_receive_file(message):
    uid = str(message.from_user.id)
    # پشتیبانی فایل‌های مختلف: document, photo, video, audio, voice
    content_type = message.content_type
    file_name = None
    file_path_on_server = None

    try:
        if content_type == "document":
            file_info = bot.get_file(message.document.file_id)
            file_name = message.document.file_name
            file_bytes = bot.download_file(file_info.file_path)
        elif content_type == "photo":
            # انتخاب بزرگترین سایز عکس
            ph = message.photo[-1]
            file_info = bot.get_file(ph.file_id)
            file_name = f"photo_{ph.file_id}.jpg"
            file_bytes = bot.download_file(file_info.file_path)
        elif content_type == "video":
            file_info = bot.get_file(message.video.file_id)
            file_name = message.video.file_name or f"video_{message.video.file_id}.mp4"
            file_bytes = bot.download_file(file_info.file_path)
        elif content_type == "audio":
            file_info = bot.get_file(message.audio.file_id)
            file_name = message.audio.file_name or f"audio_{message.audio.file_id}.mp3"
            file_bytes = bot.download_file(file_info.file_path)
        else:
            bot.send_message(uid, "نوع فایل پشتیبانی نمی‌شود، لطفاً به صورت فایل ارسال کنید.")
            return
    except Exception as e:
        bot.send_message(uid, "خطا در دریافت فایل. دوباره سعی کنید.")
        return

    # ذخیره فایل در پوشه
    file_path_on_server = os.path.join(FILES_PATH, file_name)
    with open(file_path_on_server, "wb") as wf:
        wf.write(file_bytes)

    # ادامه مراحل: کپشن
    msg = bot.send_message(uid, "کپشن (توضیح) فایل را وارد کنید:")
    # ذخیره موقت در users تحت کلید 'admin_tmp' (مخصوص ادمین)
    users[uid].setdefault("admin_tmp", {})
    users[uid]["admin_tmp"]["file_name"] = file_name
    users[uid]["admin_tmp"]["file_path"] = file_path_on_server
    save_json(USERS_FILE, users)
    bot.register_next_step_handler(msg, admin_receive_caption)

def admin_receive_caption(message):
    uid = str(message.from_user.id)
    caption = message.text or ""
    users[uid]["admin_tmp"]["caption"] = caption
    save_json(USERS_FILE, users)
    msg = bot.send_message(uid, "مقدار امتیاز فایل را وارد کنید (عدد):")
    bot.register_next_step_handler(msg, admin_receive_price)

def admin_receive_price(message):
    uid = str(message.from_user.id)
    try:
        price = int(message.text.strip())
    except:
        bot.send_message(uid, "مقدار امتیاز نامعتبر است. عملیات کنسل شد.")
        users[uid].pop("admin_tmp", None)
        save_json(USERS_FILE, users)
        return
    users[uid]["admin_tmp"]["price"] = price
    save_json(USERS_FILE, users)
    msg = bot.send_message(uid, "کد یکتا برای فایل را وارد کنید (مثال: FILE123):")
    bot.register_next_step_handler(msg, admin_finalize_file)

def admin_finalize_file(message):
    uid = str(message.from_user.id)
    code = message.text.strip()
    tmp = users[uid].get("admin_tmp", {})
    if not tmp:
        bot.send_message(uid, "خطا: داده موقت پیدا نشد. عملیات کنسل شد.")
        return
    # ثبت نهایی در files
    files[code] = {
        "name": tmp["file_name"],
        "caption": tmp.get("caption",""),
        "price": tmp.get("price", 0),
        "file_path": tmp["file_path"]
    }
    save_json(FILES_FILE, files)
    users[uid].pop("admin_tmp", None)
    save_json(USERS_FILE, users)
    bot.send_message(uid, f"✅ فایل ثبت شد با کد `{code}`", parse_mode="Markdown")

# =======================
# افزودن امتیاز — مراحل
# =======================
def admin_receive_points_userid(message):
    uid = str(message.from_user.id)
    try:
        target = str(int(message.text.strip()))
    except:
        bot.send_message(uid, "ایدی نامعتبر. عملیات کنسل شد.")
        return
    msg = bot.send_message(uid, "مقدار امتیاز برای اضافه شدن را وارد کنید (عدد):")
    bot.register_next_step_handler(msg, lambda m, t=target: admin_finalize_add_points(m, t))

def admin_finalize_add_points(message, target_userid):
    uid = str(message.from_user.id)
    try:
        points = int(message.text.strip())
    except:
        bot.send_message(uid, "مقدار نامعتبر.")
        return
    # اگر کاربر وجود نداشت، ایجادش کن
    ensure_user(target_userid)
    users[target_userid]["points"] = users[target_userid].get("points",0) + points
    save_json(USERS_FILE, users)
    bot.send_message(uid, f"✅ {points} امتیاز به کاربر {target_userid} اضافه شد.")

# =======================
# پیام پیش‌فرض — فقط برای کاربران عادی
# =======================
@bot.message_handler(func=lambda m: True)
def fallback(message):
    uid = str(message.from_user.id)
    # ادمین را نادیده بگیر (یا منوی ادمین را نمایش بده)
    if is_admin(uid):
        send_admin_menu(uid)
        return

    # اگر عضو کانال نیست => یادآوری عضویت
    if not is_member_all(uid):
        markup = types.InlineKeyboardMarkup()
        for ch in CHANNELS:
            markup.add(types.InlineKeyboardButton(f"عضویت در {ch}", url=f"https://t.me/{ch[1:]}"))
        markup.add(types.InlineKeyboardButton("✅ بررسی عضویت", callback_data="check_membership"))
        bot.send_message(uid, "برای استفاده ابتدا عضو کانال(ها) شوید:", reply_markup=markup)
        return

    # در غیر این صورت منوی کاربری را نشان بده
    send_user_menu(uid)

# =======================
# اجرا
# =======================
if __name__ == "__main__":
    print("✅ Bot is running…")
    bot.infinity_polling()
