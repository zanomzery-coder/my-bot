import os
import re
import io
import sqlite3
import logging
import requests
import yt_dlp
import telebot
from telebot import types

BOT_TOKEN = os.environ["BOT_TOKEN"]
bot = telebot.TeleBot(BOT_TOKEN)

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)

DB_PATH = "history.db"
HISTORY_LIMIT = 10
DAILY_LIMIT = 10
MAX_SIZE_MB = 49

LIMIT_MESSAGE = (
    "مخابن! لیمیتا تە یا پاقژ و بێبەرامبەر بۆ ئەڤڕۆ (١٠ ڤیدیۆ) ب دوماهی هات. ⚠️\n"
    "بۆ ئەکتیڤکرنا ئەکاونتێ VIP و داگرتنا ب بێ لێمیت، تەنێ مانگانە ٣,٠٠٠ دیناران ب فاست پەی (FastPay) یان کۆڕەک/ئاسیا بفرێسه بۆ ژمارەیا مە و وەسڵێ بنێره بۆ خودانێ بۆتی: @z_14x"
)

# Telegram user_id of the bot admin (the only one who can add/remove VIPs)
# Set to your own Telegram user_id. Find it by messaging @userinfobot.
ADMIN_ID: int = 6241334692

BROWSER_UA = (
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
    "AppleWebKit/537.36 (KHTML, like Gecko) "
    "Chrome/125.0.0.0 Safari/537.36"
)
BROWSER_HEADERS = {"User-Agent": BROWSER_UA, "Accept-Language": "en-US,en;q=0.9"}

SUPPORTED_DOMAINS = (
    "tiktok.com", "vm.tiktok.com", "vt.tiktok.com",
    "instagram.com", "instagr.am",
    "youtube.com", "youtu.be", "www.youtube.com",
)
YOUTUBE_DOMAINS = ("youtube.com", "youtu.be", "www.youtube.com")
TIKTOK_DOMAINS  = ("tiktok.com", "vm.tiktok.com", "vt.tiktok.com")


# ── Database ──────────────────────────────────────────────────────────────────

def db_connect() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def db_init():
    with db_connect() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS history (
                id         INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id    INTEGER NOT NULL,
                url        TEXT NOT NULL,
                platform   TEXT NOT NULL,
                title      TEXT,
                media_type TEXT NOT NULL DEFAULT 'video',
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)
        conn.execute("CREATE INDEX IF NOT EXISTS idx_user ON history(user_id, created_at DESC)")
        conn.execute("""
            CREATE TABLE IF NOT EXISTS vip_users (
                user_id    INTEGER PRIMARY KEY,
                note       TEXT,
                added_at   DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS known_users (
                user_id  INTEGER PRIMARY KEY,
                username TEXT COLLATE NOCASE,
                name     TEXT,
                seen_at  DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)

def db_add(user_id: int, url: str, platform: str, title: str, media_type: str):
    with db_connect() as conn:
        conn.execute(
            "INSERT INTO history (user_id, url, platform, title, media_type) VALUES (?,?,?,?,?)",
            (user_id, url, platform, title or "", media_type),
        )

def db_get_history(user_id: int) -> list:
    with db_connect() as conn:
        return conn.execute(
            "SELECT * FROM history WHERE user_id=? ORDER BY created_at DESC LIMIT ?",
            (user_id, HISTORY_LIMIT),
        ).fetchall()

def db_clear(user_id: int):
    with db_connect() as conn:
        conn.execute("DELETE FROM history WHERE user_id=?", (user_id,))

def db_count_today(user_id: int) -> int:
    with db_connect() as conn:
        row = conn.execute(
            "SELECT COUNT(*) FROM history WHERE user_id=? AND date(created_at)=date('now')",
            (user_id,),
        ).fetchone()
        return row[0] if row else 0

def db_is_vip(user_id: int) -> bool:
    with db_connect() as conn:
        row = conn.execute(
            "SELECT 1 FROM vip_users WHERE user_id=?", (user_id,)
        ).fetchone()
        return row is not None

def db_add_vip(user_id: int, note: str = ""):
    with db_connect() as conn:
        conn.execute(
            "INSERT OR REPLACE INTO vip_users (user_id, note) VALUES (?,?)",
            (user_id, note),
        )

def db_remove_vip(user_id: int) -> bool:
    with db_connect() as conn:
        cur = conn.execute("DELETE FROM vip_users WHERE user_id=?", (user_id,))
        return cur.rowcount > 0

def db_list_vip() -> list:
    with db_connect() as conn:
        return conn.execute(
            "SELECT user_id, note, added_at FROM vip_users ORDER BY added_at DESC"
        ).fetchall()

def db_track_user(user):
    """Record username→user_id mapping from any Telegram User object."""
    username = (user.username or "").lower().lstrip("@") or None
    name = " ".join(filter(None, [user.first_name, user.last_name])) or None
    with db_connect() as conn:
        conn.execute(
            """INSERT INTO known_users (user_id, username, name, seen_at)
               VALUES (?,?,?,CURRENT_TIMESTAMP)
               ON CONFLICT(user_id) DO UPDATE SET
                   username=excluded.username,
                   name=excluded.name,
                   seen_at=excluded.seen_at""",
            (user.id, username, name),
        )

def db_resolve_username(username: str) -> int | None:
    """Return user_id for a @username if we've seen them, else None."""
    clean = username.lower().lstrip("@")
    with db_connect() as conn:
        row = conn.execute(
            "SELECT user_id FROM known_users WHERE username=?", (clean,)
        ).fetchone()
        return row["user_id"] if row else None

def db_get_stats() -> dict:
    with db_connect() as conn:
        total_downloads = conn.execute("SELECT COUNT(*) FROM history").fetchone()[0]
        total_users     = conn.execute("SELECT COUNT(DISTINCT user_id) FROM history").fetchone()[0]
        total_known     = conn.execute("SELECT COUNT(*) FROM known_users").fetchone()[0]
        total_vip       = conn.execute("SELECT COUNT(*) FROM vip_users").fetchone()[0]
        today_downloads = conn.execute(
            "SELECT COUNT(*) FROM history WHERE date(created_at)=date('now')"
        ).fetchone()[0]
        today_users = conn.execute(
            "SELECT COUNT(DISTINCT user_id) FROM history WHERE date(created_at)=date('now')"
        ).fetchone()[0]
        hit_limit_today = conn.execute(
            """SELECT COUNT(*) FROM (
                SELECT user_id FROM history
                WHERE date(created_at)=date('now')
                GROUP BY user_id HAVING COUNT(*) >= ?
            )""", (DAILY_LIMIT,)
        ).fetchone()[0]
        top_users = conn.execute(
            """SELECT h.user_id,
                      COALESCE(k.username, '') AS username,
                      COALESCE(k.name, '')     AS name,
                      COUNT(*)                 AS total,
                      SUM(CASE WHEN date(h.created_at)=date('now') THEN 1 ELSE 0 END) AS today,
                      COALESCE(v.user_id IS NOT NULL, 0) AS is_vip
               FROM history h
               LEFT JOIN known_users k ON k.user_id = h.user_id
               LEFT JOIN vip_users   v ON v.user_id = h.user_id
               GROUP BY h.user_id
               ORDER BY total DESC
               LIMIT 10""",
        ).fetchall()
        platform_counts = conn.execute(
            "SELECT platform, COUNT(*) AS cnt FROM history GROUP BY platform ORDER BY cnt DESC"
        ).fetchall()
        type_counts = conn.execute(
            "SELECT media_type, COUNT(*) AS cnt FROM history GROUP BY media_type"
        ).fetchall()
    return {
        "total_downloads": total_downloads,
        "total_users": total_users,
        "total_known": total_known,
        "total_vip": total_vip,
        "today_downloads": today_downloads,
        "today_users": today_users,
        "hit_limit_today": hit_limit_today,
        "top_users": top_users,
        "platform_counts": platform_counts,
        "type_counts": type_counts,
    }


# ── URL helpers ───────────────────────────────────────────────────────────────

def is_supported(url: str) -> bool:
    return any(d in url for d in SUPPORTED_DOMAINS)

def is_youtube(url: str) -> bool:
    return any(d in url for d in YOUTUBE_DOMAINS)

def is_tiktok(url: str) -> bool:
    return any(d in url for d in TIKTOK_DOMAINS)

def detect_platform(url: str) -> str:
    if is_tiktok(url):   return "TikTok"
    if "instagram" in url: return "Instagram"
    if is_youtube(url):  return "YouTube"
    return "Unknown"

def extract_url(text: str) -> str | None:
    for u in re.findall(r"https?://[^\s]+", text):
        if is_supported(u):
            return u
    return None


# ── Stream URL into memory buffer ─────────────────────────────────────────────

def stream_to_buffer(url: str, extra_headers: dict | None = None) -> io.BytesIO:
    """Download a URL into an in-memory BytesIO buffer (no disk I/O)."""
    headers = {**BROWSER_HEADERS, **(extra_headers or {})}
    resp = requests.get(url, headers=headers, stream=True, timeout=60)
    resp.raise_for_status()

    buf = io.BytesIO()
    size = 0
    for chunk in resp.iter_content(chunk_size=512 * 1024):
        buf.write(chunk)
        size += len(chunk)
        if size > MAX_SIZE_MB * 1024 * 1024:
            raise ValueError(f"File exceeds {MAX_SIZE_MB} MB limit")
    buf.seek(0)
    return buf


# ── TikTok: tikwm.com free public API ────────────────────────────────────────

def tiktok_get_url(url: str) -> tuple[str, str]:
    resp = requests.post(
        "https://www.tikwm.com/api/",
        data={"url": url, "hd": "1"},
        headers=BROWSER_HEADERS,
        timeout=20,
    )
    resp.raise_for_status()
    body = resp.json()
    if body.get("code") != 0:
        raise RuntimeError(f"tikwm error: {body.get('msg', 'unknown')}")
    d = body.get("data") or {}
    video_url = d.get("hdplay") or d.get("play") or ""
    if not video_url:
        raise RuntimeError("tikwm returned no video URL")
    return video_url, d.get("title", "")


# ── yt-dlp: extract direct stream URL (no disk download) ─────────────────────

YDL_COMMON = {
    "quiet": True,
    "no_warnings": True,
    "nocheckcertificate": True,
    "http_headers": BROWSER_HEADERS,
}

YDL_YT_EXTRACTOR_ARGS = {
    "youtube": {
        "player_client": ["tv_embedded", "android", "web"],
        "player_skip": ["webpage", "configs"],
    }
}

def _pick_best_format(info: dict, prefer_height: int = 720) -> tuple[str, dict]:
    """
    Pick the best single-file (non-DASH) format from yt-dlp info.
    Returns (direct_url, format_dict).
    """
    # Unwrap playlist/search results
    if "entries" in info:
        info = next(iter(info["entries"]), info)

    # If info itself has a direct url (already resolved single format)
    if info.get("url") and info.get("ext") not in ("m3u8", "mpd"):
        return info["url"], info

    formats = info.get("formats") or []

    # Prefer combined audio+video formats (not DASH fragments)
    combined = [
        f for f in formats
        if f.get("url")
        and f.get("vcodec", "none") not in ("none", None, "")
        and f.get("acodec", "none") not in ("none", None, "")
        and f.get("ext") not in ("m3u8", "mpd", "mhtml")
        and "manifest" not in (f.get("url") or "").lower()
    ]

    if combined:
        # Sort by height then tbr (total bitrate)
        def sort_key(f: dict) -> tuple:
            h = f.get("height") or 0
            tbr = f.get("tbr") or 0
            return (min(h, prefer_height), tbr)
        best = max(combined, key=sort_key)
        return best["url"], best

    # Fallback: any format with a URL
    for f in reversed(formats):
        if f.get("url"):
            return f["url"], f

    raise RuntimeError("No usable direct video URL found in yt-dlp output")

def ydl_extract_video_url(url: str) -> tuple[str, str, dict]:
    """Returns (direct_video_url, title, format_dict) without downloading."""
    opts = {
        **YDL_COMMON,
        "format": "bestvideo[height<=720][ext=mp4]+bestaudio[ext=m4a]/best[height<=720][ext=mp4]/best[height<=720]/best",
    }
    if is_youtube(url):
        opts["extractor_args"] = YDL_YT_EXTRACTOR_ARGS

    with yt_dlp.YoutubeDL(opts) as ydl:
        info = yt_dlp.YoutubeDL({"quiet": True}).sanitize_info(
            ydl.extract_info(url, download=False)
        )

    title = info.get("title", "")
    direct_url, fmt = _pick_best_format(info)
    return direct_url, title, fmt

def ydl_download_audio_buffer(url: str) -> tuple[io.BytesIO, str]:
    """Download and convert audio to MP3 in a temp dir, return as BytesIO."""
    import tempfile
    with tempfile.TemporaryDirectory() as tmpdir:
        out_tmpl = os.path.join(tmpdir, "audio.%(ext)s")
        opts = {
            **YDL_COMMON,
            "outtmpl": out_tmpl,
            "format": "bestaudio/best",
            "postprocessors": [{
                "key": "FFmpegExtractAudio",
                "preferredcodec": "mp3",
                "preferredquality": "192",
            }],
        }
        if is_youtube(url):
            opts["extractor_args"] = YDL_YT_EXTRACTOR_ARGS

        with yt_dlp.YoutubeDL(opts) as ydl:
            info = ydl.extract_info(url, download=True)
        title = info.get("title", "")

        mp3 = next((f for f in os.listdir(tmpdir) if f.endswith(".mp3")), None)
        if not mp3:
            raise RuntimeError("MP3 conversion failed — no output file found")

        with open(os.path.join(tmpdir, mp3), "rb") as f:
            buf = io.BytesIO(f.read())
        buf.seek(0)
        return buf, title


# ── Core send logic ───────────────────────────────────────────────────────────

def _edit(chat_id: int, msg_id: int, text: str):
    try:
        bot.edit_message_text(text, chat_id=chat_id, message_id=msg_id)
    except Exception:
        pass

def _classify_error(err: str) -> str:
    err_l = err.lower()
    if "private" in err_l or "login" in err_l or "sign in" in err_l:
        return "❌ This content is private or requires a login."
    if "not available" in err_l or "unavailable" in err_l:
        return "❌ This content is not available in your region."
    if "exceed" in err_l or "too large" in err_l or "mb" in err_l:
        return f"❌ File is larger than {MAX_SIZE_MB} MB — Telegram can't receive it."
    return "❌ Download failed. The content may be private, deleted, or geo-restricted."

def _process(url: str, audio_only: bool, user_id: int, chat_id: int, status_id: int):
    platform = detect_platform(url)

    if not db_is_vip(user_id) and db_count_today(user_id) >= DAILY_LIMIT:
        _edit(chat_id, status_id, LIMIT_MESSAGE)
        return

    try:
        # ── Audio (YouTube / Instagram) ──────────────────────────────────────
        if audio_only:
            _edit(chat_id, status_id, "⏳ Extracting audio (this may take a moment)...")
            buf, title = ydl_download_audio_buffer(url)
            buf.name = "audio.mp3"
            _edit(chat_id, status_id, "✅ Sending audio...")
            bot.send_audio(
                chat_id, buf,
                title=title[:64] if title else None,
                caption=f"🎵 {title[:200]}" if title else None,
            )
            bot.delete_message(chat_id, status_id)
            db_add(user_id, url, platform, title, "audio")
            return

        # ── TikTok: use tikwm.com direct CDN URL ────────────────────────────
        if is_tiktok(url):
            _edit(chat_id, status_id, "⏳ Fetching TikTok video...")
            video_url, title = tiktok_get_url(url)
            _edit(chat_id, status_id, "✅ Sending video...")
            bot.send_video(
                chat_id, video_url,
                caption=f"📹 {title[:200]}" if title else None,
                supports_streaming=True,
            )
            bot.delete_message(chat_id, status_id)
            db_add(user_id, url, platform, title, "video")
            return

        # ── Instagram / YouTube: yt-dlp URL extraction ───────────────────────
        _edit(chat_id, status_id, f"⏳ Resolving {platform} video URL...")
        direct_url, title, fmt = ydl_extract_video_url(url)

        # For YouTube, CDN URLs are IP-signed → stream into memory buffer
        # and send as a file so Telegram receives the actual bytes
        if is_youtube(url):
            _edit(chat_id, status_id, "⏳ Streaming video into memory...")
            extra_hdrs = {k: v for k, v in (fmt.get("http_headers") or {}).items()}
            buf = stream_to_buffer(direct_url, extra_headers=extra_hdrs)
            buf.name = "video.mp4"
            _edit(chat_id, status_id, "✅ Sending video...")
            bot.send_video(
                chat_id, buf,
                caption=f"📹 {title[:200]}" if title else None,
                supports_streaming=True,
            )
        else:
            # Instagram CDN URLs are open — Telegram can fetch directly
            _edit(chat_id, status_id, "✅ Sending video...")
            bot.send_video(
                chat_id, direct_url,
                caption=f"📹 {title[:200]}" if title else None,
                supports_streaming=True,
            )

        bot.delete_message(chat_id, status_id)
        db_add(user_id, url, platform, title, "video")

    except Exception as e:
        logger.error("[%s] %s — %s", platform, url, e, exc_info=True)
        _edit(chat_id, status_id, _classify_error(str(e)))


# ── Keyboards ─────────────────────────────────────────────────────────────────

def yt_keyboard(url: str) -> types.InlineKeyboardMarkup:
    kb = types.InlineKeyboardMarkup()
    kb.row(
        types.InlineKeyboardButton("🎬 Video (MP4)", callback_data=f"video|{url}"),
        types.InlineKeyboardButton("🎵 Audio (MP3)", callback_data=f"audio|{url}"),
    )
    return kb

def redownload_keyboard(url: str, media_type: str) -> types.InlineKeyboardMarkup:
    kb = types.InlineKeyboardMarkup()
    if is_youtube(url):
        kb.row(
            types.InlineKeyboardButton("🎬 Video", callback_data=f"video|{url}"),
            types.InlineKeyboardButton("🎵 Audio", callback_data=f"audio|{url}"),
        )
    elif media_type == "audio":
        kb.row(types.InlineKeyboardButton("🎵 Re-download MP3", callback_data=f"audio|{url}"))
    else:
        kb.row(types.InlineKeyboardButton("🎬 Re-download", callback_data=f"video|{url}"))
    return kb


# ── Handlers ──────────────────────────────────────────────────────────────────

@bot.message_handler(commands=["start", "help"])
def handle_start(message):
    db_track_user(message.from_user)
    bot.reply_to(
        message,
        "👋 *Video Downloader Bot*\n\n"
        "Send me a link from:\n"
        "• TikTok\n"
        "• Instagram (Reels, posts)\n"
        "• YouTube (videos, Shorts)\n\n"
        "For YouTube I'll ask: *video* or *audio (MP3)*.\n\n"
        "📋 Commands:\n"
        "/history — your last 10 downloads\n"
        "/clearhistory — delete your history\n\n"
        "Videos play directly inside Telegram — no redirects! 🎬🎵",
        parse_mode="Markdown",
    )

@bot.message_handler(commands=["history"])
def handle_history(message):
    db_track_user(message.from_user)
    rows = db_get_history(message.from_user.id)
    if not rows:
        bot.reply_to(message, "📋 Your download history is empty.")
        return
    bot.reply_to(message, f"📋 *Your last {len(rows)} downloads:*", parse_mode="Markdown")
    for i, row in enumerate(rows, 1):
        icon = "🎵" if row["media_type"] == "audio" else "🎬"
        caption = (
            f"{i}. {icon} [{row['platform']}] "
            f"{(row['title'] or 'Untitled')[:60]}\n"
            f"🗓 {row['created_at'][:10]}"
        )
        bot.send_message(message.chat.id, caption,
                         reply_markup=redownload_keyboard(row["url"], row["media_type"]))

@bot.message_handler(commands=["clearhistory"])
def handle_clear(message):
    db_track_user(message.from_user)
    db_clear(message.from_user.id)
    bot.reply_to(message, "🗑 Your download history has been cleared.")


# ── Admin: VIP management ─────────────────────────────────────────────────────

def _is_admin(message) -> bool:
    if ADMIN_ID is None:
        bot.reply_to(message, "⚠️ ADMIN_ID is not set in main.py. Edit the file and set your Telegram user_id.")
        return False
    if message.from_user.id != ADMIN_ID:
        bot.reply_to(message, "❌ You are not authorised to use this command.")
        return False
    return True

def _resolve_target(arg: str) -> tuple[int | None, str]:
    """
    Resolve a /addvip or /removevip argument to (user_id, display).
    Accepts:  123456789   or   @username
    Returns (user_id, display_string) or (None, error_message).
    """
    if arg.lstrip("-").isdigit():
        return int(arg), arg
    if arg.startswith("@"):
        uid = db_resolve_username(arg)
        if uid:
            return uid, f"{arg} (`{uid}`)"
        return None, (
            f"⚠️ Username {arg} not found in cache.\n"
            "The user must send at least one message to the bot first so their ID can be recorded."
        )
    return None, f"⚠️ Invalid argument `{arg}`. Use a numeric user_id or @username."

@bot.message_handler(commands=["addvip"])
def handle_addvip(message):
    if not _is_admin(message):
        return
    parts = message.text.strip().split(maxsplit=2)
    if len(parts) < 2:
        bot.reply_to(message, "Usage: /addvip <user_id or @username> [note]")
        return
    uid, display = _resolve_target(parts[1])
    if uid is None:
        bot.reply_to(message, display, parse_mode="Markdown")
        return
    note = parts[2] if len(parts) == 3 else ""
    db_add_vip(uid, note)
    bot.reply_to(
        message,
        f"✅ {display} added to VIP list." + (f"\nNote: {note}" if note else ""),
        parse_mode="Markdown",
    )

@bot.message_handler(commands=["removevip"])
def handle_removevip(message):
    if not _is_admin(message):
        return
    parts = message.text.strip().split()
    if len(parts) < 2:
        bot.reply_to(message, "Usage: /removevip <user_id or @username>")
        return
    uid, display = _resolve_target(parts[1])
    if uid is None:
        bot.reply_to(message, display, parse_mode="Markdown")
        return
    removed = db_remove_vip(uid)
    if removed:
        bot.reply_to(message, f"✅ {display} removed from VIP list.", parse_mode="Markdown")
    else:
        bot.reply_to(message, f"⚠️ {display} was not in the VIP list.", parse_mode="Markdown")

@bot.message_handler(commands=["viplist"])
def handle_viplist(message):
    if not _is_admin(message):
        return
    rows = db_list_vip()
    if not rows:
        bot.reply_to(message, "📋 No VIP users yet.")
        return
    lines = ["👑 *VIP Users:*\n"]
    for row in rows:
        note = f" — {row['note']}" if row["note"] else ""
        lines.append(f"• `{row['user_id']}`{note} _(added {row['added_at'][:10]})_")
    bot.reply_to(message, "\n".join(lines), parse_mode="Markdown")

@bot.message_handler(commands=["stats"])
def handle_stats(message):
    if not _is_admin(message):
        return
    s = db_get_stats()

    # ── Platform breakdown ───────────────────────────────────────────────────
    platforms = " | ".join(
        f"{r['platform']} {r['cnt']}" for r in s["platform_counts"]
    ) or "—"
    types = " | ".join(
        f"{'🎵' if r['media_type']=='audio' else '🎬'} {r['media_type']} {r['cnt']}"
        for r in s["type_counts"]
    ) or "—"

    # ── Top users table ──────────────────────────────────────────────────────
    top_lines = []
    for i, u in enumerate(s["top_users"], 1):
        handle = f"@{u['username']}" if u["username"] else f"`{u['user_id']}`"
        vip_tag = " 👑" if u["is_vip"] else ""
        top_lines.append(
            f"{i}. {handle}{vip_tag} — {u['total']} total, {u['today']} today"
        )
    top_block = "\n".join(top_lines) or "No downloads yet."

    text = (
        "📊 *Bot Statistics*\n\n"
        f"📅 *Today*\n"
        f"  Downloads: {s['today_downloads']}\n"
        f"  Active users: {s['today_users']}\n"
        f"  Hit daily limit: {s['hit_limit_today']}\n\n"
        f"📦 *All time*\n"
        f"  Downloads: {s['total_downloads']}\n"
        f"  Unique downloaders: {s['total_users']}\n"
        f"  Known users (ever messaged): {s['total_known']}\n"
        f"  VIP users: {s['total_vip']}\n\n"
        f"🎯 *By platform:* {platforms}\n"
        f"🎞 *By type:* {types}\n\n"
        f"🏆 *Top 10 users:*\n{top_block}"
    )
    bot.reply_to(message, text, parse_mode="Markdown")

@bot.message_handler(commands=["broadcast"])
def handle_broadcast(message):
    if not _is_admin(message):
        return

    # Everything after "/broadcast " is the message body
    parts = message.text.split(maxsplit=1)
    if len(parts) < 2 or not parts[1].strip():
        bot.reply_to(
            message,
            "Usage: /broadcast <message text>\n\n"
            "Sends your text to every user who has ever messaged the bot.\n"
            "Supports Markdown formatting.",
        )
        return

    broadcast_text = parts[1].strip()

    with db_connect() as conn:
        rows = conn.execute("SELECT user_id FROM known_users").fetchall()
    user_ids = [r["user_id"] for r in rows]

    if not user_ids:
        bot.reply_to(message, "📭 No known users to broadcast to yet.")
        return

    progress = bot.reply_to(message, f"📡 Broadcasting to {len(user_ids)} users...")

    sent = blocked = failed = 0
    for uid in user_ids:
        try:
            bot.send_message(uid, broadcast_text, parse_mode="Markdown")
            sent += 1
        except Exception as e:
            err = str(e).lower()
            if "blocked" in err or "deactivated" in err or "not found" in err or "forbidden" in err:
                blocked += 1
            else:
                failed += 1
        # Respect Telegram's 30 messages/sec limit
        import time
        time.sleep(0.05)

    summary = (
        f"✅ *Broadcast complete*\n\n"
        f"📨 Sent: {sent}\n"
        f"🚫 Blocked/deleted: {blocked}\n"
        f"❌ Other errors: {failed}\n"
        f"👥 Total attempted: {len(user_ids)}"
    )
    bot.edit_message_text(summary, chat_id=message.chat.id,
                          message_id=progress.message_id, parse_mode="Markdown")


@bot.message_handler(func=lambda m: True, content_types=["text"])
def handle_message(message):
    db_track_user(message.from_user)
    url = extract_url(message.text.strip())
    if not url:
        bot.reply_to(message, "Please send a valid video URL from TikTok, Instagram, or YouTube.")
        return
    if is_youtube(url):
        bot.reply_to(message, "🎬 YouTube link detected! What would you like?",
                     reply_markup=yt_keyboard(url))
        return
    platform = detect_platform(url)
    status = bot.reply_to(message, f"⏳ Fetching from {platform}...")
    _process(url, audio_only=False, user_id=message.from_user.id,
             chat_id=message.chat.id, status_id=status.message_id)

@bot.callback_query_handler(func=lambda c: c.data.startswith("video|") or c.data.startswith("audio|"))
def handle_callback(call: types.CallbackQuery):
    db_track_user(call.from_user)
    bot.answer_callback_query(call.id)
    action, url = call.data.split("|", 1)
    try:
        bot.edit_message_reply_markup(call.message.chat.id, call.message.message_id, reply_markup=None)
    except Exception:
        pass
    label = "Fetching audio..." if action == "audio" else "Fetching video..."
    status = bot.send_message(call.message.chat.id, f"⏳ {label}")
    _process(url, audio_only=(action == "audio"), user_id=call.from_user.id,
             chat_id=call.message.chat.id, status_id=status.message_id)


# ── Entry point ───────────────────────────────────────────────────────────────

if __name__ == "__main__":
    db_init()
    bot.delete_webhook(drop_pending_updates=True)
    logger.info("Bot started. Polling...")
    bot.infinity_polling(timeout=20, long_polling_timeout=10)
