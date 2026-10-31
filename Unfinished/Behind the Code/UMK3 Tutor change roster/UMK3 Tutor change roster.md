# Туториал как изменить Ростер в UMK3

Здесь я раскажу по шагам как можно подробней как изменить ростер персонажей(последовательность персонажей на экране выбора персонажей) на тот который вы хотите
Наша конечная цель изменить ростер на аркадный.

## 1. Изменение последовательности

нужно перейти по адресу `0x00DFC8` и изменить последовательность на ту которая вам нужна.

### Легкий способ

использовать imhex

1. использовать imhex
2. загрузить паттерн

### Пример

Если вы хотите изменить Ростер на аркадный нужно:

1. перейти по адресу `0x00DFC8`
2. изменить\
`00 18 00 13 00 05 00 02 00 03 00 10 00 17 00 01 00 00 00 11 00 1A 00 04 00 15 00 09 00 07 00 0F 00 14 00 12 00 08 00 0A 00 06 00 0E 00 0D 00 0C`\
на\
`00 0F 00 13 00 01 00 02 00 03 00 10 00 12 00 16 00 00 00 11 00 14 00 15 00 04 00 18 00 07 00 06 00 05 00 08 00 09 00 0A 00 0B 00 0C 00 0D 00 0E`

## 2. Изменение портретов

First of all you need change Sequence of characters else portraits will not match. [[1]] If you want to swap the portraits that are already on the character selection screen, you only need to replace the references to the TileSet and TileMap used by the portraits.

### Базовый алгоритм

1. Find the portrait header data that you want to modify.
2. In the first CHAR's portrait header, replace `ref_of_TileSet` with the value of `ref_of_TileSet` from the second CHAR's portrait header. Repeat the same process for the second CHAR.
3. In the first CHAR's portrait header, replace `ref_of_TileMap` with the value of `ref_of_TileMap` from the second CHAR's portrait header. Repeat the same process for the second CHAR.
4. In the first CHAR's portrait header, replace `TileSet_Size` with the value of `TileSet_Size` from the second CHAR's portrait header. Repeat the same process for the second CHAR.

### Пример

по шагам

For example, if you want to swap:

Rain portrait
with Sonya portrait
you need to:

1. go to 0xaa732
2. change ref of TileSet from 00 0A A3 B4 to 00 0a c0 54
3. go to 0xaa73A
4. change ref of TileMap from 00 0A A7 6E to 00 0A C4 E8
5. go to 0xaa762
6. change TileSet Size from 00 00 00 2C to 00 00 00 30

7. go to 0xAC4AC
8. change ref of TileSet from 00 0A C0 54 to 00 0A A3 B4
9. go to 0xAC4B
10. change ref of TileMap from 00 0A C4 E8 to 00 0A A7 6E
11. go to 0xAC4DC
12. change TileSet Size from 00 00 00 30 to 00 00 00 2C