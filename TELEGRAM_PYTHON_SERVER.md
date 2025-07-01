
# Telegram Python Server Setup

Этот документ описывает настройку внешнего Python сервера для работы с Telegram User Bot API.

## Требования

- Python 3.8+
- Telethon библиотека
- FastAPI
- Uvicorn

## Установка

1. Создайте новый проект Python:
```bash
mkdir telegram-server
cd telegram-server
python -m venv venv
source venv/bin/activate  # или venv\Scripts\activate на Windows
```

2. Установите зависимости:
```bash
pip install telethon fastapi uvicorn python-dotenv
```

3. Создайте файл `main.py`:

```python
from fastapi import FastAPI, HTTPException, Depends, Header
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from telethon import TelegramClient
from telethon.errors import PhoneCodeInvalidError, PhoneNumberInvalidError, SessionPasswordNeededError
import os
import json
import asyncio
from typing import Optional
import logging

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Telegram User Bot API", version="1.0.0")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Модели данных
class RequestCodeRequest(BaseModel):
    api_id: int
    api_hash: str
    phone_number: str

class VerifyCodeRequest(BaseModel):
    api_id: int
    api_hash: str
    phone_number: str
    code: str
    phone_code_hash: str

class SendMessageRequest(BaseModel):
    api_id: int
    api_hash: str
    session_string: dict
    message: str
    target_username: Optional[str] = None
    target_phone: Optional[str] = None

class GetStatusRequest(BaseModel):
    api_id: int
    api_hash: str
    session_string: dict

# Аутентификация
async def verify_api_key(authorization: str = Header(...)):
    expected_key = os.getenv('API_KEY', 'default-key')
    if not authorization.startswith('Bearer '):
        raise HTTPException(status_code=401, detail="Invalid authorization header")
    
    token = authorization.replace('Bearer ', '')
    if token != expected_key:
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    return token

# Временное хранилище для phone_code_hash
temp_storage = {}

@app.post("/request_code")
async def request_code(request: RequestCodeRequest, api_key: str = Depends(verify_api_key)):
    try:
        logger.info(f"Requesting code for phone: {request.phone_number}")
        
        client = TelegramClient(
            f"session_{request.phone_number}", 
            request.api_id, 
            request.api_hash
        )
        
        await client.connect()
        
        # Отправляем код
        sent_code = await client.send_code_request(request.phone_number)
        
        # Сохраняем phone_code_hash для последующего использования
        temp_storage[request.phone_number] = {
            'phone_code_hash': sent_code.phone_code_hash,
            'client_session': client.session.save()
        }
        
        await client.disconnect()
        
        return {
            "success": True,
            "phone_code_hash": sent_code.phone_code_hash,
            "message": "Verification code sent"
        }
        
    except PhoneNumberInvalidError:
        raise HTTPException(status_code=400, detail="Invalid phone number")
    except Exception as e:
        logger.error(f"Error requesting code: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/verify_code")
async def verify_code(request: VerifyCodeRequest, api_key: str = Depends(verify_api_key)):
    try:
        logger.info(f"Verifying code for phone: {request.phone_number}")
        
        # Восстанавливаем клиент из сохраненной сессии
        temp_data = temp_storage.get(request.phone_number)
        if not temp_data:
            raise HTTPException(status_code=400, detail="No pending code request found")
        
        client = TelegramClient(
            f"session_{request.phone_number}", 
            request.api_id, 
            request.api_hash
        )
        
        await client.connect()
        
        # Авторизуемся с кодом
        await client.sign_in(
            phone=request.phone_number, 
            code=request.code,
            phone_code_hash=request.phone_code_hash
        )
        
        # Получаем информацию о пользователе
        me = await client.get_me()
        
        # Сохраняем сессию
        session_string = client.session.save()
        
        await client.disconnect()
        
        # Очищаем временные данные
        temp_storage.pop(request.phone_number, None)
        
        return {
            "success": True,
            "data": {
                "session_string": session_string,
                "user_info": {
                    "id": str(me.id),
                    "username": me.username or "",
                    "first_name": me.first_name or "",
                    "last_name": me.last_name or ""
                }
            }
        }
        
    except PhoneCodeInvalidError:
        raise HTTPException(status_code=400, detail="Invalid verification code")
    except SessionPasswordNeededError:
        raise HTTPException(status_code=400, detail="Two-factor authentication required")
    except Exception as e:
        logger.error(f"Error verifying code: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/send_message")
async def send_message(request: SendMessageRequest, api_key: str = Depends(verify_api_key)):
    try:
        logger.info(f"Sending message to {request.target_username or request.target_phone}")
        
        client = TelegramClient(
            f"temp_session", 
            request.api_id, 
            request.api_hash
        )
        
        # Загружаем сессию
        client.session.load(request.session_string)
        
        await client.connect()
        
        # Определяем получателя
        target = request.target_username or request.target_phone
        
        # Отправляем сообщение
        message = await client.send_message(target, request.message)
        
        await client.disconnect()
        
        return {
            "success": True,
            "message_id": str(message.id)
        }
        
    except Exception as e:
        logger.error(f"Error sending message: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/get_status")
async def get_status(request: GetStatusRequest, api_key: str = Depends(verify_api_key)):
    try:
        client = TelegramClient(
            f"temp_session", 
            request.api_id, 
            request.api_hash
        )
        
        # Загружаем сессию
        client.session.load(request.session_string)
        
        await client.connect()
        
        # Получаем информацию о пользователе
        me = await client.get_me()
        
        await client.disconnect()
        
        return {
            "success": True,
            "data": {
                "id": str(me.id),
                "username": me.username or "",
                "first_name": me.first_name or "",
                "last_name": me.last_name or ""
            }
        }
        
    except Exception as e:
        logger.error(f"Error getting status: {e}")
        return {
            "success": False,
            "error": str(e)
        }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "telegram-server"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

4. Создайте файл `.env`:
```
API_KEY=your-secure-api-key-here
```

5. Запустите сервер:
```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

## Развертывание

### Railway (рекомендуется)

1. Создайте аккаунт на [Railway](https://railway.app)
2. Подключите GitHub репозиторий
3. Добавьте переменную окружения `API_KEY`
4. Railway автоматически развернет сервер

### DigitalOcean App Platform

1. Создайте аккаунт на DigitalOcean
2. Используйте App Platform для развертывания
3. Настройте переменные окружения

### Docker (для любого облачного провайдера)

Создайте `Dockerfile`:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Создайте `requirements.txt`:
```
telethon==1.34.0
fastapi==0.104.1
uvicorn==0.24.0
python-dotenv==1.0.0
```

## Настройка в Supabase

После развертывания Python сервера, добавьте в Supabase секреты:

1. `PYTHON_TELEGRAM_SERVER_URL` - URL вашего Python сервера (например: https://your-app.railway.app)
2. `PYTHON_SERVER_API_KEY` - API ключ для доступа к серверу

## Тестирование

Проверьте работу сервера:
```bash
curl -X GET https://your-server-url/health
```

Должен вернуть:
```json
{"status": "healthy", "service": "telegram-server"}
```
