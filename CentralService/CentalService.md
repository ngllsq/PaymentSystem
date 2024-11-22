# Импорт библиотек
__fastapi:__ Фреймворк для создания веб-сервисов.  
__HTTPException:__  Класс для обработки исключений в FastAPI.  
__BaseModel:__  Класс из Pydantic для валидации данных.  
__Optional:__  Для указания необязательных параметров.  
__KafkaProducer:__  Для отправки сообщений в Kafka.  
__json:__  Для работы с JSON-данными.  
__redis:__  Для работы с Redis.  
__asyncpg:__  Асинхронный драйвер для PostgreSQL.  

```Python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
from kafka import KafkaProducer
import json
import redis
import asyncpg
```
# Создание приложения FastAPI и устанавление соединения с Redis-сервером
```Python
red = redis.Redis(host='localhost', port= 6379)
```

# Функция получения баланса из базы данных
```Python
async def get_user_balance_from_db(user_id: int, currency: str, pool: asyncpg.pool.Pool):
    async with pool.acquire() as conn:
        row = await conn.fetchrow("""
            SELECT balance
            FROM users
            WHERE id = $1 AND currency = $2;
        """, user_id, currency)
        if row:
            return row['balance']
        else:
            return 0.0
```

# Функция проверки баланса
```Python
async def check_user_balance(sender_id: int, amount: float, currency: str):
    # Ключ для кэша Redis
    balance_key = f"balance:{sender_id}:{currency}"
    # Получаем баланс из Redis
    balance = red.get(balance_key)
    # Если баланс не найден в Redis
    if balance is None:
        # Получаем баланс из базы данных
        balance = await get_user_balance_from_db(sender_id, currency)
        # Сохраняем баланс в Redis
        red.set(balance_key, balance)
    # Проверяем, достаточно ли средств
    return float(balance) >= amount
```

## Инициализация KafkaProducer
```Python
producer = KafkaProducer(bootstrap_servers=['localhost:9092'], value_serializer=lambda v: json.dumps(v).encode('utf-8'))
```
## Модель данных для запроса на перевод
```Python
class TransferRequest(BaseModel):
    sender_id: int  # ID отправителя
    receiver_id: int # ID получателя
    amount: float  # Сумма перевода
    currency: str  # Валюта
    description: Optional[str] = None
```

## Обработчик запроса на перевод
```Python
@app.post("/transfers")
async def initiate_transfer(request: TransferRequest):
    # Проверяем баланс отправителя
    if not await check_user_balance(request.sender_id, request.amount, request.currency):
        raise HTTPException(status_code=400, detail="Недостаточно средств")
```

## Создание сообщения для Kafka
```Python
    message = {
        "sender_id": request.sender_id,
        "receiver_id": request.receiver_id,
        "amount": request.amount,
        "currency": request.currency,
        "description": request.description,
        "transaction_id": "some_unique_transaction_id" # Сгенерировать уникальный ID
    }
```
## Отправка сообщения в Kafka
```Python
    try:
        producer.send('transfer_topic', message)
        return {"status": "success", "transaction_id": message["transaction_id"]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Ошибка отправки сообщения в Kafka: {e}")
```
