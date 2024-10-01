## 1. **Интерфейс `iCart`**

Интерфейс задает три метода: `calcVat()`, `notify()`, и `makeOrder($discount = 1.0)`.

**Проблемы:**

- **Отсутствие типизации методов:** Интерфейс не указывает, какие типы должны принимать параметры методов и что они должны возвращать. Это делает реализацию менее предсказуемой и создает возможность ошибок при неверной интерпретации типов.

**Рекомендация:**

- Указать типы параметров и возвращаемые значения для всех методов, что сделает интерфейс более четким и строгим.

```PHP
public function calcVat(): float;
public function notify(): void;
public function makeOrder(float $discount = 1.0): void;
```

## 2. **Класс `Cart`**

**Атрибуты `items` и `order`:**

- **Проблема:** Эти атрибуты не определены явно и не типизированы, что может привести к ошибкам. Например, если кто-то передаст что-то, кроме массива, в `$items`, это вызовет ошибку в методах класса.

**Рекомендация:**

- Добавить явное объявление этих свойств с типами. Также можно инициализировать `items` как пустой массив, чтобы избежать ошибок доступа к неопределенному свойству.

```PHP
public array $items = [];
public ?Order $order = null;
```

## 3. **Метод `calcVat()`**

**Проблемы:**

- **Дублирование кода:** НДС считается в этом методе и в других местах кода, что создает дублирование.
- **Отсутствие гибкости:** Метод жестко использует ставку НДС 18%. Если ставка изменится, придется переписывать этот метод.

**Рекомендация:**

- Сделать ставку НДС параметром или константой класса и вынести расчет в отдельный метод, чтобы избежать дублирования кода.

```PHP
const VAT_RATE = 0.18;

public function calcVat(): float
{
    $vat = 0;
    foreach ($this->items as $item) {
        $vat += $item->getPrice() * self::VAT_RATE;
    }
    return $vat;
}
```

## 4. **Метод `notify()` и `sendMail()`**

**Проблемы:**

- **Жестко закодированные данные:** В методе `sendMail()` создается объект `SimpleMailer` с жестко прописанными значениями пользователя и пароля, что нарушает принцип гибкости кода и усложняет тестирование.
- **Логика метода:** Логика отправки уведомлений смешана с расчетом суммы заказа и НДС, что нарушает принцип единственной ответственности (SRP).

**Рекомендации:**

- Внедрить зависимости через конструктор (Dependency Injection), передавая объекты почтового сервиса в конструкторе. Это позволит легче тестировать код и менять реализацию почтового сервиса при необходимости.
- Вынести расчет суммы заказа в отдельный метод, чтобы сделать код более чистым.

## 5. **Метод `makeOrder($discount)`**

**Проблемы:**

- **Отсутствие проверки аргументов:** Переменная `$discount` никак не проверяется. Например, отрицательное значение скидки может привести к ошибкам при расчете итоговой цены.
- **Дублирование расчета:** Метод заново считает итоговую цену, что дублирует логику, уже реализованную в других методах.
- **Нарушении инкапсуляции**: Прямое присваивание заказа через `$this->order` нарушает инкапсуляцию.

**Рекомендации:**

- Добавить проверку для аргумента `$discount`.
- Вынести расчет итоговой цены с учетом скидки и НДС в отдельный метод, чтобы уменьшить дублирование кода.
- Добавить геттеры и сеттеры для Order

```PHP
public function getOrder(): Order
{
	return $this->order;
}

public function setOrder(Order $order): void 
{ 
	$this->order = $order; 
}

public function makeOrder(float $discount): void
{
    if ($discount <= 0 || $discount > 1) {
        throw new InvalidArgumentException('Invalid discount value');
    }

    $totalPrice = $this->calculateTotalPriceWithDiscount($discount);
    $this->setOrder(new Order($this->items, $totalPrice));
    $this->notify();
}

private function calculateTotalPriceWithDiscount(float $discount): float
{
    $totalPrice = 0;
    foreach ($this->items as $item) {
        $totalPrice += $item->getPrice() * self::VAT_RATE * $discount;
    }
    return $totalPrice;
}
```

## 6. **Общие проблемы**

- **Отсутствие типизации:** Весь код страдает от отсутствия типизации аргументов и возвращаемых значений, что может привести к трудноуловимым ошибкам.
    
- **Жестко закодированные значения:** Использование жестко прописанных данных для логина и пароля (например, 'cartuser', 'j049lj-01') делает код негибким и небезопасным. Лучше вынести эти данные в настройки приложения.
    
- **Тестируемость:** Код сложен для тестирования, так как напрямую создает объекты внешних сервисов (например, `SimpleMailer`). Использование Dependency Injection упростит создание тестов для класса.


##### Это лишь первый аспект, который можно отметить. Теперь перейдем к рассмотрению более сложных и глубоких проблем:

## **1. Нарушение принципа единственной ответственности**

Класс `Cart` не только выполняет расчет стоимости с учетом налога, но также отвечает за отправку уведомлений, ограниченных только отправкой email, и за создание заказов. Однако его ответственность должна ограничиваться управлением списком товаров, добавленных пользователем в корзину. Такое распределение обязанностей является явным нарушением принципа единственной ответственности.

**Рассмотрим методы по отдельности:**

### 1. Метод `calcVat()`

#### Описание изменений:

1. **Создание класса `VatManager`**:
    - **Задача класса**: Обработка расчетов, связанных с НДС. Этот класс будет независимым компонентом, который можно использовать для любых операций, где требуется налог.
    - **Метод `applyVat`**: Выполняет расчет налога для переданной стоимости.
2. **Удаление метода `calcVat` из класса `Cart`**:
    - Класс `Cart` теперь будет полагаться на сервис `VatManager` для работы с налогами, что соответствует SRP.

#### Пример реализации:

##### 1. Класс `VatManager`:

```PHP
class VatManager
{
    private const float DEFAULT_VAT_RATE = 0.18;  // НДС по умолчанию 18%
    private float $vatRate;

    public function __construct(float $vatRate = self::DEFAULT_VAT_RATE)
    {
        if ($vatRate < 0 || $vatRate > 1) {
            throw new InvalidArgumentException('Invalid VAT rate');
        }
        $this->vatRate = $vatRate;
    }

    public function applyVat(float $price): float
    {
        return $price * (1 + $this->vatRate);  // Применение НДС
    }

    public function getVatRate(): float
    {
        return $this->vatRate;
    }
}
```

##### Изменения:

- **Константа `DEFAULT_VAT_RATE`**: НДС по умолчанию вынесен в константу, что делает код более предсказуемым и удобным для модификации.
- **Гибкость**: Теперь, если ставка НДС по умолчанию изменится, достаточно будет обновить значение в одном месте, что упростит обслуживание и тестирование системы.

##### Преимущества:

1. **Централизованное управление ставкой НДС**: Если ставка НДС изменится, можно легко обновить константу без необходимости менять логику в других частях системы.
2. **Улучшенная читаемость и поддерживаемость**: Использование констант делает код более понятным и предсказуемым, так как значение НДС теперь можно изменить в одном месте.

Таким образом, использование константы для ставки НДС улучшает модульность и гибкость кода, делая его более соответствующим стандартам разработки.

#### 2. Класс Cart 

```PHP
class Cart implements iCart 
{ 
	public array $items = []; 
	public ?Order $order = null;
	
	public function __construct(
		private iMailer $mailer,
		private VatManager $vatManager,
	) {}
	
	public function getOrder(): Order
	{
		return $this->order;
	}
	
	public function setOrder(Order $order): void 
	{ 
		$this->order = $order; 
	}
	
	public function notify(): void 
	{ 
		$this->sendMail(); 
	} 

	public function makeOrder($discount) 
	{ 
		if ($discount <= 0 || $discount > 1) { 
			throw new InvalidArgumentException('Invalid discount value'); 
		} 
		
		$totalPrice = $this->calculateTotalPriceWithDiscount($discount); 
		$this->setOrder(new Order($this->items, $totalPrice)); 
		$this->notify();
	}

	private function sendMail(): void 
	{ 
		$totalPrice = $this->calculateTotalPrice(); 
		$message = "<p><b>" . $this->getOrder()->id() . "</b> " . number_format($totalPrice, 2) . " .</p>"; 
		
		$this->mailer->sendToManagers($message); 
	}
	
	private function calculateTotalPriceWithDiscount(float $discount): float 
	{ 
		$totalPrice = 0;
		 
		foreach ($this->items as $item) { 
			$totalPrice += this->vatManager->applyVat($item->getPrice()) * (1 - $discount); 
		} 
		
		return $totalPrice; 
	}
}
```

### 2. Методы `notify()` и `sendMail()`

1. **Нарушение принципа единственной ответственности (SRP)**:  
    - Метод `notify` и, в особенности, `sendMail` нарушают принцип **SRP** (Single Responsibility Principle). Эти методы не только отправляют уведомления, но и содержат логику для составления сообщения и вычисления стоимости товаров. Это делает код менее модульным и усложняет его поддержку.
    - **Решение**: Следует разделить логику. Метод `sendMail` должен отвечать только за отправку письма, а логику расчета цены и генерации сообщений нужно вынести в отдельные классы или сервисы.
2. **Жесткая привязка к конкретному способу уведомления**:
    - В текущей реализации класс `Cart` использует метод `sendMail`, который жестко привязан к отправке уведомлений через email. Если нужно будет изменить способ уведомления (например, использовать SMS или Push-уведомления), придется модифицировать сам класс `Cart`.
    - **Решение**: Необходимо внедрить абстракцию для уведомлений. Например, можно использовать интерфейс `NotificationService`, который будет отвечать за отправку уведомлений, а уже конкретные реализации будут заниматься отправкой email, SMS и других типов уведомлений.
3. **Дублирование кода в расчетах стоимости**:
    
    - Метод `sendMail` повторно вычисляет стоимость товаров с учетом НДС, что дублирует логику, уже реализованную в других местах кода (например, в методе `makeOrder`). Это нарушение принципа **DRY** (Don't Repeat Yourself).
    - **Решение**: Расчет цены с НДС и скидкой должен быть вынесен в отдельный метод (например, в `PriceCalculator`), чтобы избежать дублирования кода.
4. **Низкая тестируемость**:
    - Прямое создание экземпляра `SimpleMailer` в методе `sendMail` усложняет тестирование класса `Cart`. Тестирование становится сложным, так как для проверки работы `sendMail` нужно будет создавать реальный объект `SimpleMailer`, который, возможно, зависит от внешних систем.
    - **Решение**: Использовать внедрение зависимостей (Dependency Injection) для `SimpleMailer`, чтобы можно было передавать mock-объекты при тестировании. Это позволит более гибко тестировать класс `Cart` без необходимости отправлять реальные письма.

#### Пример рефакторинга:

 Интерфейс для уведомлений: 

```PHP
interface iNotificationService
{
    public function sendOrderNotification(Order $order, float $totalPrice): void;
}
```

Реализация сервиса для email:

```PHP
class EmailNotificationService implements iNotificationService
{
    public function __construct(
	    private iMailer $mailer,
    ) {}

    public function sendOrderNotification(Order $order, float $totalPrice): void
    {
        $message = "<p><b>Order ID: " . $order->id() . "</b> Total: " . number_format($totalPrice, 2) . ".</p>";
        $this->mailer->sendToManagers($message);
    }
}
```

Обновление класса `Cart`:

```PHP
class Cart implements iCart 
{ 
	public array $items = []; 
	public ?Order $order = null;
	
	public function __construct(
		private iNotificationService $notificationService,
		private VatManager $vatManager,
	) {}

	public function getOrder(): Order
		{
			return $this->order;
		}
	
	public function setOrder(Order $order): void 
	{ 
		$this->order = $order; 
	}
	
	public function makeOrder($discount) 
	{ 
		if ($discount <= 0 || $discount > 1) { 
			throw new InvalidArgumentException('Invalid discount value'); 
		} 
		
		$totalPrice = $this->calculateTotalPriceWithDiscount($discount); 
		$this->setOrder(new Order($this->items, $totalPrice));
		$this->notificationService->sendOrderNotification($this->getOrder(), $totalPrice);
	}
	
	private function calculateTotalPriceWithDiscount(float $discount): float 
	{ 
		$totalPrice = 0;
		 
		foreach ($this->items as $item) { 
			$totalPrice += this->vatManager->applyVat($item->getPrice()) * (1 - $discount); 
		} 
		
		return $totalPrice; 
	}
}
```

#### Итог:

- Методы `notify` и `sendMail` нарушают несколько принципов разработки, таких как **SRP**, **DRY**, и принцип инверсии зависимостей.
- Внедрение паттерна "Стратегия" для уведомлений и использование внедрения зависимостей сделает код более гибким и легко тестируемым.
- Вынесение логики отправки сообщений и расчета цен в отдельные классы улучшит читаемость и модульность кода, а также упростит добавление новых способов уведомлений.

### 3. Метод `makeOrder`

#### 1. **Нарушение принципа единственной ответственности (SRP)**

- **Проблема**: Метод одновременно отвечает за несколько задач:
    
    - Рассчитывает общую стоимость товаров с учетом скидок.
    - Создает объект заказа.
    - Отправляет уведомления через email.
    
    Это нарушение принципа **единственной ответственности**, так как метод должен выполнять только одну задачу. Смешение логики усложняет поддержку и тестирование кода.
    
- **Решение**: Разделить обязанности метода. Выделить отдельные сервисы для:
    
    - Расчета итоговой стоимости (через `PriceCalculator`).
    - Создания заказов (через конструктор класса `Order`).
    - Уведомлений (через `NotificationService`).

#### 2. **Жесткое связывание с конкретной реализацией уведомлений**

- **Проблема**: Метод жестко привязан к отправке email-уведомлений через метод `sendMail()`. Если потребуется изменить тип уведомлений (например, отправлять SMS или push-уведомления), придется модифицировать сам метод, что нарушает принцип **открытости/закрытости** (OCP).
    
- **Решение**: Использовать паттерн "Стратегия" или внедрение зависимостей, чтобы метод был независим от конкретной реализации уведомлений. Например, можно передавать в конструктор класса `Cart` объект, реализующий интерфейс `NotificationService`.
    

#### 3. **Дублирование логики расчета стоимости**

- **Проблема**: Метод заново рассчитывает стоимость товаров с учетом скидки и НДС, что может дублировать логику, уже реализованную в других местах (например, в методе `calcVat` или других расчетных методах).
    
- **Решение**: Вынести логику расчета стоимости в отдельный класс (например, `PriceCalculator`) и использовать его внутри метода `makeOrder`. Это устранит дублирование кода и повысит его переиспользуемость.
    

#### 4. **Неявная зависимость от скидок**

- **Проблема**: Метод принимает параметр `$discount`, который влияет на расчеты. Однако метод не проверяет корректность переданных значений (например, отрицательные скидки или значения выше 100%).
    
- **Решение**: Использовать паттерн "Стратегия" для работы со скидками через интерфейс `DiscountStrategy`, а также добавить проверку входных данных, чтобы избежать некорректных значений скидки.
    

#### 5. **Отсутствие обработки ошибок**

- **Проблема**: Метод не учитывает возможные исключительные ситуации, такие как ошибки при создании заказа, расчете стоимости или отправке уведомлений. Это может привести к некорректной работе системы и трудноуловимым ошибкам.
    
- **Решение**: Добавить обработку исключений и корректную реакцию на ошибки, чтобы метод был более устойчив к проблемам на уровне выполнения.
    

#### Пример рефакторинга:

Интерфейс `DiscountStrategy`

```PHP
interface DiscountStrategy
{
    public function applyDiscount(float $price): float;
}
```

Класс `PercentageDiscount`

```PHP
class PercentageDiscount implements DiscountStrategy
{
    private const float DEFAULT_DISCOUNT_PERCENTAGE = 0.1;  // Скидка по умолчанию: 10%
    private float $percentage;

    public function __construct(?float $percentage = self::DEFAULT_DISCOUNT_PERCENTAGE)
    {
        if ($this->percentage < 0 || $this->percentage > 1) {
            throw new InvalidArgumentException('Процент скидки должен быть в диапазоне от 0 до 1.');
        }
    }
    
    public function applyDiscount(float $price): float
    {
        return $price * (1 - $this->percentage);
    }
}
```

Класс `PriceCalculator`

```PHP
class PriceCalculator
{
    public function __construct(
	    private VatManager $vatManager,
    ) {}
    
    public function calculatePriceWithDiscountAndVat(float $price, DiscountStrategy $discountStrategy): float
    {
        $discountedPrice = $discountStrategy->applyDiscount($price);
        
        return $this->vatManager->applyVat($discountedPrice);
    }
    
    public function calculateTotalPriceWithDiscount(array $items, DiscountStrategy $discountStrategy): float
    {
        $totalPrice = 0;
        
        foreach ($items as $item) {
            $price = $item->getPrice();
            $totalPrice += $this->calculatePriceWithDiscountAndVat($price, $discountStrategy);
        }
        
        return $totalPrice;
    }
}
```

Обновление класса `Cart`:

```PHP
class Cart implements iCart 
{ 
	public array $items = []; 
	public ?Order $order = null;
	
	public function __construct(
		private iNotificationService $notificationService,
		private PriceCalculator $priceCalculator,
	) {}
	
	public function getOrder(): Order
	{
		return $this->order;
	}
	
	public function setOrder(Order $order): void 
	{ 
		$this->order = $order; 
	}
	
	public function makeOrder(DiscountStrategy $discountStrategy): void 
	{ 
		try { 
			$totalPrice = $this->priceCalculator->calculateTotalPriceWithDiscount($this->items, $discountStrategy); 
			$this->setOrder(new Order($this->items, $totalPrice));  
			$this->notificationService->sendOrderNotification($this->order, $totalPrice); 
		} catch (\Exception $e) { 
			throw new RuntimeException("Ошибка при создании заказа: " . $e->getMessage()); 
		} 
	}
}
```

Интерфейс `iCart` выглядит следующим образом:

```PHP
interface iCart
{
    public function makeOrder(DiscountStrategy $discountStrategy): void;
}
```