import discord
from discord.ext import commands, tasks
import asyncio
from datetime import datetime, timedelta
import json
import os
import sys
import logging
from dotenv import load_dotenv

# Raspberry Pi用のログ設定（最適化）
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s | %(levelname)s | %(message)s',
    handlers=[
        logging.FileHandler('/home/pi/amongus_bot.log', encoding='utf-8'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# システム情報表示
logger.info("🍓 Raspberry Pi環境でBot起動中...")

# .envファイルから環境変数を読み込み
load_dotenv()

# 環境変数からトークンを取得
TOKEN = os.getenv('DISCORD_BOT_TOKEN')

# トークンが設定されているかチェック
if not TOKEN:
    logger.error("❌ エラー: DISCORD_BOT_TOKENが設定されていません")
    logger.info("📝 .envファイルを作成して、以下の形式でトークンを設定してください：")
    logger.info("DISCORD_BOT_TOKEN=あなたのボットトークン")
    sys.exit(1)

logger.info("✅ 環境変数からトークンを正常に読み込みました")
logger.info("🍓 Raspberry Pi環境での起動準備完了")

# Botの設定（Raspberry Pi最適化）
intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix='!', intents=intents)

# 投票データを保存するファイル（Raspberry Pi用パス）
POLL_DATA_FILE = '/home/pi/amongus_poll_data.json'
BACKUP_DIR = '/home/pi/backups/'

# バックアップディレクトリを作成
os.makedirs(BACKUP_DIR, exist_ok=True)

class AmongUsPollBot:
    def __init__(self):
        self.poll_channel_id = None
        self.poll_message_id = None
        self.poll_dates = []
        self.user_votes = {}
        self.reminder_enabled = True
        self.auto_bump_enabled = True
        self.start_time = datetime.now()
        self.error_count = 0
        self.pi_temp = 0.0  # Raspberry Pi温度
        
    def get_pi_temperature(self):
        """Raspberry Piの温度を取得"""
        try:
            with open('/sys/class/thermal/thermal_zone0/temp', 'r') as f:
                temp = float(f.read()) / 1000.0
                self.pi_temp = temp
                return temp
        except:
            return 0.0
    
    def get_pi_info(self):
        """Raspberry Pi システム情報を取得"""
        info = {}
        try:
            # CPU使用率
            with os.popen('vcgencmd measure_temp') as f:
                temp_str = f.read()
                if 'temp=' in temp_str:
                    info['temp'] = temp_str.split('=')[1].strip()
                else:
                    info['temp'] = f"{self.get_pi_temperature():.1f}°C"
            
            # メモリ使用量
            with os.popen('free -m') as f:
                lines = f.readlines()
                if len(lines) > 1:
                    mem_line = lines[1].split()
                    if len(mem_line) > 2:
                        total = int(mem_line[1])
                        used = int(mem_line[2])
                        info['memory'] = f"{used}/{total}MB ({used/total*100:.1f}%)"
            
            # ディスク使用量
            with os.popen('df -h / | tail -1') as f:
                disk_line = f.read().split()
                if len(disk_line) > 4:
                    info['disk'] = f"{disk_line[2]}/{disk_line[1]} ({disk_line[4]})"
            
            # 稼働時間
            with os.popen('uptime -p') as f:
                info['uptime'] = f.read().strip()
                
        except Exception as e:
            logger.error(f"❌ システム情報取得エラー: {e}")
        
        return info
        
    def save_poll_data(self):
        """投票データをファイルに保存"""
        data = {
            'channel_id': self.poll_channel_id,
            'message_id': self.poll_message_id,
            'dates': self.poll_dates,
            'user_votes': {str(k): v for k, v in self.user_votes.items()},
            'reminder_enabled': self.reminder_enabled,
            'auto_bump_enabled': self.auto_bump_enabled,
            'last_save': datetime.now().isoformat()
        }
        try:
            with open(POLL_DATA_FILE, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            logger.info(f"📁 投票データを保存しました")
        except Exception as e:
            logger.error(f"❌ データ保存エラー: {e}")
    
    def load_poll_data(self):
        """投票データをファイルから読み込み"""
        try:
            with open(POLL_DATA_FILE, 'r', encoding='utf-8') as f:
                data = json.load(f)
                self.poll_channel_id = data.get('channel_id')
                self.poll_message_id = data.get('message_id')
                self.poll_dates = data.get('dates', [])
                self.user_votes = {int(k): v for k, v in data.get('user_votes', {}).items()}
                self.reminder_enabled = data.get('reminder_enabled', True)
                self.auto_bump_enabled = data.get('auto_bump_enabled', True)
            logger.info("📂 既存の投票データを読み込みました")
        except FileNotFoundError:
            logger.info("📄 新規データファイルを作成します")
        except Exception as e:
            logger.error(f"⚠️ データ読み込みエラー: {e}")

    def create_backup(self):
        """データのバックアップを作成（Raspberry Pi用）"""
        try:
            backup_name = f"{BACKUP_DIR}backup_amongus_poll_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
            if os.path.exists(POLL_DATA_FILE):
                with open(POLL_DATA_FILE, 'r', encoding='utf-8') as src:
                    with open(backup_name, 'w', encoding='utf-8') as dst:
                        dst.write(src.read())
                logger.info(f"💾 バックアップを作成しました: {backup_name}")
                
                # 古いバックアップファイルを削除（7日以上古いもの）
                self.cleanup_old_backups()
        except Exception as e:
            logger.error(f"❌ バックアップ作成エラー: {e}")
    
    def cleanup_old_backups(self):
        """古いバックアップファイルを削除"""
        try:
            import glob
            backup_files = glob.glob(f"{BACKUP_DIR}backup_amongus_poll_*.json")
            cutoff_time = datetime.now() - timedelta(days=7)
            
            for backup_file in backup_files:
                file_time = datetime.fromtimestamp(os.path.getmtime(backup_file))
                if file_time < cutoff_time:
                    os.remove(backup_file)
                    logger.info(f"🗑️ 古いバックアップを削除: {backup_file}")
        except Exception as e:
            logger.error(f"❌ バックアップ掃除エラー: {e}")

poll_bot = AmongUsPollBot()

@bot.event
async def on_ready():
    logger.info(f'🎉 {bot.user} がRaspberry Piでログインしました！')
    logger.info(f'🌐 サーバー数: {len(bot.guilds)}')
    logger.info(f'⏰ 起動時刻: {datetime.now()}')
    
    # Raspberry Pi 情報を表示
    pi_info = poll_bot.get_pi_info()
    logger.info(f'🍓 Raspberry Pi温度: {pi_info.get("temp", "取得失敗")}')
    logger.info(f'💾 メモリ使用量: {pi_info.get("memory", "取得失敗")}')
    logger.info(f'💿 ディスク使用量: {pi_info.get("disk", "取得失敗")}')
    logger.info(f'🍓 24時間稼働開始（省電力モード）...')
    
    # データ読み込み
    poll_bot.load_poll_data()
    
    # バックアップ作成
    poll_bot.create_backup()
    
    # 各タスクを開始（Raspberry Pi用に間隔調整）
    repost_poll_task.start()
    reminder_task.start()
    auto_bump_task.start()
    weekly_new_poll_task.start()
    pi_health_check_task.start()  # Raspberry Pi専用ヘルスチェック

@tasks.loop(hours=2)  # Raspberry Pi用：2時間おきに軽めのチェック
async def pi_health_check_task():
    """Raspberry Pi専用ヘルスチェック"""
    try:
        temp = poll_bot.get_pi_temperature()
        pi_info = poll_bot.get_pi_info()
        
        logger.info(f"🍓 Pi ヘルスチェック | 温度: {temp:.1f}°C | 稼働時間: {datetime.now() - poll_bot.start_time}")
        
        # 温度警告（80度以上で警告）
        if temp > 80.0:
            logger.warning(f"🌡️ 高温警告: {temp:.1f}°C - ファンの確認をしてください")
        
        # 定期バックアップ（12時間おき）
        current_time = datetime.now()
        if current_time.hour % 12 == 0 and current_time.minute < 5:
            poll_bot.create_backup()
        
    except Exception as e:
        logger.error(f"❌ Piヘルスチェックエラー: {e}")

def get_week_dates():
    """今週の日付リストを取得（月曜日から日曜日）"""
    today = datetime.now()
    monday = today - timedelta(days=today.weekday())
    dates = []
    
    for i in range(7):
        date = monday + timedelta(days=i)
        day_names = ['月', '火', '水', '木', '金', '土', '日']
        date_str = f"{date.month}/{date.day}({day_names[i]})"
        dates.append(date_str)
    
    return dates

@bot.command(name='amongus')
async def create_amongus_poll(ctx):
    """アモングアス日程投票を作成するコマンド"""
    logger.info(f"📊 投票作成要求: {ctx.author} in {ctx.guild.name}")
    
    try:
        # 今週の日付を取得
        week_dates = get_week_dates()
        
        # 投票情報を保存
        poll_bot.poll_dates = week_dates
        poll_bot.poll_channel_id = ctx.channel.id
        poll_bot.user_votes = {}
        
        # 投票メッセージを作成
        poll_message = await create_amongus_poll_message(ctx.channel, week_dates)
        poll_bot.poll_message_id = poll_message.id
        
        # メッセージをピン留め
        try:
            await poll_message.pin()
            await ctx.send("📌 投票をピン留めしました！", delete_after=5)
            logger.info("📌 投票をピン留めしました")
        except discord.Forbidden:
            await ctx.send("⚠️ ピン留め権限がありません。", delete_after=10)
            logger.warning("⚠️ ピン留め権限不足")
        except discord.HTTPException:
            await ctx.send("⚠️ ピン留めに失敗しました。", delete_after=10)
            logger.warning("⚠️ ピン留め失敗")
        
        poll_bot.save_poll_data()
        await ctx.message.delete()
        
    except Exception as e:
        logger.error(f"❌ 投票作成エラー: {e}")
        await ctx.send(f"❌ 投票作成中にエラーが発生しました: {e}")

async def create_amongus_poll_message(channel, dates):
    """アモングアス投票メッセージを作成して送信"""
    day_emojis = ['🟦', '🟩', '🟨', '🟧', '🟪', '🟫', '⬜']
    
    # 投票メッセージの作成
    embed = discord.Embed(
        title="🚨 **AMONG US 日程投票** 🚨",
        description="## 🎮 **今週Among Usやりませんか？** 🎮\n\n" +
                   "### 📅 **参加可能な日を選んでください！**\n" +
                   "**複数選択OK！みんなで楽しもう！** 🎉\n\n",
        color=0xff0000,
        timestamp=datetime.now()
    )
    
    # 日付オプションを表示
    date_options = ""
    for i, date in enumerate(dates):
        date_options += f"## {day_emojis[i]} **{date}**\n"
    
    embed.add_field(name="📋 **参加可能日程**", value=date_options, inline=False)
    
    # 投票方法の説明
    embed.add_field(
        name="🎯 **投票方法**", 
        value="### 👆 **下の絵文字をクリック！**\n" +
              "• **複数の日程を選択可能**\n" +
              "• **再度クリックで投票取り消し**\n" +
              "• **`!amongusresults`で結果確認**", 
        inline=False
    )
    
    # 注意書きを追加
    embed.add_field(
        name="⏰ **重要**",
        value="投票は **1日おきに自動更新** されます\n" +
              "**参加者が多い日程で開催予定！**\n" +
              "🍓 **Raspberry Piで省電力24時間稼働中**",
        inline=False
    )
    
    embed.set_footer(text="🔥 みんなの参加を待ってます！ | Powered by Raspberry Pi")
    
    # アナウンスメッセージ
    announcement = "🚨 **@everyone AMONG US 投票開始！** 🚨\n" + \
                  "👆 **上の投票に参加してください！** 👆"
    
    await channel.send(announcement)
    message = await channel.send(embed=embed)
    
    # リアクションを追加（Raspberry Pi用：少し遅めに）
    for i in range(len(dates)):
        await message.add_reaction(day_emojis[i])
        await asyncio.sleep(0.5)  # Raspberry Pi用レート制限対策
    
    return message

@tasks.loop(hours=6)  # 6時間おきにリマインダー
async def reminder_task():
    """定期的にリマインダーを送信"""
    if not poll_bot.reminder_enabled or not poll_bot.poll_channel_id:
        return
    
    try:
        channel = bot.get_channel(poll_bot.poll_channel_id)
        if not channel:
            return
        
        # 投票数を確認
        unique_voters = len(poll_bot.user_votes)
        
        if unique_voters < 5:  # 投票者が5人未満の場合のみリマインダー
            reminder_messages = [
                "🔔 **Among Us投票** まだの人はお忘れなく！",
                "🎮 **Among Us日程投票** 参加者募集中！",
                "⏰ **Among Us投票** 締切前にご参加ください！",
                "🚀 **Among Us** みんなで宇宙船を修理しよう！投票お願いします！"
            ]
            
            import random
            message = random.choice(reminder_messages)
            await channel.send(f"{message}\n`!amongusresults` で現在の状況確認可能", delete_after=300)
            logger.info(f"🔔 リマインダー送信: 投票者数 {unique_voters}人")
            
    except Exception as e:
        logger.error(f"❌ リマインダーエラー: {e}")
        poll_bot.error_count += 1

@tasks.loop(hours=4)  # 4時間おきに投票を上げる
async def auto_bump_task():
    """投票メッセージを定期的に上に表示"""
    if not poll_bot.auto_bump_enabled or not poll_bot.poll_message_id:
        return
    
    try:
        channel = bot.get_channel(poll_bot.poll_channel_id)
        if not channel:
            return
        
        bump_messages = [
            "⬆️ **Among Us投票** 上げ！",
            "📈 **投票中** みんな参加してね！",
            "🔝 **Among Us** 日程投票実施中！",
            "⬆️ **参加者募集** 投票お忘れなく！"
        ]
        
        import random
        message = random.choice(bump_messages)
        bump_msg = await channel.send(message)
        
        # 3分後に削除
        await asyncio.sleep(180)
        try:
            await bump_msg.delete()
        except:
            pass
        
        logger.info(f"📈 自動バンプ実行")
            
    except Exception as e:
        logger.error(f"❌ 自動バンプエラー: {e}")
        poll_bot.error_count += 1

@tasks.loop(hours=24)  # 24時間おきにチェック
async def weekly_new_poll_task():
    """月曜日の朝9時に新しい週の投票を自動作成"""
    now = datetime.now()
    
    # 月曜日の朝9時かチェック
    if now.weekday() == 0 and 8 <= now.hour <= 10:
        if poll_bot.poll_channel_id:
            try:
                channel = bot.get_channel(poll_bot.poll_channel_id)
                if channel:
                    logger.info(f"🗓️ 週次投票更新開始")
                    
                    # 前の投票のピン留めを解除
                    if poll_bot.poll_message_id:
                        try:
                            old_message = await channel.fetch_message(poll_bot.poll_message_id)
                            await old_message.unpin()
                            await old_message.edit(content="⏰ **この投票は終了しました**\n👇 新しい週の投票が下にあります", embed=old_message.embeds[0])
                            logger.info("🗂️ 前週の投票を終了しました")
                        except Exception as e:
                            logger.error(f"⚠️ 前の投票処理エラー: {e}")
                    
                    # 新しい週の投票を作成
                    week_dates = get_week_dates()
                    poll_bot.poll_dates = week_dates
                    poll_bot.user_votes = {}
                    
                    new_message = await create_amongus_poll_message(channel, week_dates)
                    poll_bot.poll_message_id = new_message.id
                    
                    try:
                        await new_message.pin()
                        logger.info("📌 新しい週の投票をピン留めしました")
                    except Exception as e:
                        logger.error(f"⚠️ 新投票ピン留めエラー: {e}")
                    
                    poll_bot.save_poll_data()
                    await channel.send("🎉 **新しい週のAmong Us投票が始まりました！**\n今週もみんなで楽しみましょう！ 🚀", delete_after=600)
                    
            except Exception as e:
                logger.error(f"❌ 週次投票作成エラー: {e}")
                poll_bot.error_count += 1

@tasks.loop(hours=48)  # 48時間 = 1日おき  
async def repost_poll_task():
    """1日おきに投票を再投稿（月曜日は除く）"""
    now = datetime.now()
    
    # 月曜日の場合は週次タスクに任せるのでスキップ
    if now.weekday() == 0:
        return
        
    if not poll_bot.poll_channel_id or not poll_bot.poll_dates:
        return
    
    try:
        channel = bot.get_channel(poll_bot.poll_channel_id)
        if not channel:
            return
        
        logger.info(f"🔄 投票再投稿開始")
        
        # 古い投票メッセージのピン留めを解除して削除
        if poll_bot.poll_message_id:
            try:
                old_message = await channel.fetch_message(poll_bot.poll_message_id)
                await old_message.unpin()
                await old_message.delete()
            except:
                pass
        
        # 新しい投票メッセージを作成
        new_message = await create_amongus_poll_message(channel, poll_bot.poll_dates)
        
        # 新しいメッセージをピン留め
        try:
            await new_message.pin()
        except:
            pass
        
        # 既存の投票を復元
        await restore_votes(new_message)
        
        poll_bot.poll_message_id = new_message.id
        poll_bot.save_poll_data()
        
        logger.info(f"✅ 投票再投稿完了")
        
    except Exception as e:
        logger.error(f"❌ 投票再投稿エラー: {e}")
        poll_bot.error_count += 1

# リアクション処理
@bot.event
async def on_raw_reaction_add(payload):
    await handle_reaction_change(payload, True)

@bot.event
async def on_raw_reaction_remove(payload):
    await handle_reaction_change(payload, False)

async def handle_reaction_change(payload, is_add):
    if payload.user_id == bot.user.id:
        return
    
    if payload.message_id != poll_bot.poll_message_id:
        return
    
    day_emojis = ['🟦', '🟩', '🟨', '🟧', '🟪', '🟫', '⬜']
    
    if str(payload.emoji) not in day_emojis:
        return
    
    user_id = payload.user_id
    date_index = day_emojis.index(str(payload.emoji))
    selected_date = poll_bot.poll_dates[date_index]
    
    if user_id not in poll_bot.user_votes:
        poll_bot.user_votes[user_id] = []
    
    if is_add and selected_date not in poll_bot.user_votes[user_id]:
        poll_bot.user_votes[user_id].append(selected_date)
        logger.info(f"✅ 投票追加: {user_id} -> {selected_date}")
    elif not is_add and selected_date in poll_bot.user_votes[user_id]:
        poll_bot.user_votes[user_id].remove(selected_date)
        logger.info(f"❌ 投票削除: {user_id} -> {selected_date}")
    
    poll_bot.save_poll_data()

async def restore_votes(message):
    """既存の投票をメッセージのリアクションに復元"""
    day_emojis = ['🟦', '🟩', '🟨', '🟧', '🟪', '🟫', '⬜']
    
    for user_id, selected_dates in poll_bot.user_votes.items():
        try:
            user = bot.get_user(user_id)
            if not user:
                continue
                
            for date in selected_dates:
                if date in poll_bot.poll_dates:
                    date_index = poll_bot.poll_dates.index(date)
                    emoji = day_emojis[date_index]
                    await message.add_reaction(emoji)
                    await asyncio.sleep(0.5)  # Raspberry Pi用レート制限対策
                    
        except Exception as e:
            logger.error(f"⚠️ 投票復元エラー (ユーザー {user_id}): {e}")

# Raspberry Pi専用コマンド
@bot.command(name='pistatus')
async def pi_status(ctx):
    """Raspberry Pi の状態を確認"""
    try:
        pi_info = poll_bot.get_pi_info()
        uptime = datetime.now() - poll_bot.start_time
        
        embed = discord.Embed(
            title="🍓 Raspberry Pi Among Us Bot 稼働状況",
            color=0x00ff00,
            timestamp=datetime.now()
        )
        
        embed.add_field(name="🖥️ システム", value="Raspberry Pi", inline=True)
        embed.add_field(name="⏰ Bot稼働時間", value=str(uptime).split('.')[0], inline=True)
        embed.add_field(name="🍓 Pi稼働時間", value=pi_info.get('uptime', '取得失敗'), inline=True)
        embed.add_field(name="🌡️ 温度", value=pi_info.get('temp', '取得失敗'), inline=True)
        embed.add_field(name="💾 メモリ", value=pi_info.get('memory', '取得失敗'), inline=True)
        embed.add_field(name="💿 ディスク", value=pi_info.get('disk', '取得失敗'), inline=True)
        embed.add_field(name="📊 サーバー数", value=len(bot.guilds), inline=True)
        embed.add_field(name="🔄 ピング", value=f"{round(bot.latency * 1000)}ms", inline=True)
        embed.add_field(name="⚠️ エラー数", value=poll_bot.error_count, inline=True)
        embed.add_field(name="🎮 アクティブ投票", value="あり" if poll_bot.poll_message_id else "なし", inline=True)
        embed.add_field(name="👥 投票者数", value=len(poll_bot.user_votes), inline=True)
        
        embed.set_footer(text="省電力24時間稼働中 on Raspberry Pi")
        
        await ctx.send(embed=embed)
        
    except Exception as e:
        await ctx.send(f"❌ Pi状態取得エラー: {e}")
        logger.error(f"❌ Pi状態エラー: {e}")

@bot.command(name='amongusresults')
async def amongus_results(ctx):
    """アモングアス投票結果を表示"""
    if not poll_bot.poll_dates:
        await ctx.send("❌ アクティブなAmong Us投票がありません。")
        return
    
    try:
        # 各日程の投票数を集計
        date_counts = {}
        user_list_by_date = {}
        
        for date in poll_bot.poll_dates:
            date_counts[date] = 0
            user_list_by_date[date] = []
        
        for user_id, selected_dates in poll_bot.user_votes.items():
            user = bot.get_user(user_id)
            username = user.display_name if user else f"ユーザー{user_id}"
            
            for date in selected_dates:
                if date in date_counts:
                    date_counts[date] += 1
                    user_list_by_date[date].append(username)
        
        # 結果を表示
        embed = discord.Embed(
            title="🚀 Among Us 投票結果",
            color=0x00ff00,
            timestamp=datetime.now()
        )
        
        # 投票数順にソート
        sorted_dates = sorted(date_counts.items(), key=lambda x: x[1], reverse=True)
        
        result_text = ""
        for date, count in sorted_dates:
            participants = user_list_by_date[date]
            if count > 0:
                participant_list = ", ".join(participants[:5])
                if len(participants) > 5:
                    participant_list += f" (+{len(participants)-5}人)"
                result_text += f"**{date}**: {count}人\n└ {participant_list}\n\n"
            else:
                result_text += f"**{date}**: {count}人\n\n"
        
        embed.description = result_text
        
        # 最も人気の日程をハイライト
        if sorted_dates and sorted_dates[0][1] > 0:
            best_date = sorted_dates[0][0]
            best_count = sorted_dates[0][1]
            embed.add_field(
                name="🏆 最多参加予定日", 
                value=f"**{best_date}** ({best_count}人)", 
                inline=False
            )
        
        embed.set_footer(text="リアルタイム結果 | Powered by Raspberry Pi")
        
        await ctx.send(embed=embed)
        
    except Exception as e:
        await ctx.send(f"❌ 結果の取得でエラーが発生しました: {e}")
        logger.error(f"❌ 結果表示エラー: {e}")

@bot.command(name='reboot')
async def reboot_pi(ctx):
    """Raspberry Pi を再起動（管理者のみ）"""
    if not ctx.author.guild_permissions.administrator:
        await ctx.send("❌ 管理者権限が必要です。")
        return
    
    await ctx.send("🔄 Raspberry Pi を再起動しています...")
    logger.info("🔚 管理者によるRaspberry Pi再起動要求")
    
    # データ保存
    poll_bot.save_poll_data()
    poll_bot.create_backup()
    
    await ctx.send("✅ データ保存完了。30秒後に再起動します。")
    await asyncio.sleep(5)
    
    # Raspberry Pi 再起動
    os.system('sudo reboot')

@bot.command(name='shutdown')
async def shutdown_bot(ctx):
    """Bot を安全にシャットダウン（管理者のみ）"""
    if not ctx.author.guild_permissions.administrator:
        await ctx.send("❌ 管理者権限が必要です。")
        return
    
    await ctx.send("🔄 Botを安全にシャットダウンしています...")
    logger.info("🔚 管理者によるシャットダウン要求")
    
    # データ保存
    poll_bot.save_poll_data()
    poll_bot.create_backup()
    
    await ctx.send("✅ シャットダウン完了。")
    await bot.close()

# エラーハンドリング（Raspberry Pi最適化）
@bot.event
async def on_error(event, *args, **kwargs):
    logger.error(f"❌ イベントエラー: {event}")
    logger.error(f"⚠️ 詳細: {args}")
    poll_bot.error_count += 1
    
    # 温度チェック（エラー発生時）
    temp = poll_bot.get_pi_temperature()
    if temp > 85.0:
        logger.critical(f"🌡️ 危険な高温: {temp:.1f}°C - 冷却を確認してください")

@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.CommandNotFound):
        return
    
    logger.error(f"❌ コマンドエラー: {error}")
    await ctx.send(f"❌ エラーが発生しました: {error}")
    poll_bot.error_count += 1

# Raspberry Pi用のシグナルハンドラー
def setup_signal_handlers():
    """Raspberry Pi用シグナルハンドラーの設定"""
    import signal
    
    def signal_handler(sig, frame):
        logger.info("🔚 シャットダウンシグナルを受信しました。安全にシャットダウンしています...")
        poll_bot.save_poll_data()
        poll_bot.create_backup()
        logger.info("💾 データ保存完了")
        sys.exit(0)
    
    signal.signal(signal.SIGINT, signal_handler)   # Ctrl+C
    signal.signal(signal.SIGTERM, signal_handler)  # システムシャットダウン

# systemd用のサービスファイル作成
def create_systemd_service():
    """systemd用サービスファイルの作成"""
    service_content = f"""[Unit]
Description=Among Us Discord Poll Bot
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi
ExecStart=/usr/bin/python3 /home/pi/amongus_bot.py
Restart=always
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
"""
    
    try:
        with open('/tmp/amongus-bot.service', 'w') as f:
            f.write(service_content)
        logger.info("📄 systemdサービスファイルを /tmp/amongus-bot.service に作成しました")
        logger.info("📋 以下のコマンドでサービスを有効化できます:")
        logger.info("sudo cp /tmp/amongus-bot.service /etc/systemd/system/")
        logger.info("sudo systemctl enable amongus-bot.service")
        logger.info("sudo systemctl start amongus-bot.service")
    except Exception as e:
        logger.error(f"❌ サービスファイル作成エラー: {e}")

# メイン実行部分
if __name__ == "__main__":
    try:
        logger.info("🚀 Raspberry PiでのBot起動を開始します...")
        logger.info("🍓 24時間省電力稼働モード")
        logger.info("📊 詳細ログを /home/pi/amongus_bot.log に記録中...")
        
        # シグナルハンドラー設定
        setup_signal_handlers()
        
        # systemdサービスファイル作成
        create_systemd_service()
        
        # 起動時の温度チェック
        startup_temp = poll_bot.get_pi_temperature()
        logger.info(f"🌡️ 起動時温度: {startup_temp:.1f}°C")
        
        if startup_temp > 80.0:
            logger.warning(f"⚠️ 起動時の温度が高めです: {startup_temp:.1f}°C")
        
        # Botを起動
        bot.run(TOKEN)
        
    except discord.LoginFailure:
        logger.error("❌ ログインに失敗しました。トークンを確認してください。")
        logger.info("🔧 .envファイルのDISCORD_BOT_TOKENが正しく設定されているか確認してください。")
        sys.exit(1)
    except KeyboardInterrupt:
        logger.info("🔚 キーボード割り込みでシャットダウン")
        poll_bot.save_poll_data()
        poll_bot.create_backup()
    except Exception as e:
        logger.error(f"❌ 予期しないエラー: {e}")
        logger.info("🔄 10秒後に再起動を試みます...")
        
        # データ保存
        poll_bot.save_poll_data()
        poll_bot.create_backup()
        
        import time
        time.sleep(10)
        
        # 自動再起動（Raspberry Pi用）
        logger.info("🔄 自動再起動を実行...")
        python = sys.executable
        os.execl(python, python, *sys.argv)
        
    finally:
        logger.info("🔚 Among Us投票Bot終了")
        # Raspberry Pi では input() を使わない（ヘッドレス運用）
