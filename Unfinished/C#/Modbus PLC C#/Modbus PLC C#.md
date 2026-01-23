# Как я пытался считать переменные с ПЛК с помощью Modbus, чтобы потом вывести в приложение на C#

## Введение

Modbus — это открытый протокол связи, широко используемый в промышленной автоматизации для обмена данными между устройствами. Протокол работает по принципу "мастер-слейв" (master-slave), где мастер запрашивает данные у подчиненных устройств (слейвов).

В данной статье описывается... 
todo добавить основную задачу которая решает статья
для работы с Modbus в среде CODESYS и чтения переменных ПЛК (программируемого логического контроллера) в приложение на C#.

## Задача

todo опиши задачу

## Подготовка

### Структура данных

Структура данных организована следующим образом:

todo добавить что заголовок нужен чтобы определять что это за проект, так как на ПЛК может заливаться несколько проектов

- **Первые 2 регистра** — заголовок устройства (тип устройства и версия)
- **Остальные регистры** — значения переменных

Имена переменных и метаданные не хранятся в ПЛК — они жестко прописаны в карте переменных на стороне C# приложения. Карта переменных — это отдельный класс, содержащий описания всех переменных с их адресами, типами и размерами.

### Класс PlcValue

todo переделать название чтобы небыло конкретного класса

Самая простая реализация класса для представления переменной ПЛК:

```cs
public class PlcValue<T> : IEquatable<PlcValue<T>>
{
    private static readonly HashSet<string> _usedNames = new();

    public PlcValue(string name, T value, ushort address)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым", nameof(name));

        // Проверяем уникальность нового имени
        if (_usedNames.Contains(name))
            throw new ArgumentException($"Имя \"{name}\" уже используется.", nameof(name));

        _usedNames.Add(name);
        Name = name;
        Value = value!;
        Address = address;

        RegSize = CalculateRegSize(typeof(T));
        ByteSize = (UInt32)RegSize * 2;
    }
    public PlcValue(string name, T value, ushort address, ushort regSize)
        : this(name, value, address)
    {
        RegSize = regSize;
        ByteSize = (UInt32)RegSize * 2;
    }
    /// <summary>
    /// Имя переменной. Уникальное среди всех экземпляров PlcValue.
    /// Не может быть null или пустой строкой.
    /// </summary>
    public string Name { get; }
    /// <summary>
    /// Значение
    /// </summary>
    public T Value { get; }
    /// <summary>
    /// Тип. НЕ null
    /// </summary>
    public Type CSType => typeof(T);
    /// <summary>
    /// Номер регистра. По умолчанию с 0
    /// </summary>
    public UInt16 Address { get; }
    /// <summary>
    /// Размер в байтах. Если не задан, вычисляется автоматически на основе типа.
    /// </summary>
    public UInt32 ByteSize { get; }
    /// <summary>
    /// Размер в регистрах. Если не задан, вычисляется автоматически на основе типа.
    /// </summary>
    public UInt16 RegSize { get; }

    public bool Equals(PlcValue<T>? other)
    {
        if (other is null)
            return false;

        if (ReferenceEquals(Value, other.Value))
            return true;

        if (Value is Array a1 && other.Value is Array a2)
            return a1.Length == a2.Length &&
                   a1.Cast<object>().SequenceEqual(a2.Cast<object>());

        return EqualityComparer<T>.Default.Equals(Value, other.Value);
    }

    public override bool Equals(object? obj)
        => obj is PlcValue<T> other && Equals(other);

    public override int GetHashCode()
        => HashCode.Combine(Name, Value, CSType, Address);

    public static bool operator ==(PlcValue<T>? left, PlcValue<T>? right)
        => object.Equals(left, right);

    public static bool operator !=(PlcValue<T>? left, PlcValue<T>? right)
        => !object.Equals(left, right);

    /// <summary>
    /// Вычисляет размер в регистрах на основе типа.
    /// </summary>
    private static ushort CalculateRegSize(Type type)
    {
        if (type.IsEnum)
            type = Enum.GetUnderlyingType(type);

        return Type.GetTypeCode(type) switch
        {
            TypeCode.Boolean or
            TypeCode.Byte or
            TypeCode.SByte or
            TypeCode.Int16 or
            TypeCode.UInt16 => 1,

            TypeCode.Int32 or
            TypeCode.UInt32 or
            TypeCode.Single or
            TypeCode.DateTime => 2,

            TypeCode.Int64 or
            TypeCode.UInt64 or
            TypeCode.Double => 4,

            _ => 1 // массивы, строки — задаются вручную
        };
    }
}
```

если хотите добавлять его в массив то используйте интерфейс IPlcValue

```cs
public interface IPlcValue
{
    /// <summary>
    /// Имя переменной. Уникальное среди всех экземпляров PlcValue.
    /// Не может быть null или пустой строкой.
    /// </summary>
    public string Name { get; }
    /// <summary>
    /// Тип. НЕ null
    /// </summary>
    public Type CSType { get; }
    /// <summary>
    /// Номер регистра. По умолчанию с 0
    /// </summary>
    public UInt16 Address { get; }
    /// <summary>
    /// Размер в регистрах. Если не задан, вычисляется автоматически на основе типа.
    /// </summary>
    public UInt16 RegSize { get; }
}
```

RegSize добавлен из за удобства программирования, если он вам не нужен можете убрать

тогда у нас остается проблема с типом. Нельзя так просто создать PlcValue не зная тип заранее, если тип определяется в рантайме

```cs
IPlcValue[] templates = GetVarTemplates();
IPlcValue[] result = new IPlcValue[templates.Length];
for (int i = 0; i < templates.Length; i++)
   IPlcValue template = templates[i];
   object? value = ModbusValueMarshaler.Marshal(slice, template.CSType);
   result[i] = new PlcValue<typeof(template.CSType) > (template.Name, (template.CSType)value, template.Address);
}
```

есть несколько вариантов как это можно исправить

1 вариант:

использовать рефлексию

```cs
result[i] = (IPlcValue)Activator.CreateInstance(
    typeof(PlcValue<>).MakeGenericType(template.CSType),
    template.Name,
    value!,
    template.Address
)!;
```

2 вариант:

Добавь фабрику в IPlcValue

```cs
public interface IPlcValue
{
    //остальной код
    IPlcValue CreateNew(object value);
}

public class PlcValue<T> : IPlcValue
{
    //остальной код
    public IPlcValue CreateNew(object value) => new PlcValue<T>(Name, (T)value, Address);
}
```

в коде

```cs
result[i] = template.CreateNew(value);
```


## Чтение данных

Для чтения данных из ПЛК можно реализовать несколько подходов:

### Вариант 1: Раздельные функции

- Отдельная функция для чтения заголовка
- Отдельная функция для чтения значений переменных

### Вариант 2: Единая функция
todo дописать что этот вариант выбран

Одна функция `ReadAll()`, которая:

- Читает все регистры одним Modbus-запросом (оптимизация производительности)
- Автоматически распределяет регистры на заголовок и значения переменных
- Возвращает структурированный объект `MBDataScheme` с заголовком и массивом значений

**Преимущества единой функции:**

- Минимум Modbus-запросов (один запрос вместо нескольких)
- Атомарность чтения — все данные читаются в один момент времени
- Простота использования

**Пример использования:**

```csharp
MBReader reader = new MBReader(master, varMap);
MBDataScheme data = reader.ReadAll(slaveId: 1);

// Доступ к заголовку
Console.WriteLine($"Device Type: {data.Header.DevType}");
Console.WriteLine($"Version: {data.Header.Version}");

// Доступ к переменным
foreach (var plcValue in data.PlcValues)
{
    Console.WriteLine($"{plcValue.Name}: {plcValue.Value}");
}
```

### Особенности работы с типами данных

**Строки:**

- Проблема: длина строки не всегда очевидна заранее
- Решение: первый регистр содержит длину строки, остальные — символы
- В карте переменных для строк необходимо явно указывать `RegSize`

**Enum:**

- Enum в CODESYS хранится как базовый целочисленный тип (обычно UInt16)
- При чтении используется базовый тип, затем значение преобразуется в enum
- Требуется явное указание базового типа в маршаллере

## Запись данных

### Базовый подход: запись всех переменных

Можно записывать все переменные одним запросом:

```csharp
MBWriter writer = new MBWriter(master);
PlcValue[] allValues = { /* все переменные */ };
writer.WriteValues(slaveId: 1, allValues);
```

### Оптимизация: запись только измененных переменных

Для повышения производительности можно записывать только те переменные, которые изменились:

**Реализация:**

- Использовать отдельный массив/список для хранения измененных значений
- Использовать словарь для быстрого поиска переменных по имени
- Имя переменной уникально, поэтому его можно использовать в качестве ключа в словаре

**Пример оптимизированной записи:**

```csharp
// Словарь для отслеживания изменений
Dictionary<string, PlcValue> changedValues = new();

// При изменении переменной
void OnVariableChanged(string name, object newValue)
{
    var template = varMap.GetVariable(name);
    var updatedValue = new PlcValue
    {
        Name = template.Name,
        Address = template.Address,
        Type = template.Type,
        Value = newValue,
        RegSize = template.RegSize
    };
    
    changedValues[name] = updatedValue;
}

// Периодическая запись изменений
void FlushChanges()
{
    if (changedValues.Count > 0)
    {
        PlcValue[] valuesToWrite = changedValues.Values.ToArray();
        writer.WriteValues(slaveId: 1, valuesToWrite);
        changedValues.Clear();
    }
}
```

**Преимущества:**

- Меньше Modbus-запросов
- Меньше нагрузка на сеть и ПЛК
- Быстрее отклик приложения

## Константы

Встает вопрос: как хранить константы, которые используются в приложении?

### Варианты хранения констант

1. **Статические поля в классе карты переменных**

```csharp

1. **Статические поля в классе карты переменных**
   ```csharp
   public class MyVarMap
   {
       public const ushort HEADER_ADDRESS = 10;
       public const ushort HEADER_SIZE = 2;
       // ...
   }
   ```

2. **Отдельный класс с константами**

```csharp
   public static class ModbusConstants
   {
       public const ushort HEADER_ADDRESS = 10;
       public const ushort HEADER_SIZE = 2;
       public const int DEFAULT_TIMEOUT_MS = 5000;
   }
   ```

3. **Конфигурационный файл** (для значений, которые могут меняться)
   - JSON, XML или appsettings.json
   - Позволяет менять значения без перекомпиляции

4. **Атрибуты на свойствах** (для метаданных)

```csharp
   [ModbusAddress(100)]
   [ModbusType(typeof(float))]
   public PlcValue Temperature { get; set; }
   ```

## Работа с байтами

В отличие от чтения файлов, где структура данных фиксирована, при работе с Modbus:

- Заголовок и структура данных могут меняться в зависимости от версии протокола
- Адреса переменных могут отличаться для разных типов устройств
- Если вы хотите обращаться к элементам через точку (например, `data.Temperature`), лучше использовать класс карты переменных с типизированными свойствами

**Пример типизированного доступа:**

```csharp
public class MyVarMap
{
    public PlcValue<float> Temperature { get; set; }
    public PlcValue<ushort> Status { get; set; }
}

// Использование
var data = reader.ReadAll(slaveId);
float temp = (float)data.PlcValues
    .First(v => v.Name == "Temperature")
    .Value;
```

## Подключение и мониторинг соединения

Так как Modbus работает по обычному TCP соединению, у него нет встроенных способов определить, разорвалось ли соединение или произошла ошибка во время выполнения программы. Для этого нужен механизм проверки соединения (heartbeat).

### Проблема с циклом while

Цикл `while` будет опрашивать соединение каждый кадр, что избыточно и создает большую нагрузку. Лучше использовать таймер с определенным интервалом.

### Варианты проверки подключения

#### 1. Проверка через чтение регистра (рекомендуется)

Вместо ping можно попытаться прочитать один регистр. Если чтение проходит успешно, значит Modbus-сервер подключен:

```csharp
try
{
    // Минимальный Modbus-запрос (heartbeat)
    _master.ReadHoldingRegisters(_slaveId, 0, 1);
    IsConnected = true;
    
    // Читаем данные
    ReadVariables();
}
catch
{
    Disconnect();
}
```

**Преимущества:**

- Проверяет не только TCP-соединение, но и работоспособность Modbus-протокола
- Минимальная нагрузка (чтение одного регистра)
- Надежный способ определения состояния устройства

#### 2. Проверка через сокет (альтернативный способ)

```csharp
private bool IsConnected()
{
    if (_client?.Client == null)
        return false;

    try
    {
        var socket = _client.Client;
        return !(socket.Poll(1, SelectMode.SelectRead) && socket.Available == 0);
    }
    catch
    {
        return false;
    }
}
```

**Недостатки:**

- Проверяет только TCP-соединение, но не Modbus-протокол
- Может давать ложные положительные результаты, если TCP-соединение есть, но Modbus-устройство не отвечает

### Реализация автоматического переподключения

Для автоматического управления соединением используется класс `ModbusReconnectionTask`:

```csharp
var reconnectionTask = new ModbusReconnectionTask(
    ipAddress: "192.168.1.100",
    port: 502,
    slaveId: 1,
    varMap: myVarMap,
    checkIntervalMs: 1000
);

// Подписка на события
reconnectionTask.ConnectionStatusChanged += (sender, isConnected) =>
{
    Console.WriteLine($"Connection: {(isConnected ? "Connected" : "Disconnected")}");
};

reconnectionTask.DataRead += (sender, data) =>
{
    // Обработка прочитанных данных
    ProcessData(data);
};

reconnectionTask.ReadError += (sender, ex) =>
{
    Console.WriteLine($"Error: {ex.Message}");
};
```

**Особенности:**

- Автоматическое переподключение при потере связи
- Настраиваемый интервал проверки
- События для уведомления об изменении состояния
- Потокобезопасная реализация

## Маршаллер (ModbusValueMarshaler)

Так как размер переменных в ПЛК не всегда соответствует размеру переменных в C#, нужен маршаллер, который будет преобразовывать сырые данные из регистров Modbus в типизированные значения C# и обратно.

### Проблема преобразования типов

Modbus работает с 16-битными регистрами (UInt16), но в C# используются различные типы:

- `bool`, `byte`, `sbyte`, `Int16`, `UInt16` — 1 регистр
- `Int32`, `UInt32`, `float`, `TimeSpan`, `DateTime` — 2 регистра
- `Int64`, `UInt64`, `double` — 4 регистра
- `string` — переменное количество регистров
- Массивы — зависят от размера элемента и длины массива

### Реализация маршаллера

Класс `ModbusValueMarshaler` предоставляет два основных метода:

#### 1. Marshal — преобразование из регистров в C# типы

```csharp
UInt16[] raw = { 0x1234, 0x5678 };
object? value = ModbusValueMarshaler.Marshal(raw, typeof(UInt32));
// Результат: 0x12345678
```

**Поддерживаемые типы:**

- Примитивные типы: `bool`, `byte`, `sbyte`, `UInt16`, `Int16`, `UInt32`, `Int32`, `UInt64`, `Int64`
- Числа с плавающей точкой: `float`, `double`
- Специальные типы: `TimeSpan` (миллисекунды), `DateTime` (Unix timestamp в секундах)
- Строки: первый регистр — длина, остальные — символы
- Массивы: преобразование каждого элемента
- Enum: преобразование через базовый тип

#### 2. Unmarshal — преобразование из C# типов в регистры

```csharp
float value = 25.5f;
UInt16[] regs = ModbusValueMarshaler.Unmarshal(value, typeof(float));
// Результат: массив из 2 регистров с битовым представлением float
```

**Особенности:**


- Порядок байтов: старшие биты в первом регистре (big-endian)
- Для строк требуется указание размера буфера
- Enum преобразуется в базовый целочисленный тип

### Примеры преобразований

**UInt32 (2 регистра):**

```csharp
// Чтение: [0x1234, 0x5678] → 0x12345678
UInt32 value = ((UInt32)raw[0] << 16) | raw[1];

// Запись: 0x12345678 → [0x1234, 0x5678]
regs[0] = (UInt16)(value >> 16);
regs[1] = (UInt16)(value & 0xFFFF);
```

**Float (2 регистра):**

```csharp
// Чтение: регистры → биты → float
UInt32 bits = ((UInt32)raw[0] << 16) | raw[1];
float value = BitConverter.Int32BitsToSingle((Int32)bits);

// Запись: float → биты → регистры
Int32 bits = BitConverter.SingleToInt32Bits(value);
UInt32 u = unchecked((UInt32)bits);
regs[0] = (UInt16)(u >> 16);
regs[1] = (UInt16)(u & 0xFFFF);
```

**Строки:**

```csharp
// Формат: [длина, символ1, символ2, ...]
// Чтение
int length = raw[0];
char[] chars = new char[length];
for (int i = 0; i < length; i++)
    chars[i] = (char)raw[i + 1];
string result = new string(chars);

// Запись
regs[0] = (UInt16)str.Length;
for (int i = 0; i < str.Length; i++)
    regs[i + 1] = str[i];
```

## Вывод

Создание библиотеки для работы с Modbus в C# требует решения нескольких задач:

1. **Структурирование данных**: Определение карты переменных с метаданными (адреса, типы, размеры)
2. **Эффективное чтение**: Минимизация количества Modbus-запросов через группировку регистров
3. **Преобразование типов**: Маршаллинг между регистрами Modbus и типами C#
4. **Управление соединением**: Автоматическое переподключение и мониторинг состояния
5. **Оптимизация записи**: Запись только измененных переменных для снижения нагрузки

### Ключевые решения

- ✅ Использование единого запроса для чтения всех данных
- ✅ Типизированная карта переменных через класс `PlcValue<T>`
- ✅ Централизованный маршаллер для преобразования типов
- ✅ Автоматическое управление переподключением через `ModbusReconnectionTask`
- ✅ Событийная модель для уведомления об изменениях

### Результат

Получилась удобная которая:

- Абстрагирует низкоуровневые детали работы с Modbus
- Предоставляет типизированный API для работы с переменными ПЛК
- Автоматически управляет соединением и переподключением
- Оптимизирует производительность через минимизацию запросов

## Ссылки

1. [CODESYS Setup - drag&bot Help](https://help.dragandbot.com/modules/protocols/modbus/03_codesys.html)
2. [NModbus Library](https://github.com/NModbus/NModbus) — библиотека для работы с Modbus в .NET
3. [Modbus Protocol Specification](https://modbus.org/specs.php) — официальная спецификация протокола Modbus
4. [CODESYS Modbus Documentation](https://help.codesys.com/) — документация по настройке Modbus в CODESYS
5. [ПЛК HCFA опыт разработки и немного эксплуатации](https://www.asutpforum.ru/viewtopic.php?t=19136)
