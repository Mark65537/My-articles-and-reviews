# Как я читал переменные из ПЛК по Modbus и выводил их в C#-приложение

## Введение

Modbus — это открытый и очень распространённый протокол обмена данными в промышленной автоматизации. Он работает по модели master–slave: мастер (например, PC-приложение) запрашивает данные у ведомого устройства (ПЛК), получая или записывая значения регистров.

На практике Modbus кажется простым — всего лишь массив 16-битных регистров. Но как только возникает задача читать типизированные переменные, поддерживать несколько проектов в одном ПЛК, минимизировать количество запросов и безопасно работать с соединением, всё быстро усложняется.

В этой статье я описываю реальный подход, который использовал для чтения переменных из ПЛК (CODESYS + Modbus TCP) и отображения их в приложении на C#.

## Задача

Необходимо организовать работу с ПЛК по Modbus TCP так, чтобы приложение на C# взаимодействовало не с «сырыми» 16-битными регистрами, а с типизированными переменными. При этом важно учитывать, что в ПЛК могут загружаться разные проекты с отличающейся структурой данных, поэтому требуется механизм идентификации проекта и проверки его версии. Система должна не только читать регистры, но и корректно записывать значения обратно в ПЛК, минимизируя количество Modbus-запросов и обеспечивая согласованность данных. Также необходимо реализовать преобразование типов между регистрами Modbus и типами C#, а также контроль соединения с возможностью автоматического переподключения. Таким образом, задача заключается в построении устойчивой и расширяемой архитектуры поверх Modbus, а не просто в выполнении операций чтения и записи.

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

#### Зачем нужен заголовок

Заголовок нужен для того, чтобы определить:

- какой проект сейчас загружен в ПЛК
- совместима ли версия проекта с приложением

Это важно, потому что:

- в один и тот же ПЛК могут загружаться разные проекты
- структура регистров может отличаться

Читать «чужую» карту переменных — прямой путь к ошибкам.

#### Формат заголовка

Здесь представлена таблица с описанием минимально необходимых полей заголовка:

| Offset from | Size | Note                                           |
|-------------|------|------------------------------------------------|
| 0           | 1    | тип проекта(Enum). например ВФУ |
| 1           | 1    | версия проекта                    |

### Описание ПЛК-переменной

Ниже приведён упрощённый пример класса, который описывает переменную ПЛК и её расположение в карте регистров. Это минимальная версия — без проверок уникальности, без перегруженных операторов сравнения и сложной логики проверки типов.

>Полную реализацию можно посмотреть в GitHub Gist

```cs
/// <summary>
/// Описывает переменную ПЛК и её расположение в Modbus-регистрах.
/// </summary>
public class ModbusVariable : INotifyPropertyChanged
{
    private object? _value;

    public ModbusVariable(string name, Type type, ushort address, ushort? regSize = null)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым", nameof(name));

        Name = name;
        CSType = type ?? throw new ArgumentNullException(nameof(type));
        Address = address;
        RegSize = regSize ?? CalculateRegSize(type);
    }

    public string Name { get; }

    public Type CSType { get; }

    public ushort Address { get; }

    public ushort RegSize { get; }

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

    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

}
```

>RegSize добавлен для удобства при расчёте смещений. Если он не нужен — его можно убрать.

#### Генерики или нет?

Идея сделать `ModbusVariable<T>` выглядит привлекательно: тип значения фиксируется на этапе компиляции, код становится строже и безопаснее. Однако в реальной задаче работы с ПЛК тип переменной часто известен только во время выполнения — он приходит из карты переменных, полученной через рефлексию.

В таком сценарии использование generics начинает усложнять архитектуру. Проще хранить тип в отдельном свойстве, а тип Value делать object, потому что именно так мы можем динамически создавать переменные на основании метаданных.

Если всё же требуется generic-реализация, без интерфейса не обойтись — иначе невозможно собрать коллекцию переменных разных типов. Например:

```cs
public interface IModbusVariable
{
    string Name { get; }
    Type CSType { get; }
    ushort Address { get; }
    ushort RegSize { get; }
}
```

Можно конечно добавлить в интерфейс `object? Value { get; set; }`, но это не имеет большого смысла, так как одна из целей использования generics — как раз уйти от универсального object. Типизированные переменные формируются на основе карты, после чего необходимо инициализировать конкретные реализации интерфейса. Проблема в том, что даже если тип хранится в свойстве `CSType`, нельзя написать что-то вроде `new ModbusVariable<typeof(template.CSType)>(...)`, потому что параметр generic-типа должен быть известен на этапе компиляции. Рассмотрим пример:

```cs
IModbusVariable[] templates = GetVarTemplates();// получаем массив ModbusVariable, преобразовав некий класс через рефлексию

for (int i = 0; i < templates.Length; i++)
{
    var template = templates[i];
    object? value = ModbusValueMarshaler.Marshal(slice, template.CSType);
    template = new ModbusVariable<typeof(template.CSType)> (template.Name, (template.CSType)value, template.Address);
}
```

Такой код не скомпилируется именно из-за того, что T нельзя определить во время выполнения. Возникает вопрос — как создать типизированный объект, если тип известен только в рантайме?
Первый вариант — использовать рефлексию:

```cs
result[i] = (IModbusVariable)Activator.CreateInstance(
    typeof(ModbusVariable<>).MakeGenericType(template.CSType),
    template.Name,
    value!,
    template.Address
)!;
```

Второй вариант — добавить фабричный метод в интерфейс, чтобы каждая реализация сама знала, как создать новый экземпляр своего типа:

```cs
public interface IModbusVariable
{
    string Name { get; }
    Type CSType { get; }
    ushort Address { get; }
    ushort RegSize { get; }

    IModbusVariable CreateNew(object value);
}
```

Реализация:

```cs
public class ModbusVariable<T> : IModbusVariable
{
    // ...

    public IModbusVariable CreateNew(object value)
        => new ModbusVariable<T>(Name, (T)value, Address);
}
```

Такой подход позволяет убрать прямое использование рефлексии из основного кода и делает архитектуру более чистой и контролируемой.

## Чтение данных

Для чтения данных из ПЛК можно реализовать несколько подходов:

**Вариант 1. Раздельные функции:**

- Отдельная функция для чтения заголовка
- Отдельная функция для чтения значений переменных

```cs
public ST_Header ReadHeader(UInt16[] regs)
{
    return new ST_Header
    {
        DevType = (DevType)regs[0],
        Version = regs[1]
    };
}

public PlcValue[] ReadValues(byte slaveId)
{
    PlcValue[] vars = GetVarTemplates();
    int countVars = vars.Length;
    if (countVars == 0)
        return Array.Empty<PlcValue>();
    ushort startAdr = vars[0].Address;
    ushort endAdr = startAdr;
    for (int i = 0; i < countVars; i++)
    {
        ushort lastAdr = (ushort)(
            vars[i].Address +
            vars[i].RegSize - 1);
        if (lastAdr < startAdr) startAdr = vars[i].Address;
        if (lastAdr > endAdr) endAdr = lastAdr;
    }
    ushort count = (ushort)(endAdr - startAdr + 1);
    ushort[] regs = _master.ReadHoldingRegisters(slaveId, startAdr, count);
    PlcValue[] result = new PlcValue[countVars];
    for (int i = 0; i < countVars; i++)
    {
        PlcValue template = vars[i];
        int varOffset = template.Address - startAdr;
        int regCount = template.RegSize;
        try
        {

            // берем столько регистров сколько нужно
            UInt16[] slice = new UInt16[regCount];
            //копирует байты в slice. Написано что этот метод быстрее
            Buffer.BlockCopy(
                regs, varOffset * sizeof(UInt16),
                slice, 0,
                regCount * sizeof(UInt16));

            template.Value = ModbusValueMarshaler.Marshal(slice, template.CSType);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Не могу конвертировать элемент с индексом: {i}", ex);
        }
    }
    return result;
}
```

**Вариант 2. Единая функция:**

Одна функция `ReadAll()`, которая:

- Читает все регистры одним Modbus-запросом
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

## Тестирование

Для полноценного тестирования можно использовать простой консольный эмулятор, который открывает TCP-соединение, принимает и возвращает массив регистров. Реализация такого эмулятора достаточно проста: по сути это TCP-сервер с минимальной логикой обработки Modbus-запросов. Однако на начальном этапе можно обойтись и без него.

Чтение регистров вполне реально протестировать изолированно. Если заранее известно, какие значения лежат в регистрах и какие данные должны получиться после преобразования, можно сформировать тестовый массив UInt16[] и проверить корректность маппинга регистров в типизированные переменные.

> NOTE
> Код получается довольно объёмным из-за больших массивов, поэтому ниже приведены только ключевые части.

```cs
    private readonly UInt16[] allRegs =
[
    1,   // [0] адрес 10 - DevType.VFU = 1 (читается ReadHeader)
    11,  // [1] адрес 11 - Version = 11 (читается ReadHeader)
    1,   // [2] адрес 12 - BoolVar = true
    13,  // [3] адрес 13 - ByteVar = 13

    // TODO остальные простые типы

    (UInt16)'H', // [33] адрес 43
    (UInt16)'e', // [34] адрес 44
    (UInt16)'l', // [35] адрес 45
    (UInt16)'l', // [36] адрес 46
    (UInt16)'o', // [37] адрес 47

    // TODO остальные сложные типы
];

    private readonly PlcValue[] templates =
{
    new PlcValue("BoolVar", typeof(bool), 12),
    new PlcValue("ByteVar", typeof(byte), 13),

    // TODO остальные типы
};

[Fact]
public void Test_RegsToPlcValues()
{
    // Arrange
    MBReader reader = new(null, null!);

    // Act
    MBDataScheme result = reader.RegsToPlcValues(allRegs, templates);

    // Assert - проверка заголовка
    Assert.NotNull(result);
    Assert.NotNull(result.Header);
    Assert.Equal(expectedHeader.DevType, result.Header.DevType);
    Assert.Equal(expectedHeader.Version, result.Header.Version);

    // Assert - проверка количества переменных
    Assert.NotNull(result.PlcValues);
    Assert.Equal(templates.Length, result.PlcValues.Length);

    // Assert - проверка значений переменных
    // BoolVar (адрес 12, индекс в массиве 2)
    Assert.Equal(true, result.PlcValues[0].Value);
    Assert.Equal("BoolVar", result.PlcValues[0].Name);
    Assert.Equal(typeof(bool), result.PlcValues[0].CSType);

    // ByteVar (адрес 13, индекс в массиве 3)
    Assert.Equal((byte)13, result.PlcValues[1].Value);
    Assert.Equal("ByteVar", result.PlcValues[1].Name);

    // UInt16Var (адрес 14, индекс в массиве 4)
    Assert.Equal((ushort)2, result.PlcValues[2].Value);
    Assert.Equal("UInt16Var", result.PlcValues[2].Name);

    // TODO остальные типы
}
```

Такой подход позволяет проверить корректность преобразования регистров в значения ПЛК, правильность расчёта адресов и соответствие типов — без реального сетевого соединения.

А вот запись без эмулятора протестировать полноценно не получится, потому что необходимо убедиться, что данные действительно отправляются и корректно принимаются другой стороной. Для полного тестирования — сначала запись, затем чтение (по сути e2e-сценарий) — требуется эмулятор или реальное устройство, которое будет играть роль ПЛК.

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

## Заключение

К сожалению данный проект не закончен и скорее всего не будет закончен так как данный ПЛК решили не использовать. 

Надеюсь, данная статья кому - нибудь поможет.

Есть еще куча материала, которая не вошла в статью, вы можете ее посмотреть по ссылке?

## Ссылки

1. [CODESYS Setup - drag&bot Help](https://help.dragandbot.com/modules/protocols/modbus/03_codesys.html)
2. [NModbus Library](https://github.com/NModbus/NModbus) — библиотека для работы с Modbus в .NET
3. [Modbus Protocol Specification](https://modbus.org/specs.php) — официальная спецификация протокола Modbus
4. [CODESYS Modbus Documentation](https://help.codesys.com/) — документация по настройке Modbus в CODESYS
5. [ПЛК HCFA опыт разработки и немного эксплуатации](https://www.asutpforum.ru/viewtopic.php?t=19136)
