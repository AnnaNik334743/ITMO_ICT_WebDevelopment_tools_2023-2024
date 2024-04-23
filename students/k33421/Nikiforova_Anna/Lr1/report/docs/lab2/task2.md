## Параллельный парсинг веб-страниц с сохранением в базу данных

### Задание

Напишите программу на Python для параллельного парсинга нескольких веб-страниц с сохранением данных в базу данных 
с использованием подходов threading, multiprocessing и async.  
Каждая программа должна парсить информацию с нескольких веб-сайтов, сохранять их в базу данных.

### Решение

Так же, как и в первом задании, было решено слегка изменить формулировку. В моей постановки задачи происходит следующее: есть многостраничный [вебсайт про алкогольные коктейли](https://luding.ru/cocktails/), необходимо спарсить содержимое с нескольких его страниц, сохранив информацию в базу данных. Информация представляет собой сведения о трёх связанных сущностях (коктейли, ингридиенты, дополнительные свойства). 

В качестве базы данных используется Postgres, для подключения используется SQLAlchemy (имеет как синхронную, так и асинхронную реализацию объекта сессии). В качестве веб-скраппера используется httpx (тоже имеет как синхронную, так и асинхронную реализацию объекта сессии), в качестве парсера - BeautifulSoup.

Реализация подключения к БД:

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlalchemy.orm import sessionmaker
from config import LOGIN, PASSWORD, HOST, PORT, DB_NAME


DATABASE_URI = f"postgresql+psycopg://{LOGIN}:{PASSWORD}@{HOST}:{PORT}/{DB_NAME}"
ASYNC_DATABASE_URI = f"postgresql+psycopg_async://{LOGIN}:{PASSWORD}@{HOST}:{PORT}/{DB_NAME}"

engine = create_engine(DATABASE_URI)
Session = sessionmaker(bind=engine)

async_engine = create_async_engine(ASYNC_DATABASE_URI)
AsyncSession = async_sessionmaker(bind=async_engine)
```

Таблицы в БД:

```python
class LowercasedString(TypeDecorator):
    impl = String
    cache_ok = True

    def process_bind_param(self, value, dialect):
        return value.lower() if isinstance(value, str) else value
    

class NullableStringValidator(TypeDecorator):
    impl = String
    cache_ok = True

    def process_bind_param(self, value, dialect):
        return value if value != '' else None


class FloatStringValidator(TypeDecorator):
    impl = Float
    cache_ok = True

    def process_bind_param(self, value, dialect):
        # Validate and convert string representation of float
        if value == '':
            return None
        if isinstance(value, str):
            value = value.replace(',', '.')
        return value


ingredient_coctail_link = Table(
    "ingredient_coctail",
    Base.metadata,
    Column("ingredient_id", Integer, ForeignKey("ingredient.id"), primary_key=True),
    Column("coctail_id", Integer, ForeignKey("coctail.id"), primary_key=True),
)


properties_coctail_link = Table(
    "properties_coctail",
    Base.metadata,
    Column("properties_id", Integer, ForeignKey("property.id"), primary_key=True),
    Column("coctail_id", Integer, ForeignKey("coctail.id"), primary_key=True),
)


class Ingredient(Base):
    __tablename__ = "ingredient"

    id = Column(Integer, primary_key=True)
    name = Column(LowercasedString)
    unit = Column(String)
    unit_value = Column(FloatStringValidator, nullable=True)
    parts = Column(FloatStringValidator, nullable=True)
    coctails = relationship("Coctail", secondary=ingredient_coctail_link, back_populates="ingredients")


class Property(Base):
    __tablename__ = "property"

    id = Column(Integer, primary_key=True)
    property_name = Column(String)
    class_name = Column(String)
    name = Column(String)
    value = Column(NullableStringValidator, nullable=True)
    coctails = relationship("Coctail", secondary=properties_coctail_link, back_populates="properties")


class Coctail(Base):
    __tablename__ = "coctail"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    name_ru = Column(NullableStringValidator, nullable=True)
    detail_text = Column(String)
    steps = Column(String)
    price = Column(FloatStringValidator, nullable=True)
    ingredients = relationship("Ingredient", secondary=ingredient_coctail_link, back_populates="coctails")
    properties = relationship("Property", secondary=properties_coctail_link, back_populates="coctails")
```

Базовые классы для парсинга. Достаточно сильно захардкожена логика парсинга данных, приходыщих со страниц сайта, однако альтернативного работающего способа найти не удалось.

```python
class AbstractBaseScrapper(ABC):
    @abstractmethod
    def __str__(self) -> str:
        pass
    
    @abstractmethod
    def get_html(self, url: str) -> str:
        pass

    def parse_html(self, html: str) -> dict:
        soup = BeautifulSoup(html, features="html.parser")
    
        script_tags = soup.findAll('script')
        desired_script_tag = None

        for script_tag in script_tags:
            string_tag_repr = script_tag.string
            if string_tag_repr is not None and "JSCocktailsProducts" in string_tag_repr:
                desired_script_tag = string_tag_repr
                break
            
        if desired_script_tag:
            start_index = desired_script_tag.find("'ITEMS':")
            end_index = desired_script_tag.find(",'MARKUP':")

            json_string = '{' + desired_script_tag[start_index:end_index+1] + '}'
            return eval(json_string)
        
        return dict()
    
    @abstractmethod
    def save_data(self, data: dict) -> None:
        pass
    
    @abstractmethod
    def parse_and_save(self, urls: list[str]) -> None:
        pass


class AbstractSyncScrapper(AbstractBaseScrapper):
    @abstractmethod
    def __str__(self) -> str:
        pass
    
    def get_html(self, url: str) -> str:
        with httpx.Client() as client:
            response = client.get(url)
        assert response.status_code == 200
        return response.text

    def parse_html(self, html: str) -> dict:
        soup = BeautifulSoup(html, features="html.parser")
    
        script_tags = soup.findAll('script')
        desired_script_tag = None

        for script_tag in script_tags:
            string_tag_repr = script_tag.string
            if string_tag_repr is not None and "JSCocktailsProducts" in string_tag_repr:
                desired_script_tag = string_tag_repr
                break
            
        if desired_script_tag:
            start_index = desired_script_tag.find("'ITEMS':")
            end_index = desired_script_tag.find(",'MARKUP':")

            json_string = '{' + desired_script_tag[start_index:end_index+1] + '}'
            return eval(json_string)
        
        return dict()
    
    def save_data(self, raw_data: dict) -> None:
        with Session() as session:
            for coctail_data in raw_data["ITEMS"]:
                # Extract coctail data and keep only valid fields
                coctail_dict = {
                    "id": int(coctail_data.get("ID")),
                    "name": coctail_data.get("NAME"),
                    "name_ru": coctail_data.get("NAME_RU"),
                    "detail_text": coctail_data.get("DETAIL_TEXT", "").replace("<p>", "").replace("<\\/p>", "").replace("\t", ""),
                    "steps": "\n".join(coctail_data.get("STEPS", "")),
                    "price": coctail_data.get("PRICE")
                }
                
                # If this coctail already exists, we do not have to add it again
                statement = select(Coctail).where(Coctail.id == coctail_dict.get("id"))
                existing_coctail = session.execute(statement).scalar()
                if existing_coctail:
                    continue
                
                # Extract ingredient data and keep only valid fields
                ingredient_data = coctail_data.get("INGREDIENTS", [])

                # Extract property data and keep only valid fields
                property_data = coctail_data.get("PROPERTIES", [])

                # Convert ingredient data into SQLAlchemy objects
                ingredients = []
                for ingredient_data_entry in ingredient_data:
                    # Check if ingredient already exists based on all available fields except for id
                    statement = select(Ingredient).where(
                        (Ingredient.name == ingredient_data_entry.get("NAME")) &
                        (Ingredient.unit == ingredient_data_entry.get("UNIT")) &
                        (Ingredient.unit_value == ingredient_data_entry.get("UNIT_VALUE")) &
                        (Ingredient.parts == ingredient_data_entry.get("PARTS"))
                    )
                    existing_ingredient = session.execute(statement).scalar()
                    
                    if not existing_ingredient:
                        ingredient = Ingredient(
                            name=ingredient_data_entry.get("NAME"),
                            unit=ingredient_data_entry.get("UNIT"),
                            unit_value=ingredient_data_entry.get("UNIT_VALUE"),
                            parts=ingredient_data_entry.get("PARTS")
                        )
                        session.add(ingredient)
                    else:
                        ingredient = existing_ingredient

                    ingredients.append(ingredient)

                # Convert property data into SQLAlchemy objects
                properties = []
                for property_key, property_value in property_data.items():
                    # Check if property already exists based on all available fields except for id
                    statement = select(Property).where(
                        (Property.property_name == property_key) &
                        (Property.class_name == property_value.get("CLASS_NAME")) &
                        (Property.name == property_value.get("NAME")) &
                        (Property.value == property_value.get("VALUE"))
                    )
                    existing_property = session.execute(statement).scalar()

                    if not existing_property:
                        property = Property(
                            property_name=property_key,
                            class_name=property_value.get("CLASS_NAME"),
                            name=property_value.get("NAME"),
                            value=property_value.get("VALUE")
                        )
                        session.add(property)
                    else:
                        property = existing_property

                    properties.append(property)

                # Update the coctail_dict with converted ingredients and properties
                coctail_dict["ingredients"] = ingredients
                coctail_dict["properties"] = properties

                # Create the Coctail object
                coctail = Coctail(**coctail_dict)
                session.add(coctail)

            # Commit changes after all coctails, ingredients, and properties are added
            session.commit()
    
    @abstractmethod
    def parse_and_save(self, urls: list[str]) -> None:
        pass
```

Конкретные реализации:

=== "sync"
    ```python
    class SimpleScrapper(AbstractSyncScrapper):
    def __str__(self) -> str:
        return "SimpleScrapper"

    def parse_and_save(self, urls: list[str]) -> float:
        start_time = time.time()
        for url in urls:
            html = self.get_html(url)
            raw_data = self.parse_html(html)
            self.save_data(raw_data)
        end_time = time.time()
        execution_time = end_time - start_time
        return execution_time
    ```
    Парсинг всех страниц осуществляется последовательно.

=== "threading"
    ```python
    class URLProcessingThread(threading.Thread):
    def __init__(self, url, scrapper_instance):
        super().__init__()
        self.url = url
        self.scrapper_instance = scrapper_instance

    def run(self):
        html = self.scrapper_instance.get_html(self.url)
        raw_data = self.scrapper_instance.parse_html(html)
        self.scrapper_instance.save_data(raw_data)


class ThreadingScrapper(AbstractSyncScrapper):
    def __str__(self) -> str:
        return "ThreadingScrapper"
    
    def parse_and_save(self, urls: list[str]) -> float:
        start_time = time.time()
        threads = []

        for url in urls:
            thread = URLProcessingThread(url, self)
            thread.start()
            threads.append(thread)

        for thread in threads:
            thread.join()

        end_time = time.time()
        execution_time = end_time - start_time
        return execution_time
    ```
    Для парсинга каждой страницы выделяется отдельный поток.

=== "multiprocessing"
    ```python
    class MultiprocessingScrapper(AbstractSyncScrapper):
    def __str__(self) -> str:
        return "MultiprocessingScrapper"

    def process_url(self, url):
        html = self.get_html(url)
        raw_data = self.parse_html(html)
        self.save_data(raw_data)

    def parse_and_save(self, urls: list[str]) -> float:
        start_time = time.time()
        with multiprocessing.Pool() as pool:
            pool.map(self.process_url, urls)

        end_time = time.time()
        execution_time = end_time - start_time
        return execution_time
    ```
    Для парсинга каждой страницы выделяется отдельный процесс.

=== "async"
    ```python
    class AsyncScrapper(AbstractBaseScrapper):
    def __str__(self) -> str:
        return "AsyncScrapper"
    
    async def get_html(self, url: str) -> str:
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
        assert response.status_code == 200
        return response.text
    
    async def save_data(self, raw_data: dict) -> None:
        async with AsyncSession() as session:
            for coctail_data in raw_data["ITEMS"]:
                # Extract coctail data and keep only valid fields
                coctail_dict = {
                    "id": int(coctail_data.get("ID")),
                    "name": coctail_data.get("NAME"),
                    "name_ru": coctail_data.get("NAME_RU"),
                    "detail_text": coctail_data.get("DETAIL_TEXT", "").replace("<p>", "").replace("<\\/p>", "").replace("\t", ""),
                    "steps": "\n".join(coctail_data.get("STEPS", "")),
                    "price": coctail_data.get("PRICE")
                }
                
                # If this coctail already exists, we do not have to add it again
                statement = select(Coctail).where(Coctail.id == coctail_dict.get("id"))
                existing_coctail = (await session.execute(statement)).scalar()
                if existing_coctail:
                    continue
                
                # Extract ingredient data and keep only valid fields
                ingredient_data = coctail_data.get("INGREDIENTS", [])
                
                # Extract property data and keep only valid fields
                property_data = coctail_data.get("PROPERTIES", [])

                # Convert ingredient data into SQLAlchemy objects
                ingredients = []
                for ingredient_data_entry in ingredient_data:
                    # Check if ingredient already exists based on all available fields except for id
                    statement = select(Ingredient).where(
                        (Ingredient.name == ingredient_data_entry.get("NAME")) &
                        (Ingredient.unit == ingredient_data_entry.get("UNIT")) &
                        (Ingredient.unit_value == ingredient_data_entry.get("UNIT_VALUE")) &
                        (Ingredient.parts == ingredient_data_entry.get("PARTS"))
                    )
                    existing_ingredient = (await session.execute(statement)).scalar()
                    
                    if not existing_ingredient:
                        ingredient = Ingredient(
                            name=ingredient_data_entry.get("NAME"),
                            unit=ingredient_data_entry.get("UNIT"),
                            unit_value=ingredient_data_entry.get("UNIT_VALUE"),
                            parts=ingredient_data_entry.get("PARTS")
                        )
                        session.add(ingredient)
                    else:
                        ingredient = existing_ingredient

                    ingredients.append(ingredient)

                # Convert property data into SQLAlchemy objects
                properties = []
                for property_key, property_value in property_data.items():
                    # Check if property already exists based on all available fields except for id
                    statement = select(Property).where(
                        (Property.property_name == property_key) &
                        (Property.class_name == property_value.get("CLASS_NAME")) &
                        (Property.name == property_value.get("NAME")) &
                        (Property.value == property_value.get("VALUE"))
                    )
                    existing_property = (await session.execute(statement)).scalar()

                    if not existing_property:
                        property = Property(
                            property_name=property_key,
                            class_name=property_value.get("CLASS_NAME"),
                            name=property_value.get("NAME"),
                            value=property_value.get("VALUE")
                        )
                        session.add(property)
                    else:
                        property = existing_property

                    properties.append(property)

                # Update the coctail_dict with converted ingredients and properties
                coctail_dict["ingredients"] = ingredients
                coctail_dict["properties"] = properties

                # Create the Coctail object
                coctail = Coctail(**coctail_dict)
                session.add(coctail)

            # Commit changes after all coctails, ingredients, and properties are added
            await session.commit()
            
    async def process_url(self, url: str) -> None:
        html = await self.get_html(url)
        raw_data = self.parse_html(html)
        await self.save_data(raw_data)

    async def parse_and_save(self, urls: list[str]) -> float:
        start_time = time.time()
        await asyncio.gather(*(self.process_url(url) for url in urls))
        end_time = time.time()
        execution_time = end_time - start_time
        return execution_time
    ```

    Используются асинхронные версии всего, чего только можно.

### Сравнение и итоги

Результаты для парсинга 5 страниц:

| Подход          | Время выполнения    |
|-----------------|---------------------|
| sync            | 20.18869686126709   |
| threading       | 5.608245134353638   |
| multiprocessing | 5.733122110366821   |
| async           | 5.6372270584106445  |

Результаты для парсинга 10 страниц:

| Подход          | Время выполнения    |
|-----------------|---------------------|
| sync            | 41.915942668914795  |
| threading       | 7.99373984336853    |
| multiprocessing | 10.207027673721313  |
| async           | 8.419097900390625   |

Как можно видеть, синхронная реализация проигрывает всем остальным. В случае мультипроцессинга мы ограничены количеством ядер процессора. Хорошо работает многопоточность и асинхронность, причем интересно, что многопоточная реализация работает даже быстрее. В принципе, это хорошо объясняется I/O-bound типом решаемой задачи (ожидание ответа от сайта, чтение и запись в БД).
