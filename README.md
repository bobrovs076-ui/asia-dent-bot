import os
import json
from datetime import datetime
from flask import Flask, request, jsonify
import requests

META_TOKEN        = os.getenv("META_TOKEN")
PHONE_NUMBER_ID   = os.getenv("PHONE_NUMBER_ID")
VERIFY_TOKEN      = os.getenv("VERIFY_TOKEN")
OPENAI_API_KEY    = os.getenv("OPENAI_API_KEY")
OPENAI_MODEL      = os.getenv("OPENAI_MODEL", "gpt-4o-mini")

WA_API_URL = f"https://graph.facebook.com/v21.0/{PHONE_NUMBER_ID}/messages"
HEADERS_WA = {"Authorization": f"Bearer {META_TOKEN}", "Content-Type": "application/json"}

app = Flask(__name__)

TRIAGE_KEYWORDS = ["кровь","сильная боль","лихорад","отёк","гной","травма","перелом","аллерг","обморок","температур","сильное кровотечение"]

def llm_answer(user_text: str) -> str:
    lower = user_text.lower()
    if any(k in lower for k in TRIAGE_KEYWORDS):
        return ("Похоже на срочную ситуацию. Рекомендую немедленно обратиться к врачу. "
                "Я могу записать вас на приём. Напишите, когда удобно.")
    return "Я ассистент стоматолога. Задайте вопрос или выберите в меню ниже."

def send_text(to_number: str, body: str):
    payload = {
        "messaging_product": "whatsapp",
        "to": to_number,
        "type": "text",
        "text": {"preview_url": False, "body": body[:4000]}
    }
    requests.post(WA_API_URL, headers=HEADERS_WA, json=payload, timeout=30)

def send_menu(to_number: str):
    payload = {
        "messaging_product": "whatsapp",
        "to": to_number,
        "type": "interactive",
        "interactive": {
            "type": "button",
            "body": {"text": "Чем могу помочь?"},
            "action": {
                "buttons": [
                    {"type": "reply", "reply": {"id": "book", "title": "Запись на приём"}},
                    {"type": "reply", "reply": {"id": "price", "title": "Прайс"}},
                    {"type": "reply", "reply": {"id": "address", "title": "Адрес"}}
                ]
            }
        }
    }
    requests.post(WA_API_URL, headers=HEADERS_WA, json=payload, timeout=30)

@app.get("/webhook")
def verify():
    if request.args.get("hub.mode") == "subscribe" and request.args.get("hub.verify_token") == VERIFY_TOKEN:
        return request.args.get("hub.challenge"), 200
    return "forbidden", 403

@app.post("/webhook")
def incoming():
    data = request.get_json(force=True, silent=True) or {}
    try:
        entry = data.get("entry",[{}])[0]
        changes = entry.get("changes",[{}])[0]
        value = changes.get("value",{})
        messages = value.get("messages",[])
        if not messages: return "ok",200
        msg = messages[0]
        from_wa = msg.get("from")
        text = msg.get("text",{}).get("body","").strip() if msg.get("type")=="text" else ""
        # Interactive replies
        if msg.get("type")=="interactive":
            reply_id = msg.get("interactive",{}).get("button_reply",{}).get("id")
            if reply_id=="book":
                send_text(from_wa, "Укажите удобный день и время для визита.")
            elif reply_id=="price":
                send_text(from_wa, "Наши цены:\n• Кариес 15-22 тыс тг\n• Пульпит 40-55 тыс тг\n• Удаление от 5 тыс (сред. 12-15 тыс)")
            elif reply_id=="address":
                send_text(from_wa, "Asia Dent — г. Усть-Каменогорск, ул. Космическая 5.\nЧасы работы: 9:00–17:00, выходные ср и чт.")
            return "ok",200
        # Показываем меню по ключевым словам
        if any(k in text.lower() for k in ["запис","приём","прайс","адрес","цена"]):
            send_menu(from_wa)
            return "ok",200
        answer = llm_answer(text)
        send_text(from_wa, answer)
    except Exception as e:
        print("Error:", e)
    return "ok",200

@app.get("/")
def root():
    return jsonify({"ok":True})

if __name__=="__main__":
    app.run(host="0.0.0.0", port=int(os.getenv("PORT",8000)))
