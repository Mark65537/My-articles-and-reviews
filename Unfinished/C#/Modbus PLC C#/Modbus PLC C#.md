# Как я читал переменные из ПЛК по Modbus и выводил их в C#-приложение

## Введение

Modbus — это открытый и очень распространённый протокол обмена данными в промышленной автоматизации. Он работает по модели master–slave: мастер (например, PC-приложение) запрашивает данные у ведомого устройства (ПЛК), получая или записывая значения регистров.

На практике Modbus кажется простым — всего лишь массив 16-битных регистров. Но как только возникает задача читать типизированные переменные, поддерживать несколько проектов в одном ПЛК, минимизировать количество запросов и безопасно работать с соединением, всё быстро усложняется.

В этой статье я описываю реальный подход, который использовал для чтения переменных из ПЛК (CODESYS + Modbus TCP) и отображения их в приложении на C#.

## Задача

Исходные требования были такие:

* Читать значения переменных из ПЛК по Modbus TCP
* Работать со всеми типами данных (bool, числа, enum, строки и т.д.)
* Поддерживать ситуацию, когда в ПЛК могут быть загружены разные проекты
* Минимизировать количество Modbus-запросов
* Иметь удобный, типизированный API на стороне C#

Обеспечить контроль соединения и автоматическое переподключение

Из этого сразу следует важный вывод:
Modbus — это транспорт, а вся структура данных должна быть на стороне приложения.

## Структура данных

Вся область регистров делится на две логические части:

1. Заголовок проекта
2. Значения переменных

>Имена переменных и их метаданные не хранятся в ПЛК.
>Они жёстко описаны в карте переменных на стороне C#.

### Заголовок

В заголовке хранится служебная информация о загруженном в ПЛК проекте.

На первом этапе он задумывался как простая структура данных, но довольно быстро стало ясно, что такой подход не задаёт никаких правил: легко забыть добавить поле, перепутать порядок или создать «неполноценный» заголовок, который формально существует, но по смыслу бесполезен. Оформление заголовка в виде класса решает эту проблему. Класс явно описывает, какие данные обязаны присутствовать в заголовке, и заставляет пользователя указать их при создании объекта. Таким образом, сам код становится документацией: из конструктора и свойств сразу видно, что именно считается корректным заголовком и без каких полей он не может существовать.

Размер заголовка остаётся фиксированным, при этом его можно вычислять автоматически, например с использованием рефлексии.

### Зачем нужен заголовок

Заголовок нужен для того, чтобы определить:

- какой проект сейчас загружен в ПЛК
- совместима ли версия проекта с приложением

Это важно, потому что:

- в один и тот же ПЛК могут загружаться разные проекты
- структура регистров может отличаться

Читать «чужую» карту переменных — прямой путь к ошибкам.

### Формат заголовка

Здесь представлена таблица с описанием минимально необходимых полей заголовка:

| Offset from | Size | Note                                           |
|-------------|------|------------------------------------------------|
| 0           | 1    | тип проекта. например ВФУ |
| 1           | 1    | версия проекта                    |

### Класс PlcValue

Todo переделать название чтобы не было конкретного класса

Самая простая реализация класса для представления переменной ПЛК:

```cs
/// <summary>
/// Представляет значение переменной ПЛК, прочитанной по Modbus.
/// </summary>
[DebuggerDisplay("Name: {Name} Value: {Value} Address: {Address} RegSize: {RegSize} ByteSize: {ByteSize}")]
public class PlcValue : IEquatable<PlcValue>, INotifyPropertyChanged
{
    private static readonly HashSet<string> _usedNames = new();

    private object? _value;

    public PlcValue(string name, Type type, UInt16 address)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым", nameof(name));

        // Проверяем уникальность нового имени
        if (!_usedNames.Add(name))
            throw new ArgumentException($"Имя \"{name}\" уже используется.", nameof(name));

        CSType = type ?? throw new ArgumentNullException(nameof(type));
        Name = name;
        Address = address;
        RegSize = CalculateRegSize(type);
        ByteSize = (UInt32)RegSize * 2;
    }
    public PlcValue(string name, object value, Type type, UInt16 address) : this(name, type, address)
    {
        Value = ConvertToCSType(value, CSType);
    }
    public PlcValue(string name, Type type, ushort address, UInt16 regSize)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым", nameof(name));

        // Проверяем уникальность нового имени
        if (!_usedNames.Add(name))
            throw new ArgumentException($"Имя \"{name}\" уже используется.", nameof(name));

        if (regSize == 0)
            throw new ArgumentOutOfRangeException(nameof(regSize), "Размер регистра должен быть больше 0.");

        CSType = type ?? throw new ArgumentNullException(nameof(type));
        Name = name;
        Address = address;

        RegSize = regSize;
        ByteSize = (uint)regSize * 2;
    }
    public PlcValue(string name, object value, Type type, UInt16 address, UInt16 regSize)
        : this(name, value, type, address)
    {
        RegSize = regSize;
        ByteSize = (UInt32)RegSize * 2;
    }
    public PlcValue(string name, object value, Type type, UInt16 address, UInt32 byteSize) :
        this(name, value, type, address)
    {
        RegSize = (ushort)((byteSize + 1) / 2);
        ByteSize = byteSize;
    }

    /// <summary>
    /// Имя переменной. Уникальное среди всех экземпляров PlcValue.
    /// Не может быть null или пустой строкой.
    /// </summary>
    public string Name { get; }
    /// <summary>
    /// Значение. Может быть не задано.
    /// При изменении поднимает событие <see cref="PropertyChanged"/>,
    /// чтобы привязки (WPF) могли обновить UI.
    /// </summary>
    public object? Value
    {
        get => _value;
        set
        {
            if (!Equals(_value, value))
            {
                _value = value;
                OnPropertyChanged(nameof(Value));
            }
        }
    }
    /// <summary>
    /// Тип. НЕ null
    /// </summary>
    public Type CSType { get; }
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

    public event PropertyChangedEventHandler? PropertyChanged;

    public bool Equals(PlcValue? other)
    {
        if (other is null)
            return false;

        if (ReferenceEquals(Value, other.Value))
            return true;

        if (Value is Array a1 && other.Value is Array a2)
            return a1.Length == a2.Length &&
                   a1.Cast<object>().SequenceEqual(a2.Cast<object>());

        return EqualityComparer<object?>.Default.Equals(Value, other.Value);
    }

    public override bool Equals(object? obj)
        => obj is PlcValue other && Equals(other);

    public override int GetHashCode()
        => HashCode.Combine(Name, Value, CSType, Address);

    public static bool operator ==(PlcValue? left, PlcValue? right)
        => Equals(left, right);

    public static bool operator !=(PlcValue? left, PlcValue? right)
        => !Equals(left, right);

    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    private static object ConvertToCSType(object value, Type targetType)
    {
        if (value == null)
            throw new ArgumentNullException(nameof(value));

        Type valueType = value.GetType();

        // Уже совместимо
        if (targetType.IsAssignableFrom(valueType))
            return value;

        try
        {
            // Enum
            if (targetType.IsEnum)
            {
                if (value is string s)
                    return Enum.Parse(targetType, s, ignoreCase: true);

                return Enum.ToObject(targetType, value);
            }

            bool sourceIsFloating =
                value is float ||
                value is double ||
                value is decimal;

            bool targetIsInteger =
                targetType == typeof(byte) ||
                targetType == typeof(sbyte) ||
                targetType == typeof(short) ||
                targetType == typeof(ushort) ||
                targetType == typeof(int) ||
                targetType == typeof(uint) ||
                targetType == typeof(long) ||
                targetType == typeof(ulong);

            // ❗ КРИТИЧЕСКАЯ ПРОВЕРКА
            if (sourceIsFloating && targetIsInteger)
            {
                double d = Convert.ToDouble(value);

                if (d % 1 != 0)
                    throw new InvalidCastException(
                        $"Дробное значение {d} не может быть приведено к целочисленному типу '{targetType.FullName}'."
                    );
            }

            return Convert.ChangeType(value, targetType);
        }
        catch (InvalidCastException)
        {
            throw;
        }
        catch (Exception ex)
        {
            throw new InvalidCastException(
                $"Невозможно привести значение типа '{valueType.FullName}' к типу '{targetType.FullName}'.",
                ex
            );
        }
    }

    /// <summary>
    /// Вычисляет размер в регистрах на основе типа.
    /// </summary>
    private static ushort CalculateRegSize(Type type)
    {
        if (type.IsEnum)
            type = Enum.GetUnderlyingType(type);

        if (type == typeof(TimeSpan))
            return 2;

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

todo переструктурируй

Если хотите чтобы тип был генерик, то вам нужен интерфейс, но я бы не советовал так делать так как потом нужно будет использовать рефлексию.

Если хотите добавлять его в массив, то используйте интерфейс IPlcValue

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

RegSize добавлен из-за удобства программирования, если он вам не нужен можете убрать

Тогда у нас остается проблема с типом. Нельзя так просто создать PlcValue не зная тип заранее, если тип определяется в рантайме

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

### Тестирование

Для полного тестирования нужен некий консольный эмулятор, который просто открывает TCP соединение и отдает и принимает регистры.
Вообще его реализация довольно простая.

Но можно обойтись без него.

если вы знаете какие  вас регистры и что вы должны получить то можно написать подобный тест:

> NOTE
> код очень длинный из-за больших массивов, поэтому я опишу только его основные части

```csharp

```

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

Встает вопрос: как хранить константы, которые нужны для протокола Modbus?

### Что точно нужно для констант

1. Адресс начала
 ...
fix можно добавить тип регистра

### Варианты хранения констант

1. **Статические поля в классе Reader**

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

5. **В классах которые им принадлежат**

например если это класс заголовка то там храняться константы связанные с заголовком

```csharp
public class ModbusHeader
{
    public const ushort HEADER_ADDRESS = 10;
    public const ushort HEADER_SIZE = 2;
}
```

## Подключение и мониторинг соединения

Так как Modbus работает по обычному TCP соединению, у него нет встроенных способов определить, разорвалось ли соединение или произошла ошибка во время выполнения программы. Для этого нужен механизм проверки соединения.

### Проблема с циклом while

todo нужно убрать этот параграф а то что внутри переместить в другое место

Цикл `while` будет опрашивать соединение каждый кадр, что избыточно и создает большую нагрузку. Лучше использовать таймер с определенным интервалом.

### Варианты проверки подключения

#### 1. Проверка через чтение регистра

Вместо ping можно попытаться прочитать один регистр. Если чтение проходит успешно, значит Modbus-сервер подключен:

```csharp
try
{
    // Минимальный Modbus-запрос (heartbeat)
    _master.ReadHoldingRegisters(_slaveId, 0, 1);
    IsConnected = true;
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

#### 2. Проверка через сокет

Данный вариант еще не тестировался так что не могу полностью утверждать о его работоспособности.

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

Для автоматического управления соединением, лучше использовать отдельный класс, назовем его: `ModbusReconnectionTask`.
Пример его использования:

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

todo надо как то один раз написать про типы

- `bool`, `byte`, `sbyte`, `Int16`, `UInt16` — 1 регистр
- `Int32`, `UInt32`, `float`, `TimeSpan`, `DateTime` — 2 регистра
- `Int64`, `UInt64`, `double` — 4 регистра
- `string` — переменное количество регистров
- Массивы — зависят от размера элемента и длины массива

> NOTE
> еще есть структуры с которыми пока непонятно как работать

### Реализация маршаллера

Класс `ModbusValueMarshaler` предоставляет два основных метода:

1. Marshal — преобразование из регистров в C# типы

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

2. Unmarshal — преобразование из C# типов в регистры

```csharp
float value = 25.5f;
UInt16[] regs = ModbusValueMarshaler.Unmarshal(value, typeof(float));
// Результат: массив из 2 регистров с битовым представлением float
```

**Особенности:**

- Порядок байтов: старшие биты в первом регистре (big-endian)
- Для строк требуется указание размера буфера
- Enum преобразуется в базовый целочисленный тип
лучше всего Enum приравнивать к UInt16

```cs
public enum DevType : UInt16
{
    VFU = 1,
}
```

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

todo переписать чтобы он не был структурированным а шел как текст

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
