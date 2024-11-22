## Импорт библиотек
```Python
from kafka import KafkaConsumer
import json
import logging
import yagmail
import redis
import asyncio
```
__yagmail__: предоставляет простой в использовании интерфейс для отправки электронных писем с помощью Gmail.

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
redis_ttl = 3600  
```

## Отправка уведомлений по электронной почте
```Python
async def send_email_notification(message):
    transaction_id = message['transaction_id']
    redis_key = f"notification:{transaction_id}"

    if redis_client.exists(redis_key): 
        logger.info(f"Уведомление о транзакции {transaction_id} уже отправлено (из кэша).")
        return

    try:
        sender_id = message['sender_id']
        receiver_id = message['receiver_id']
        amount = message['amount']
        currency = message['currency']
        description = message.get('description', '')

        sender_name = f"User {sender_id}"  
        receiver_name = f"User {receiver_id}" 

        body = f"""
        Уведомление о транзакции:

        ID транзакции: {transaction_id}
        Отправитель: {sender_name}
        Получатель: {receiver_name}
        Сумма: {amount} {currency}
        Описание: {description}
        """


        yag = yagmail.SMTP("email@gmail.com", "password") 

        yag.send(to=["receiver_email@example.com"],  
                 subject=f"Уведомление о транзакции {transaction_id}",
                 contents=body)

        redis_client.set(redis_key, 1, ex=redis_ttl) 
        logger.info(f"Уведомление об успешном переводе {transaction_id} отправлено.")

    except Exception as e:
        logger.error(f"Ошибка отправки уведомления о транзакции {transaction_id}: {e}")
```

## Получение сообщения из Kafka и отправка уведомлений"""
```Python
async def consume_and_notify():
    async for message in consumer:
        await send_email_notification(message.value)
```
