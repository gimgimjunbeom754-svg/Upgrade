import sys
import threading
import sqlite3
import requests
import random
from datetime import datetime

from iris import ChatContext, Bot
from iris.bot.models import ErrorContext
from iris.decorators import *
from iris.kakaolink import IrisLink

# 외부 모듈 임포트
from bots.gemini import get_gemini
from bots.pyeval import python_eval, real_eval
from bots.stock import create_stock_image
from bots.imagen import get_imagen
from bots.lyrics import get_lyrics, find_lyrics
from bots.replyphoto import reply_photo
from bots.text2image import draw_text
from bots.coin import get_coin_info
from helper.BanControl import ban_user, unban_user
from bots.detect_nickname_change import detect_nickname_change

# --- [ 설정 및 API 키 ] ---
WEATHER_API_KEY = "4ec3817a115114f01070af7c9df2c5c1"

# --- [ 데이터베이스 관리 ] ---
class BotDatabase:
    def __init__(self, db_path='iris_data.db'):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self.cursor = self.conn.cursor()
        self._init_db()

    def _init_db(self):
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS users 
            (user_id TEXT PRIMARY KEY, chat_count INTEGER DEFAULT 0, 
             points INTEGER DEFAULT 1000, level INTEGER DEFAULT 0, last_check TEXT)''')
        self.conn.commit()

    def get_user(self, user_id):
        self.cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
        user = self.cursor.fetchone()
        if not user:
            self.cursor.execute("INSERT INTO users (user_id) VALUES (?)", (user_id,))
            self.conn.commit()
            return (user_id, 0, 1000, 0, None)
        return user

    def execute(self, query, params=()):
        self.cursor.execute(query, params)
        self.conn.commit()

db = BotDatabase()

# --- [ 기능 함수 ] ---
def get_weather(city_name):
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city_name}&appid={WEATHER_API_KEY}&lang=kr&units=metric"
    try:
        res = requests.get(url, timeout=5).json()
        if res.get("cod") != 200: 
            return f"도시를 찾을 수 없습니다: {city_name}"
        
        weather_desc = res["weather"][0]["description"]
        temp = res["main"]["temp"]
        humidity = res["main"]["humidity"]
        return (f"[ {res['name']} 날씨 정보 ]\n"
                f"- 온도: {temp}도\n"
                f"- 상태: {weather_desc}\n"
                f"- 습도: {humidity}%")
    except:
        return "날씨 정보를 가져오는 중 오류가 발생했습니다."

# --- [ 봇 초기화 ] ---
if len(sys.argv) < 2:
    print("Usage: python main.py [IRIS_URL]")
    sys.exit(1)

iris_url = sys.argv[1]
bot = Bot(iris_url)
kl = IrisLink(iris_url)

@bot.on_event("message")
@is_not_banned
def on_message(chat: ChatContext):
    user_id = chat.sender.name
    user_data = db.get_user(user_id)
    
    # 활동량 기록
    db.execute("UPDATE users SET chat_count = chat_count + 1 WHERE user_id = ?", (user_id,))

    command = chat.message.command
    full_text = chat.message.msg
    args_text = full_text.replace(command, "", 1).strip()

    try:
        match command:
            # 기본 시스템
            case "!도움말" | "!help":
                help_msg = (
                    "[ Iris Bot 명령어 안내 ]\n\n"
                    "1. 정보조회: !날씨 [도시], !주식 [종목], !코인, !순위\n"
                    "2. AI/생성: !gi [질문], !ig [문구], !분석, !텍스트 [문구]\n"
                    "3. 포인트: !출첵, !강화, !내정보\n"
                    "4. 기타: !가사찾기, !노래가사, !프사"
                )
                chat.reply(help_msg)

            case "!내정보":
                _, count, pts, lvl, _ = db.get_user(user_id)
                chat.reply(f"사용자: {user_id}\n채팅횟수: {count}\n포인트: {pts}\n강화레벨: +{lvl}")

            # 포인트 및 게임
            case "!출첵":
                today = datetime.now().strftime("%Y-%m-%d")
                if user_data[4] == today:
                    chat.reply("이미 오늘 출석체크를 완료했습니다.")
                else:
                    reward = random.randint(100, 500)
                    db.execute("UPDATE users SET points = points + ?, last_check = ? WHERE user_id = ?", (reward, today, user_id))
                    chat.reply(f"출석 완료. {reward}포인트가 지급되었습니다.")

            case "!강화":
                points, level = user_data[2], user_data[3]
                cost = 200
                if points < cost:
                    return chat.reply(f"포인트가 부족합니다. (필요: {cost})")
                
                new_points = points - cost
                success_rate = max(0.1, 0.7 - (level * 0.07))
                
                if random.random() < success_rate:
                    new_level = level + 1
                    db.execute("UPDATE users SET points=?, level=? WHERE user_id=?", (new_points, new_level, user_id))
                    chat.reply(f"강화 성공! 현재 레벨: +{new_level}\n잔액: {new_points}")
                else:
                    new_level = max(0, level - 1)
                    db.execute("UPDATE users SET points=?, level=? WHERE user_id=?", (new_points, new_level, user_id))
                    chat.reply(f"강화 실패. 현재 레벨: +{new_level}\n잔액: {new_points}")

            case "!순위":
                db.cursor.execute("SELECT user_id, chat_count FROM users ORDER BY chat_count DESC LIMIT 5")
                ranks = db.cursor.fetchall()
                rank_msg = "[ 채팅 활동 순위 ]\n" + "\n".join([f"{i+1}위: {r[0]} ({r[1]}회)" for i, r in enumerate(ranks)])
                chat.reply(rank_msg)

            # 외부 모듈 연동
            case "!날씨":
                city = args_text if args_text else "Seoul"
                chat.reply(get_weather(city))

            case "!gi" | "!i2i" | "!분석": get_gemini(chat)
            case "!ig": get_imagen(chat)
            case "!코인" | "!내코인" | "!바낸" | "!김프": get_coin_info(chat)
            case "!주식": create_stock_image(chat)
            case "!가사찾기": find_lyrics(chat)
            case "!노래가사": get_lyrics(chat)
            case "!ipy": python_eval(chat)
            case "!iev": real_eval(chat, kl)
            case "!tt" | "!프사": reply_photo(chat, kl)
            case "!텍스트" | "!사진": draw_text(chat)
            case "!ban": ban_user(chat)
            case "!unban": unban_user(chat)
            case "!hhi": chat.reply(f"Hello {chat.sender.name}")

    except Exception as e:
        print(f"Error: {e}")

@bot.on_event("error")
def on_error(err: ErrorContext):
    print(f"System Error: {err.exception}")

if __name__ == "__main__":
    # 닉네임 변경 감지 스레드
    threading.Thread(target=detect_nickname_change, args=(bot.iris_url,), daemon=True).start()
    
    print(f"Bot 시작됨 (연결 주소: {iris_url})")
    bot.run()
