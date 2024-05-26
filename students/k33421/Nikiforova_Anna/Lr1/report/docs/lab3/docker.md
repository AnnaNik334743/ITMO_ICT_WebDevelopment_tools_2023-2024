Реализованное приложение было упаковано в Docker, чтобы во фразе "it works on my machine" не возникало слово "only".

Dockerfile:
```
FROM python:3.10

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt
```

docker-compose.yml:
```
version: "1.0"

services:
  db:
    image: postgres:14.1
    hostname: ${DB_HOST}
    container_name: db
    restart: always
    volumes:
      - ./db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    env_file:
      - .env

  redis:
    image: redis:7.2.4
    hostname: ${REDIS_HOST}
    container_name: redis
    ports:
      - "6379:6379"
    restart: always

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app
    command: python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
    restart: always
    volumes:
      - ./app_data:/var/lib/data
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db

  celery:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: celery
    command: python -m celery -A celery_tasks.app_worker worker
    restart: always
    volumes:
      - ./app_data:/var/lib/data
    env_file:
      - .env
    depends_on:
      - app
      - redis
```

Нужно отметить, что глобальыне переменные должны лежать в файлике .env в корне проекта. 
