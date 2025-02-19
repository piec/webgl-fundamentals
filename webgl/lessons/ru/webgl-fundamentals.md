Title: Основы WebGL
Description: Ваш первый урок по WebGL начнётся с его основ
TOC: Основы WebGL


О WebGL часто думают, как о API для 3D. Люди думают "Я буду использовать WebGL и *магия* получится классное 3D".
На самом деле WebGL - это просто средство растеризации. Он отображает точки, линии и треугольники на основе
написанного кода. Чтобы получить что-то от WebGL, вам нужно написать код, где, используя точки, линии
и треугольники, и вы достигнете своей цели.

WebGL выполняется на графическом процессоре компьютера. То есть вам нужно написать код, который выполняется
на этом процессоре. Код представлен в виде пар функций. Эти две функции - вершинный и фрагментный шейдер,
и обе они написаны на очень строго типизированном языке, подобному C/C++, который называется
[GLSL](webgl-shaders-and-glsl.html). (GL Shader Language). Вместе эта пара функций называется *программа*

Задача вершинного шейдера - вычислять положения вершин. Основываясь на положениях вершин, которые возвращает функция,
WebGL затем может растеризовать различные примитивы, включая точки, линии или треугольники.
В процессе растеризации этих примитивов WebGL прибегает к использованию второй функции - фрагментному шейдеру.
Задача фрагментного шейдера - вычислять цвет для каждого пикселя примитива, который в данный момент отрисовывается.

Практически всё API WebGL заключается в настройке состояния для работы этих двух функций.
Вы устанавливаете настройки для каждого объекта, который хотите отрисовать, а затем выполняете эти
две функции через вызов `gl.drawArrays` или `gl.drawElements`, которые выполнят шейдеры на графическом процессоре.

Любые данные, которые вы хотите использовать в этих двух функциях, должны быть переданы на графический процессор.
Есть 4 способа, которыми шейдер может получить данные.

1. Атрибуты и буферы

   Буферы - это массивы бинарных данных, загруженных в графический процессор. Обычно буферы содержат
   вещи вроде положений вершин, нормалей, координат текстур, цветов вершин и т.д., хотя
   вы вольны положить в них что угодно.

   Атрибуты определяют, каким образом данные из ваших буферов передаются в вершинный шейдер.
   Например, вы можете поместить положения вершин в буфер как три 32-битных числа с плавающей точкой
   на одно положение. Вы указываете конкретному атрибуту, откуда брать положения вершин, какой тип
   данных используется (три 32-битных числа с плавающей точкой), начиная с какого индекса в буфере
   начинаются положения вершин и какое количество байтов нужно получить от одного положения до следующего.

   Доступ к буферам не произвольный. Вместо этого вершинный шейдер выполняется заданное
   количество раз и каждый раз, когда он выполняется, выбирается следующее значение каждого
   из указанных буферов и назначается атрибуту.

2. Uniform-переменные

   Uniform-переменные - это глобальные переменные, которые устанавливаются перед выполнением программы шейдера.

3. Текстуры

   Текстуры - это массивы данных, к которым есть произвольный доступ в программе шейдера.
   Чаще всего в текстуру помещается картинка, но текстура - это просто набор данных и вы
   можете запросто поместить в неё что-то отличное от набора цветов.

4. Varying-переменные

   Varying-переменные позволяют передавать данные из вершинного шейдера фрагментному шейдеру.
   Во фрагментном шейдере мы получим интерполированные значения вершинного шейдера - зависит
   от того, отображаем ли мы точки, линии или треугольники.


## Hello World на WebGL

Для WebGL важны 2 вещи. Это координата пространства отсечения и цвет.
Ваша задача как программиста - обеспечить для WebGL эти 2 вещи.
И для этого у вас есть 2 шейдера. Вершинный шейдер задаёт координаты
пространства отсечения, а фрагментный шейдер отвечает за цвет.

Координаты пространства отсечения всегда находятся в диапазоне от -1 до +1 вне
зависимости от размера canvas. Рассмотрим простейший пример использования WebGL.

Начнём с вершинного шейдера

    // атрибут, который будет получать данные из буфера
    attribute vec4 a_position;

    // все шейдеры имеют функцию main
    void main() {

      // gl_Position - специальная переменная вершинного шейдера,
      // которая отвечает за установку положения
      gl_Position = a_position;
    }

Если бы весь код был написан на JavaScript вместо GLSL,
то он бы мог выглядеть примерно так:

    // *** ПСЕВДОКОД!! ***

    var positionBuffer = [
      0, 0, 0, 0,
      0, 0.5, 0, 0,
      0.7, 0, 0, 0,
    ];
    var attributes = {};
    var gl_Position;

    drawArrays(..., offset, count) {
      var stride = 4;
      var size = 4;
      for (var i = 0; i < count; ++i) {
         // копировать следующие 4 значения из positionBuffer в атрибут a_position
         attributes.a_position = positionBuffer.slice((offset + i) * stride, size);
         runVertexShader();
         ...
         doSomethingWith_gl_Position();
    }

В действительности всё не так просто, потому что `positionBuffer` понадобилось бы конвертировать
в бинарные данные (см. ниже), поэтому настоящее вычисление для получения данных буфера
немного бы отличалось, но надеюсь этот код дал вам представление о работе вершинного шейдера.

Далее нам понадобится фрагментный шейдер

    // фрагментные шейдеры не имеют точности по умолчанию, поэтому нам необходимо её
    // указать. mediump подойдёт для большинства случаев. Он означает "средняя точность"
    precision mediump float;

    void main() {
      // gl_FragColor - специальная переменная фрагментного шейдера.
      // Она отвечает за установку цвета.
      gl_FragColor = vec4(1, 0, 0.5, 1); // вернёт красновато-фиолетовый
    }

Здесь мы установили `gl_FragColor` в значение `1, 0, 0.5, 1` - 1 для красного, 0 для зелёного,
0.5 для синего и 1 для прозрачности. Цвета в WebGL принимают значения от 0 до 1.

Теперь, когда мы написали 2 функции шейдеров, давайте займёмся самим WebGL.

Для начала нам понадобится HTML-элемент canvas

     <canvas id="c"></canvas>

Далее получаем ссылку на него из JavaScript

     var canvas = document.getElementById("c");

Теперь мы можем получить объект WebGLRenderingContext - контекст отрисовки WebGL

     var gl = canvas.getContext("webgl");
     if (!gl) {
        // у вас не работает webgl!
        ...

Далее необходимо скомпилировать наши шейдеры, чтобы передать их на видеокарту, но сначала нужно
преобразовать их в строки. Строки для GLSL создаются так же, как и строки в JavaScript. Например, через конкатенацию,
или используя AJAX для их загрузки, или используя шаблонные строки, ну или в нашем случае помещая строки в специальные
теги с типом, не равным "JavaScript"

    <script id="2d-vertex-shader" type="notjs">

      // атрибут, который будет получать данные из буфера
      attribute vec4 a_position;

      // все шейдеры имеют функцию main
      void main() {

        // gl_Position - специальная переменная вершинного шейдера,
        // которая отвечает за установку положения
        gl_Position = a_position;
      }

    </script>

    <script id="2d-fragment-shader" type="notjs">

      // фрагментные шейдеры не имеют точности по умолчанию, поэтому нам необходимо её
      // указать. mediump подойдёт для большинства случаев. Он означает "средняя точность"
      precision mediump float;

      void main() {
        // gl_FragColor - специальная переменная фрагментного шейдера.
        // Она отвечает за установку цвета.
        gl_FragColor = vec4(1, 0, 0.5, 1); // вернёт красновато-фиолетовый
      }

    </script>

На самом деле большинство 3D движков генерируют шейдеры GLSL на лету, используя различные шаблоны, объединения и так далее.
Однако, ни один пример на этом сайте не обладает такой сложностью, чтобы потребовалось генерация на лету.

Ещё нам понадобится функция, которая создаст шейдер, загрузит код GLSL и скомпилирует шейдер.
Для неё я не писал комментариев, так как из названий функций должно быть понятно, что происходит.
(а я всё-таки напишу - прим.пер.)

    function createShader(gl, type, source) {
      var shader = gl.createShader(type);   // создание шейдера
      gl.shaderSource(shader, source);      // устанавливаем шейдеру его программный код
      gl.compileShader(shader);             // компилируем шейдер
      var success = gl.getShaderParameter(shader, gl.COMPILE_STATUS);
      if (success) {                        // если компиляция прошла успешно - возвращаем шейдер
        return shader;
      }

      console.log(gl.getShaderInfoLog(shader));
      gl.deleteShader(shader);
    }

Теперь создадим 2 шейдера с помощью этой функции

    var vertexShaderSource = document.getElementById("2d-vertex-shader").text;
    var fragmentShaderSource = document.getElementById("2d-fragment-shader").text;

    var vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
    var fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);

Далее мы должны *связать* эти 2 шейдера с *программой*

    function createProgram(gl, vertexShader, fragmentShader) {
      var program = gl.createProgram();
      gl.attachShader(program, vertexShader);
      gl.attachShader(program, fragmentShader);
      gl.linkProgram(program);
      var success = gl.getProgramParameter(program, gl.LINK_STATUS);
      if (success) {
        return program;
      }

      console.log(gl.getProgramInfoLog(program));
      gl.deleteProgram(program);
    }

И вызвать эту функцию

    var program = createProgram(gl, vertexShader, fragmentShader);

Теперь, когда мы создали программу на видеокарте, нам нужно снабдить её данными.
Большая часть WebGL API занимается установкой состояния для последующей передачи данных в нашу программу GLSL.
В нашем случае единственными входными данными программы является атрибут `a_position`.
Первое, что мы должны сделать - получить ссылку на атрибут для только что созданной программы

    var positionAttributeLocation = gl.getAttribLocation(program, "a_position");

Получение ссылки на атрибут (и ссылки на uniform-переменную) следует
выполнять во время инициализации, но не во время цикла отрисовки.

Атрибуты получают данные от буферов, поэтому нам нужно создать буфер

    var positionBuffer = gl.createBuffer();

WebGL позволяет управлять многими своими ресурсами через глобальные точки связи.
Точки связи - это что-то вроде внутренних глобальных переменных в WebGL.
Сначала вы привязываете ресурс к точке связи. А после все остальные функции
обращаются к этому ресурсу через его точку связи. Итак, привяжем буфер положений.

    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

Теперь можно наполнить буфер данными, указав буфер через его точку связи

    // три двумерных точки
    var positions = [
      0, 0,
      0, 0.5,
      0.7, 0,
    ];
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);

Здесь происходит несколько вещей. Сперва у нас есть JavaScript-массив `positions`.
Но для WebGL нужны строго типизированные данные, поэтому нам нужно явно создать
массив 32-битных чисел с плавающей точкой через `new Float32Array(positions)`, куда
скопируются значения из массива `positions`. Далее `gl.bufferData` копирует типизированные
данные в `positionBuffer` на видеокарте. Копирование происходит в буфер положений,
потому что мы привязали его к точке связи `ARRAY_BUFFER` выше.

Через последний аргумент `gl.STATIC_DRAW` мы указываем, как WebGL должен использовать данные.
WebGL может использовать эту подсказку для оптимизации определённых вещей. `gl.STATIC_DRAW`
говорит о том, что скорей всего мы не будем менять эти данные.

Код до этого момента предназначен для *инициализации*. Это код, который запускается один раз
при загрузке страницы. Далее идёт код *рендеринга*, который будет выполняться каждый раз, когда
происходит отрисовка.

## Рендеринг

Перед отрисовкой нам нужно изменить размер canvas, чтобы он соответствовал размеру экрана. Canvas очень похож
на изображение и имеет два размера. Один размер - это количество пикселей в исходном изображении, второй - размер
на HTML-странице. CSS определяет размер на HTML-странице. **Всегда следует задавать требуемый размер через CSS**,
так как этот метод несравнимо удобнее по сравнению с другими методами.

Чтобы количество пикселей в canvas совпадало с резмером на HTML-странице,
я использую функцию-помощник, о которой вы можете прочесть [по этой ссылке](webgl-resizing-the-canvas.html).

Практически во всех примерах здесь canvas имеет размер 400х300 пикселей, если пример работает
в собственном окне, и растягивается на всё доступное пространство, если canvas находится внутри
iframe, как на этой странице. Если определять размер через CSS, а зачем подстраивать размер
элемента HTML-страницы, пример будет работать в этих обоих случаях.

    webglUtils.resizeCanvasToDisplaySize(gl.canvas);

Нам нужно указать, как координаты из пространства отсечения, которые мы задаём
через `gl_Position`, преобразовать в пиксели, часто называемыми экранными координатами.
Для этого мы вызовем `gl.viewport` и передадим текущий размер canvas.

    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

Это укажет WebGL, что диапазон координат отсечения от -1 до +1 соответствует от 0 до `gl.canvas.width` по x
и от 0 до `gl.canvas.height` по y.

Теперь очищаем canvas. `0, 0, 0, 0`  - красный, зелёный, синий и прозрачность,
то есть наш canvas полностью прозрачный.

    // очищаем canvas
    gl.clearColor(0, 0, 0, 0);
    gl.clear(gl.COLOR_BUFFER_BIT);

Указываем WebGL, какую шейдерную программу нужно выполнить

    // говорим использовать нашу программу (пару шейдеров)
    gl.useProgram(program);

Теперь нужно сказать WebGL, как извлечь данные из буфера, который мы настроили ранее, и передать их
в атрибут шейдера. Для начала необходимо включить атрибут

    gl.enableVertexAttribArray(positionAttributeLocation);

А затем требуется указать, как извлекать данные из буфера

    // Привязываем буфер положений
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

    // Указываем атрибуту, как получать данные от positionBuffer (ARRAY_BUFFER)
    var size = 2;          // 2 компоненты на итерацию
    var type = gl.FLOAT;   // наши данные - 32-битные числа с плавающей точкой
    var normalize = false; // не нормализовать данные
    var stride = 0;        // 0 = перемещаться на size * sizeof(type) каждую итерацию для получения следующего положения
    var offset = 0;        // начинать с начала буфера
    gl.vertexAttribPointer(
        positionAttributeLocation, size, type, normalize, stride, offset)

У `gl.vertexAttribPointer` есть не видная на первый взгляд особенность - он привязывает
к атрибуту текущий `ARRAY_BUFFER`. Другими словами, этот атрибут привязан к `positionBuffer`.
Это означает, что мы теперь можем свободно привязать что-нибудь другое к точке связи
`ARRAY_BUFFER`, но атрибут останется привязанным к `positionBuffer`.

Обратите внимание, что с точки зрения вершинного шейдера GLSL атрибут `a_position` имеет тип `vec4`

    attribute vec4 a_position;

`vec4` - это 4 значения с плавающей точкой. В JavaScript это бы выглядело примерно как
`a_position = {x: 0, y: 0, z: 0, w: 0}`. Немного выше мы установили `size = 2`. Значениями
по умолчанию для атрибута являются `0, 0, 0, 1`, поэтому атрибут получит его первые два
значения (x и y) из буфера, а z и w примут значения по умолчанию 0 и 1 соответственно.

И в итоге после всего этого мы говорим WebGL выполнить нашу программу GLSL

    var primitiveType = gl.TRIANGLES;
    var offset = 0;
    var count = 3;
    gl.drawArrays(primitiveType, offset, count);

Так как count имеет значение 3, наш вершинный шейдер выполнится 3 раза. Первый раз `a_position.x` и `a_position.y`
в вершинном шейдере примут значение первых двух значений из positionBuffer.
Второй раз в `a_position.xy` попадёт вторая пара значений из positionBuffer.
В последний раз в атрибут атрибут попадёт последняя пара значений.

Так как мы задали значение `primitiveType` для `gl.TRIANGLES`, каждый раз, когда вершинный
шейдер выполняется 3 раза, WebGL будет рисовать треугольник на основе 3 значений, которые
мы установили для `gl_Position`. Неважно, какого размера будет наш canvas, так как значения
находятся в координатах пространства отсечения, занимающих диапазон от -1 до +1 в каждом направлении.

Наш вершинный шейдер просто копирует значения positionBuffer в `gl_Position`, поэтому
треугольник будет отрисован в координатах пространства отсечения

      0, 0,
      0, 0.5,
      0.7, 0,

Далее координаты пространства отсечения преобразуются в экранные координаты,
и если размер canvas будет 400х300, то мы получим следующее:

    пространство       экранные
     отсечения        координаты
       0, 0       ->   200, 150
       0, 0.5     ->   200, 225
       0.7, 0     ->   340, 150

Теперь WebGL займётся отрисовкой треугольника. Для каждого выводимого пикселя WebGL вызовет фрагментный шейдер.
Наш фрагментный шейдер просто установит `gl_FragColor` в значение `1, 0, 0.5, 1`. Так как canvas содержит
8 бит на канал, итоговым значением цвета на canvas будет `[255, 0, 127, 255]`.

Демонстрация работы

{{{example url="../webgl-fundamentals.html" }}}

В рассмотренном выше примере вершинный шейдер занимается только одним делом -
передаёт данные о положении напрямую. Так как данные о положении уже в
пространстве отсечения, больше от шейдера ничего и не требуется. *Если вы хотите
3D, вам самим нужно позаботится о написании шейдеров, которые конвертируют координаты
из пространства отсечения, так как WebGL - это только API растеризации.*

Возможно, вы задаётесь вопросом, почему треугольник начинается в центре и лежит в верхней правой
части. Пространство отсечений по `x` идёт от -1 до +1. Это означает, что 0 находится прямо в центре,
а положительные значения x будут справа.

То же самое и по вертикали - значение -1 находится на нижней границе, а значение +1 - в самом верху.
Другими словами, 0 находится в центре, а позитивные значения будут находится выше центра.

Для работы в 2D-пространстве вы скорей всего будете сразу работать в пиксельных
координатах, нежели в координатах пространства отсечения, поэтому давайте изменим
шейдер соответствующим образом, чтобы он мог конвертировать координаты за нас.
Новый вершинный шейдер выглядит следующим образом

    <script id="2d-vertex-shader" type="notjs">

    -  attribute vec4 a_position;
    *  attribute vec2 a_position;

    +  uniform vec2 u_resolution;

      void main() {
    +    // преобразуем положение в пикселях к диапазону от 0.0 до 1.0
    +    vec2 zeroToOne = a_position / u_resolution;
    +
    +    // преобразуем из 0->1 в 0->2
    +    vec2 zeroToTwo = zeroToOne * 2.0;
    +
    +    // преобразуем из 0->2 в -1->+1 (пространство отсечения)
    +    vec2 clipSpace = zeroToTwo - 1.0;
    +
    *    gl_Position = vec4(clipSpace, 0, 1);
      }

    </script>

Стоит обратить внимание на некоторые изменения. Для `a_position` мы задали тип `vec2`, так
как мы всё равно используем только `x` и `y`. `vec2` похож на `vec4`, но содержит лишь `x` и `y`.

Далее мы добавили uniform-переменную `u_resolution`. Получаем на неё ссылку, чтобы в
дальнейшем мы смогли установить ей значение.

    var resolutionUniformLocation = gl.getUniformLocation(program, "u_resolution");

Остальное должно быть понятным по комментариям. Через установку для `u_resolution` значения,
равному разрешению canvas, шейдер может принимать положения из `positionBuffer` в пиксельных
координатах и преобразовывать их в координаты пространства отсечения.

Теперь изменим значения положений на пиксельные. На этот раз мы отрисуем прямоугольник,
состоящий из 2 треугольников, по 3 точки в каждом.

    var positions = [
    *  10, 20,
    *  80, 20,
    *  10, 30,
    *  10, 30,
    *  80, 20,
    *  80, 30,
    ];
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);

После того как мы установили, какую программу использовать, мы можем устанавливать значение
созданной uniform-переменной. `gl.useProgram` похоже на использование `gl.bindBuffer`.
После выбора программы все функции `gl.uniformXXX` задают значения uniform-переменных текущей программы.

    gl.useProgram(program);

    ...

    // установка разрешения
    gl.uniform2f(resolutionUniformLocation, gl.canvas.width, gl.canvas.height);

И конечно же для отрисовки 2 треугольников нам нужно, чтобы WebGL вызывал наш вершинный шейдер
6 раз, поэтому нужно изменить значение `count` на `6`.

    // отрисовка
    var primitiveType = gl.TRIANGLES;
    var offset = 0;
    *var count = 6;
    gl.drawArrays(primitiveType, offset, count);

И вот наш прямоугольник

Примечание: Этот и следующие примеры используют [`webgl-utils.js`](/webgl/resources/webgl-utils.js),
где содержатся функции для компиляции и компоновки шейдеров. Не будем захламлять примеры
[шаблонным кодом](webgl-boilerplate.html).

{{{example url="../webgl-2d-rectangle.html" }}}

И снова вы могли обратить внимание, что прямоугольник находится внизу. Для WebGL положительное
значение Y направлено вверх, а отрицательное - вниз. В пространстве отсечения нижний левый угол
имеет координаты -1,-1. Мы не меняли знаков, поэтому с нашей текущей математикой координаты 0, 0
будут находиться в нижнем левом углу. Для того, чтобы начало координат находилось в более
привычном верхнем левом углу, как это принято в 2D-системах, мы можем просто перевернуть
y-координату пространства отсечения.

    *   gl_Position = vec4(clipSpace * vec2(1, -1), 0, 1);

И теперь наш прямоугольник находится на своём месте

{{{example url="../webgl-2d-rectangle-top-left.html" }}}

Теперь давайте вынесем код создания прямоугольника в функцию, чтобы
мы могли вызывать её для прямоугольников разного размера. И заодно
сделаем, чтобы цвет можно было выбирать.

Сначала сделаем фрагментный шейдер, который принимает цвет через uniform-переменную

    <script id="2d-fragment-shader" type="notjs">
      precision mediump float;

    +  uniform vec4 u_color;

      void main() {
    *    gl_FragColor = u_color;
      }
    </script>

И теперь код, который создаёт 50 прямоугольников в произвольных местах со случайным цветом.

      var colorUniformLocation = gl.getUniformLocation(program, "u_color");
      ...

      // создаём 50 прямоугольников в произвольных местах со случайным цветом
      for (var ii = 0; ii < 50; ++ii) {
        // задаём произвольный прямоугольник
        // Запись будет происходить в positionBuffer,
        // так как он был привязан последник к
        // точке связи ARRAY_BUFFER
        setRectangle(
            gl, randomInt(300), randomInt(300), randomInt(300), randomInt(300));

        // задаём случайный цвет
        gl.uniform4f(colorUniformLocation, Math.random(), Math.random(), Math.random(), 1);

        // отрисовка прямоугольника
        gl.drawArrays(gl.TRIANGLES, 0, 6);
      }
    }

    // возврат случайного целого числа значением от 0 до range-1
    function randomInt(range) {
      return Math.floor(Math.random() * range);
    }

    // заполнение буфера значениями, которые определяют прямоугольник

    function setRectangle(gl, x, y, width, height) {
      var x1 = x;
      var x2 = x + width;
      var y1 = y;
      var y2 = y + height;

      // ПРИМ.: gl.bufferData(gl.ARRAY_BUFFER, ...) воздействует
      // на буфер, который привязан к точке привязке `ARRAY_BUFFER`,
      // но таким образом у нас будет один буфер. Если бы нам понадобилось
      // несколько буферов, нам бы потребовалось привязать их сначала к `ARRAY_BUFFER`.

      gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([
         x1, y1,
         x2, y1,
         x1, y2,
         x1, y2,
         x2, y1,
         x2, y2]), gl.STATIC_DRAW);
    }

И наши прямоугольники:

{{{example url="../webgl-2d-rectangles.html" }}}

Я надеюсь вы увидели, что WebGL на самом деле достаточно простое API.
Ну ладно, "простое", возможно, неподходящее слово. Но то, чем занимается WebGL -
достаточно просто. Он всего лишь выполняет 2 указанные пользователем функции -
вершинный и фрагментный шейдер, - и отрисовывает треугольники, линии и точки.
В то же время вы как программист можете делать более сложные шейдеры,
чтобы выполнять более сложные задачи 3D. Но WebGL сам по себе - просто
средство растеризации и концептуально довольно лёгок в понимании.

Мы рассмотрели небольшой пример, где увидели, как передавать данные атрибутам и 2
uniform-переменным. Обычно используется несколько атрибутов и множество uniform-переменным.
В начале статьи мы упоминали *varying-переменные* и *текстуры*. Их мы затронем в следующих уроках.

Перед тем, как двигаться дальше, я бы хотел обратить внимание, что *большинство* приложений
обновляет данные в буфере не так, как мы делали в `setRectangle`. Я использовал этот пример,
потому что его проще объяснить, так как он работает с пиксельными координатами на входе и
содержит мало математики в GLSL. Он не ошибочный, нет - есть множество ситуаций, когда так
можно и нужно делать, но советую [продолжить чтение и узнать более распространённый способ
для позиционирования, направления и масштабирования объектов в WebGL](webgl-2d-translation.html).

Если вы ранее не имели опыта в веб-разработке, и даже если такой опыт был, взгляните на
[Настройку и установку](webgl-setup-and-installation) для советов по разработке WebGL

Если вы абсолютный новичок в WebGL и не имеете представления о GLSL, шейдерах или графическом
процессоре, посмотрите на [основы работы WebGL](webgl-how-it-works.html).

Также рекомендую хотя бы кратко ознакомиться с [шаблоном используемого здесь кода](webgl-boilerplate.html),
который используется в большинстве примеров. Так же хотя бы по диагонали взгляните на то,
[как отрисовать несколько объектов](webgl-drawing-multiple-things.html), что даст вам понимание,
как более типичные приложения WebGL построены архитектурно. К сожалению, практически все примеры
отрисовывают только один объект и поэтому не показывают структуру приложения.

Иначе вы можете продолжить обучение в 2 направлениях. Если вам интересна обработка изображений,
я покажу, [как выполнить обработку 2D-изображения](webgl-image-processing.html).
Если вам интересно узнать о переносе, повороте, масштабе и в конце концов 3D,
тогда [начните здесь](webgl-2d-translation.html).

<div class="webgl_bottombar">
<h3>Что означает type="notjs"?</h3>
<p>
Теги <code>&lt;script&gt;</code> по умолчанию содержат код JavaScript.
Вы можете не указывать тип, или указать <code>type="javascript"</code>,
или <code>type="text/javascript"</code>, и браузер будет понимать содержимое
как JavaScript. Если вы укажете что-либо другое для <code>type</code>, браузер
проигнорирует содержимое тега скрипта. Другими словами, <code>type="notjs"</code>
или <code>type="foobar"</code> не имеют значения для браузера

<p>В таком виде шейдеры легко редактировать.
Альтернативным способом является конкатенация строк, например</p>
<pre class="prettyprint">
  var shaderSource =
    "void main() {\n" +
    "  gl_FragColor = vec4(1,0,0,1);\n" +
    "}";
</pre>
<p>или мы бы могли загружать шейдеры через ajax, но это медленно, ещё и асинхронность добавится.</p>
<p>Более современной альтернативой было бы использование шаблонных строк</p>
<pre class="prettyprint">
  var shaderSource = `
    void main() {
      gl_FragColor = vec4(1,0,0,1);
    }
  `;
</pre>
<p>Шаблонные строки работают во всех браузерах, поддерживающих WebGL.
К сожалению, они не работают в действительно старых браузерах, поэтому
если вам нужно поддерживать устаревшие браузеры тоже, вероятно вы не
захотите использовать шаблонные строки или воспользуетесь <a href="https://babeljs.io/">препроцессором</a>.
</p>
</div>
