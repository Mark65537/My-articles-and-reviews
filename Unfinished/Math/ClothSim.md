# Симуляция ткани на WebGPU

## Постановка задачи

Цель проекта — реализовать простую симуляцию ткани с использованием метода **Position Based Dynamics (PBD)**. Ткань представляет собой квадратную сетку, состоящую из небольших квадратов. Каждый квадрат, в свою очередь, разбивается на два треугольника.

![image.png](./example.png)

Требования к симуляции:

* четыре угловые вершины должны быть закреплены;
* одна внутренняя вершина должна перемещаться по закону синуса;
* движение центральной вершины должно происходить именно через изменение её координаты, а не через приложение силы;
* должна присутствовать гравитация;
* гравитацию необходимо включать и выключать через пользовательский интерфейс;
* ткань должна визуализироваться с помощью WebGPU;
* проект должен быть написан на JavaScript, HTML и CSS;
* сторонние 3D-движки, например Three.js или Blend4Web, использоваться не должны.

## Подготовка проекта

Начнём с HTML. Для начала нам нужен будет просто канвас без какого либо интерфейса:

```html
<html lang="RU">

<head>
    <title>PBD-Fabric</title>
    <meta charset="UTF-8">
</head>

<link rel="stylesheet" href="style.css">

<body>

    <canvas id="canvas"></canvas>

    <script src="main.js"></script>

</body>

</html>
```

```css
body{
    margin:0;
    overflow:hidden;
}

canvas{
    width:100vw;
    height:100vh;
}
```

### Проверка поддержки WebGPU

В JavaScript сначала проверяем наличие WebGPU:

```js
if (!navigator.gpu) {
    alert("WebGPU не поддерживается");
}
```

Если браузер не поддерживает WebGPU, продолжать работу бессмысленно.

## Рисуем один треугольник

## Render

## Рисуем квадрат

## Шейдеры

## Генерирование сетки

## Анимация

## Ограничения(Constraints)

## Камера

## Интегратор

## Пользовательский интерфейс

## Заключение

## Ссылки

1. [Заготовка для старта](http://medium.com/@carmencincotti/drawing-a-triangle-with-webgpu-53d48fb1ba8)
2. [WebGPU](https://www.w3.org/TR/webgpu)
3. [Язык шейдеров](https://www.w3.org/TR/WGSL/#intro)
4. [Статья по compute шейдерам](https://developer.chrome.com/articles/gpu-compute)
5. [WebGPU Cloth Simulation is in GitHub](https://carmencincotti.com/2022-09-12/webgpu-cloth-simulation-is-in-github/)
6. [Похожий проект(полностью рабочий)](https://github.com/AntonyFomichev/pbd-cloth)
7. [Похожий проект](https://github.com/aabal/webgpu-pbd-cloth-simulation)
8. [похожий проект](https://github.com/vadik260/cloth-pbd)
9. [text](https://github.com/g30613740/cloth_sim)
10. [text](https://github.com/Catnivetur/WebGPU-PBD-cloth-simulator)
11. [Real-Time Cloth Simulation Using WebGPU: Evaluating Limits](https://arxiv.org/html/2507.11794)

