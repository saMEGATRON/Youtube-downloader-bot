import os
import time
import asyncio
from pyrogram import Client, filters, enums
from pyrogram.types import InlineKeyboardMarkup, InlineKeyboardButton, CallbackQuery
import yt_dlp

# 📌 Fix Python Event Loop
try:
    asyncio.get_event_loop()
except RuntimeError:
    asyncio.set_event_loop(asyncio.new_event_loop())

# 📌 Your Telegram Credentials
API_ID = 24593873
API_HASH = "2414f6b12bfe6f9435802683e49d2c38"
BOT_TOKEN = "8919936123:AAF2gycKUdXPJRFLoHhsyW8bsj9MMYn8eYg"

app = Client("fast_yt_bot", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

# Dictionary to store user links
user_links = {}

# 📊 Function to create Progress Bar
def get_progress_bar(percentage):
    filled = int(percentage // 10)
    blank = 10 - filled
    return "█" * filled + "▱" * blank

# 🛡️ Safe Message Edit (To avoid Telegram flood/limit errors)
async def edit_status_safe(client, chat_id, message_id, text):
    try:
        await client.edit_message_text(
            chat_id=chat_id,
            message_id=message_id,
            text=text,
            parse_mode=enums.ParseMode.HTML
        )
    except Exception:
        pass

# 1️⃣ Welcome Message (For /start command)
@app.on_message(filters.command("start"))
async def start_cmd(client: Client, message):
    welcome_text = (
        "<b>👋 Hello! Welcome to YouTube Downloader Bot!</b>\n\n"
        "<b>✨ Features:</b>\n"
        "• High-speed video downloads up to 2GB 🎬\n"
        "• 360p, 480p, 720p HD, 1080p Full HD 🎥\n"
        "• High-quality MP3 Audio Converter 🎵\n"
        "• Ultra-fast Multi-Thread Engine ⚡\n\n"
        "🚀 <b>How to use?</b>\n"
        "Just send me any YouTube video or Shorts link!"
    )
    await message.reply_text(welcome_text, parse_mode=enums.ParseMode.HTML)

# 2️⃣ YouTube Link Handling
@app.on_message(filters.text)
async def handle_youtube_link(client: Client, message):
    url = message.text.strip()
    
    if any(domain in url.lower() for domain in ["youtube.com", "youtu.be"]):
        user_links[message.chat.id] = url
        
        buttons = InlineKeyboardMarkup([
            [
                InlineKeyboardButton("🎬 360p", callback_data="dl_360"),
                InlineKeyboardButton("🎬 480p", callback_data="dl_480")
            ],
            [
                InlineKeyboardButton("🎬 720p HD", callback_data="dl_720"),
                InlineKeyboardButton("1080p Full HD 🎬", callback_data="dl_1080")
            ],
            [
                InlineKeyboardButton("🎵 MP3 Audio", callback_data="dl_mp3")
            ]
        ])
        
        await message.reply_text("<b>👇 Please select your desired quality:</b>", reply_markup=buttons, parse_mode=enums.ParseMode.HTML)

# 3️⃣ Download & Upload Process with Live Progress
@app.on_callback_query(filters.regex(r"^dl_"))
async def process_download(client: Client, callback: CallbackQuery):
    try:
        await callback.answer("⚡ Processing...", show_alert=False)
    except Exception:
        pass

    chat_id = callback.message.chat.id
    url = user_links.get(chat_id)
    
    if not url:
        await callback.message.edit_text("❌ Link not found. Please send it again.")
        return

    quality_type = callback.data.replace("dl_", "")
    status_msg = await callback.message.edit_text("<b>⏳ Preparing to download...</b>", parse_mode=enums.ParseMode.HTML)

    loop = asyncio.get_running_loop()
    last_update_time = [0]

    # 📥 Live Download Progress Hook
    def download_hook(d):
        if d['status'] == 'downloading':
            now = time.time()
            if now - last_update_time[0] >= 2.5:  # Update every 2.5 seconds
                last_update_time[0] = now
                total = d.get('total_bytes') or d.get('total_bytes_estimate') or 0
                downloaded = d.get('downloaded_bytes', 0)
                
                if total > 0:
                    percentage = (downloaded / total) * 100
                    bar = get_progress_bar(percentage)
                    speed = d.get('speed', 0) or 0
                    speed_mb = speed / (1024 * 1024) if speed else 0
                    
                    text = (
                        f"<b>⏳ Downloading...</b>\n\n"
                        f"<code>[{bar}]</code> <b>{percentage:.1f}%</b>\n"
                        f"⚡ <b>Speed:</b> {speed_mb:.2f} MB/s"
                    )
                    asyncio.run_coroutine_threadsafe(
                        edit_status_safe(client, chat_id, status_msg.id, text), loop
                    )

    ydl_opts = {
        'outtmpl': f'file_{chat_id}_%(title)s.%(ext)s',
        'concurrent_fragment_downloads': 10,
        'nocheckcertificate': True,
        'quiet': True,
        'progress_hooks': [download_hook],
        'max_filesize': 2000 * 1024 * 1024
    }

    if quality_type == "mp3":
        ydl_opts['format'] = 'bestaudio/best'
        ydl_opts['postprocessors'] = [{
            'key': 'FFmpegExtractAudio',
            'preferredcodec': 'mp3',
            'preferredquality': '192',
        }]
    else:
        res_val = int(quality_type)
        ydl_opts['format'] = f'bestvideo[height<={res_val}][ext=mp4]+bestaudio[ext=m4a]/best[height<={res_val}][ext=mp4]/best'

    def download_file():
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            filename = ydl.prepare_filename(info)
            if quality_type == "mp3":
                filename = os.path.splitext(filename)[0] + ".mp3"
            return filename, info.get('title', 'Media')

    file_path = None
    try:
        file_path, title = await asyncio.to_thread(download_file)

        if file_path and os.path.exists(file_path):
            last_upload_time = [0]

            # 📤 Live Upload Progress Callback
            async def upload_progress(current, total):
                now = time.time()
                if now - last_upload_time[0] >= 2.5 or current == total:
                    last_upload_time[0] = now
                    percentage = (current / total) * 100
                    bar = get_progress_bar(percentage)
                    
                    text = (
                        f"<b>🚀 Uploading to Telegram...</b>\n\n"
                        f"<code>[{bar}]</code> <b>{percentage:.1f}%</b>"
                    )
                    await edit_status_safe(client, chat_id, status_msg.id, text)

            # Notification caption advising users to save/forward the file
            caption_note = (
                f"\n\n⚠️ <i>Note: Please save or forward this file, as it is temporary "
                f"and will be cleared from the server shortly!</i>"
            )

            if quality_type == "mp3":
                await client.send_audio(
                    chat_id=chat_id, 
                    audio=file_path, 
                    caption=f"🎧 <b>{title}</b>{caption_note}", 
                    parse_mode=enums.ParseMode.HTML,
                    progress=upload_progress
                )
            else:
                await client.send_video(
                    chat_id=chat_id, 
                    video=file_path, 
                    caption=f"🎬 <b>{title}</b>{caption_note}", 
                    parse_mode=enums.ParseMode.HTML,
                    progress=upload_progress
                )

            await status_msg.delete()
        else:
            await status_msg.edit_text("❌ File could not be downloaded.")

    except Exception as e:
        await status_msg.edit_text(f"❌ <b>Error:</b> {str(e)}", parse_mode=enums.ParseMode.HTML)
    
    finally:
        # 🧹 Clean up file immediately to protect server storage
        if file_path and os.path.exists(file_path):
            try:
                os.remove(file_path)
            except Exception:
                pass
        
        # Clean up any leftover temporary files for this chat
        for item in os.listdir("."):
            if item.startswith(f"file_{chat_id}_"):
                try:
                    os.remove(item)
                except Exception:
                    pass

print("🚀 Real-Time Progress Downloader Bot Started Successfully!")
app.run()
