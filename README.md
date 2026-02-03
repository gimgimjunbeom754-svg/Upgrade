from iris import ChatContext, Bot
from iris.bot.models import ErrorContext
from bots.gemini import get_gemini
from bots.pyeval import python_eval, real_eval
from bots.stock import create_stock_image
from bots.imagen import get_imagen
from bots.lyrics import get_lyrics, find_lyrics
from bots.replyphoto import reply_photo
from bots.text2image import draw_text
from bots.coin import get_coin_info

from iris.decorators import *
from helper.BanControl import ban_user, unban_user
from iris.kakaolink import IrisLink

from bots.detect_nickname_change import detect_nickname_change
import sys, threading, requests

# --- [ 설정 및 API 키 연결 ] ---
WEATHER_API_KEY = "4ec3817a115114f01070af7c9df2c5c1"

def get_weather(city_name):
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city_name}&appid={WEATHER_API_KEY}&lang=kr&units=metric"
    try:
        res = requests.get(url, timeout=5).json()
        if res.get("cod") != 200: return f"도시를 찾을 수 없습니다: {city_name}"
        weather_desc = res["weather"][0]["description"]
        temp = res["main"]["temp"]
        return f"[ {res['name']} 날씨 ]\n온도: {temp}°C\n상태: {weather_desc}"
    except:
        return "날씨 정보를 가져오는 중 오류가 발생했습니다."

iris_url = sys.argv[1]
bot = Bot(iris_url)

@bot.on_event("message")
@is_not_banned
def on_message(chat: ChatContext):
    try:
        # chat.message.text 속성을 활용하여 명령어 뒤의 텍스트 추출
        # 예: "!날씨 서울" 입력 시 command는 "!날씨", args_text는 "서울"이 됩니다.
        full_text = chat.message.text
        command = chat.message.command
        args_text = full_text.replace(command, "", 1).strip()

        match command:
            case "!hhi":
                chat.reply(f"Hello {chat.sender.name}")

            case "!tt" | "!ttt" | "!프사" | "!프사링":
                reply_photo(chat, kl)

            case "!iris":
                chat.reply_media("res/help.png")

            case "!gi" | "!i2i" | "!분석":
                get_gemini(chat)
            
            case "!ipy":
                python_eval(chat)
            
            case "!iev":
                real_eval(chat, kl)
            
            case "!ban":
                ban_user(chat)
            
            case "!unban":
                unban_user(chat)

            case "!주식":
                create_stock_image(chat)

            case "!ig":
                get_imagen(chat)
            
            case "!가사찾기":
                find_lyrics(chat)

            case "!노래가사":
                get_lyrics(chat)

            case "!텍스트" | "!사진" | "!껄무새" | "!멈춰" | "!지워" | "!진행" | "!말대꾸" | "!텍스트추가":
                draw_text(chat)
            
            case "!코인" | "!내코인" | "!바낸" | "!김프" | "!달러" | "!코인등록" | "!코인삭제":
                get_coin_info(chat)
            
            # --- 추가된 날씨 명령어 (chat.message.text 활용) ---
            case "!날씨":
                city = args_text if args_text else "Seoul"
                chat.reply(get_weather(city))
            
    except Exception as e :
        print(f"메시지 처리 중 오류: {e}")

#입장감지
@bot.on_event("new_member")
def on_newmem(chat: ChatContext):
    pass

#퇴장감지
@bot.on_event("del_member")
def on_delmem(chat: ChatContext):
    pass

@bot.on_event("error")
def on_error(err: ErrorContext):
    print(err.event, "이벤트에서 오류가 발생했습니다", err.exception)

if __name__ == "__main__":
    nickname_detect_thread = threading.Thread(target=detect_nickname_change, args=(bot.iris_url,))
    nickname_detect_thread.start()
    kl = IrisLink(bot.iris_url)
    bot.run()

