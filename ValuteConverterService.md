## Импорт библиотек
```Python
from fastapi import FastAPI, HTTPException
from apscheduler.schedulers.background import BackgroundScheduler
import requests
import json
import redis
import logging
```

## Настройка логирования.  Уровень логирования INFO, формат сообщения: дата-время - уровень - сообщение
```Python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)
```

## Создание FastAPI приложение и подключение к Redis
```Python
app = FastAPI() 
redis_client = redis.Redis(host='localhost', port=6379)
```
## Получаение курса валют в реальном времени
```Python
def CurrencyExchangeRate(from_currency, to_currency, api_key):

    base_url = "https://www.alphavantage.co/query"
    params = {
        "function": "CURRENCY_EXCHANGE_RATE",
        "from_currency": from_currency,
        "to_currency": to_currency,
        "apikey": api_key
    }

    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status() 
        data = response.json() 
        if "Error Message" in data: 
            raise Exception(f"Ошибка Alpha Vantage API: {data['Error Message']}")
        exchange_rate = data["Realtime Currency Exchange Rate"]["5. Exchange Rate"] 
        return float(exchange_rate)
    except requests.exceptions.RequestException as e:
        logger.error(f"Ошибка сети: {e}") 
        return None
    except KeyError as e:
        logger.error(f"Неверный ответ API: Отсутствует ключ {e}") 
        return None
    except Exception as e:
        logger.error(f"Произошла ошибка: {e}") 
        return None
```

## Обновление курса валют и сохранение их в Redis
```Python
async def update_exchange_rates(api_key):
    try:
        rates = {}
        rates["USD_RUB"] = CurrencyExchangeRate("USD", "RUB", api_key)
        rates["EUR_RUB"] = CurrencyExchangeRate("EUR", "RUB", api_key)
        rates["GBP_RUB"] = CurrencyExchangeRate("GBP", "RUB", api_key)

        redis_client.set('exchange_rates', json.dumps(rates))
        logger.info("Курсы валют успешно обновлены.")
    except Exception as e:
        logger.error(f"Ошибка обновления курсов валют: {e}")

@app.get("/exchange_rates/{from_currency}_{to_currency}")
async def convert_currency(from_currency_to_currency: str):
    rates_json = redis_client.get('exchange_rates')
    if rates_json is None: 
        try:
            await update_exchange_rates('YOUR_API_KEY')
        except Exception as e:
            logger.error(f"Ошибка обновления курсов валют при первом запросе: {e}")
            raise HTTPException(status_code=503, detail="Сервис обмена валют недоступен")

        rates_json = redis_client.get('exchange_rates')
        if rates_json is None:
            raise HTTPException(status_code=503, detail="Сервис обмена валют недоступен") 

    try:
        rates = json.loads(rates_json) 
        rate = rates.get(from_currency_to_currency) 
        if rate is None:
            return {"error": "Курс валюты не найден"} 
        return {"exchange_rate": rate} 
    except json.JSONDecodeError:
        logger.error("Неверные данные курсов валют в Redis") 
        raise HTTPException(status_code=500, detail="Внутренняя ошибка сервера")
```

## Конвертация суммы перевода в указанную валюту
```Python
@app.post("/convert_transfer/{transaction_id}/{to_currency}")
async def convert_transfer(transaction_id: str, to_currency: str):
    transfer_data_json = redis_client.get(f"transfer:{transaction_id}") 
    if transfer_data_json is None:
        return {"error": "Данные о переводе не найдены"} 

    try:
        transfer_data = json.loads(transfer_data_json) 
        from_currency = transfer_data.get("currency") 
        amount = transfer_data.get("amount") 

        if from_currency is None or amount is None:
            return {"error": "Неполные данные о переводе"} 

        rates_json = redis_client.get('exchange_rates') 
        if rates_json is None:
            await update_exchange_rates('YOUR_API_KEY') 
            rates_json = redis_client.get('exchange_rates')
            if rates_json is None:
                return {"error": "Курсы валют недоступны"} 

        rates = json.loads(rates_json) 
        from_rate = rates.get(from_currency) 
        to_rate = rates.get(to_currency) 

        if from_rate is None or to_rate is None:
            return {"error": "Неверный код валюты"} 

        converted_amount = amount  (to_rate / from_rate) 
        return {"transaction_id": transaction_id, "from_currency": from_currency, "to_currency": to_currency, "amount": amount, "converted_amount": converted_amount} # Возвращаем результат

    except (json.JSONDecodeError, KeyError) as e:
        logger.error(f"Ошибка обработки данных: {e}") 
        return {"error": f"Ошибка обработки данных: {e}"} 
```

## Создание планировщика задач для обновления курса валют
```Python
scheduler = BackgroundScheduler() 
scheduler.add_job(update_exchange_rates, 'interval', hours=4, kwargs={'api_key': 'YOUR_API_KEY'}) # Задаем задачу на обновление каждые 4 часа.  Замените YOUR_API_KEY на ваш API ключ!
scheduler.start()
```
