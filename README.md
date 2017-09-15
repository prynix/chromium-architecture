# Chromium архитектура

Chromium — графический веб-браузер с открытым исходным кодом, основанный на движке Blink и разрабатываемый корпорацией Google совместно с сообществом The Chromium Authors, и некоторыми другими корпорациями. На основе Chromium создан браузер Google Chrome, на сегодняшний день его доля превысила 60%. Поэтому хотелось бы поговорить о нем чуть больше, чем об остальных, хотя про Firefox, Edge, Safari и их организацию было бы тоже здорово посмотреть, но не сегодня. Отмечу, что не стоит под Chromium подразумевать Google Chrome, так как одно определение дополняет другое. Google Chrome = Chromium + Google Update + закрытые плагины и кодеки + отправка отчетов и статистики.

<h1 id="#Browser_Process">Browser process</h1>

![image](http://szeged.github.io/sprocket/img/arch/Browser_Process.png)

- **Startup**
  - Это точка входа всего приложения (Main)
  - Main запускает ContentMain и BrowserMain
  - BrowserMain запускает MainMessageLoop

- **Threads**

  - Сразу после запуска запускаются необходимые потоки
  - **I/O thread**
    - Обеспечивает коммуникацию с задачами рендеринга
  - **DB thread**
    - Подключение базы данных sqlite и выполнение запросов
  - **Cache thread**
    - Хранение и извлечение кеша
  - **Worker threads**
    - Рабочие потоки запускают задачи, которые не требуют отдельных потоков или собственного цикла сообщений (event loop)
    - Существует пул потоков, называемый WorkerPool, который динамически добавляет потоки (при необходимости) для обработки всех задач
    - Существуют различные реализации для POSIX и не-POSIX-систем

- **Loop**
  - MainMessageLoop
    - Используется для обработки событий определенного потока
    - Помещает входящие сообщения, а также задачи в очередь
    - Выдает задачу из очереди и запускает ее
    - Обеспечивает прочное соединениние с IPC
    - Имеет защиту от повторной установки задачи
      - Повторная задача не может быть запущена до завершения предыдущей задачи
      
- **IPC / Mojo**
  - Это структура, используемая для межпроцессного взаимодействия
  - Подключается непосредственно к MainMessageLoop
  - Предоставляет каналы связи, через которые могут быть отправлены сообщения
  - Обеспечивает создание, отправку и получение сообщений
  - Обеспечивает асинхронную обработку сообщений

- **TaskAnnotator**
  - Все входящие задачи проходят через TaskAnnotator, который аннотирует задачу перед выполнением
  - Реализует общие отладочные аннотации для размещенных задач. Сюда входят такие данные, как происхождение задачи, длительность очередей и использование памяти
  - Выполняет ранее поставленную задачу
  
- **ResourceLoader**
  - Диспетчеризация ресурсов
  - Получает запросы от дочерних процессов (Renderer, Worker и т.д.)
  - Отправляет полученные запросы в URLRequests
  - Пересылает сообщения из URLRequests в корректный процесс обработки

- **URL**
  - Эта группа содержит все соответствующие URL-адреса
  - Производит замену URL и/или расширяет URL-адреса
  - Автозаполнение URL
  - Извлечение поисковых запросов из URL-адреса
  - Разбор URL
  - Обеспечивает нормализацию (канонизация) URL
  - Использование Omnibox (многофункциональная адресная строка Chromium)
  
- **SQL**
  - Включает наборв классов, которые взаимодействуют с базой данных sqlite3
  - Загрузка/обновление url при автозаполнении
  - Загрузка сохраненных favicons

- **net**
  - Работа с NetworkDelegate
    - Выполняет действия до запуска URLRequest
  - Работа с URLRequests
  - Обработка файлов cookie
    - Загружает асинхронно все файлы cookie для заданного URL-адреса 
    - Устанавливает все файлы cookie для данного URL-адреса
  - Работа с SSL-сертификатами
    - Обрабатывает действия, связанные с SSL
    - Подтверждение SSL
    - Подтверждение сертификации
    - Проверка подписи
   
- **Compositor (cc)**
  - PaintFrame
    - Прорисовка основного кадра
    - Подготовка тайлов
    - Обновление слоев кадра
    - Обновление дополнительных слоев изображения
    - Работа со списком отображения обновлений (отрисовка, обновление)
    - Вызов ауры изображения
    - Работа с интерфейсным стеком Aura для работы с вкладками браузера
    
  - Swap
    - Работа с 2D-плоскостью
    - Работа со swap-буфером
  
  - RasterTask
    - Выполнение растеризации
    - Представление задач отрисовки в виде графа:
      - грани: это зависимости
      - узлы: являются задачами, вес узла определяет ее приоритет
    - Элементы в из списка отображения выводятся на 2D-плоскость
    - Растеризация вызывает определенные функции Skia, чтобы правильно отрисовать текущий холст (drawColor, drawPicture, drawRect, fillRect, и т.д.)
    
- **X11/Windows/Mac**
  - Захватывание событий мыши, клавиатуры и передача их Chromium
  
- **UIEvent**
  - Работа с классами представленными в пространстве имен ui (обеспечивают работу с пользовательским интерфейсом)
  - Одной из важных обязанностей является обработка событий пользовательского интерфейса, например, mousemove, mouseclick, keypress и т. д.
  - Эти события передаются из библиотеки рабочего окна
  - События попадают в цепочку обработки событий
  - Если пользователь вводит URL-адрес в адресную строку (URLBar), эти классы выполняют вставку символов в текстовое поле URLBar
  - Aura:
    - UI framework, диспетчер окон рабочего стола и среды оболочки
    - Кроссплатформенный
    - Также используется в качестве графической оболочки для Chrome OS / Chrome / Chromium
    - Chrome OS использует его, а также Chrome / Chromium
    - Aura обеспечивает взаимодействие окон и разных типов событий, контролирует поведение
    
<br><br><br>
<hr>

<h1 id="#Zygote_Process">Zygote process</h1>

![image](http://szeged.github.io/sprocket/img/arch/Zygote_Process.png)

- **Startup**
  - Запускается после запуска родительского процесс "Browser process"

- **Zygote**
  - После запуска родительского процесса создается Zygote-процесс

- **Fork**
  - Zygote-процесс разветвляется (forked), чтобы создать процессы Renderer

- **Loop**
  - MainMessageLoop
    - Используется для обработки событий определенного потока
    - Помещает входящие сообщения, а также задачи в очередь
    - Выдает задачу из очереди и запускает ее
    - Обеспечивает прочное соединениние с IPC
    - Имеет защиту от повторной установки задачи
      - Повторная задача не может быть запущена до завершения предыдущей задачи
      
- **IPC / Mojo**
  - Это структура, используемая для межпроцессного взаимодействия
  - Подключается непосредственно к MainMessageLoop
  - Предоставляет каналы связи, через которые могут быть отправлены сообщения
  - Обеспечивает создание, отправку и получение сообщений
  - Обеспечивает асинхронную обработку сообщений

- **TaskAnnotator**
  - Все входящие задачи проходят через TaskAnnotator, который аннотирует задачу перед выполнением
  - Реализует общие отладочные аннотации для размещенных задач. Сюда входят такие данные, как происхождение задачи, длительность очередей и использование памяти
  - Выполняет ранее поставленную задачу
  
- **callback**
  - Этот блок представляет собой обратные вызовы, связанные с системой или сообщением
  
- **Scheduler**
  - Пакет, который содержит набор классов работающих по расписанию
  - TaskQueueManager
    - Диспетчер очереди задач предоставляет N очередей задач и интерфейс селектора для выбора задачи из очереди. Каждая очередь задач состоит из двух дополнительных очередей:
      - Входящая очередь задач
      - Рабочая очередь

- **blink::HTMLDocumentParser**
  - Парсинг HTML-документа
  - Построение дерева DOM
   
- **autofill**
  - AutofillAgent занимается связью с Autofill, связанной между WebKit и браузером
  - Подсчет APR (AutofillAgent per RenderFrame)
  - Autofill охватывает:
    - Работа с Autocomplete
    - Работа с заполнение формы пароля, называемое автозаполнением пароля, и заполнение всей формы на основе одной записи поля, называемой формой автозаполнения.
    
- **icu**
  - Это библиотека для работы с unicode в том числе содержит ряд инструментов для сравнения строк, получения даты и времени в самых разных форматах и иные
  - В текущем контексте icu используется для сопоставления шаблонов регулярных выражений, чтобы определить, соответствует ли автозаполнение конкретному регулярному выражению

- **content::ResourceDispatcher**
  - Этот комопонент служит интерфейсом для связи с диспетчером ресурсов
  - Он может использоваться из любого дочернего процесса
  - Отправляет входящие сообщения
  
- **blink::TimerBase**
  - Базовый таймер

- **blink::FrameLoader**
  - Управляет загрузкой определенного кадра (страницы)
  
- **blink::EventHandler**
  - Обрабатывает события, такие как выделение, перетаскивание, жесты, перемещение мыши, нажатия клавиатуры
  - Выполняет hit-тесты (hit detection, picking, pick correlation)
  
- **blink::EventDispatcher**
  - Рассылает простые события, скоординированные события или имитируемые клики
 
- **blink::Document**
  - blink::DocumentLoader отвечает за загрузку документа
  - После применение CSS стилей (blink::StyleResolver) и расчет макета обновленяет графические слои
 
- **media::DecoderStream**
  - Обертывает DemuxerStream и список декодеров, и предоставляет вывод для клиента (Audio/VideoRendererImpl)
  
- **blink::ScriptRunner**
  - Выполнение JavaScript

- **content::RenderThread**
  - Поток, который используется для рендеринга задач в процессе Renderer и Zygote (запускается в однопоточном режиме)
  
- **blink::V8Initializer**
  - Имеет только статические методы для инициализации контекста движка V8
    - В основном потоке
    - В рабочем потоке
    
- **extensions**
  - Представляет собой extensions-модуль для работы с расширениями браузера
  - Реализует основные части расширения Chrome и может взаимодействовать с любым content-модулем

- **blink::WorkerThread**
  - Поток, который может выполнять только определенные задачи
  - Определенные задачи могут быть отправлены в WorkerThread
  - Вызывает WorkerScriptController::initializeContextifNeeded для выполнения JavaScript посредством V8

- **Compositor (cc)**
  - Использует несколько хранилищ резервных копий для кэширования и группирования фрагментов дерева рендеринга
  - Избегает ненужной перерисовки
  - Выполняет первичные задачи компоновки:
    - Определяет, как группировать содержимое в резервном хранилище (композитные слои)
    - Выполнение закрашивания каждого композитного слоя
    - Отрисовка окончательного изображения
  - Отрисовывает содержимое слоев из списка отображения
  - Обрабатывает обновления слоев
  
- **Skia**
  - Использование Blink-библиотек отрисовки 
  - Растеризация вызывает определенные функции Skia, для правильной отрисовки холста (drawColor, drawPicture, drawRect, fillRect и т.д.)
  
- **blink::ImageDecoder**
  - ImageDecoder является основой для всех декодеров изображений, специфичных для определенных форматов (например, JPEGImageDecoder). Он управляет кешем ImageFrame.

- **content::WebGraphicsContext3DCommandBuffer**
  - Выполнение методов 3D отрисовки для графики
  - Отправляет инструкции в GpuChannelHost, которые:
    - Инкапсулирует IPC-канал между клиентом и одним GPU
    - На стороне GPU имеется соответствующий GpuChannel
    - Каждый метод может быть вызван в любом потоке, за исключением потока ввода-вывода
 
- **RasterTask** 
  - Выполнение растеризации
  - Представление задач отрисовки в виде графа:
    - грани: это зависимости
    - узлы: являются задачами, вес узла определяет ее приоритет
  - Растеризация вызывает gpu::gles2::QueryTracker методы для создания запросов к графическому процессору  

- **gpu::gles2**
  - QueryTracker
    - Отслеживает запросы клиентской части командного буфера
  - QueryManager
    - Отслеживает запросы и их состояние, поскольку запросы не используются, есть один QueryManager для каждого контекста
  - Отправляет запросы в content::CommandBufferProxyImpl
    - Это клиентский прокси-сервер, который синхронно пересылает сообщения в CommandBufferStub
    
<br><br>
<hr>

<h1 id="#Renderer_Process">Renderer process</h1>

![image](http://szeged.github.io/sprocket/img/arch/Renderer_Process.png)

- **Startup**
  - Запускается после запуска родительского процесс "Browser process" (и запущенного Zygote-процесс)

- **Renderer**
  - Renderer-процесс является разветвлением (forked) Zygote-процесса

- **Loop**
  - MainMessageLoop
    - Используется для обработки событий определенного потока
    - Помещает входящие сообщения, а также задачи в очередь
    - Выдает задачу из очереди и запускает ее
    - Обеспечивает прочное соединениние с IPC
    - Имеет защиту от повторной установки задачи
      - Повторная задача не может быть запущена до завершения предыдущей задачи
      
- **IPC / Mojo**
  - Это структура, используемая для межпроцессного взаимодействия
  - Подключается непосредственно к MainMessageLoop
  - Предоставляет каналы связи, через которые могут быть отправлены сообщения
  - Обеспечивает создание, отправку и получение сообщений
  - Обеспечивает асинхронную обработку сообщений

- **TaskAnnotator**
  - Все входящие задачи проходят через TaskAnnotator, который аннотирует задачу перед выполнением
  - Реализует общие отладочные аннотации для размещенных задач. Сюда входят такие данные, как происхождение задачи, длительность очередей и использование памяти
  - Выполняет ранее поставленную задачу

- **Scheduler**
  - Пакет, который содержит набор классов работающих по расписанию
  - TaskQueueManager
    - Диспетчер очереди задач предоставляет N очередей задач и интерфейс селектора для выбора задачи из очереди. Каждая очередь задач состоит из двух дополнительных очередей:
      - Входящая очередь задач
      - Рабочая очередь
      
- **content::MessageRouter**
  - MessageRouter обрабатывает все входящие сообщения, отправленные ему, перенаправляя их правильному подписчику, основываясь на идентификаторе маршрутизации сообщения
  - Маршрутизация основана на идентификаторе маршрутизации сообщения
  - Поскольку идентификаторы маршрутизации обычно назначаются асинхронно в процессе работы браузера, MessageRouter имеет ожидающие идентификаторы для подписчиков, которым еще не присвоен идентификатор маршрутизации
  
- **content::RenderWidget**
  - RenderWidget обеспечивает коммуникационный мост между WebWidget и RenderWidgetHost, последний из которых работает в другом процессе
  - Обрабатывает входящее сообщение в методе OnMessageReceived
  - Обрабатывает входящие события в цепочке blink::handleInputEvent
  - В случае наведения события мыши на требуемый элемент, у которого установлен tooltip, выполняется hit-тест
  - Отправляет ответы через IPC

- **content::InputHandler**
  - content::InputHandlerManager
    - Управляет экземплярами InputHandlerProxy для WebView в рендеринге
  - content::InputHandlerProxy
    - Этот класс является прокси-сервером фильтрирующим события ввода и обработки композиции. Экземпляры InputHandlerProxy полностью связаны с потоком компоновщика. Каждый экземпляр InputHandler обрабатывает входные события, предназначенные для определенного WebWidget
    - Вызывает конкретные функции Compositor компонента в результате входного события

- **Compositor (cc)**
  - Compositor вызывает конкретные функции gfx и GL/GLES для выполнения правильной отрисовки

- **blink::ScriptController**
  - Интерпретирует JavaScript и получает возвращаемое значение через объект V8ScriptRunner
  
- **V8**
  - Это JavaScript-движок встроенный в Blink
  - JavaScript
    - Парсит
    - Компилирует
    - Выполняет
  - Выполняет обратные вызовы для изменения DOM-дерева
  - Работает с памятью
    - Выделение памяти
    - Сборка мусора

<br><br><br>
<hr>

<h1 id="#Utility_process">Utility process</h1>

![image](http://szeged.github.io/sprocket/img/arch/Utility_process.png)

<br><br><br>
<hr>

<h1 id="#GPU_Process">GPU process</h1>

![image](http://szeged.github.io/sprocket/img/arch/GPU_Process.png)
