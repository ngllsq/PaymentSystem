## Импорт библиотек
```Python
from kafka import KafkaConsumer
import json
import asyncpg
import logging
import redis
```

## Настройка логирования
```Python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)
```

## Подключение к Kafka и Redis
```Python
consumer = KafkaConsumer('transfer_topic', bootstrap_servers=['localhost:9092'],
                         value_deserializer=lambda v: json.loads(v))
redis_client = redis.Redis(host='localhost', port=6379)
redis_ttl = 60
```

## Получение баланса пользователя из Redis
```Python
async def get_user_balance(user_id, currency, pool):
    redis_key = f"balance:{user_id}:{currency}"
    balance = redis_client.get(redis_key)
    if balance:
        return float(balance)

    try:
        async with pool.acquire() as conn:
            row = await conn.fetchrow("""
                SELECT balance FROM users WHERE id = $1 AND currency = $2
            """, user_id, currency)
            if row:
                balance = row['balance']
                redis_client.set(redis_key, balance, ex=redis_ttl)
                return float(balance)
            else:
                return 0.0
    except Exception as e:
        logger.error(f"Ошибка получения баланса пользователя {user_id}: {e}")
        return None

```

## Обрабатка сообщения о переводе
```Python
async def process_transaction(message, pool):
    try:
        transaction_id = message['transaction_id']
        sender_id = message['sender_id']
        receiver_id = message['receiver_id']
        amount = message['amount']
        currency = message['currency']
        description = message.get('description', '')
        original_amount = message.get('original_amount')
        original_currency = message.get('original_currency')

        sender_balance = await get_user_balance(sender_id, currency, pool)
        receiver_balance = await get_user_balance(receiver_id, currency, pool)


        if sender_balance is None or receiver_balance is None:
            raise Exception("Ошибка получения баланса пользователя")
        if sender_balance < amount:
            raise Exception("Недостаточно средств у отправителя")


        async with pool.transaction():
            await pool.execute("""
                INSERT INTO transactions (transaction_id, sender_id, receiver_id, amount, currency, description, original_amount, original_currency, created_at)
                VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
            """, transaction_id, sender_id, receiver_id, amount, currency, description, original_amount, original_currency)

            await pool.execute("""
                UPDATE users SET balance = balance - $1 WHERE id = $2
            """, amount, sender_id)
            await pool.execute("""
                UPDATE users SET balance = balance + $1 WHERE id = $2
            """, amount, receiver_id)

            redis_client.delete(f"balance:{sender_id}:{currency}")
            redis_client.delete(f"balance:{receiver_id}:{currency}")

        logger.info(f"Транзакция {transaction_id} обработана успешно.")

    except asyncpg.exceptions.UniqueViolationError:
        logger.warning(f"Транзакция {transaction_id} уже существует. Пропускаем.")
    except Exception as e:
        logger.error(f"Ошибка обработки транзакции {transaction_id}: {e}")

```

## Обработка сообщения из Kafka
```Python
async def consume_and_process(pool):
    async for message in consumer:
        await process_transaction(message.value, pool)
```
