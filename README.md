# 專案名稱：LINE + GPT + Google Sheet 回報系統（Python 版）
# 說明：會員透過 LINE 傳送訊息 → GPT 回應 → 任務回報寫入 Google Sheet 或查詢分數

from flask import Flask, request, abort
import os
import openai
import gspread
from oauth2client.service_account import ServiceAccountCredentials
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import MessageEvent, TextMessage, TextSendMessage

app = Flask(__name__)

# 初始化 LINE Bot API
LINE_CHANNEL_ACCESS_TOKEN = os.getenv("LINE_CHANNEL_ACCESS_TOKEN")
LINE_CHANNEL_SECRET = os.getenv("LINE_CHANNEL_SECRET")
line_bot_api = LineBotApi(LINE_CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(LINE_CHANNEL_SECRET)

# 初始化 OpenAI
openai.api_key = os.getenv("OPENAI_API_KEY")

# 初始化 Google Sheet API
scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
creds = ServiceAccountCredentials.from_json_keyfile_name("google-credentials.json", scope)
client = gspread.authorize(creds)
sheet = client.open("BNI任務回報").sheet1

# Webhook 接收來自 LINE 的訊息
@app.route("/callback", methods=['POST'])
def callback():
    signature = request.headers['X-Line-Signature']
    body = request.get_data(as_text=True)
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)
    return 'OK'

# GPT 呼叫函式
def ask_gpt(prompt):
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content.strip()

# 處理文字訊息
@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    user_text = event.message.text
    user_id = event.source.user_id

    if user_text.startswith("回報任務"):
        task = user_text.replace("回報任務", "").strip()
        sheet.append_row([user_id, task])
        reply = f"✅ 任務『{task}』已成功回報！"

    elif user_text == "查詢分數":
        records = sheet.get_all_records()
        count = sum(1 for r in records if r['user_id'] == user_id)
        reply = f"📊 您目前累積任務完成次數為：{count} 次"

    else:
        reply = ask_gpt(user_text)

    line_bot_api.reply_message(
        event.reply_token,
        TextSendMessage(text=reply)
    )

# 啟動伺服器
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=int(os.getenv('PORT', 5000)))
# line-gpt-sheet-bot
