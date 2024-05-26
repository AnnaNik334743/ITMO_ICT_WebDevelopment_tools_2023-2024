Начнем с того, зачем вообще в подобном приложении нужна очередь. 

ML-модели требовательны к ресурсам, их работа может занимать довольно много времени. Несмотря на то, что представленная в данной работе модель работает быстро, это не означает, что в дальнейшем ситуация не изменится. Запуск тяжелых вычислений через очередь позволяет не блокировать приложение (что важно в случае большого количества пользователей), а также создавать несколько инстансов модели для распределения нагрузки.

Итак, очередь релаизована через Redis + Celery, где Redis выступает в том числе как брокер сообщений. Была попытка поднять RabbitMQ, но он, хотя поднимался локально, выкидывал странные ошибки с путями в докере, поэтому было решено пойти по пути наименьшего сопротивления. API иммет три эндпоинта, связанных с обработкой задач в фоновом режиме: создание задачи, проверка статуса, получение результата: 

```python

@app.post('/api/process')
async def process(ingredients_joined: str = ''):
    try:    
        task = {}
        try:
            task_id = predict.delay(ingredients_joined)
            task['task_id'] = str(task_id)
            task['status'] = 'PROCESSING'
            task['url_result'] = f'/api/result/{task_id}'
        except Exception as ex:
            task['task_id'] = str(task_id)
            task['status'] = 'ERROR'
            task['url_result'] = ''
        return JSONResponse(status_code=202, content=[task])
    except Exception as ex:
        return JSONResponse(status_code=400, content=[])

@app.get('/api/result/{task_id}', response_model=Prediction)
async def result(task_id: str):
    task = AsyncResult(task_id)
    if not task.ready():
        return JSONResponse(status_code=202, content={'task_id': str(task_id), 'status': task.status, 'result': ''})
    task_result = task.get()
    result = task_result.get('result')
    return JSONResponse(status_code=200, content={'task_id': str(task_id), 'status': task_result.get('status'), 'result': result})

@app.get('/api/status/{task_id}', response_model=Prediction)
async def status(task_id: str):
    task = AsyncResult(task_id)
    return JSONResponse(status_code=200, content={'task_id': str(task_id), 'status': task.status, 'result': ''})

```

А вот так выглядит выполняемая задача: 

```python

class PredictTask(Task):
    def __init__(self):
        super().__init__()
        self.tokenizer = None
        self.model = None
        self.base_mask = None
        self.optional_mask = None

    def __call__(self, *args, **kwargs):
        
        if not self.tokenizer: 
            self.tokenizer = load_tokenizer('celery_tasks/models/tokenizer.pkl')
            if len(BASE_WORDS_WHITELIST):
                self.base_mask = create_prediction_mask(self.tokenizer, BASE_WORDS_WHITELIST)
            if len(OPTIONAL_WORDS_WHITELIST):
                self.optional_mask = create_prediction_mask(self.tokenizer, OPTIONAL_WORDS_WHITELIST)
                
        if not self.model:
            self.model = load_model('celery_tasks/models/model.h5')
            
        return self.run(*args, **kwargs)


@app.task(ignore_result=False, bind=True, base=PredictTask)
def predict(self, ingredients_joined: str) -> dict[str]:
    try:
        if ingredients_joined == '':  # first ingredient is always base
            prediction = predict_next_word(self.tokenizer, self.model, ingredients_joined, mask=self.base_mask)
        else:
            prediction = predict_next_word(self.tokenizer, self.model, ingredients_joined, mask=self.optional_mask)
        return {'status': 'SUCCESS', 'result': prediction}
    except Exception as ex:
        try:
            self.retry(countdown=2)
        except MaxRetriesExceededError as ex:
            return {'status': 'FAIL', 'result': 'max retried achieved'}

```

Небольшой пример работы:

![](static/image2.png)
![](static/image3.png)