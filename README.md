# Автоматическая система торговли криптовалютой

## Краткая суть проекта 
Реализовать алгоритм с ИИ торговли на Binance + для удобства вывод в тг информации о сделках и прочие приколы.

![image](https://user-images.githubusercontent.com/113302248/211331783-14212b2f-33bd-4aa3-95be-9a0a84dd8d18.png)

### Get data
#### Training 

У Binance есть API(со своей документацией). Было решено использовать данные начиная с "16 Jun,2022". Нам нужны только ```
['open_time','open', 'high', 'low', 'close', 'volume','close_time', 'qav','num_trades','taker_base_vol','taker_quote_vol', 'ignore']``` поэтому остальное дропаем. 

```python
import pandas as pd
from binance.client import Client
import datetime as dt

api_key = ''
api_secret = ''
client = Client(api_key, api_secret)
symbol = "BTCUSDT"
interval='1m'
Client.KLINE_INTERVAL_15MINUTE 
klines = client.get_historical_klines(symbol, interval, "16 Jun,2022")
data = pd.DataFrame(klines)

data.columns = ['open_time','open', 'high', 'low', 'close', 'volume','close_time', 'qav','num_trades','taker_base_vol','taker_quote_vol', 'ignore']

data.index = [dt.datetime.fromtimestamp(x/1000.0) for x in data.close_time]
data.to_csv(symbol+'_2022_Jun.csv', index = None, header=True)
```
#### Realtime data

Это нам понадобится, для того чтобы предсказывать в релаьном времени точку входа в сделку. 
```python
cl = Spot()
r = cl.klines('BTCUSDT', '1m', limit =  500)
df = DataFrame(r).iloc[:, :9]
df = df.drop([6, 7], axis = 1)
df.columns = ('timestamp', 'Open', 'High', 'Low', 'Close', 'Volume', 'Count')
```
Тут ничего интересного. Можно отметить только момент с ```limit```, он не больше 1000. То есть, если длина последовательности для LSTM больше 1000, то надо придумывать махинации, а учитывая что индикаторы 'съедают'(например индикаторы, которым надо много исторических данных, ну или nan на нули менять или по умному заполнять, но кажется это лажа и проще просто откинуть строки с nan). Ограничений на количество запросов на скачивание я не получал так что пока опустим момент с ограничениями. 

### Preprocessing data
После того как мы получили данные из прошлого пункта нам необходимо проверить их на пробелы и наны. 
```(df_train.index[1:]-df_train.index[:-1]).value_counts()``` эта штука проверяет на наличие пробелов. Почему они есть?
Вообще говоря зависит от даты с которой скачать тренировочные данные. Если качать с июля 2022 то их не будет, но если скачать с 
июля 2021 они будут. 
```data = data.reindex(range(data.index[0],data.index[-1]+60,60),method='pad')``` - это решает проблему. 
Также сосдаем колонку 'Target' который определяем так:

```python
for i in range (len(df['Open'])):
    open_price= df['Open'][i]
    max = df['High'][i:i+15:].max()
    if max >= open_price + 25:
        df['Target'][i] = 1
    else: 
        df['Target'][i] = 0
```
Логика такая. Если в близжайшие 15 минут есть high_price > 25, в этом случае мы бы заработали денег. Почему + 25?
Во-первых потому что сейчас даже + 5 за 250 минут встречается редко а значит выход из модели будет равен 0(не входим  сделку), и так далее. Почему не + 5? Тогда таргетов с 1 >> чем с 0. А значит нужно думать над перебалансировкой. Так и было, я думал что в данном проекте данный паттерн реализуем, но чисто по-человечески: если модель предсказывает + 5 то разве это то что хотя бы в теории может заработать денег? Так что было принято решение оставить + 25. 

Дальше по файлу всякие типичные штучки и ничего интересного. 

### Processing data

Необходимо получить индикаторы из того, что есть. 

Какие индикаторы используем?

Вот такие:

```python
# Commodity Channel Index 
def CCI(data, ndays): 
    TP = (data['High'] + data['Low'] + data['Close']) / 3 
    CCI = pd.Series((TP - TP.rolling(ndays).mean()) / (0.015 * TP.rolling(ndays).std()), name = 'CCI') 
    data = data.join(CCI) 
    return data

# Ease of Movement 
def EVM(data, ndays): 
    dm = ((data['High'] + data['Low'])/2) - ((data['High'].shift(1) + data['Low'].shift(1))/2)
    br = (data['Volume'] / 100000000) / ((data['High'] - data['Low']))
    EVM = dm / br 
    EVM_MA = pd.Series(EVM.rolling(ndays).mean(), name = 'EVM') 
    data = data.join(EVM_MA) 
    return data 

# Simple Moving Average 
def SMA(data, ndays): 
    SMA = pd.Series(data['Close'].rolling(ndays).mean(), name = 'SMA') 
    data = data.join(SMA) 
    return data

# Exponentially-weighted Moving Average 
def EWMA(data, ndays): 
    EMA = pd.Series(data['Close'].ewm(span = ndays, min_periods = ndays - 1).mean(), 
                    name = 'EWMA_' + str(ndays)) 
    data = data.join(EMA) 
    return data

def BBANDS(data, window):
    MA = data.Close.rolling(window).mean()
    SD = data.Close.rolling(window).std()
    data['UpperBB'] = MA + (2 * SD) 
    data['LowerBB'] = MA - (2 * SD)
    return data

# Force Index 
def ForceIndex(data, ndays): 
    FI = pd.Series(data['Close'].diff(ndays) * data['Volume'], name = 'ForceIndex') 
    data = data.join(FI) 
    return data 

# Rate of Change (ROC)
def ROC(data,n):
    N = data['Close'].diff(n)
    D = data['Close'].shift(n)
    ROC = pd.Series(N/D,name='Rate of Change')
    data = data.join(ROC)
    return data 
```

Почему их? Потому что каждый из них выглядит логично. По сути это комбинации стратегий торговли. В файле тренировки модели есть графики с этими индикаторами. 

#### Нормализация данных. 

Нормализуем данные. Это нужно для того чтобы модель могла понять, что делать если вдруг цена на входе упадет на 100 и станет равная условно 14900, в тренировачном наборе данных такого может не быть поэтому номарлизуем чтоб предотвратить это. Этот  

```prolog
d = preprocessing.normalize(X_train, axis=0)
X_train = pd.DataFrame(d, columns=X_train.columns)
```

#### Выбор библиотеки

Сначла я пытался все делать в tf, потом в  pytorch. Поиспользовав то и то выбор неожиданно пал на pytorch lightning. 
C Pytorch lightning цикл обучения мне стал более понятным + всякие приколы с графиками + lightning_logs + checkpoint_callback, все так просто. 

Каких-то недостатков я не заметил. Кроме того что в интернете по этой библиотеке меньше примеров. В целом, юзабельно.

#### Постановка задачи 

Регрессия или мультиклассификация или бинарная или что.

Тут все хаотично.

Этап 1. 

Так как сначало планировалось решить соревнование с kaggle https://www.kaggle.com/competitions/g-research-crypto-forecasting, то и проекте планировалось предсказывать лог доходность, а занчит и регрессию . Позже стало ясно, что не понятно где использовать ее в алгоритме торговли. Тут можно прикрутить её юзабельность, но есть и по лучше идеи.  

Этап 2.

Бинарная классификация. Предсказывать вход в потанциадльную сделку. Тут я решил использовать CrossEntropyLoss(), почему не BCEWithLogitsLoss()? 

Этап 3.

Потому что я хотел проверить паттерн нейтрального класса в данной задаче. Он не сработал. Модель не улучшилась, а значит бинарная классификация то, что нам нужно. А
CrossEntropyLoss() можно использовать и при ```n_class = 2```

Итог: хотим посторить модель, которая предсказывает потенциально выигрышный вход в сделку. 

#### Выбор модели

До этого проекта я не работал с временными рядами, а в процессеобучения в deep learning school от МФТИ + mlcourse.ai не уделял должного внимания временным рядам. Но я всегда знал про lstm и что они хорошо справляются с времеными рядами. 

! Важно отметить, что не получилось релаизовать в данном разделе. Я подумал, что круто было бы реализвоать LSTM + ССА или для предобработки последовательности использовать гауссовские процессы. Но потратив недельку понял, что  времени не хватит. 
Поэтому ограничились LSTM. 

#### Обучение

С помощью бибилиотеки optuna. На раннем этапе проекта было выявленнf примерная оптимальная длина последовательности. ```{'hidden_size': 118, 'num_layers': 4, 'dropout_prob': 0.3622464677409225, 'learning_rate': 0.00022486809854740082, 'batch_size': 632, 'seq_length': 69}. Best is trial 4 with value: 5.5725564531409566e-06.``` Но на самом деле это неважно, так как очевидно что чем больше длина последовательности тем лучше(в данном проекте). Если брать длину последовательности меньше 250 то ничего не выйдет так как некоторые индикаторы основываются только на 250 данных ДО. Поэтому длина последовательности, передоваемая в модель, равна 250(то есть примерно 4 часа), что для скальпинга вполне логично.  Почему тогда optuna выдала 69? Так как финальный accuracy 0.508, а лосс не падает, можно сделать вывод что на вход модели подаются хаотичные данные, и задача не решаема ИИ. Но как показала практика модель способна находить точки входа. Может это связано с тем, что в тренировачном наборе данных много примеров с target = 1, а на практике сейчас за 250 минут обычно от 0 до 20 точек входа для ИИ.


![image](https://user-images.githubusercontent.com/113302248/211375830-4ad1e8ce-a5c6-4f1d-af3c-78cefbb3f7e6.png)

![image](https://user-images.githubusercontent.com/113302248/211368510-6374313d-f282-4808-bb04-6467b57d33b4.png)

![image](https://user-images.githubusercontent.com/113302248/211375706-eb91384b-c20c-4611-8bdb-d1b29338f51b.png)

Может я что-то упустил но в целом обучение модели закончилось. Если честно я думал, что это все займет недельку максимум две, так как я предствалял что куда и как делать, но оказалось, что это все куда сложнее, чем просто взять готовый датасет и обучить на нем нейронку.
Для того чтобы это все сделать понадобилось очень много времени. Один только pytorch lightning + обучение больше месяца. 

Идеи которые можно было бы реализовать в будущем: 
- подумать над умной нармализацией
-  использовать ансамбли моделей
-  использовать каскады, например моедль которая по новостям предсказывает куда пойдет цена -> еще какая-то модель->...->...->...-> наша модель-> финальный предикт
-  подумать над перебалансировкой. Как это повлияет на реальную практику. 
### Алгоритм торговли.

Раз в минуту с binance запрашиваем 500 строк(минут) новейших данных. Из которых 250 съедают индикаторы, а другие 250 идут на предикт в модель. Там происходит обработка 
данных таким же образом как и перед тренировкой модоли. То есть мы создаем мини датасет для того, чтобы модель предсказала 0 или 1. Все это происходит в функции ```trading``` в файле ```consumer.py```. Как только модель предсказала 1, создается запрос на покупку ордера по текущей цене, он исполняется сразу же. 
```python
def place_order(order_type):
        if(order_type == 'buy'):
            client.create_order(symbol=SYMBOL, side='buy', type='MARKET', quantity= QNTY)
        else:
            client.create_order(symbol=SYMBOL, side='sell', type='MARKET', quantity= QNTY)
            
 ...
 if (predictions[0] == 1) and (buy == False):
                    place_order('buy')
                    price_buy = get_data()[-1]
                    send(f' Покупаем\n Цена покупки: {price_buy}')
                    buy = True
                    sell = False
                    oredr_sell = price_buy
```
Дальше начинается сам алгоритм. Обычный скальпинг. Проверяем каждую секунду цену. Если она выросла то при минус одном пнутке продаем, если она ни разу не стала выше той, за которую мы купили, то рано или поздно сработает стоп-лосс 

```python
elif (price_current < price_buy  - 15) and (price_current < oredr_sell):
                        place_order('sell')
                        send(f'Продаём GG: {(100*price_current)/price_buy}\n Цена продажи: {price_current}')
                        number_of_trades[0] += 1
                        buy = False
                        sell = True
```
Монжо выставить стоп-лосс по времени. Но очевидно, что и то и то можно улучшить и объеденить. Например сделать продажу при увеличении цены или при уменьшее не линейно, то есть продовать частями. Но это выходит за рамки данного проекта. 

В целом на этом все, что касается начинки этого проекта. Без деплоя, это работает отлично. Достаточно скачать ```___init___.py``` вставить api tg bot + id пользователя, создать config с ```api_key, api_secret``` и запустить прогру. 
Какие плюсы:
- бот работает без проблем 24 на 7, если не выключать программу. 
- если вдруг что-то не так на рынке и хочется отключить бота, то достаточно прекратить работу ```___init___.py```, все завершается без перебоев со стороны binance.
Какие минусы: 
- Если далеко от компьютера с программой и срочно нужно отключить алгоритм торговли, то нужно бежать к нему либо зайти в преложение binnace и поменять api ключи. 

## Деплой 
Деплой было решено делать в виде тг бота. Сразу проблема. Никакой дурак не скинет свой ```api_key и api_secret``` какому-то боту. Этот проект делается для нас и друзей так, что это +- ок.  В качетсве связующего звена между тг ботом и алгоритом торголви был выбран rabbitmq. Мы пытались это реализовать с помощью RPC. Но возникла проблема.  Функция  ```trading``` в  ```consumer.py``` может выполнять бескончено, например если пользователь укажет количество сделок = очень много. Но тогда rabbitmq  первет связь ```[error] <0.668.0> missed heartbeats from client, timeout: 60s``` это фиксится в конфиге:
```
[{rabbit, 
    [{heartbeat, 0}]
    }].
```
Но это не работает поэтому было принято решение обойтись без RPC и реализовать просто ```producer -> consumer ```. 
Что касается message_handler. В ```bot.py``` ипсользуеются состояния, поэтому ```storage=MemoryStorage()```, не redis так как хранить в нем нечего + нагрузка на безопастность. Самое важно происходит тут:
```python
async def num_of_trades(state:FSMContext ,id: int, message: str, api_key: str, api_secret: str, num_trades: int, stop: bool) -> None:
    binance_rpc = BinanceClient()
    await binance_rpc.call(message, id, api_key, api_secret, num_trades, stop)
    await state.finish()
    
@dp.message_handler(state=Gen.wait_for_input_num_of_trades)
async def answer_on_input_num_of_trades(message: types.Message, state: FSMContext):
    await state.finish()
    await Gen.wait_for_answer.set()
    await num_of_trades(state, message.from_user.id, message = 'BTCUSDT', api_key= api_key, api_secret=api_secret, num_trades = message['text'], stop=False)
```
Тут пользователь отправляет количество сделок, которые совершит бот. После чего ``` consumer ``` конектится к binence и начинает торговать, переодически сообщая ползователю о сделках. К сожалению функцию принудительной остановки торговли не получилось реализовать, некий /stop. 

### Docker 
Всего два docker. Один для бота другой для консьюмера(сервера). Тут ничего необычного. requirements.txt Загружаем -> устанавливаем все библиотеки. --no-cache-dir Для того чтобы докер образ был как можно меньше. 

```
FROM python:3.8

WORKDIR /bot/

ADD ./requirements.txt /bot/requirements.txt
RUN apt-get update
RUN pip install --no-cache-dir -r ./requirements.txt

ADD . /bot/
CMD ["python", "bot.py"]
```
Тут тоже все просто. Единственное, установка torch необходима отдельной командой. Так как в противном случае он установит не ту подборку что надо. Тут CPU просто потму что он меньше весит. Но в кончено если деплоить окончательно на сервак стоит поставить cuda gpu так как будет работать в раз 100 быстрее (выполнять предикт). + Загружаем модельку ранее обученую.  
```
FROM python:3.8 as builder

WORKDIR /server/

ADD ./requirements.txt /server/requirements.txt
ADD ./model_weights_epochs_500.pth /server/model_weights_epochs_500.pth
RUN apt-get update 
RUN pip install --no-cache-dir torch==1.12.1+cpu torchvision==0.13.1+cpu torchaudio==0.12.1 --extra-index-url https://download.pytorch.org/whl/cpu
RUN pip install --no-cache-dir -r ./requirements.txt

ADD . /server/
CMD ["python", "consumer.py"]
```
### Docker compose
Я ранее ни разу не пытался написать что-то подобное. Данный синтаксис мне был не понятен, но через время стал более менее понятен. Первым делом обозначаем с какой версией python мы работаем. 3.7 - надежный выбор. Далее обозначаем все сервисы. Раньше их было 4. Но как не крути redis пока в данном проекте не нужен. 
- rabbit: образ rabbitmq:3-management-alpine. Все работает надежно кроме того, что он отваливается хотя в конфиге прописано, что не должен отваливаться. Но самое главное он делает - доставляет количество сделок, которое должен совершить бот. 
- server - это consumer.py, он же торговый алгоритм с ИИ. Принимает:
        ```
        params = {
                        "user_id": id,
                        "symbol": symbol,
                        "api_key": api_key,
                        "api_secret": api_secret,
                        "num_trades": num_trades,
                        "stop": stop
                    }
        ```


```
version: "3.7"

services:
    rabbit:
        image: rabbitmq:3-management-alpine
        container_name: "rabbitmq"
        ports:
            - 5672:5672
            - 15672:15672
        hostname: rabbit
        volumes:
            - ./rabbitmq/advanced.config:/etc/rabbitmq/advanced.config
        networks:
            - rabbit_net
        restart: on-failure
    server:
        build: server/
        command: python ./consumer.py
        depends_on:
            - rabbit
            - bot
        environment:
            AMQP_URL: 'amqp://guest:guest@rabbit:5672/?name=Server%20connection'
        restart: on-failure
        networks:
            - rabbit_net
    bot:
        build: bot/
        command: python ./bot.py
        environment:
            AMQP_URL: 'amqp://guest:guest@rabbit:5672/?name=Bot%20connection'
            TOKEN: '5822952565:AAH9tX6qJUJYAtN8RjlntQ1gIyrPxD0vTFo'
        depends_on:
            - rabbit
        networks:
            - rabbit_net
        restart: on-failure
networks:
    rabbit_net:
        driver: bridge
```
