# **Работа с dxf/dwg файлами C\#**

## **Введение**

как то мне нужно было считать dxf и dwg файлы и изменить в них текстовые данные

Задача:

1\. прочитать текст из dxf/dwg файла

2\. изменить его в формат unicode

3\. создать новый dxf/dwg файл в котором будет тоже самое содержимое но с измененным текстом из 2 пункта

4\. записать новый unicode текст в файл в той же папке где содержится оригинальный файл и добавить к названию файла префикс.

Для начала немного теории.

## **Структура dwg файла**

DWG это один из широко используемых файловых форматов в ряде приложений и имеет надежную файловую структуру. Поскольку DWG является двоичным форматом файла, он не читается человеком, как простой формат файла ASCII [DXF](https://docs.fileformat.com/ru/cad/dxf/). Файлы DWG обычно содержат информацию о координатах изображения и любые связанные с ним метаданные. Полные [спецификации](https://www.opendesign.com/files/guestdownloads/OpenDesign_Specification_for_.dwg_files.pdf) формата файла DWG доступны для загрузки компанией OpenDesign. Файловая структура формата файла DWG выглядит следующим образом.

**Заголовок:** Заголовок файла состоит из переменных заголовка DWG и информации о проверке циклическим избыточным кодом (CRC), которая используется для обнаружения ошибок. Каждый подраздел представляет собой специализированный вектор, в котором биты разной длины используются для представления разных меток. Например, первые 6 бит переменной заголовка DWG обозначают строку версии.

Определения классов. Файл DWG может содержать множество классов в зависимости от конкретного назначения файла .dwg. Информация, такая как размер метаданных класса области данных класса, номер класса и контрольная сумма в дополнение к другим.

**Шаблон:** это необязательный параметр, а для версий R15 и R15 этот раздел находится под разделом «Свободное пространство объекта».

**Заполнение:** к метаданным добавляются суффиксы и постфиксы с определенным количеством байтов, что делает старые версии программного обеспечения AutoCAD совместимыми с новым форматом файлов DWG.

**Данные изображения:** метаданные для этого раздела зависят от конкретного типа файла .dwg. Для пользователей R14 и R15 этот раздел является необязательным.

**Данные объекта:** Данные объекта состоят из полного списка сущностей таблицы, записей словаря и т. д., соответствующих существующему списку объектов.

**Карта объектов:** в этом разделе файла указывается местоположение каждого объекта в файле. Большинство метаданных в этом разделе представляют собой дескрипторы файлов, которые играют роль в идентификации и повторной визуализации объекта.

**Свободное пространство объекта:** этот раздел является необязательным для всех пользователей.

**Второй заголовок:** дубликат раздела заголовка файла ближе к концу файла DWG.

Спецификация формата DWG от некоммерческого консорциума Open Design Alliance размещена в открытом доступе[\[42\]](https://ru.wikipedia.org/wiki/DWG#cite_note-43).

Спецификация формата DWG от компании Autodesk доступна только для зарегистрированных и утверждённых компанией Autodesk групп разработчиков.

## Способы решения

Есть несколько способов как это сделать

### **Способ 1\. Использовать готовую библиотеку**

Мы будем использовать библиотеку [ACadSharp](https://github.com/DomCR/ACadSharp/tree/master), так как она бесплатная, с открытым исходным кодом и в ней есть весь необходимый функционал. У нее есть недостаток она работает не со всеми версиями AutoCAD, а именно с: MC0\_0, AC1\_2, AC1\_4, AC1\_50, AC2\_10, AC1002, AC1003, AC1004, AC1006, AC1009, AC1012, AC1021. То есть работает в основном с последними версиями.

Внимание: используйте только стабильную версию библиотеки так как нумерация версий не по порядку и самая последняя это самая стабильная.

ни каждая программа открывает преобразованный файл, точнее почти ни одна программа не открывает, открывает только [этот сайт](https://products.groupdocs.app/ru/viewer/dwg).

все файлы которые я открывал в программе программе ODA Drawings Explorer 25.8.0, выпадал мессаджбокс который предлагал восстановить файл и при восстановлении файлы открывались.

есть ограничение, длина строки не должна превышать 256 символов, насколько я понял это не ограничения библиотеки, а ограничения dxf/dwg файлов, возможно если текст в файле больше то он разбивается на несколько блоков. Для того чтобы не превышать это значение можно использовать транслитерацию, об этом можно прочитать [здесь]().

Принцип работы у всех библиотек примерно одинаковый, нужно взять из файла сущности, а потом что то с ними делать

лучше использовать WW но она платная

решил использовать библиотеку ACadSharp так как она бесплатна и совместима с WW то есть не надо переделывать метод чтения текста.

### **Способ 2\. Использовать AutoCAD API**

Данный способ подойдет не всем так как для его использования на компьютере должен быть установлен AutoCAD и библиотека  Aspose.CAD и это не бесплатно.

Насколько я понял она скрытно запускает процесс AutoCAD. как например при работе с Excel в Net.

#### **Aspose.CAD for .NET**

* **Описание**: Коммерческая библиотека для работы с DWG и DXF, поддерживает чтение, преобразование в изображения или PDF, доступ к слоям, текстам, линиям и другим элементам.  
* **Поддержка DWG**: Полная, включая DWG и DXF.  
* **Стоимость**: Платная, но есть бесплатная пробная версия.  
  **Установка**:

```bash  
Install-Package Aspose.CAD 
```

**Пример использования**:

```cs
  using Aspose.CAD;  
  using Aspose.CAD.FileFormats.Cad;
  using Aspose.CAD.FileFormats.Cad.CadObjects;
   
  using (var cadImage = (CadImage)Image.Load("file.dwg")) 
  {  
     foreach (var entity in cadImage.Entities)  
     {  
         // Обработка объектов, например, текста  
     }  
  }
```

### **Способ 3\. Написать парсер самому**

Это очень сложная и трудоемкая задача, для этого вам либо нужно изучать формат самому используя документ [OpenDesign\_Specification\_for\_.dwg\_files.pdf](http://www.opendesign.com/files/guestdownloads/OpenDesign_Specification_for_.dwg_files.pdf) (возможно нужен VPN)  
либо смотреть как устроена библиотека ACadSharp

## Чтение

Мы будем использовать 1-й способ.

С чтением все просто. Мы должны выбрать соответствующий тип DxfReader или DwgReader и передать в его статический метод Read, путь к нашему файлу. После этого мы получаем CadDocument из которого потом можем извлечь необходимые нам сущности.

Код:

```cs
CadDocument document = DwgReader.Read(filePath);

// Извлекаем текстовые объекты из файла  
var textEntities = document.Entities.OfType<TextEntity>().ToList();

TextListBox.Items.Clear();

foreach (var text in textEntities)  
{ 
    TextListBox.Items.Add(text.Value);  
}   
fileExtension = Path.GetExtension(filePath).ToLower();  
CadDocument? document = null;

if (fileExtension == ".dxf")  
{  
    document = DxfReader.Read(filePath);  
}  
else if (fileExtension == ".dwg")  
{  
    document = DwgReader.Read(filePath);  
}  
else  
{  
    MessageBox.Show("Неподдерживаемый формат файла.");  
    return;  
}

List<TextEntity> textEntities = document.Entities.OfType<TextEntity>().ToList();

StringBuilder textBuilder = new();  
foreach (TextEntity? text in textEntities)  
{  
    textBuilder.Append(text.Value).Append(' ');  
}

TextDataItem? tdi = TextDataItems.FirstOrDefault(x => x.FileName == Path.GetFileName(filePath));  
if(tdi != null)  
    tdi.Text = textBuilder.ToString();
```

## Запись

С записью пришлось повозиться  
просто так изменить текст не получилось. в dwg просмотрщиках, либо выдает ошибку, либо выводит такой же текст что и до перевода в unicode.

есть 2 способа. первый это создать новый документ и склонировать в него все сущности. Второй способ изменить сущности в текущем документе.

Просто так перенести все сущности не получиться так как у каждой из них есть владелец и просто так перекинуть нельзя. Для того чтобы перенести все сущности нужно создать их клон с помощью
var clonedEntity = entity.Clone() as Entity;

Код:

```cs  
CadDocument newDocument = new();  
CadDocument document;

if (fileExtension == ".dxf")  
{  
    document = DxfReader.Read(filePath);  
}  
else if (fileExtension == ".dwg")  
{  
    document = DwgReader.Read(filePath);  
}  
else  
{  
    MessageBox.Show("Неподдерживаемый формат файла.");  
    return;  
}

// Извлекаем все сущности из исходного документа и добавляем их в новый документ  
foreach (var entity in document.Entities)  
{  
    // Создаем клон каждого объекта  
    var clonedEntity = entity.Clone() as Entity;

    if (clonedEntity is TextEntity textEntity)  
    {  
        // Преобразуем текст в Unicode  
        string unicodeText = string.Join(" ", textEntity.Value.Select(c =\> $"\\\\u{(int)c:X4}"));  
        //string unicodeText = "изменен";

        // Создаем новый TextEntity с преобразованным текстом  
        var newTextEntity = new TextEntity  
        {  
            Value = unicodeText,  
            InsertPoint = textEntity.InsertPoint, // Сохраняем точку вставки  
            Height = textEntity.Height // Сохраняем высоту текста  
        };

        // Добавляем новый текстовый объект в новый документ  
        newDocument.Entities.Add(newTextEntity);  
    }  
    else  
    {  
        // Добавляем все другие объекты без изменений  
        newDocument.Entities.Add(clonedEntity);  
    }  
}

// Сохраняем новый файл с префиксом в названии  
string? outputDirectory = Path.GetDirectoryName(filePath);  
if (outputDirectory == null)  
{  
    MessageBox.Show($"не существует папки: {outputDirectory}.");  
    return;  
}  
string outputFileName = $"{Path.GetFileNameWithoutExtension(filePath)}{prefix}.dwg";  
string outputPath = Path.Combine(outputDirectory, outputFileName);

using (DwgWriter writer = new(outputPath, newDocument))  
{  
    writer.Write();  
}  
TextDataItems[0].IsReplaced = true;

private void LoadTextFromFile()  
{  
    if (_filePaths == null || _filePaths.Length == 0)  
    {  
        MessageBox.Show("Пожалуйста, выберите файл.");  
        return;  
    }

    try  
    {  
        foreach (string? filePath in _filePaths)  
        {  
            CadDocument document = ReadDocument(filePath);

            var textEntities = document.Entities.OfType<TextEntity>();

            StringBuilder textBuilder = new();  
            StringBuilder unicodeTextBuilder = new();  
            foreach (TextEntity? text in textEntities)  
            {  
                textBuilder.Append(text.Value).Append(' ');  
                unicodeTextBuilder.Append(RuEncode_UnicodeEscapeSequence(text.Value)).Append(' ');  
            }

            CadDataItem? tdi = CadDataItems.FirstOrDefault(x => x.FileName == Path.GetFileName(filePath));  
            if (tdi != null)  
            {  
                //tdi.Text = textBuilder.ToString();  
                //tdi.UnicodeText = unicodeTextBuilder.ToString(); // Сохраняем преобразованный текст в Unicode  
            }  
        }  
    }  
    catch (Exception ex)  
    {  
        MessageBox.Show($"Ошибка при загрузке файла: {ex.Message}");  
    }  
}
```

## Результаты конвертирования

Проверял на программе ODA Drawings Explorer 25.8.0.

Преобразование dwg в их же версию.  
AC1014  не работает  
AC1015 работает если нажать восстановить, сайт не открыл. Самый малый размер файла  
AC1018 не работает  
AC1024 не работает  
AC1027 не работает  
AC1032 не работает

Ниже таблицы где файлы разных версий конвертировались в версию AC1015, так как она меньше всего весит. и тот же самый результат при конвертации в версию AC1032.

### DXF

|                    | Версия  | Версия в файле | Результат                                                                            |
| :----------------- | :------ | :------------- | :----------------------------------------------------------------------------------- |
| :heavy_check_mark: | 2000    | AC1015         | Открывает                                                                            |
| :heavy_check_mark: | 2004    | AC1018         | Открывает                                                                            |
| :heavy_check_mark: | 2007    | AC1021         | Открывает. Если не конвертировать MText, то русский текст будет в странном формате   |
| :heavy_check_mark: | 2010    | AC1024         | Открывает. Если не конвертировать MText, то русский текст будет в странном формате   |
| :heavy_check_mark: | 2013    | AC1027         | Открывает. Если не конвертировать MText, то русский текст будет в странном формате   |
| :heavy_check_mark: | 2018    | AC1032         | Открывает. Меняет положение MText и отображает только первую строку. Пропадает Hatch |
| -                  | R12     | AC1009         | Открывает, но ничего не отображает                                                   |
| :x:                | Pronest | AC1018         | не конвертирует                                                                      |

### DWG

| Версия | Версия в файле | Pезультат                                                 |
| :----- | :------------- | :-------------------------------------------------------- |
| 2000   | AC1015         | требует восстановить. не преобразовывает MText            |
| 2004   | AC1018         | Открывает. Требует восстановления.  не переводит MText    |
| 2007   | AC1021         | Открывает. Требует восстановления. Не конвертирует MText. |
| 2010   | AC1024         | Открывает. Требует восстановления. Не конвертирует MText. |
| 2013   | AC1027         | Открывает. Требует восстановления. Не конвертирует MText. |
| 2018   | AC1032         | Открывает. Требует восстановления. Не конвертирует MText. |
| R14    | AC1014         | Открывает. Ничего не отображает                           |

### Нерабочие преобразования

| тип файла | Версия       | В какую версию конвертирую | Результат                                                                                         |
| :-------- | :----------- | :------------------------- | :------------------------------------------------------------------------------------------------ |
| dwg       | 2000         | AC1032                     | не открывается после восстановления Error Recovering Database...Object was erased: \<object\> (C) |
| dxf       | 2018(AC1032) | MC0\_0                     | не открывается после восстановления. Error Recovering Database...Invalid file version             |
| dxf       | 2018(AC1032) | AC1\_2                     | не открывается после восстановления. Error Recovering Database...Invalid file version             |
|           |              |                            |                                                                                                   |

## Вывод

Если нужно читать только dfx, то отлично подойдет **netDxf**, она мало весит и легка в использовании.  
Если нужен еще формат dwg, то рекомендую использовать

Формат dwg является бинарным закрытым форматом, доступ к нему есть только у вступивших в клуб AutoCAD

## **Ссылки**

1. [https://github.com/DomCR/ACadSharp](https://github.com/DomCR/ACadSharp)  
2. [http://www.opendesign.com/files/guestdownloads/OpenDesign\_Specification\_for\_.dwg\_files.pdf](http://www.opendesign.com/files/guestdownloads/OpenDesign_Specification_for_.dwg_files.pdf)
