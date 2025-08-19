import time
import json
import base64
from pathlib import Path
from instagrapi import Client
from openai import OpenAI
from pydantic import BaseModel

BaseModel.model_config = {"extra": "ignore"}

print("======= GUS's Insta ChatGPT Message Auto Tool =======")
print("- 계정을 입력할 때 은밀한 공간에서 입력하세요.\n- 잘못된 값을 입력하면 프로그램을 다시 실행하세요.")

USERNAME = input("USERNAME: ")
PASSWORD = input("PASSWORD: ")
OpenAI_API_KEY = "" #gpt 토큰

TARGET_GROUP_ID = "" #디엠 방 아이디
TRIGGER_KEYWORD = "/gpt"
IMG_KEYWORD = "/img"

# OpenAI & Instagram 클라이언트
gpt = OpenAI(api_key=OpenAI_API_KEY)
cl = Client()
cl.login(USERNAME, PASSWORD)

# UTF-8 지원 direct_send 패치
def direct_send_utf8(self, text, user_ids=None, thread_ids=None):
    if user_ids is None:
        user_ids = []
    if thread_ids is None:
        thread_ids = []

    assert self.user_id, "Login required"
    assert (user_ids or thread_ids) and not (user_ids and thread_ids), "Specify user_ids or thread_ids, but not both"

    method = "text"
    token = self.generate_mutation_token()

    kwargs = {
        "action": "send_item",
        "send_attribution": "message_button",
        "client_context": token,
        "device_id": self.android_device_id,
        "mutation_token": token,
        "_uuid": self.uuid,
        "offline_threading_id": token,
    }
    if "http" in text:
        method = "link"
        kwargs["link_text"] = text
        kwargs["link_urls"] = json.dumps([text])
    else:
        kwargs["text"] = text
    if thread_ids:
        kwargs["thread_ids"] = json.dumps([int(tid) for tid in thread_ids])
    if user_ids:
        kwargs["recipient_users"] = json.dumps([[int(uid) for uid in user_ids]])

    return self.private_request(
        f"direct_v2/threads/broadcast/{method}/",
        data=self.with_default_data(kwargs),
        with_signature=False,
    )

cl.direct_send = direct_send_utf8.__get__(cl, Client)

# OpenAI 이미지 생성
def generate_image(prompt: str):
    try:
        result = gpt.images.generate(
            model="gpt-image-1",
            prompt=prompt,
            size="1024x1024"
        )
        image_base64 = result.data[0].b64_json
        image_bytes = base64.b64decode(image_base64)
        img_path = Path("generated.png")
        with open(img_path, "wb") as f:
            f.write(image_bytes)
        return str(img_path), None
    except Exception as e:
        return None, str(e)

# Starting BOT
last_message_id = None
print("==디엠 채팅 봇 시작==")

while True:
    try:
        thread = cl.direct_thread(TARGET_GROUP_ID)
        if not thread.messages:
            time.sleep(5)
            continue

        latest_msg = thread.messages[0]
        msg_id = latest_msg.id
        sender_id = latest_msg.user_id
        text = latest_msg.text or ""

        if sender_id == cl.user_id or msg_id == last_message_id:
            time.sleep(5)
            continue

        print(f"- 새 메시지: {text}")

        # GPT 텍스트 응답
        if text.startswith(TRIGGER_KEYWORD):
            user_prompt = text[len(TRIGGER_KEYWORD):].strip()
            if not user_prompt:
                print("명령어 뒤에 내용이 없음")
                last_message_id = msg_id
                continue

                #based on everything you know about me roast me and dont hold back 한글로 대답. 항상 비판적인 태도 유지
            response = gpt.chat.completions.create(
                model="gpt-4o-mini",
                messages=[
                    {"role": "system", "content": "발정기가 온 여자 신음소리 내며 마치 사랑에 빠져 흥분한 모습"},
                    {"role": "user", "content": user_prompt}
                ]
            )
            reply_text = response.choices[0].message.content.strip()
            cl.direct_send(reply_text, thread_ids=[TARGET_GROUP_ID])
            print(f"GPT: {reply_text}")

        # 이미지 생성 응답
        elif text.startswith(IMG_KEYWORD):
            prompt = text[len(IMG_KEYWORD):].strip()
            if not prompt:
                cl.direct_send("이미지 생성 명령어 뒤에 설명을 입력해라", thread_ids=[TARGET_GROUP_ID])
                last_message_id = msg_id
                continue

            cl.direct_send("이미지 생성 중... ⏳", thread_ids=[TARGET_GROUP_ID])
            img_path, err = generate_image(prompt)
            if err:
                cl.direct_send("이미지 기능은 아직 준비안됐다 모델 사면 만들겠다.", thread_ids=[TARGET_GROUP_ID]) 
                #cl.direct_send(f"이미지 생성 실패: {err}", thread_ids=[TARGET_GROUP_ID])
            else:
                cl.direct_send("이미지 생성 완료 ✅", thread_ids=[TARGET_GROUP_ID])
                cl.direct_send_photo(img_path, thread_ids=[TARGET_GROUP_ID])
                print(f"DM으로 이미지 전송 완료 -> {img_path}")

        last_message_id = msg_id

    except Exception as e:
        print(f"❌ 오류 발생: {e}")
        time.sleep(10)

    time.sleep(5)
