# Как я читал переменные из ПЛК по Modbus и выводил их в C#-приложение

## Введение

Modbus — это открытый и очень распространённый протокол обмена данными в промышленной автоматизации. Он работает по модели master–slave: мастер (например, PC-приложение) запрашивает данные у ведомого устройства (ПЛК), получая или записывая значения регистров.

На практике Modbus кажется простым — всего лишь массив 16-битных регистров. Но как только возникает задача читать типизированные переменные, поддерживать несколько проектов в одном ПЛК, минимизировать количество запросов и безопасно работать с соединением, всё быстро усложняется.

В этой статье я описываю реальный подход, который использовал для чтения и записи переменных из ПЛК и отображения их в приложении на C#.

## Задача

Необходимо организовать работу с ПЛК по Modbus TCP так, чтобы приложение на C# взаимодействовало не с «сырыми» 16-битными регистрами, а с типизированными переменными. При этом важно учитывать, что в ПЛК могут загружаться разные проекты с отличающейся структурой данных, поэтому требуется механизм идентификации проекта и проверки его версии. Система должна не только читать регистры, но и корректно записывать значения обратно в ПЛК, минимизируя количество Modbus-запросов и обеспечивая согласованность данных. Также необходимо реализовать преобразование типов между регистрами Modbus и типами C#, а также контроль соединения с возможностью автоматического переподключения. Таким образом, задача заключается в построении устойчивой и расширяемой архитектуры поверх Modbus, а не просто в выполнении операций чтения и записи.

## Используемые инструменты

В работе использовались следующие инструменты и библиотеки:

- **CODESYS** — среда программирования ПЛК, в которой создавалась программа для обмена данными по Modbus
- **Modbus TCP** — вариант протокола Modbus, работающий поверх TCP/IP
- **NModbus** — .NET-библиотека для работы с Modbus (реализация master-устройства на стороне ПЛК)
- **Modbus Slave** — программа-эмулятор Modbus-устройства для тестирования (использовалась для отладки вместо реального ПЛК)

## Особенности Modbus и работы с ним в CODESYS

Ключевые особенности протокола:

1. Modbus оперирует только 16-битными регистрами
2. за один запрос можно прочитать не более 125 регистров
3. каждое устройство имеет уникальный Slave ID

>[!NOTE]
Если в качестве сервера используется программа [Modbus Slave](https://www.modbustools.com/), чтение регистров приходится начинать с 10-го адреса — это практическая особенность используемого инструмента или его реализации протокола.

При работе с CODESYS необходимо учитывать дополнительные нюансы:

1. всего доступно 4096 регистров
2. если переменная замаплена в Holding Register, то после старта ПЛК её значение автоматически сбрасывается в ноль
3. если в регистр сопоставить массив, фактически будет доступен только первый элемент
4. строки в регистрах сохраняются в обратном порядке

Поскольку Modbus работает только с 16-битными ячейками, значения большего размера (например, 32 или 64 бита) распределяются по нескольким регистрам. Чтобы не разбивать данные логически по разным адресам и упростить чтение и запись, удобно использовать единый буферный массив регистров, в который последовательно упаковываются все переменные, а уже поверх него выполнять типизированное преобразование.

## Работа с Modbus в C#

Для работы с Modbus в C# нужно использовать стороннюю библиотеку.
В данном проекте была выбрана библиотека [Nmodbus](https://www.nuget.org/packages/NModbus/). Это одна из первых библиотек, которая попалась при поиске, при этом у неё достаточно простой и понятный API, поэтому её оказалось удобно использовать на практике.

Со временем выяснилось ограничение стандартного Modbus — за один запрос можно читать не более 125 регистров. В связи с этим возник вопрос, существуют ли библиотеки, которые автоматически разбивают большие запросы на несколько. В документации библиотеки [PL.Modbus](https://www.nuget.org/packages/PL.Modbus) указано, что она умеет выполнять такие мультизапросы. При необходимости можно рассмотреть её использование, однако в рамках данного проекта она не применялась, поэтому оценить её работу на практике не удалось. Причины, по которым не был выполнен переход на другую библиотеку, описаны в разделе [Заключение](#Заключение).

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

### ПЛК-переменные

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

## Маршаллинг

Так как размер переменных в ПЛК не всегда соответствует размеру переменных в C#, нужен маршаллер, который будет преобразовывать сырые данные из регистров Modbus в типизированные значения C# и обратно.

### Особенности работы с типами данных

С точки зрения обмена по Modbus и представления данных в ПЛК, все типы можно условно разделить на две группы: простые и сложные. Разница между ними не в синтаксисе, а в стратегии хранения и преобразования.

**Простые типы** — это базовые скалярные типы ПЛК: `BOOL`, `BYTE`, `WORD`, `DWORD`, `LWORD`, `SINT`, `INT`, `DINT`, `LINT`, `USINT`, `UINT`, `UDINT`, `ULINT`, `REAL`, `LREAL`. Работа с ними прямолинейна: Modbus оперирует 16-битными регистрами (UInt16), поэтому если тип в C# занимает меньше 2 байт, он всё равно размещается в одном полном регистре — уплотнять несколько переменных в один регистр не стоит. Если тип больше 2 байт, просто используется столько регистров, сколько требуется для его размера. В целом правило простое: размер типа в байтах делится на 2 и округляется вверх до количества регистров.

**Сложные типы** — это `TIME`, `DATE`, `TOD`, `DT`, `STRING`, `STRUCT`, `ARRAY`. С ними уже нет единственно очевидной стратегии, и приходится принимать архитектурные решения.

Типы времени (`TIME`, `DATE`, `TOD`, `DT`) в C# напрямую не соответствуют типам ПЛК. В .NET фактически используются два базовых типа: `DateTime` и `TimeSpan`. `TIME` удобно сопоставлять с `TimeSpan`, учитывая, что в ПЛК он хранится как количество миллисекунд — при чтении выполняется преобразование миллисекунд в `TimeSpan`, при записи — обратная конвертация. `DATE` представляет дату без времени (формат вида `YYYYMMDD`), а `TOD` — время без даты (`HHMMSS`); для них можно использовать либо `DateTime` с фиксированной датой, либо `TimeSpan` — в зависимости от выбранной модели. `DT` обычно маппится на `DateTime`.

Со строками основная сложность — кодировка и определение длины. В ПЛК используется кодировка [Latin-1](https://ru.wikipedia.org/wiki/ISO_8859-1) (`ISO-8859-1`, также известная как `Windows-1252`). Работа с кириллицей может быть неоднозначной, поэтому этот момент желательно отдельно проверять, особенно если строки активно используются. Вторая проблема — длина строки. Возможны два подхода: либо первый регистр хранит длину строки, а далее идут символы, либо строка завершается терминальным символом `\0`. Допустимо комбинировать оба способа для повышения надёжности. Поскольку длина строки не всегда известна заранее, в карте переменных для строк необходимо явно указывать `RegSize`, чтобы понимать, сколько регистров резервируется под данные.

**Enum** в CODESYS хранится как базовый целочисленный тип длиной 2 байта.и передаётся как обычное число в одном регистре. Рекомендуется явно указывать базовый тип UInt16, чтобы избежать неоднозначностей:

```cs
public enum DevType : UInt16
{
    VFU = 1,
}
```

**Массивы** требуют явного указания длины — без фиксированного размера корректно работать с ними невозможно. Количество регистров для массива определяется как произведение размера элемента на длину массива.

Отдельно остаются структуры (`STRUCT`) — с ними стратегия работы зависит от договорённости о порядке полей и выравнивании, и этот вопрос требует отдельного проектного решения.

### Реализация маршаллера

Класс маршаллера отвечает за двустороннее преобразование данных между 16-битными регистрами Modbus (`UInt16[]`) и типами C#. Его задача — полностью изолировать логику упаковки и распаковки регистров, чтобы остальной код оперировал привычными C#-типами и не задумывался о битовых сдвигах и порядке байтов.

Базовая сигнатура выглядит так:

```csharp
public static object Marshal(UInt16[] raw, Type targetType)
```

Метод проверяет входные параметры, после чего по `targetType` выбирает стратегию преобразования. Для простых однорегистровых типов (`bool`, `byte`, `sbyte`, `UInt16`, `Int16`) используется прямое приведение из `raw[0]`. Для 32-битных типов (`UInt32`, `Int32`) объединяются два регистра через сдвиг старшего слова на 16 бит. Для 64-битных типов (`UInt64`, `Int64`) объединяются уже четыре регистра.

```csharp
// Простые типы (однорегистровые)
if (targetType == typeof(bool)) return raw[0] != 0;
if (targetType == typeof(byte)) return (byte)raw[0];
if (targetType == typeof(UInt16)) return raw[0];
if (targetType == typeof(Int16)) return (Int16)raw[0];
if (targetType == typeof(sbyte)) return (sbyte)raw[0];

// 32-битные типы
if (targetType == typeof(UInt32))
{
    if (raw.Length < 2)
        throw new ArgumentException("недостаточно регистров для UInt32");

    return ((UInt32)raw[0] << 16) | raw[1];
}

if (targetType == typeof(Int32))
{
    if (raw.Length < 2)
        throw new ArgumentException("недостаточно регистров для Int32");

    return (Int32)(((UInt32)raw[0] << 16) | raw[1]);
}

// 64-битные типы
if (targetType == typeof(UInt64))
{
    if (raw.Length < 4)
        throw new ArgumentException("недостаточно регистров для UInt64");

    return ((UInt64)raw[0] << 48) |
           ((UInt64)raw[1] << 32) |
           ((UInt64)raw[2] << 16) |
           raw[3];
}

if (targetType == typeof(Int64))
{
    if (raw.Length < 4)
        throw new ArgumentException("недостаточно регистров для Int64");

    UInt64 value =
        ((UInt64)raw[0] << 48) |
        ((UInt64)raw[1] << 32) |
        ((UInt64)raw[2] << 16) |
        raw[3];

    return (Int64)value;
}
```

Числа с плавающей точкой (`float`, `double`) сначала собираются в целочисленное представление (`UInt32`/`UInt64`), после чего преобразуются через `BitConverter.Int32BitsToSingle` и `BitConverter.Int64BitsToDouble`.

```csharp
// Числа с плавающей точкой
if (targetType == typeof(float))
{
    if (raw.Length < 2)
        throw new ArgumentException("недостаточно регистров для float");

    UInt32 bits = ((UInt32)raw[0] << 16) | raw[1];
    return BitConverter.Int32BitsToSingle((Int32)bits);
}

if (targetType == typeof(double))
{
    if (raw.Length < 4)
        throw new ArgumentException("недостаточно регистров для double");

    UInt64 bits =
        ((UInt64)raw[0] << 48) |
        ((UInt64)raw[1] << 32) |
        ((UInt64)raw[2] << 16) |
        raw[3];

    return BitConverter.Int64BitsToDouble((Int64)bits);
}
```

Специальные типы интерпретируются по договорённости: `TimeSpan` — как количество миллисекунд в 32 битах, `DateTime` — как количество секунд от `UnixEpoch`. Строка читается по схеме «первый регистр — длина, далее символы». Массивы обрабатываются в зависимости от типа элемента: для `UInt16[]` возвращается исходный массив, для остальных создаётся новый массив с приведением каждого элемента. Для `enum` сейчас возвращается числовое значение регистра (при необходимости можно восстановить `enum` через базовый тип).

```csharp
// Специальные типы
if (targetType == typeof(TimeSpan))
{
    if (raw.Length < 2)
        throw new ArgumentException("недостаточно регистров для TimeSpan");
    UInt32 milliseconds = ((UInt32)raw[0] << 16) | raw[1];
    return TimeSpan.FromMilliseconds(milliseconds);
}
if (targetType == typeof(DateTime))
{
    if (raw.Length < 2)
        throw new ArgumentException("недостаточно регистров для DateTime");
    UInt32 seconds = ((UInt32)raw[0] << 16) | raw[1];
    return DateTime.UnixEpoch.AddSeconds(seconds);
}
if (targetType == typeof(string))
{
    int length = raw[0];
    if (length <= 0 || raw.Length < length + 1)
        return string.Empty;
    char[] chars = new char[length];
    for (int i = 0; i < length; i++)
        chars[i] = (char)raw[i + 1];
    return new string(chars);
}
if (targetType.IsArray)
{
    Type elementType = targetType.GetElementType()!;
    if (elementType == typeof(UInt16))
        return raw;
    Array result = Array.CreateInstance(elementType, raw.Length);
    for (int i = 0; i < raw.Length; i++)
    {
        result.SetValue(Convert.ChangeType(raw[i], elementType), i);
    }
    return result;
}

if (targetType.IsEnum || targetType == typeof(Enum))
{
    return raw[0];
}
```

Обратное преобразование выполняет метод:

```csharp
public static UInt16[] Unmarshal(object? value, Type? targetType)
```

Логика симметрична `Marshal`:

- однорегистровые типы возвращаются как массив из одного `UInt16`;
- 32-битные и 64-битные типы разбиваются на 2 или 4 регистра с помощью побитовых сдвигов;
- `float` и `double` сначала переводятся в битовое представление через `BitConverter`, затем режутся на 16-битные части;
- `TimeSpan`сериализуется как количество миллисекунд, `DateTime` — как количество секунд от `UnixEpoch`;
- для массивов рассчитывается количество регистров на элемент и формируется итоговый буфер;
- `enum` приводится к базовому целочисленному типу и записывается в один регистр.

Дополнительно реализован вспомогательный метод `ConvertToType<T>`, который аккуратно приводит входное значение к нужному типу, включая поддержку конвертации из строк с использованием `InvariantCulture`. Это позволяет безопасно сериализовать данные, полученные, например, из UI или конфигурационных файлов.

```csharp
private static T ConvertToType<T>(object value)
{
    if (value is T directValue)
        return directValue;

    if (value is string strValue)
    {
        // Пытаемся конвертировать строку в нужный тип
        var targetType = typeof(T);

        var inv = System.Globalization.CultureInfo.InvariantCulture;
        const System.Globalization.NumberStyles numStyle = System.Globalization.NumberStyles.Any;

        if (targetType == typeof(bool))
        {
            if (bool.TryParse(strValue, out bool b))
                return (T)(object)b;
            if (int.TryParse(strValue, numStyle, inv, out int i))
                return (T)(object)(i != 0);
            throw new ArgumentException($"не удалось конвертировать строку \"{strValue}\" в тип {targetType.Name}");
        }

        if (targetType == typeof(TimeSpan))
        {
            if (TimeSpan.TryParse(strValue, inv, out TimeSpan ts))
                return (T)(object)ts;
            throw new ArgumentException($"не удалось конвертировать строку \"{strValue}\" в тип {targetType.Name}");
        }

        if (targetType == typeof(DateTime))
        {
            // Для строк без таймзоны считаем, что это UTC (чтобы не было сдвига на локальный TZ)
            const System.Globalization.DateTimeStyles dtStyle =
                System.Globalization.DateTimeStyles.AssumeUniversal |
                System.Globalization.DateTimeStyles.AdjustToUniversal;

            if (DateTime.TryParse(strValue, inv, dtStyle, out DateTime dt))
                return (T)(object)dt;
            throw new ArgumentException($"не удалось конвертировать строку \"{strValue}\" в тип {targetType.Name}");
        }

        // Числовые типы
        if (targetType == typeof(byte) && byte.TryParse(strValue, numStyle, inv, out byte u8)) return (T)(object)u8;
        if (targetType == typeof(sbyte) && sbyte.TryParse(strValue, numStyle, inv, out sbyte i8)) return (T)(object)i8;
        if (targetType == typeof(UInt16) && UInt16.TryParse(strValue, numStyle, inv, out UInt16 u16)) return (T)(object)u16;
        if (targetType == typeof(Int16) && Int16.TryParse(strValue, numStyle, inv, out Int16 i16)) return (T)(object)i16;
        if (targetType == typeof(UInt32) && UInt32.TryParse(strValue, numStyle, inv, out UInt32 u32)) return (T)(object)u32;
        if (targetType == typeof(Int32) && Int32.TryParse(strValue, numStyle, inv, out Int32 i32)) return (T)(object)i32;
        if (targetType == typeof(UInt64) && UInt64.TryParse(strValue, numStyle, inv, out UInt64 u64)) return (T)(object)u64;
        if (targetType == typeof(Int64) && Int64.TryParse(strValue, numStyle, inv, out Int64 i64)) return (T)(object)i64;
        if (targetType == typeof(float) && float.TryParse(strValue, numStyle, inv, out float f)) return (T)(object)f;
        if (targetType == typeof(double) && double.TryParse(strValue, numStyle, inv, out double d)) return (T)(object)d;

        throw new ArgumentException($"не удалось конвертировать строку \"{strValue}\" в тип {targetType.Name}");
    }

    // Пытаемся использовать стандартную конвертацию
    return (T)Convert.ChangeType(value, typeof(T), System.Globalization.CultureInfo.InvariantCulture);
}
```

Таким образом, маршаллер инкапсулирует всю логику преобразования типов, обеспечивая единый и предсказуемый механизм работы с данными при чтении и записи через Modbus.

## Чтение данных

Для чтения данных по Modbus просто используется функция

```cs
ushort[] ReadHoldingRegisters(byte slaveAddress, ushort startAddress, ushort numberOfPoints);
```

При чтении данных из ПЛК можно реализовать несколько подходов:

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
- Возвращает структурированный объект(`MBDataScheme`) с заголовком и массивом значений

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

## Запись данных

Механизм записи строится по тем же принципам, что и чтение: прикладной код работает с типизированными значениями, а перед отправкой в ПЛК данные преобразуются в массив `UInt16[]` через маршаллер. Далее используется стандартная функция записи нескольких регистров Modbus.

Далее возможны две стратегии организации записи.

**Вариант 1. запись всех переменных:**

Самый простой подход — формировать общий буфер регистров и отправлять его целиком одним запросом. Обычно это реализуется через единый массив, в котором каждая переменная имеет заранее известное смещение.

Особенности подхода:

- минимальная логика на стороне клиента;
- нет необходимости отслеживать изменения;
- предсказуемое поведение — ПЛК всегда получает полное состояние.

Недостаток — избыточный трафик. Даже если изменилось одно значение, передаётся весь блок данных. При большом количестве переменных это создаёт лишнюю нагрузку на сеть и ПЛК и вдобавок может записывать переменные которые были изменены во время редактирования, из-за этого могут храниться неправельные значения.

**Вариант 2. запись только измененных переменных:**

Более оптимальный вариант — отправлять только те значения, которые действительно были изменены приложением. Для этого требуется дополнительная инфраструктура:

Алгоритм работы обычно следующий:

1. Пользователь изменяет значение переменной.
2. Значение сравнивается с предыдущим.
3. Если оно отличается — переменная добавляется в специальный список где храняться все изменённые переменные.
4. Запись переменных из списка в ПЛК.

Преимущества такого подхода:

- меньше Modbus-запросов
- снижение сетевой нагрузки
- уменьшение времени цикла записи
- более быстрый отклик интерфейса

Таким образом, выбор стратегии зависит от требований к производительности и сложности системы. Для небольших проектов достаточно полной записи всего блока, тогда как в более нагруженных системах оправдано внедрение механизма отслеживания изменений и выборочной передачи данных.

## Тестирование

Для полноценного тестирования можно использовать простой консольный эмулятор, который открывает TCP-соединение, принимает и возвращает массив регистров. Реализация такого эмулятора достаточно проста: по сути это TCP-сервер с минимальной логикой обработки Modbus-запросов. Однако на начальном этапе можно обойтись и без него.

Чтение регистров вполне реально протестировать изолированно. Если заранее известно, какие значения лежат в регистрах и какие данные должны получиться после преобразования, можно сформировать тестовый массив `UInt16[]` и проверить корректность маппинга регистров в типизированные переменные.

>[!NOTE]
Код получается довольно объёмным из-за больших массивов, поэтому ниже приведены только ключевые части.

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

## Подключение и мониторинг соединения

Поскольку Modbus TCP работает поверх обычного TCP-соединения, сам протокол не предоставляет встроенного механизма уведомления о разрыве связи или «зависшем» устройстве. Соединение может быть формально установлено, но ПЛК уже не отвечает на запросы. Поэтому необходим собственный механизм контроля состояния и автоматического переподключения.

Важно, чтобы проверка соединения выполнялась не в бесконечном while-цикле с опросом «каждый кадр», а по таймеру с заданным интервалом. Постоянный активный опрос создаёт избыточную нагрузку на CPU и сеть. Оптимальный вариант — периодическая проверка (например, раз в 500–1000 мс).

Ниже приведены два практических способа проверки подключения.

### Проверка через чтение регистра (heartbeat)

Самый надёжный способ — выполнить минимальный Modbus-запрос, например чтение одного Holding Register. Если операция проходит успешно, значит:

- TCP-соединение активно;
- Modbus-сервер доступен;
- устройство корректно обрабатывает запросы.

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

**Преимущества подхода:**

- Проверяет не только TCP-соединение, но и работоспособность Modbus-протокола
- Минимальная нагрузка — читается всего один регистр
- корректно выявляются ситуации, когда сокет открыт, но ПЛК не отвечает.

### 2. Проверка через сокет

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

## Заключение

К сожалению данный проект не закончен и скорее всего не будет закончен так как данный ПЛК решили не использовать.

данная программа не тестировалась в реальных условияс с подключением к ПЛК.

Надеюсь, данная статья кому - нибудь поможет.

Есть еще куча материала, которая не вошла в статью, вы можете ее посмотреть по ссылке?

## Ссылки

1. [CODESYS Setup - drag&bot Help](https://help.dragandbot.com/modules/protocols/modbus/03_codesys.html)
2. [NModbus Library](https://github.com/NModbus/NModbus) — библиотека для работы с Modbus в .NET
3. [Modbus Protocol Specification](https://modbus.org/specs.php) — официальная спецификация протокола Modbus
4. [CODESYS Modbus Documentation](https://help.codesys.com/) — документация по настройке Modbus в CODESYS
5. [ПЛК HCFA опыт разработки и немного эксплуатации](https://www.asutpforum.ru/viewtopic.php?t=19136)
6. [ModbusSlaveSimulation](https://github.com/GitHubDragonFly/ModbusSlaveSimulation)
7. [Сайт по проге Modbus Slave](https://www.modbustools.com/)
