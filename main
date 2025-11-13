from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, MessageHandler, filters, ContextTypes
from telegram.error import TelegramError, Forbidden, BadRequest
from flask import Flask
from threading import Thread
import asyncio
import os
import logging

# 1. Configure logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# 2. Retrieve the BOT TOKEN from the environment variable (Must be named TELEGRAM_BOT_TOKEN on Render)
TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")

# 3. Setup Flask server (Keep-Alive mechanism for Render Free Tier)
app = Flask(__name__)

@app.route('/')
def home():
    # Render uses this endpoint for health checks.
    return "Telegram Bot is running (Keep-Alive enabled)!"

def run_flask_server():
    # Render provides the PORT environment variable; retrieve it
    port = int(os.environ.get("PORT", 10000))
    # Flask must run on host 0.0.0.0 to be accessible externally
    app.run(host='0.0.0.0', port=port)
    logger.info(f"Flask server running on port {port}")

def start_keep_alive_thread():
    # Run the Flask server in a separate thread
    t = Thread(target=run_flask_server)
    t.start()
    logger.info(f"Flask keep-alive server started on thread {t.name}.")

# --- Telegram Bot Handlers ---

async def welcome_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # Ensure this handler runs only in group chats when new members join
    if not update.message or not update.message.new_chat_members:
        logger.warning("Received update without message or new_chat_members")
        return
    
    for member in update.message.new_chat_members:
        # Ignore bot joining message itself
        if member.is_bot and member.username == context.bot.username:
            continue

        try:
            # Generate HTML mention and welcome message
            mention = f'<a href="tg://user?id={member.id}">{member.first_name}</a>'
            msg = (
                f" আসসালামু আলাইকুম \n\n"
                f"{mention}   আপনাকে আমাদের গ্রুপে আন্তরিক স্বাগতম \n\n"
                f"এখান থেকে পাবেন:\n"
                f" আপডেট, সহায়তা ও দরকারি তথ্য\n"
                f" গ্রুপটি পিন করুন\n\n"
                f" একসাথে শিখি ও এগিয়ে চলি! \n"
                f" সব আপডেট পেতে চ্যানেলে যোগ দিন "
            )
            keyboard = InlineKeyboardMarkup([
                [InlineKeyboardButton(" Join Channel", url="https://t.me/CardArenaOfficial")]
            ])
            
            # Send the welcome message
            sent_msg = await update.message.reply_html(msg, reply_markup=keyboard)
            logger.info(f"Welcome message sent to {member.first_name} (ID: {member.id}) in chat {update.effective_chat.title}")
            
            # Wait 60 seconds and attempt to delete the message
            await asyncio.sleep(60)
            
            try:
                await context.bot.delete_message(
                    chat_id=update.effective_chat.id, 
                    message_id=sent_msg.message_id
                )
                logger.info(f"Deleted welcome message for {member.first_name}")
            except Forbidden:
                logger.warning(f"Cannot delete message - bot lacks admin rights in chat {update.effective_chat.id}. Ensure the bot has 'Delete messages' permission.")
            except BadRequest as e:
                # This often happens if the message was already deleted by an admin
                logger.warning(f"Cannot delete message: {e}")
            except TelegramError as e:
                logger.error(f"Telegram error while attempting to delete message: {e}")
                
        except TelegramError as e:
            logger.error(f"Failed to send welcome message to {member.first_name}: {e}")
        except Exception as e:
            logger.error(f"Unexpected error in welcome_message processing: {e}", exc_info=True)

async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # Log all errors from the bot framework
    logger.error(f"Exception while handling an update: {context.error}", exc_info=context.error)

# --- Main Bot Setup ---

def main():
    if not TOKEN:
        raise ValueError("TELEGRAM_BOT_TOKEN environment variable is not set! Deployment will fail.")
    
    # 1. Start the Flask server thread to keep the service alive
    start_keep_alive_thread()
    
    # 2. Initialize and start the Telegram bot (polling)
    app_bot = ApplicationBuilder().token(TOKEN).build()
    app_bot.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, welcome_message))
    app_bot.add_error_handler(error_handler)
    
    logger.info("Starting Telegram bot with Long Polling...")
    # Use reasonable polling settings
    app_bot.run_polling(poll_interval=1.0, timeout=20, drop_pending_updates=True) 

if __name__ == "__main__":
    main()
