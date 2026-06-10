# Настройке Django-Filer и Easy-Thumbnails с поддержкой HEIC/HEIF и автоконве­ртацией в WebP в проектах на Django

Django-Filer — популярная библиотека для управления файлами и изобра­жениями в Django. Она использует Easy-Thumbnails
для создания миниатюр изображений. Но, к сожалению, в документации не все описано. Но главное, что поддержка
изображений HEIC/HEIF вообще не преду­смотрена. С этими форматами вообще не все так просто в мире Python.
Наиболее популярная библиотека PIL (Pillow) из коробки **не поддерживает HEIC/HEIF**. Это форматы от Apple для iPhone и большинство приложекний для работы с графикой мира Wiindows, Linux и даже MacOS не поддерживают их.
Но есть сторонняя библиотека `pillow-heif`, которая добавляет поддержку HEIC/HEIF в Pillow. И мы можем использовать
её для обработки этих форматов в Django-Filer и Easy-Thumbnails.

## Содержание
1. [Установка пакетов](#установка-пакетов)
2. [Настройка FILER_STORAGES](#настройка-filer_storages)
3. [Конфигурация Thumbnails](#конфигурация-thumbnails)
4. [Автоко­нвертация JPEG/HEIC в WebP](#автоко­нвертация-jpegheic-в-webp)
5. [Решение проблем](#решение-проблем)

## Так же рекомендую сверяться с официальной докуме­нтацией:
- [Настройки Django-Filer](https://django-filer.readthedocs.io/en/latest/settings.html)
- [Настройки Easy-Thumbnails](https://easy-thumbnails.readthedocs.io/en/latest/ref/settings/)
- [Опции Pillow-Heif](https://pillow-heif.readthedocs.io/en/latest/options.html)

—

## Установка пакетов

### Системные пакеты

Для поддержки HEIC/HEIF в Pillow нам нужно установить в системк `libheif` и его зависимости. На разных системах это
делается по-разному:

- **Ubuntu/Debian**:
  ```bash
  sudo apt-get update
  sudo apt-get install -y libheif-dev libde265-dev
  ```
- **Fedora**:
  ```bash
  sudo dnf install -y libheif-devel libde265-devel
  ```
- **MacOS (через Homebrew)**:
  ```bash
  brew install libheif
  ```

### Установка django-filer и pillow-heif

При установке `django-filer` он автома­тически добавит `pillow` и `easy-thumbnails`. Для поддержки HEIC/HEIF нам
нужно дополни­тельно установить `pillow-heif`:

- **Установка через pip**:
  ```bash
  pip install django-filer pillow-heif
  ```
- **Установка через Poetry**:
  ```bash
  poetry add django-filer pillow-heif
  ```

### Почему нужен pillow-heif?

`pillow-heif` регистрирует себя как плагин к Pillow и добавляет эту поддержку:

```python
import pillow_heif
pillow_heif.register_heif_opener()  # Теперь PIL может открывать HEIC/HEIF
```

—

## Настройка FILER_STORAGES

У django-filer много настроек который делаются в `settings.py` вашего django-проекта, но не все из них докуме­нтированы.
В частности `FILER_STORAGES` содержит не описаный ключ `base_dir`, котрый задает имя промежу­точного каталог для хранеия миниатюр.

Например:

```python
FILER_STORAGES = {
    'public': {
        'main': {
            'ENGINE': 'filer.storage.PublicFileSystemStorage',
            'OPTIONS': {
                'location': MEDIA_ROOT / 'flr',       # где хранятся загруженные файлы
                'base_url': MEDIA_URL + 'flr/',       # URL публичного доступа
            },
            'UPLOAD_TO': 'filer.utils.generate_filename.randomized',
            'UPLOAD_TO_PREFIX': '',
        },
        'thumbnails': {
            'ENGINE': 'filer.storage.PublicFileSystemStorage',
            'OPTIONS': {
                'location': MEDIA_ROOT / 'flrm',      # где хранятся миниатюры
                'base_url': MEDIA_URL + 'flrm/',     # URL для миниатюр
            },
            # ВАЖНО: Переопре­деляем THUMBNAIL_OPTIONS
            'THUMBNAIL_OPTIONS': {
                'base_dir': '',  # Отключаем default prefix 'filer_public_thumbnails'
            },
        },
    },
}
```

### Проблема с `base_dir`

**Проблема:** В официальной документации filer **нет упоминания** о ключе `base_dir` в `THUMBNAIL_OPTIONS`.

**Почему это важно:** По умолчанию filer добавляет prefix-каталог `'filer_public_thumbnails'` ко всем миниатюрам:
```
flrm/
  ├── filer_public_thumbnails/
  │   └── af
  │         └── a3
  │             └── 6aa36f69–8a98–4414–9e9a-f3a849da1223
  │                 └── [миниатюры]
```

Пути и URL у миниатюр (thumbnails) и так длинные, и добавление еще одного  уровня вложенности каталогов, да еще
с таким длинным именем, может нешуточно раздуть резуль­тирующий html-файл или вызвать проблемы с файловой системой
(особенно актуально для Windows).

**Решение:** Явно указываем `'base_dir': ''` чтобы миниатюры лежали прямо в `flrm/`:
```
flrm/
  ├── af
  │   └── a3
  │       └── 6aa36f69–8a98–4414–9e9a-f3a849da1223
  │           └── [миниатюры]
```

Это намного удобнее и экономит уровни вложенности папок. — ## Конфигурация Thumbnails

### Easy-Thumbnails настройки

```python
# Основные параметры
THUMBNAIL_PRESERVE_FORMAT = False      # Не сохранять исходный формат
THUMBNAIL_FORMAT = 'WEBP'              # Создавать миниатюры в WebP (экономия места на диске и трафика в 3–4 раза)
THUMBNAIL_QUALITY = 80                 # Качество сжатия (0–100)
THUMBNAIL_WEBP_QUALITY = THUMBNAIL_QUALITY  # Наша кастомная настройка… Качество сжатие исходных изображений в WebP

# Движок обработки
THUMBNAIL_ENGINE = 'easy_thumbnails.engines.pil_engine.PilEngine'

# КРИТИЧНО: Для этих файлов будет сохранен формат при генерации миниатюр
THUMBNAIL_PRESERVE_EXTENSIONS = ['png', 'gif', 'webp']

# Нейминг миниатюр… Формат, что-то типа: you-image.webp__40×40_q85_crop_subsampling-2.jpg
THUMBNAIL_NAMER = 'easy_thumbnails.namers.default'

# Отладка (устана­вливает в соответствии с уровнем отладки всего проекта django)
THUMBNAIL_VERBOSE = DEBUG

# Преду­становки размеров. Можно вообще не объявлять, т. к. алиасы миниатюр можно настроить из админки filer, а размеры
# миниатюр для самой filer-админки зашиты внутрь библиотекb и не изменяются (это 40×40, 210×210 и 420×420)
THUMBNAIL_ALIASES = {
    '': {
        'small': {'size': (256, 256), 'crop': True},
        'medium': {'size': (512, 512), 'crop': True},
        'large': {'size': (1024, 1024), 'crop': 'smart'},
    },
}
```

### Важные параметры для Filer

Для поддержки HEIC/HEIF важно добавить в «белый список» MMIME-типы и равсширения файлов:

```python
# Типы файлов
MIME_TYPE_WHITELIST = (
    'image/jpeg',
    'image/png',
    'image/gif',
    'image/svg+xml',
    'image/webp',
    'image/heic',      # HEIC поддержка
    'image/heif',      # HEIF поддержка
    'application/pdf',
    # … другие типы
)

# Расширения файлов
FILER_WHITELIST_FOR_PATH_ACCESS = (
    '.jpg', '.jpeg', '.png', '.gif', '.svg', '.webp',
    '.heic', '.heif',  # HEIC/HEIF поддержка
    '.doc', '.docx', '.pdf', '.xls', '.xlsx', '.txt', '.csv',
)

# Ограничения загрузки
FILER_UPLOADER_MAX_FILE_SIZE = 100 * 1024 * 1024  # 100 MB
FILER_MAX_IMAGE_PIXELS = 4096 * 4096              # Макс разрешение
```

—

## Автоко­нвертация JPEG/HEIC/HEIF в WebP

### Проблема

Когда пользователь загружает HEIC или JPEG:
1. Браузер отправляет оригинальный файл (тяжелый).
2. Filer сохраняет его как есть.
3. Результат: большие файлы в хранилище, медленная загрузка, много трафика.
4. Для HEIC/HEIF никаких не создаётся вообще, т. к. fille не считает их картинками (A), PIL не может их открыть (B) и, 
   браузеры их вообще не поддерживают ©.

### Решение: Патчинг MultiStorageFieldFile.save()

Переопре­деляем метод сохранения файлов filer в `apps.py`:

```python
# frontend/apps.py

import hashlib
import os
import pillow_heif
from io import BytesIO
from django.apps import AppConfig
from django.core.exceptions import ValidationError
from django.core.files.base import ContentFile, File
from PIL import Image as PILImage
from typing import Tuple
from НАШ_ПРОЕКТ.settings import (
    THUMBNAIL_WEBP_QUALITY,
    FILER_ENABLE_PERMISSIONS,
    FILER_UPLOADER_MAX_FILE_SIZE,
    FILER_WHITELIST_FOR_PATH_ACCESS,
    MIME_TYPE_WHITELIST,
    FILE_VALIDATORS,
)

# Регистрируем плагин HEIF в Pillow для поддержки HEIC/HEIF файлов
pillow_heif.register_heif_opener()

# Настройка парал­лельного декоди­рования HEIF/HEIC файлов
# DECODE_THREADS: количество потоков для декоди­рования
# - 0 = использовать количество ядер процессора (рекоме­ндуется)
# - N &gt; 0 = использовать N потоков (больше потоков = быстрее, но больше памяти)
pillow_heif.options.DECODE_THREADS = 0

# Дополни­тельные опции Pillow для обработки изображений
# LOAD_TRUNCATED_IMAGES: загружать некорректно завершенные изображения
# (полезно для некачес­твенных или поврежденных файлов)
PILImage.LOAD_TRUNCATED_IMAGES = True

logger = logging.getLogger(__name__)

# … конфигурация основного django-приложения …
# …
# …

# Патчинг filer для автоко­нвертации в WebP и поддержки HEIC/HEIF
class CustomFilerConfig(AppConfig):
    name = 'filer'
    verbose_name = 'Медиафайлы'  # Очень удобно, что можно переопре­делить название приложения в админке
    
    @staticmethod
    def _convert_to_webp_if_needed(
        name: str, content: File
    ) -&gt; Tuple[File, str, bool, int | None, int | None]:
        """
        Конвертирует JPEG/PNG/BMP/TIFF/HEIC/HEIF в WebP.
        
        Returns:
            (content, new_name, was_converted, image_width, image_height)
        """
        _, original_ext = os.path.splitext(name)
        convertible_extensions = [".jpg», ".jpeg», ".png», ".bmp», ".tiff», ".heic», ".heif»]
        
        if original_ext.lower() in convertible_extensions:
            try:
                # Открываем исходный файл
                content.seek(0)
                img = PILImage.open(BytesIO(content.read()))
                
                # Получаем размеры ПЕРЕД конвертацией
                # (они не меняются, меняется только качество)
                image_width, image_height = img.size
                
                # Конвертируем CMYK в RGB
                if img.mode == 'CMYK':
                    img = img.convert('RGB')
                
                # Сохраняем в WebP
                webp_buffer = BytesIO()
                img.save(webp_buffer, format="WEBP», quality=THUMBNAIL_WEBP_QUALITY)
                webp_data = webp_buffer.getvalue()
                
                new_name = os.path.splitext(name)[0] + ".webp»
                return ContentFile(webp_data), new_name, True, image_width, image_height
                
            except Exception as e:
                # логируем и выбразывсаем ошибку валидации в админку django
                error_msg = f"Ошибка при конвертации '{name}' в WebP: {str(e)}"
                logger.error(error_msg, exc_info=True)
                raise ValidationError(error_msg)
        
        # Если не конвертируем, возвращаем исходный файл
        content.seek(0)
        return content, name, False, None, None

    def ready(self) -&gt; None:
        ""«Патчим MultiStorageFieldFile.save() для WebP конвертации.»""
        from filer.fields.multistorage_file import MultiStorageFieldFile
        from filer import settings as filer_settings
        
        original_save = MultiStorageFieldFile.save
        
        # ВАЖНО: Добавляем HEIC/HEIF в IMAGE_EXTENSIONS, чтобы filer распознавал их как изображения
        filer_settings.IMAGE_EXTENSIONS = list(
            set(filer_settings.IMAGE_EXTENSIONS + ['.heic', '.heif'])
        )
        filer_settings.IMAGE_MIME_TYPES = list(
            set(filer_settings.IMAGE_MIME_TYPES + ['heic', 'heif'])
        )
        
        def patched_save(self_instance, name: str, content: File, save: bool = True) -&gt; str | None:
            ""«Патчи­рованный save() с конвертацией в WebP.»""
            new_content, new_name, converted, img_width, img_height = (
                CustomFilerConfig._convert_to_webp_if_needed(name, content)
            )
            
            if converted:
                # Устана­вливаем MIME тип WebP
                self_instance.instance.mime_type = «image/webp»
                
                # Обновляем original_filename
                if hasattr(self_instance.instance, 'original_filename'):
                    if self_instance.instance.original_filename:
                        original_filename, _ = os.path.splitext(
                            self_instance.instance.original_filename
                        )
                        self_instance.instance.original_filename = f"{original_filename}.webp»
                
                # КРИТИЧНО: Обновляем _file_size и sha1 на основе уже сконверти­рованного WebP, а не исходника
                new_content.seek(0)
                webp_bytes = new_content.read()
                new_content.seek(0)
                self_instance.instance._file_size = len(webp_bytes)
                self_instance.instance.sha1 = hashlib.sha1(webp_bytes).hexdigest()
            
            # Вызываем оригинальный save()
            result = original_save(self_instance, new_name, new_content, save)
            
            # Гарантируем что mime_type=image/webp в БД
            if converted and new_name.lower().endswith('.webp'):
                if self_instance.instance.id:
                    self_instance.instance.__class__.objects.filter(
                        id=self_instance.instance.id
                    ).update(mime_type='image/webp')
            
            return result
        
        MultiStorageFieldFile.save = patched_save
```

—

## Почему это было непросто с атрибутами файла?

### Проблема 1: Instance.id = None при первом сохранении 
**Проблема:**
```python
# После original_save() instance.id все еще None!
original_save(self_instance, new_name, new_content, save=False)
if self_instance.instance.id:  # ← None!
    # Это не выполнится
```

**Причина:** При `save=False` файл не сохраняется в БД, поэтому `id` не присва­ивается.

**Решение:** Обновляем в отдельном запросе ПОСЛЕ `original_save()`:
```python
if converted:
    # После ориги­нального save()
    self_instance.instance.__class__.objects.filter(
        id=self_instance.instance.id
    ).update(mime_type='image/webp')
```

### Проблема 2: mime_type должен быть установлен ДО сохранения

**Проблема:** Если установить `mime_type` после `original_save()`, filer уже решил создать `File` или `Image`.

**Решение:** Устана­вливаем ПЕРЕД вызовом `original_save()`:
```python
if converted:
    self_instance.instance.mime_type = «image/webp»  # ← ДО save()
    # Теперь filer видит image/webp и создает Image запись
```

### Проблема 3: _file_size и sha1 из исходного файла

**Проблема:** Django-filer читает `_file_size` и `sha1` из исходных данных:
```python
# Это будет размер ИСХОДНОГО файла (тяжелого HEIC)
self_instance.instance._file_size = len(original_bytes)
self_instance.instance.sha1 = hash(original_bytes)
```

**Результат:** В БД записывается неправильный размер и хеш исходного файла, а не WebP.

**Решение:** Обновляем ПЕРЕД `original_save()`:
```python
if converted:
    new_content.seek(0)
    webp_bytes = new_content.read()
    new_content.seek(0)
    self_instance.instance._file_size = len(webp_bytes)
    self_instance.instance.sha1 = hashlib.sha1(webp_bytes).hexdigest()
    # Теперь original_save() будет использовать ПРАВИЛЬНЫЕ значения
```

### Проблема 4: Размеры изображения из преобра­зованного файла

**Проблема:** Обычно размеры не меняются при конвертации, но нужно быть осторожным. HEIС/HEIF могут содержать
несколько изображений (например, Live Photos), параметры трехмерной сцены и вообще не иметь «основного» разрешения
и переуста­навливать его динамически в зависимости от устройства или слоев редакти­рования. Из-за этого Pillow
может не всегда корректно считать размеры HEIС/HEIF-картинки. На праактике, ему это почти никогда не удается, и он просто возвращает None.

Поэтому мы не можем полагаться на размеры из исходного файла, а должны брать их из уже конверти­рованного WebP,
который гаранти­рованно будет иметь правильные размеры.

**Решение:** Получаем размеры из исходного PIL Image **перед** конвертацией:
```python
img = PILImage.open(BytesIO(content.read()))
image_width, image_height = img.size  # Получаем размеры

# Теперь конвертируем
img.convert('RGB').save(webp_buffer, format="WEBP»)

# Размеры image_width и image_height останутся теми же
# (пиксели не меняются, меняется только качество)
```

### Проблема 5: IMAGE_EXTENSIONS - глобальные константы

**Проблема:** Нельзя переопре­делить через `settings.py`:
```python
# В settings.py это НЕ будет работать:
FILER_IMAGE_EXTENSIONS = ['.heic', '.heif']  # ← игнорируется

# Потому что filer делает:
from filer.settings import IMAGE_EXTENSIONS  # ← импортирует, не читает из settings
```

**Решение:** Модифицируем после импорта в `ready()`:
```python
from filer import settings as filer_settings

filer_settings.IMAGE_EXTENSIONS = list(
    set(filer_settings.IMAGE_EXTENSIONS + ['.heic', '.heif'])
)
```

—

## Полный результат

При загрузке HEIC файла в админ filer:

```
Исходный HEIC файл (2.5 MB)
    ↓
_convert_to_webp_if_needed() преобразует в WebP (0.3 MB)
    ↓
patched_save() обновляет метаданные:
  - mime_type = «image/webp»
  - _file_size = 0.3 MB (WebP, не исходный!)
  - sha1 = хеш WebP (не исходного!)
  - original_filename = IMG.webp
    ↓
Filer сохраняет файл в БД
    ↓
Миниатюры генерируются автома­тически (40×40, 210×210, 420×420)
    ↓
В админе отображается миниатюра и размеры
Экономия места: 2.5 MB → 0.3 MB (~88% сокращение)
```

—

## Тестирование

В админе filer загрузить HEIC файл и проверить:
2. Файл сохранен как .webp 
2. Миниатюра 40×40 видна в списке
3. Размеры (в пикселях и вес картинки) указаны правильно
4. При открытии картинки на редакти­рованече через админку filer видно:
   * что это WebP (не HEIC) и отображается корректно;
   * автома­тически создались миниатюры 210×210 и 420×420;
   * SHA1 хеш из WebP данных (не исходного HEIC);
   * MIME-тип установлен как `image/webp`.



