from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException, WebDriverException, NoSuchElementException
import time
import random
import os
import pyautogui
import json
import zipfile
import base64
from webdriver_manager.chrome import ChromeDriverManager

# --- НАСТРОЙКИ ---
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
]

# --- СПИСОК ВАШИХ ПРОКСИ ДЛЯ ТЕСТИРОВАНИЯ (теперь SOCKS5 с отдельными полями) ---
PROXY_LIST_TO_TEST = [
    {"ip": "46.3.230.83", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "46.3.251.100", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "46.3.251.205", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "46.3.230.175", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "46.3.251.155", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "46.3.251.135", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "92.240.196.83", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "92.240.198.231", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "92.240.199.62", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
    {"ip": "92.240.198.212", "port": 16837, "user": "user311331", "password": "5am2yp", "protocol": "socks5"},
]

# --- Вспомогательные функции ---
def get_random_user_agent():
    return random.choice(USER_AGENTS)

def random_sleep(min_sec=1, max_sec=3):
    sleep_time = random.uniform(min_sec, max_sec)
    time.sleep(sleep_time)

def create_proxy_extension(proxy_data):
    """
    Создает временное расширение Chrome для настройки SOCKS5 прокси с аутентификацией.
    Возвращает путь к папке расширения.
    """
    plugin_dir = os.path.join(os.getcwd(), 'proxy_extension')

    # Удаляем старую папку расширения, если существует
    if os.path.exists(plugin_dir):
        import shutil
        shutil.rmtree(plugin_dir)
    os.makedirs(plugin_dir)

    manifest_json = {
        "version": "1.0.0",
        "manifest_version": 2,
        "name": "Chrome Proxy",
        "permissions": [
            "proxy",
            "tabs",
            "unlimitedStorage",
            "storage",
            "<all_urls>",
            "webRequest",
            "webRequestBlocking"
        ],
        "background": {
            "scripts": ["background.js"]
        },
        "minimum_chrome_version": "76.0.0.0"
    }

    background_js = """
    var config = {
            mode: "fixed_servers",
            rules: {
              singleProxy: {
                scheme: "%(protocol)s",
                host: "%(host)s",
                port: parseInt(%(port)s)
              },
              bypassList: ["localhost"]
            }
          };

    chrome.proxy.settings.set({value: config, scope: "regular"}, function() {});

    function callbackFn(details) {
        return {
            authCredentials: {
                username: "%(username)s",
                password: "%(password)s"
            }
        };
    }

    chrome.webRequest.onAuthRequired.addListener(
                callbackFn,
                {urls: ["<all_urls>"]},
                ['blocking']
    );
    """ % {
        "host": proxy_data["ip"],
        "port": proxy_data["port"],
        "username": proxy_data["user"],
        "password": proxy_data["password"],
        "protocol": proxy_data["protocol"]
    }

    with open(os.path.join(plugin_dir, "manifest.json"), "w") as f:
        json.dump(manifest_json, f)
    with open(os.path.join(plugin_dir, "background.js"), "w") as f:
        f.write(background_js)
    
    return plugin_dir

def setup_driver_anti_detect(proxy_data=None):
    service = Service(ChromeDriverManager().install())
    options = Options()

    if proxy_data:
        proxy_extension_path = create_proxy_extension(proxy_data)
        options.add_argument(f'--load-extension={proxy_extension_path}')

    options.add_argument("--disable-blink-features=AutomationControlled")
    options.add_experimental_option("excludeSwitches", ["enable-automation"])
    options.add_experimental_option('useAutomationExtension', False)
    options.add_argument(f"user-agent={get_random_user_agent()}")
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-notifications")
    options.add_experimental_option("prefs", {
        "profile.default_content_setting_values.notifications": 2,
        "profile.default_content_setting_values.geolocation": 2,
        "profile.default_content_setting_values.media_stream": 2,
    })
    
    driver = webdriver.Chrome(service=service, options=options)
    
    # Дополнительные скрипты для обхода антидетект
    driver.execute_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")
    driver.execute_script("Object.defineProperty(navigator, 'plugins', {get: () => [1, 2, 3, 4, 5]})")
    driver.execute_script("Object.defineProperty(navigator, 'languages', {get: () => ['ru-RU', 'ru']})")

    return driver

# --- Вспомогательная функция: Извлечение и печать информации об элементе ---
def get_element_info(driver, element, description="Элемент"):
    """
    Извлекает и печатает максимально доступную информацию о заданном элементе.
    Требует объекта driver для выполнения JavaScript.
    """
    print(f"\n--- ВЫВОД ИНФОРМАЦИИ ОБ {description} ---")
    try:
        print(f"  Тег HTML: {element.tag_name}")
        print(f"  Видимый текст: '{element.text.strip()}'")
        print(f"  ID атрибут: '{element.get_attribute('id')}'")
        print(f"  Класс атрибут: '{element.get_attribute('class')}'")
        print(f"  Имя (name) атрибут: '{element.get_attribute('name')}'")
        print(f"  Value (значение) атрибут: '{element.get_attribute('value')}'")
        print(f"  HREF (для ссылок) атрибут: '{element.get_attribute('href')}'")
        print(f"  Атрибут data-test-id (если есть): '{element.get_attribute('data-test-id')}'")
        print(f"  Элемент доступен (кликабелен): {element.is_enabled()}")
        print(f"  Элемент виден на странице: {element.is_displayed()}")
        print(f"  Элемент выбран (для чекбоксов/радиокнопок): {element.is_selected()}")
        print(f"  Размер (ширина, высота): {element.size}")
        print(f"  Позиция (X, Y): {element.location}")

        # Попытка получить XPath и CSS Selector с помощью JavaScript
        try:
            xpath = driver.execute_script("function getXPath(element) { if (element.id !== '') return 'id(\"' + element.id + '\")'; if (element === document.body) return element.tagName; var ix = 0; var siblings = element.parentNode.childNodes; for (var i = 0; i < siblings.length; i++) { var sibling = siblings[i]; if (sibling === element) return getXPath(element.parentNode) + '/' + element.tagName + '[' + (ix + 1) + ']'; if (sibling.nodeType === 1 && sibling.tagName === element.tagName) ix++; } } return getXPath(arguments[0]);", element)
            print(f"  XPath (сгенерированный): {xpath}")
        except Exception:
            print("  Не удалось сгенерировать XPath программно.")
        
        try:
            css_selector = driver.execute_script("var selector = ''; for (var node = arguments[0]; node.parentNode; node = node.parentNode) { var tagName = node.tagName.toLowerCase(); var id = node.id ? '#' + node.id : ''; var classNames = Array.from(node.classList).map(cls => '.' + cls).join(''); var selectorPart = tagName + id + classNames; if (node.parentNode.children.length > 1 && !id) { var index = Array.from(node.parentNode.children).indexOf(node) + 1; selectorPart += ':nth-child(' + index + ')'; } selector = selectorPart + (selector ? '>' + selector : ''); } return selector;", element)
            print(f"  CSS Selector (сгенерированный): {css_selector}")
        except Exception:
            print("  Не удалось сгенерировать CSS Selector программно.")

    except Exception as e:
        print(f"  Произошла ошибка при получении данных элемента: {e}")
    print("-------------------------------------------------")


def automate_ozon_after_auth(proxy_data):  # Основная функция автоматизации Ozon после авторизации
    """ Автоматизация Ozon после авторизации пользователя.
    Параметры:
    proxy_data (dict): Данные прокси-сервера с ключами 'ip', 'port', 'user', 'password', 'protocol'.
    Возвращает True, если сценарий выполнен успешно, иначе False.
    """
    driver = None
    finally_status = False
    
    proxy_ip = proxy_data["ip"]
    proxy_string_for_print = f"{proxy_data['protocol']}://{proxy_data['user']}:***@{proxy_data['ip']}:{proxy_data['port']}"
    print(f"\n--- ЗАПУСК СЦЕНАРИЯ С ПРОКСИ: {proxy_string_for_print} ---")
    
    try:
        driver = setup_driver_anti_detect(proxy_data)
        
        # 1. ПЕРЕХОД НА СТРАНИЦУ Ozon
        print("ШАГ 1/4: Переходим на Ozon.ru...")
        driver.get("https://www.ozon.ru/")
        random_sleep(5, 8)

        # --- ОЖИДАНИЕ ДЕЙСТВИЯ ПОЛЬЗОВАТЕЛЯ (АВТОРИЗАЦИЯ) ---
        print("\n--- ОЖИДАНИЕ: Пожалуйста, вручную авторизуйтесь на Ozon в браузере и нажмите 'OK' для продолжения. ---")
        pyautogui.confirm(text='Пожалуйста, авторизуйтесь на Ozon в открывшемся браузере и нажмите "OK" для продолжения.', title='Авторизация Ozon', buttons=['OK'])
        random_sleep(5, 8) # Дадим дополнительное время на полную загрузку после авторизации
        print("--- АВТОРИЗАЦИЯ ПРЕДПОЛАГАЕТСЯ ЗАВЕРШЕННОЙ. СКРИПТ ПРОДОЛЖАЕТ РАБОТУ. ---")

        # 2. ПЕРЕХОД НА СТРАНИЦУ ТОВАРА (для кнопки "В корзину")
        product_url = "https://www.ozon.ru/product/1406391275/"
        print(f"\nШАГ 2/4: Переходим на страницу товара: {product_url}...")
        driver.get(product_url)
        random_sleep(5, 8) # Пауза для загрузки страницы товара
        
        # 3. ПОИСК И НАЖАТИЕ КНОПКИ "В КОРЗИНУ" / "ДОБАВИТЬ В КОРЗИНУ"
        print("\nШАГ 3/4: Ищем и нажимаем кнопку 'Добавить в корзину' / 'В корзину'...")

        # Список локаторов, которые будем пробовать, в порядке убывания надежности
        locators_to_try = [
            # 1. Наиболее надежные XPath (по тексту и контексту виджета)
            (By.XPATH, "//div[@data-widget='webAddToCart']//button[contains(., 'Добавить в корзину')]"),
            (By.XPATH, "//div[@data-widget='webAddToCart']//button[contains(., 'В корзину')]"), 

            # 2. Просто по тексту кнопки (все еще очень надежно)
            (By.XPATH, "//button[contains(., 'Добавить в корзину')]"),
            (By.XPATH, "//button[contains(., 'В корзину')]"), 

            # 3. По тексту вложенного span (если кнопка содержит текст внутри span)
            (By.XPATH, "//button[.//span[contains(., 'Добавить в корзину')]]"),
            (By.XPATH, "//button[.//span[contains(., 'В корзину')]]"),

            # 4. CSS-селекторы по классам (менее надежно из-за динамики классов Ozon)
            (By.CSS_SELECTOR, "button.j3q_27.qj7_27.b25_3_1-a0"), # Комбинация нескольких классов
            (By.CSS_SELECTOR, "button.j3q_27"),
            (By.CSS_SELECTOR, "button.qj7_27"),

            # 5. По имени (name attribute) - крайне маловероятно, но по запросу
            (By.NAME, "addToCartButton"), # Пример: если бы name='addToCartButton'
            (By.NAME, "buyButton"), # Еще пример

            # 6. Общий XPath для блока добавления в корзину
            (By.XPATH, "//div[contains(@data-widget, 'webAddToBasket')]//button")
        ]

        found_button = None
        for locator_type, locator_value in locators_to_try:
            try:
                # Исправлено: используем locator_type без .value
                print(f"Попытка найти кнопку с помощью By.{locator_type.split('.')[-1]} и локатора: '{locator_value}'")
                found_button = WebDriverWait(driver, 5).until( # Ждем до 5 секунд для каждого метода
                    EC.element_to_be_clickable((locator_type, locator_value))
                )
                print(f"Кнопка успешно найдена по By.{locator_type.split('.')[-1]}: '{locator_value}'")
                break
            except Exception as e:
                print(f"Не удалось найти кнопку по By.{locator_type.split('.')[-1]} '{locator_value}'. Пробуем следующий метод...")
        
        if found_button:
            # Выводим подробную информацию о найденной кнопке перед кликом
            get_element_info(driver, found_button, "Кнопка 'Добавить в корзину'")
            found_button.click()
            print("Кнопка 'Добавить в корзину' успешно нажата.")
            random_sleep(3, 5) # Пауза после клика
        else:
            print("Не удалось найти кнопку 'Добавить в корзину' после перебора всех методов.")
            print("Пожалуйста, проверьте страницу Ozon: возможно, разметка изменилась или товар недоступен.")
            raise NoSuchElementException("Кнопка 'Добавить в корзину' не найдена.") # Выбросим исключение, чтобы сценарий завершился неудачно

        # 4. ПОИСК И НАЖАТИЕ КНОПКИ "ПЕРЕЙТИ К ОФОРМЛЕНИЮ" (или аналогичной)
        print("\nШАГ 4/4: Ищем и нажимаем кнопку 'Перейти к оформлению' (или аналогичную)...")
        # Здесь тебе нужно будет добавить аналогичный блок поиска с перебором локаторов
        # для кнопки, которая появляется после добавления товара в корзину (например,
        # кнопка в всплывающем окне корзины или кнопка в шапке сайта, ведущая в корзину/оформление).

        # Пример локаторов для кнопки "Перейти к оформлению" (нужно будет уточнить на сайте)
        checkout_locators_to_try = [
            (By.XPATH, "//a[contains(., 'Перейти в корзину')]"), # Иногда это ссылка
            (By.XPATH, "//button[contains(., 'Перейти к оформлению')]"),
            (By.CSS_SELECTOR, "a[href*='checkout']"), # Ссылка на оформление
            (By.CSS_SELECTOR, "button.go-to-checkout-button"), # Пример класса
        ]

        checkout_button = None
        for locator_type, locator_value in checkout_locators_to_try:
            try:
                # Исправлено: используем locator_type без .value
                print(f"Попытка найти кнопку оформления с помощью By.{locator_type.split('.')[-1]} и локатора: '{locator_value}'")
                checkout_button = WebDriverWait(driver, 5).until(
                    EC.element_to_be_clickable((locator_type, locator_value))
                )
                print(f"Кнопка оформления успешно найдена по By.{locator_type.split('.')[-1]}: '{locator_value}'")
                break
            except Exception as e:
                print(f"Не удалось найти кнопку оформления по By.{locator_type.split('.')[-1]} '{locator_value}'. Пробуем следующий метод...")
        
        if checkout_button:
            get_element_info(driver, checkout_button, "Кнопка 'Перейти к оформлению'")
            checkout_button.click()
            print("Кнопка 'Перейти к оформлению' успешно нажата.")
            random_sleep(3, 5)
            finally_status = True # Если дошли досюда, считаем сценарий успешным для прокси
        else:
            print("Не удалось найти кнопку 'Перейти к оформлению' после перебора всех методов.")
            raise NoSuchElementException("Кнопка 'Перейти к оформлению' не найдена.")


    except WebDriverException as e:
        print(f"Ошибка WebDriver (возможно, прокси не работает, таймаут или проблема с драйвером): {e}")
        finally_status = False
    except Exception as e:
        print(f"Произошла непредвиденная ошибка в основном блоке automate_ozon_after_auth: {e}")
        finally_status = False
    finally:
        if driver:
            print(f"\n--- ЗАВЕРШЕНИЕ: Браузер останется открытым на 30 секунд для наблюдения за прокси {proxy_ip}. ---")
            time.sleep(30)
            driver.quit()
        return finally_status

if __name__ == "__main__":
    print("Начинаем автоматизацию Ozon с прокси.")
    print("ВНИМАНИЕ: Браузер будет запускаться в 'чистом' режиме каждый раз (без загрузки профиля).")
    print("Пожалуйста, убедитесь, что у вас установлен Chrome и драйвер ChromeDriver.")
    print("Если возникнут проблемы, проверьте настройки и установку ChromeDriver.") 
    print("Запуск первого сценария через 5 секунд...")
    
    # Можно запустить сценарий для каждого прокси из списка
    results = {}
    for i, proxy_data_item in enumerate(PROXY_LIST_TO_TEST):
        print(f"\n\n=========== ЗАПУСК СЦЕНАРИЯ ДЛЯ ПРОКСИ №{i+1} ============")
        success = automate_ozon_after_auth(proxy_data_item)
        results[f"Прокси {i+1} ({proxy_data_item['ip']}:{proxy_data_item['port']})"] = "УСПЕШНО" if success else "НЕУДАЧНО"
        random_sleep(2, 5) # Пауза между тестами прокси

    print("\n--- Результаты автоматизации по всем прокси ---")
    for proxy, status in results.items():
        print(f"{proxy}: {status}")

    print("\nАвтоматизация завершена.")
    input("Нажмите Enter, чтобы закрыть программу.")
