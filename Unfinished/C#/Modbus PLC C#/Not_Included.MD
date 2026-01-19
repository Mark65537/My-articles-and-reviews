
# Названия

- Как я пытался считать переменные с ПЛК с помощью Modbus, чтобы потом вывести в приложение на C#

## Чтение данных

### Вариант 1: Раздельные функции

- Третья функция-обертка, которая вызывает обе функции для удобного использования

## Marshal и PlcValue

так как у нас переменные считываются не сразу а карта переменных изместна заранее то инициализация переменных происходит в 2 этанпа.Сначала инициализация в карте, потом считывание всех значений, поэтому value может быть null
Так сделано чтобы запрос считывание всех регистров сделать один раз и не нагружать ПЛК.

если не хочешь чтобы Маршал возвращал object то можно сделать так

```cs
public static class ModbusValueMarshaler
{
    public static T Marshal<T>(ushort[] regs)
    {
        // Примеры
        if (typeof(T) == typeof(int))
            return (T)(object)((regs[0] << 16) | regs[1]);

        if (typeof(T) == typeof(float))
            return (T)(object)BitConverter.ToSingle(
                BitConverter.GetBytes(
                    (uint)((regs[0] << 16) | regs[1])
                ), 0);

        throw new NotSupportedException(typeof(T).FullName);
    }
}
```

и использовать

```cs
foreach (var template in templates)
{
    ushort[] slice = ReadSlice(...);

    object value = template switch
    {
        PlcValue<int> v    => ModbusValueMarshaler.Marshal<int>(slice),
        PlcValue<float> v  => ModbusValueMarshaler.Marshal<float>(slice),
        PlcValue<bool> v   => ModbusValueMarshaler.Marshal<bool>(slice),
        _ => throw new NotSupportedException()
    };
}
```

но это мало что даст так как у value все равно будет тип object

есть еще вариант вставить метод read в PlcValue

```cs
public sealed class PlcValue<T> :
    IPlcValue,
    IEquatable<PlcValue<T>>
{

    public IPlcValue Read(UInt16[] regs)
    {
        T value = ModbusValueMarshaler.Marshal<T>(regs);

        return new PlcValue<T>(
            Name,
            value,
            Address,
            RegSize
        );
    }

}
```

Я считаю что если делать PlcValue<T> то у тебя будет от этого больше проблем чем пользы. Да у тебя тогда идет контроль типов то есть CSType равен типу value, но из за этого тебе нужно создавать лишний интерфейс, а интерфейы я считаю больше зло так как почти всегдаа трудно найти реализацию интерфейса и так же нельзя будет написать типа такого кода

```cs
if (template is PlcValue<T>)
{
    template.Value = ModbusValueMarshaler.Marshal(slice, template.CSType);
}
```
