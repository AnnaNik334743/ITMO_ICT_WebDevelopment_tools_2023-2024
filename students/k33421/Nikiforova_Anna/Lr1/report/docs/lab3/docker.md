Реализованное приложение было упаковано в Docker, чтобы во фразе "it works on my machine" не возникало слово "only".

Dockerfile celery_app:
```
FROM python:3.10

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000"]
```

Dockerfile fastapi_app:
```
FROM python:3.10

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

docker-compose.yml:
```
version: "3.0"

services:
  db:
    image: postgres:14.1
    container_name: db
    restart: always
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    env_file:
      - .env

  redis:
    image: redis:7.2.4
    container_name: redis
    ports:
      - "6379:6379"
    restart: always

  fastapi_app:
    build:
      context: ./fastapi_app
    container_name: fastapi_app
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    restart: always
    volumes:
      - ./fastapi_app:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db
      - redis
      - celery_app

  celery_app:
    build:
      context: ./celery_app
    container_name: celery_app
    command: uvicorn main:app --host 0.0.0.0 --port 5000
    restart: always
    volumes:
      - ./celery_app:/app
    ports:
      - "5000:5000"
    env_file:
      - .env
    depends_on:
      - redis

  celery_worker:
    build:
      context: ./celery_app
    container_name: celery_worker
    command: celery -A celery_worker worker --loglevel=info
    restart: always
    volumes:
      - ./celery_app:/app
    env_file:
      - .env
    depends_on:
      - redis
      - celery_app

volumes:
  db_data:
```

Нужно отметить, что глобальыне переменные должны лежать в файлике .env в корне проекта. 
