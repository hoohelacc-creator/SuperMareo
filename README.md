import json
import os
import subprocess
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes

TOKEN = '8622316758:AAGrxJFd8Ut03ojSCmSB8zGzzevHO2mIrgI'
ADMIN_ID = 8304323630
DB_FILE = "store_data.json"

# تحميل البيانات
if os.path.exists(DB_FILE):
    with open(DB_FILE, "r") as f: data = json.load(f)
else:
    data = {"users": {}, "files": []}

def save_db():
    with open(DB_FILE, "w") as f: json.dump(data, f)

# نظام الأمان: فحص الملفات
def is_safe(filename):
    # نمنع الملفات التنفيذية أو التي تنتهي بـ sh, bat, exe
    forbidden = ['.exe', '.sh', '.bat', '.pyc']
    return not any(filename.endswith(ext) for ext in forbidden)

# أمر رفع الملف
async def upload_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    doc = update.message.document
    if not is_safe(doc.file_name):
        await update.message.reply_text("❌ الملف غير قانوني أو غير آمن!")
        return
    
    file = await doc.get_file()
    path = f"uploads/{doc.file_name}"
    await file.download_to_drive(path)
    
    data["files"].append({"name": doc.file_name, "path": path, "user": update.effective_user.id})
    save_db()
    await update.message.reply_text(f"✅ تم رفع {doc.file_name} بأمان.")

# أمر تشغيل ملف بايثون (للأدمن فقط)
async def run_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    filename = context.args[0]
    path = f"uploads/{filename}"
    
    if os.path.exists(path):
        # تشغيل الملف
        result = subprocess.run(['python3', path], capture_output=True, text=True)
        await update.message.reply_text(f"📊 نتيجة التشغيل:\n{result.stdout}\n{result.stderr}")
    else:
        await update.message.reply_text("❌ الملف غير موجود.")

# أمر عرض الملفات (للأدمن فقط)
async def list_files(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    msg = "📂 قائمة الملفات المرفوعة:\n" + "\n".join([f["name"] for f in data["files"]])
    await update.message.reply_text(msg)

app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler("run", run_file))
app.add_handler(CommandHandler("list", list_files))
app.add_handler(MessageHandler(filters.Document.ALL, upload_file))

print("البوت يعمل...")
app.run_polling()
