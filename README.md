#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import time
import sqlite3
import datetime
import pytz
from collections import defaultdict

from telegram import (
    Update,
    ChatPermissions,
)
from telegram.ext import (
    ApplicationBuilder,
    ContextTypes,
    MessageHandler,
    CallbackContext,
    filters,
)

# === إعداد المتغيرات الثابتة ===
TOKEN = '7267693922:AAHlYSosDZQe8KjU4wltrrd65ca4y5-m6UE'
ALLOWED_CHATS = [-1002566816917, -1009876543210]
ADMIN_GROUP_ID = -1002443915584
ADMIN_IDS = [7695570550, 7266144312]
VIP_IDS = []  # إضافة إيدي المستخدمين المميزين هنا

# ربط الأوامر العربية بالإنجليزية
CMD_MAP = {
    "حظر":         "ban",
    "الغاء حظر":   "unban",
    "كتم":         "mute",
    "الغاء كتم":   "unmute",
    "تقييد":       "restrict",
    "الغاء تقييد": "unrestrict",
    "تحذير":       "warn",
    "مسح تحذير":  "clear_warnings",
    "فتح الشات":   "open_chat",
    "اغلاق الشات": "close_chat",
    "احصائيات":    "stats",
    "لوحة":        "panel",
    "كشف":         "inspect",
}

# دوال المعالجات بالإنجليزية
async def cmd_ban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target = update.message.reply_to_message.from_user
    await context.bot.ban_chat_member(update.effective_chat.id, target.id)
    await update.message.reply_text(f"⛔ @{target.username or target.id} تم حظره.")
    await log_action(context, f"⛔ حظر @{target.username or target.id} ({target.id})")

async def cmd_unban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = context.args
    if args:
        target_id = int(args[0])
    else:
        target_id = update.message.reply_to_message.from_user.id
    await context.bot.unban_chat_member(update.effective_chat.id, target_id)
    await update.message.reply_text(f"✅ تم إلغاء الحظر عن `{target_id}`.")
    await log_action(context, f"✅ إلغاء حظر {target_id}")

async def cmd_mute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target = update.message.reply_to_message.from_user
    muted_users.add(target.id)
    await update.message.reply_text(f"🤐 @{target.username or target.id} تم كتمه.")
    await log_action(context, f"🤐 كتم @{target.username or target.id} ({target.id})")

async def cmd_unmute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target = update.message.reply_to_message.from_user
    muted_users.discard(target.id)
    await update.message.reply_text(f"✅ @{target.username or target.id} تم إلغاء الكتم.")
    await log_action(context, f"✅ إلغاء كتم @{target.username or target.id} ({target.id})")

async def cmd_restrict(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target = update.message.reply_to_message.from_user
    await context.bot.restrict_chat_member(
        chat_id=update.effective_chat.id,
        user_id=target.id,
        permissions=ChatPermissions(can_send_messages=False)
    )
    restricted_users[target.id] = None
    await update.message.reply_text(f"🔒 @{target.username or target.id} تم تقييده.")
    await log_action(context, f"🔒 تقييد @{target.username or target.id} ({target.id})")

async def cmd_unrestrict(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target = update.message.reply_to_message.from_user
    await context.bot.restrict_chat_member(
        chat_id=update.effective_chat.id,
        user_id=target.id,
        permissions=ChatPermissions(
            can_send_messages=True,
            can_send_media_messages=True,
            can_send_other_messages=True,
            can_add_web_page_previews=True
        )
    )
    restricted_users.pop(target.id, None)
    await update.message.reply_text(f"✅ @{target.username or target.id} تم إلغاء التقييد.")
    await log_action(context, f"✅ إلغاء تقييد @{target.username or target.id} ({target.id})")

async def cmd_warn(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await handle_warning_logic(update, context, update.message.reply_to_message.from_user.id)

async def cmd_clear_warnings(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target = update.message.reply_to_message.from_user
    cur.execute('UPDATE users SET warnings = 0 WHERE user_id = ?', (target.id,))
    conn.commit()
    await update.message.reply_text(f"🗑️ تم مسح تحذيرات @{target.username or target.id}.")
    await log_action(context, f"🗑️ مسح تحذيرات @{target.username or target.id} ({target.id})")

async def cmd_open_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await context.bot.set_chat_permissions(
        chat_id=update.effective_chat.id,
        permissions=ChatPermissions(
            can_send_messages=True,
            can_send_media_messages=True,
            can_send_other_messages=True,
            can_add_web_page_previews=True
        )
    )
    await update.message.reply_text("🔓 تم فتح الدردشة للجميع.")
    await log_action(context, f"🔓 فتح الدردشة في {update.effective_chat.id}")

async def cmd_close_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await context.bot.set_chat_permissions(
        chat_id=update.effective_chat.id,
        permissions=ChatPermissions(can_send_messages=False)
    )
    await update.message.reply_text("🔒 تم إغلاق الدردشة للجميع.")
    await log_action(context, f"🔒 إغلاق الدردشة في {update.effective_chat.id}")

async def cmd_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    cid = update.effective_chat.id
    count = await context.bot.get_chat_member_count(cid)
    chat = await context.bot.get_chat(cid)
    open_status = "مفتوحة" if chat.permissions.can_send_messages else "مغلقة"
    n_banned = len(banned_users)
    n_restricted = len(restricted_users)
    n_muted = len(muted_users)
    txt = (
        f"📊 إحصائيات المجموعة:\n"
        f"• عدد الأعضاء: {count}\n"
        f"• حالة الدردشة: {open_status}\n"
        f"• المحظورين: {n_banned}\n"
        f"• المقيدين: {n_restricted}\n"
        f"• المكتومين: {n_muted}\n"
    )
    await update.message.reply_text(txt)

async def cmd_panel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ADMIN_GROUP_ID:
        return
    text = "📋 لوحة التحكم:\n\n"
    def fmt(uids):
        lines = []
        for i, uid in enumerate(uids, 1):
            member = context.bot.get_chat_member(ADMIN_GROUP_ID, uid)
            name = member.user.full_name
            lines.append(f"{i}. {name} ({uid})")
        return "\n".join(lines) or "لا يوجد"
    text += "⛔ المحظورين:\n" + fmt(banned_users) + "\n\n"
    text += "🔒 المقيدين:\n" + fmt(restricted_users.keys()) + "\n\n"
    text += "🤐 المكتومين:\n" + fmt(muted_users) + "\n\n"
    for cid in ALLOWED_CHATS:
        ch = await context.bot.get_chat(cid)
        cnt = await context.bot.get_chat_member_count(cid)
        text += f"📍 '{ch.title}': {cnt} عضو\n"
    await update.message.reply_text(text)

async def cmd_inspect(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        target = update.message.reply_to_message.from_user
    else:
        txt = update.message.text.strip()
        parts = txt.split(maxsplit=1)
        target = await context.bot.get_chat_member(update.effective_chat.id, int(parts[1]))
        target = target.user
    now = time.time()
    ten_sec_ago = now - 10
    cur.execute('SELECT COUNT(*) FROM messages WHERE user_id = ? AND timestamp > ?', (target.id, ten_sec_ago))
    msgs_10 = cur.fetchone()[0]
    cur.execute('SELECT total_messages, warnings, join_date FROM users WHERE user_id = ?', (target.id,))
    row = cur.fetchone() or (0,0,'غير معروف')
    total_msgs, warns, jd = row
    name = target.full_name
    uname = target.username or ''
    text = (
        f"🕵️ كشف حساب {name}\n"
        f"👤 الاسم: {name}\n"
        f"🔖 اليوزر: {uname}\n"
        f"🆔 `{target.id}`\n"
        f"📅 انضم: {jd}\n"
        f"💬 رسائل خلال 10 ث: {msgs_10}\n"
        f"⚠️ تحذيرات: {warns}\n"
        f"📝 إجمالي الرسائل: {total_msgs}\n"
    )
    await update.message.reply_text(text)

# ربط المعالجات
HANDLER_MAP = {
    "ban":            cmd_ban,
    "unban":          cmd_unban,
    "mute":           cmd_mute,
    "unmute":         cmd_unmute,
    "restrict":       cmd_restrict,
    "unrestrict":     cmd_unrestrict,
    "warn":           cmd_warn,
    "clear_warnings": cmd_clear_warnings,
    "open_chat":      cmd_open_chat,
    "close_chat":     cmd_close_chat,
    "stats":          cmd_stats,
    "panel":          cmd_panel,
    "inspect":        cmd_inspect,
}

# === إعداد قاعدة البيانات ===
conn = sqlite3.connect('bot_data.db', check_same_thread=False)
cur = conn.cursor()
cur.execute('''
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    name TEXT,
    username TEXT,
    join_date TEXT,
    warnings INTEGER DEFAULT 0,
    total_messages INTEGER DEFAULT 0,
    last_message_ts REAL
)
''')
cur.execute('''
CREATE TABLE IF NOT EXISTS messages (
    user_id INTEGER,
    chat_id INTEGER,
    timestamp REAL
)
''')
conn.commit()

# === هياكل البيانات في الذاكرة ===
recent_msgs     = defaultdict(list)
muted_users     = set()
restricted_users= {}
banned_users    = set()

# === دوال مساعدة ===
def is_admin(user_id: int) -> bool:
    return user_id in ADMIN_IDS

def is_vip(user_id: int) -> bool:
    return user_id in VIP_IDS

async def log_action(context: ContextTypes.DEFAULT_TYPE, text: str):
    await context.bot.send_message(chat_id=ADMIN_GROUP_ID, text=text)

# === ترحيب الأعضاء الجدد + تسجيلهم ===
async def welcome_new(update: Update, context: ContextTypes.DEFAULT_TYPE):
    for member in update.message.new_chat_members:
        if member.is_bot:
            await context.bot.ban_chat_member(update.effective_chat.id, member.id)
            await log_action(context, f"🤖 طُرد بوت: {member.full_name} ({member.id})")
        else:
            name = member.full_name
            uname = member.username or ''
            jd = datetime.datetime.now(pytz.timezone('Asia/Damascus')).strftime('%Y-%m-%d')
            cur.execute('''
                INSERT OR IGNORE INTO users(user_id,name,username,join_date,last_message_ts)
                VALUES (?, ?, ?, ?, ?)
            ''', (member.id, name, uname, jd, time.time()))
            conn.commit()
            txt = (
                f"🎉 مرحباً {name}!\n"
                "📋 قوانين المجموعة:\n"
                "• لا سبام أكثر من 4 رسائل خلال 3 ثواني\n"
                "• ممنوع إرسال روابط\n"
                "• الالتزام بقائمة الكلمات الممنوعة\n"
                "• ممنوع إعادة التوجيه\n"
            )
            await context.bot.send_message(update.effective_chat.id, txt)

# === مراقبة الرسائل ===
async def monitor(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message
    uid = msg.from_user.id
    cid = update.effective_chat.id
    now = time.time()
    if cid not in ALLOWED_CHATS:
        return
    cur.execute('INSERT INTO messages(user_id,chat_id,timestamp) VALUES (?,?,?)', (uid, cid, now))
    cur.execute('UPDATE users SET total_messages = total_messages + 1, last_message_ts = ? WHERE user_id = ?', (now, uid))
    conn.commit()
    if is_vip(uid):
        return
    text = msg.text or ""
    # سبام
    lst = [t for t in recent_msgs[uid] if now - t < 3]
    lst.append(now)
    recent_msgs[uid] = lst
    if len(lst) > 4:
        await handle_warning_logic(update, context, uid)
        return
    # روابط
    if msg.entities:
        for ent in msg.entities:
            if ent.type in ("url", "text_link"):
                member = await context.bot.get_chat_member(cid, uid)
                if member.status not in ("administrator", "creator"):
                    await msg.delete()
                    await context.bot.send_message(cid, f"🚫 @{msg.from_user.username or uid} ممنوع إرسال روابط!")
                    await log_action(context, f"🚫 حذف رابط من @{msg.from_user.username or uid}")
                return
    # إعادة توجيه
    if msg.forward_date or msg.forward_from_chat:
        await msg.delete()
        await context.bot.send_message(cid, f"🔄 @{msg.from_user.username or uid} ممنوع إعادة التوجيه!")
        await handle_warning_logic(update, context, uid)
        return
    # كلمات ممنوعة
    FORBIDDEN = ['كلمة1', 'كلمة2', 'حرف_ممنوع']
    for w in FORBIDDEN:
        if w in text:
            await msg.delete()
            await context.bot.send_message(cid, f"⚠️ @{msg.from_user.username or uid} ممنوع استخدام هذه الكلمة!")
            await log_action(context, f"⚠️ كلمة ممنوعة من @{msg.from_user.username or uid}: {w}")
            await handle_warning_logic(update, context, uid)
            return
    # كتم
    if uid in muted_users:
        await msg.delete()
        return

# === منطق التحذيرات ===
async def handle_warning_logic(update: Update, context: ContextTypes.DEFAULT_TYPE, uid: int):
    cur.execute('UPDATE users SET warnings = warnings + 1 WHERE user_id = ?', (uid,))
    conn.commit()
    warns = cur.execute('SELECT warnings FROM users WHERE user_id = ?', (uid,)).fetchone()[0]
    cid = update.effective_chat.id
    uname = update.message.from_user.username or ''
    if warns == 1:
        await context.bot.send_message(cid, f"⚠️ @{uname} هذه هي المرة الأولى لتحذيرك.")
    elif warns == 2:
        until = datetime.datetime.utcnow() + datetime.timedelta(hours=1)
        await context.bot.restrict_chat_member(cid, uid, permissions=ChatPermissions(can_send_messages=False), until_date=until)
        restricted_users[uid] = until.timestamp()
        await context.bot.send_message(cid, f"🔒 @{uname} تم تقييدك لمدة ساعة.")
    elif warns == 3:
        until = datetime.datetime.utcnow() + datetime.timedelta(hours=5)
        await context.bot.restrict_chat_member(cid, uid, permissions=ChatPermissions(can_send_messages=False), until_date=until)
        restricted_users[uid] = until.timestamp()
        await context.bot.send_message(cid, f"🔒 @{uname} تم تقييدك لمدة 5 ساعات.")
    else:
        await context.bot.ban_chat_member(cid, uid)
        banned_users.add(uid)
        await context.bot.send_message(cid, f"⛔ @{uname} تم حظرك نهائياً.")
    await log_action(context, f"🚨 تحذير #{warns} لـ @{uname} ({uid}) في {cid}")

# === التعامل مع الأوامر العربية مباشرة ===
async def handle_text_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    if uid not in ADMIN_IDS:
        return
    text = update.message.text.strip()
    # البحث عن أطول مفتاح يطابق البداية
    for ar_cmd in sorted(CMD_MAP.keys(), key=len, reverse=True):
        if text.startswith(ar_cmd):
            eng = CMD_MAP[ar_cmd]
            handler = HANDLER_MAP.get(eng)
            # استخراج الوسيط إذا وُجد
            arg = text[len(ar_cmd):].strip()
            context.args = [arg] if arg else []
            await handler(update, context)
            return

# === وظائف جداول المهام الدورية مع ردود ذكية ===

async def daily_top3(context: ContextTypes.DEFAULT_TYPE):
    tz = pytz.timezone('Asia/Damascus')
    now = datetime.datetime.now(tz)
    start_ts = datetime.datetime(now.year, now.month, now.day, tzinfo=tz).timestamp()
    cur.execute(
        'SELECT user_id, COUNT(*) as cnt FROM messages '
        'WHERE timestamp > ? GROUP BY user_id '
        'ORDER BY cnt DESC LIMIT 3',
        (start_ts,)
    )
    top3 = cur.fetchall()
    
    txt = "🏆 تقرير اليوم الذهبي! أكثر 3 نجوم تفاعل:\n"
    for i, (uid, cnt) in enumerate(top3, 1):
        member = await context.bot.get_chat_member(ALLOWED_CHATS[0], uid)
        name = member.user.username or uid
        txt += f"{i}. @{name}: {cnt} رسالة 💬\n"
    txt += "\n🔥 استمروا على هذا الأداء المذهل!"
    
    await context.bot.send_message(ALLOWED_CHATS[0], txt)


async def check_inactivity(context: ContextTypes.DEFAULT_TYPE):
    now = time.time()
    for user_id, username, last_ts in cur.execute(
        'SELECT user_id, username, last_message_ts FROM users'
    ):
        if now - last_ts > 24 * 3600:
            mention = f"@{username}" if username else str(user_id)
            msg = (
                f"{mention} يبدو أنك دخلت طور الـ\"مشاهد فقط\"! 😏\n"
                "شاركنا ولو بكلمة واحدة، المجموعة تنتظرك 😉"
            )
            await context.bot.send_message(ALLOWED_CHATS[0], msg)

# === نقطة الدخول الرئيسية ===
def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, welcome_new))
    app.add_handler(MessageHandler(filters.ALL & ~filters.COMMAND, monitor))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text_cmd))

    jq = app.job_queue
    tz = pytz.timezone('Asia/Damascus')
    jq.run_daily(daily_top3, time=datetime.time(hour=11, minute=59, tzinfo=tz))
    jq.run_repeating(check_inactivity, interval=3600, first=60)

    app.run_polling()

if __name__ == '__main__':
    main()
