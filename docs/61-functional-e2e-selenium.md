!!! danger "ВНИМАНИЕ"
    Теперь использование данного конспекта является платным. I am Michael from Microsoft support, send 5000$ to my PayPal account

# Билет 61. Функциональное и End-to-End тестирование. Selenium

## Ответ

### Функциональное тестирование

**Функциональное тестирование** — проверка того, что система реализует требуемые функции. Тестируется поведение системы с точки зрения пользователя: входные данные → ожидаемый выход.

Черный ящик: тестировщик знает **что** должна делать система, но не **как** она это делает внутри.

Примеры:
- «При вводе верного логина/пароля система открывает личный кабинет».
- «При заказе товара система присылает email с подтверждением».

### End-to-End (E2E) тестирование

**E2E тестирование** — проверка полного пользовательского сценария от начала до конца через реальную (или максимально близкую к реальной) систему.

```
[Браузер/Клиент]
        ↓
[Frontend / UI]
        ↓
[Backend API]
        ↓
[База данных / Внешние сервисы]
```

Все компоненты реальные. Тест проходит весь стек.

**Отличие от интеграционного:** интеграционный тест проверяет стык двух компонентов; E2E — весь путь пользователя.

### Selenium

**Selenium** — инструмент для автоматизации браузера в тестировании веб-приложений.

```
Тест-скрипт (Java/Python/JS)
        ↓
  Selenium WebDriver API
        ↓
    ChromeDriver / FirefoxDriver
        ↓
       Браузер (Chrome/Firefox)
        ↓
   Веб-приложение
```

**Пример (Java, Selenium 4):**

```java
WebDriver driver = new ChromeDriver();
driver.get("https://example.com/login");

driver.findElement(By.id("username")).sendKeys("user@test.com");
driver.findElement(By.id("password")).sendKeys("secret");
driver.findElement(By.cssSelector("button[type=submit]")).click();

// Дождаться появления элемента
WebElement welcome = new WebDriverWait(driver, Duration.ofSeconds(5))
    .until(ExpectedConditions.visibilityOfElementLocated(By.id("welcome")));

assertEquals("Добро пожаловать, User!", welcome.getText());
driver.quit();
```

### Локаторы элементов

| Метод | Пример |
|-------|--------|
| `By.id` | `By.id("username")` |
| `By.name` | `By.name("email")` |
| `By.cssSelector` | `By.cssSelector(".btn-primary")` |
| `By.xpath` | `By.xpath("//button[@type='submit']")` |
| `By.linkText` | `By.linkText("Войти")` |

---

## Подробно

### Почему E2E тесты медленные и нестабильные

E2E тесты работают с реальным браузером, реальной сетью, реальной БД. Любая задержка сети, анимация на странице, изменение вёрстки могут сломать тест. Поэтому:
- В пирамиде тестирования E2E-тестов мало.
- Они запускаются реже (не на каждом коммите, а раз в день или перед релизом).

### Явное vs неявное ожидание

Браузер рендерит страницы асинхронно. Ждать через `Thread.sleep(3000)` — плохо (медленно и ненадёжно). Правильно:

```java
// Явное ожидание: ждать элемент до 10 секунд
WebElement button = new WebDriverWait(driver, Duration.ofSeconds(10))
    .until(ExpectedConditions.elementToBeClickable(By.id("submit")));
```

### Page Object Model (POM)

Паттерн для поддержки Selenium-тестов: каждая страница описывается как класс с методами.

```java
class LoginPage {
    private WebDriver driver;
    
    LoginPage(WebDriver driver) { this.driver = driver; }
    
    LoginPage enterUsername(String username) {
        driver.findElement(By.id("username")).sendKeys(username);
        return this;
    }
    
    HomePage clickLogin() {
        driver.findElement(By.id("submit")).click();
        return new HomePage(driver);
    }
}

// В тесте:
new LoginPage(driver)
    .enterUsername("user@test.com")
    .clickLogin()
    .assertWelcomeMessageVisible();
```

POM изолирует локаторы в одном месте: если изменился id элемента, меняем только в классе страницы.

### Headless-режим

```java
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless");
WebDriver driver = new ChromeDriver(options);
```

В CI-среде (без GUI) браузер запускается в headless-режиме: работает так же, но без окна.
