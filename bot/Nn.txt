from aiofiles import open as aiopen
from aiofiles.os import path as aiopath, remove
from asyncio import gather, create_subprocess_exec
from os import execl as osexecl
from psutil import (
    disk_usage,
    cpu_percent,
    swap_memory,
    cpu_count,
    virtual_memory,
    net_io_counters,
    boot_time,
)
from pyrogram.filters import command
from pyrogram.handlers import MessageHandler
from signal import signal, SIGINT
from sys import executable
from time import time
from flask import Flask

from bot import (
    bot,
    botStartTime,
    LOGGER,
    intervals,
    config_dict,
    scheduler,
    sabnzbd_client,
)
from .helper.ext_utils.telegraph_helper import telegraph
from .helper.ext_utils.bot_utils import (
    cmd_exec,
    sync_to_async,
    create_help_buttons,
    new_task,
)
from .helper.ext_utils.db_handler import database
from .helper.ext_utils.files_utils import clean_all, exit_clean_up
from .helper.ext_utils.jdownloader_booter import jdownloader
from .helper.ext_utils.status_utils import get_readable_file_size, get_readable_time
from .helper.listeners.aria2_listener import start_aria2_listener
from .helper.mirror_leech_utils.rclone_utils.serve import rclone_serve_booter
from .helper.telegram_helper.bot_commands import BotCommands
from .helper.telegram_helper.button_build import ButtonMaker
from .helper.telegram_helper.filters import CustomFilters
from .helper.telegram_helper.message_utils import send_message, edit_message, send_file
from .modules import (
    authorize,
    cancel_task,
    clone,
    exec,
    file_selector,
    gd_count,
    gd_delete,
    gd_search,
    mirror_leech,
    status,
    ytdlp,
    shell,
    users_settings,
    bot_settings,
    help,
    force_start,
)

# Flask server setup for health check
app = Flask(__name__)

@app.route('/health', methods=['GET'])
def health_check():
    return "Bot is running smoothly", 200

@new_task
async def stats(_, message):
    if await aiopath.exists(".git"):
        last_commit = await cmd_exec(
            "git log -1 --date=short --pretty=format:'%cd <b>From</b> %cr'", True
        )
        last_commit = last_commit[0]
    else:
        last_commit = "No UPSTREAM_REPO"
    total, used, free, disk = disk_usage("/")
    swap = swap_memory()
    memory = virtual_memory()
    stats = (
        f"<b>Commit Date:</b> {last_commit}\n\n"
        f"<b>Bot Uptime:</b> {get_readable_time(time() - botStartTime)}\n"
        f"<b>OS Uptime:</b> {get_readable_time(time() - boot_time())}\n\n"
        f"<b>Total Disk Space:</b> {get_readable_file_size(total)}\n"
        f"<b>Used:</b> {get_readable_file_size(used)} | <b>Free:</b> {get_readable_file_size(free)}\n\n"
        f"<b>Upload:</b> {get_readable_file_size(net_io_counters().bytes_sent)}\n"
        f"<b>Download:</b> {get_readable_file_size(net_io_counters().bytes_recv)}\n\n"
        f"<b>CPU:</b> {cpu_percent(interval=0.5)}%\n"
        f"<b>RAM:</b> {memory.percent}%\n"
        f"<b>DISK:</b> {disk}%\n\n"
        f"<b>Physical Cores:</b> {cpu_count(logical=False)}\n"
        f"<b>Total Cores:</b> {cpu_count(logical=True)}\n\n"
        f"<b>SWAP:</b> {get_readable_file_size(swap.total)} | <b>Used:</b> {swap.percent}%\n"
        f"<b>Memory Total:</b> {get_readable_file_size(memory.total)}\n"
        f"<b>Memory Free:</b> {get_readable_file_size(memory.available)}\n"
        f"<b>Memory Used:</b> {get_readable_file_size(memory.used)}\n"
    )
    await send_message(message, stats)

# Flask app run with specified port
def run_flask():
    app.run(host='0.0.0.0', port=8080)

# Bot main function
async def main():
    if config_dict["DATABASE_URL"]:
        await database.db_load()
    await gather(
        jdownloader.initiate(),
        sync_to_async(clean_all),
        bot_settings.initiate_search_tools(),
        restart_notification(),
        telegraph.create_account(),
        rclone_serve_booter(),
        sync_to_async(start_aria2_listener, wait=False),
    )
    create_help_buttons()

    bot.add_handler(
        MessageHandler(
            start, filters=command(BotCommands.StartCommand, case_sensitive=True)
        )
    )
    bot.add_handler(
        MessageHandler(
            log,
            filters=command(BotCommands.LogCommand, case_sensitive=True)
            & CustomFilters.sudo,
        )
    )
    bot.add_handler(
        MessageHandler(
            restart,
            filters=command(BotCommands.RestartCommand, case_sensitive=True)
            & CustomFilters.sudo,
        )
    )
    bot.add_handler(
        MessageHandler(
            ping,
            filters=command(BotCommands.PingCommand, case_sensitive=True)
            & CustomFilters.authorized,
        )
    )
    bot.add_handler(
        MessageHandler(
            bot_help,
            filters=command(BotCommands.HelpCommand, case_sensitive=True)
            & CustomFilters.authorized,
        )
    )
    bot.add_handler(
        MessageHandler(
            stats,
            filters=command(BotCommands.StatsCommand, case_sensitive=True)
            & CustomFilters.authorized,
        )
    )

    LOGGER.info("Bot Started!")
    signal(SIGINT, exit_clean_up)

# Flask and bot loop execution
if __name__ == "__main__":
    from threading import Thread
    Thread(target=run_flask).start()
    bot.loop.run_until_complete(main())
    bot.loop.run_forever()
