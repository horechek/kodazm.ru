+++
date = "2014-12-14T17:42:05+03:00"
draft = false
title = "Модели конкурентности в Go. Конвейеры."

+++

<p>Перевод статьи из официального блога Go <a href="http://blog.golang.org/pipelines">"Go Concurrency Patterns: Pipelines and cancellation"</a>. Автор Sameer Ajmani</p>

<h3>Примечания.</h3>

<p>Давайте сразу договоримся, как переводить "сoncurrency". В русском языке аналога пока нет. Самое близкое - это "параллельность". Но нам это не подходит, потому что с технической стороны "сoncurrency" это совсем не "параллельность". Давайте будем называть это "конкурентность". Такая же проблема и с "pipeline". Как мне кажется, наиболее точный аналог это "конвейер".</p>

<h3>Введение</h3>

<p>Примитивы конкурентности в Go позволяют построить эффективные конвейеры потоковой обработки данных, которые эффективно используют CPU и I/O. В этой статье описаны приемы создания таких конвейеров, тонкости использования и некоторые способы решения возникающих проблем.</p>

<h3>Что такое конвейеры(pipeline)?</h3>

<p>В Go нет четкого понятия для конвейера, это всего лишь один из видов параллельного программирования. Неформальное определение конвейера - это ряд этапов, связанных между собой каналами, где каждый этап это набор go-рутин выполняющих определенную функцию. На каждом этапе выполняются определенные действия:</p>

<ul>
<li>Получить значения с предыдущего этапа.</li>
<li>Выполняются какие либо действия над этими значениями. Как правило создаются новые значения.</li>
<li>Значения отправляются на следующий этап через выходные каналы.</li>
</ul>

<p>На всех этапах может быть сколько угодно входящих и выходящих каналов, кроме первого и последнего, на которых есть только выходящий и входящий канал. Первый этап иногда называют источник(source) или отправитель(producer). Последний этап называется потребитель(consumer, sink).</p>

<p>Мы начнем с простого примера конвейера для быстрого понимания принципов и идей. Позже разберем более реалистичный пример использования такого подхода.</p>

<h3>Возведение в квадрат</h3>

<p>Рассмотри трехэтапный конвейер.</p>

<p>Превый этап. <code>gen</code> это функция, которая преобразует список целых чисел в канал, который посылает числа из этого списка. Внутри этой функции запускается go-рутина, которая отправляет целые числа в канал и закрывает этот канал, когда все числа отправлены:</p>

<pre><code class="go">func gen(nums ...int) &lt;-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out &lt;- n
        }
        close(out)
    }()
    return out
}
</code></pre>

<p>Второй этап. <code>sq</code> забирает числа из канала и возвращает новый канал, который отдает квадрат каждого полученного числа. После того как входящий канал закрыт и все значения на этом шаге отправлены в исходящий канал, то исходящий канал закрывается:</p>

<pre><code class="go">func sq(in &lt;-chan int) &lt;-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out &lt;- n * n
        }
        close(out)
    }()
    return out
}
</code></pre>

<p>Функция <code>main</code> оперирует каналами и реализует последний этап. Принимает значения, полученные на втором этапе, и выводит каждое, пока канал не закроется:</p>

<pre><code class="go">func main() {
    // Создаем необходимые каналы.
    c := gen(2, 3)
    out := sq(c)

    // Выводим значения.
    fmt.Println(&lt;-out) // 4
    fmt.Println(&lt;-out) // 9
}
</code></pre>

<p>Функция <code>sq</code> принимает и возвращает каналы одинакового типа. А это значит, что мы можем компоновать эти функции сколько угодно. Кроме того, мы можем переписать функцию <code>main</code> с использованием <code>range</code>:</p>

<pre><code class="go">func main() {
    // Создаем необходимые каналы и выводим значения.
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16 затем 81
    }
}
</code></pre>

<h3>Fan-out, fan-in</h3>

<p><em>Прим.</em> "fan" здесь, вероятно, стоит переводить как "лопастной вентилятор" или, с технической стороны, как "револьверный барабан".</p>

<p>Несколько функций могут читать из одного канала пока он не закроется. Это называется <em>fan-out</em>. Такой подход дает возможность распределять задачи между так называемыми воркерами(исполнителями).</p>

<p>Функция может читать из нескольких входных каналов, мультиплексировать все в один канал и обрабатывать, пока входные каналы не будут закрыты. Это называется <em>fan-in</em>.</p>

<p>Мы можем изменить наш пример так, чтобы запускались два экземпляра <code>sq</code>. Каждый экземпляр читает данные из входного канала. Мы добавим еще одну функцию - <em>merge</em>. Эта функция будет реализовывать <em>fan-in</em> для наших результатов:</p>

<pre><code class="go">func main() {
    in := gen(2, 3)

    // Распределяем работу между двумя воркерами для считывания данных из `in`.
    c1 := sq(in)
    c2 := sq(in)

    // Объединяем вывод из c1 и c2.
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4 затем 9, или 9 затем 4
    }
}
</code></pre>

<p>Функция <code>merge</code> преобразует несколько каналов в один канал, запуская go-рутину для каждого входного канала. Внутри этих рутин значения копируются в один выходной канал. После того как все go-рутины, формирующие выходной канал, запущены, стартует еще одна go-рутина, котора нужна для закрытия выходного канала после отправки в него всех данных.</p>

<p>Отправка данных в закрытый канал спровоцирует панику. Поэтому, очень важно убедиться, что все данные отправлены до вызова <code>close</code>. В нашем случае используется <a href="http://golang.org/pkg/sync/#WaitGroup"><code>sync.WaitGroup</code></a>. Этот способ обеспечивает простую синхронизацию:</p>

<pre><code class="go">func merge(cs ...&lt;-chan int) &lt;-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Запуск go-рутины для каждого входного канала из `cs`. `output`
    // копирует значения из входного канала `с` пока `с` не будет 
    // закрыт. Затем вызывается `wg.Done`.
    output := func(c &lt;-chan int) {
        for n := range c {
            out &lt;- n
        }
        wg.Done()
    }
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // Запуск go-рутины, которая закроет `out` канал после 
    // завершения всех  `output` go-рутин. Этот код должен 
    // выполняться только после вызова `wg.Add`.
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
</code></pre>

<h3>Быстрая остановка(Stopping short)</h3>

<p>До этого мы рассматривали паттерны в которых четко обозначены этапы:</p>

<ul>
<li>Закрыть исходящие каналы, после выполнения всех операций.</li>
<li>Получить значения из входящих каналов, пока они не будут закрыты.</li>
</ul>

<p>Такой подход позволяет на каждом этапе получать значения с помощью <code>range</code> и гарантирует, что все go-рутины завершатся после того как все значения будут отправлены.</p>

<p>Но в реальном мире нам не всегда нужно ожидать все отправленные значения. Иногда это может быть нюансами дизайна, когда приемнику нужны только часть значений. Чаще всего, такое происходит, если есть ошибки на более раннем этапе. В таком случае, нам не нужно ждать, пока все значения будут получены и можно прекратить их обработку на более раннем шаге.</p>

<p>В нашем примере конвейера, если на каком-то шаге не получается получить значения, то все попытки отправить новые значения будут заблокированы:</p>

<pre><code class="go">    // Получаем первое значение из выходного канала.
    out := merge(c1, c2)
    fmt.Println(&lt;-out) // 4 or 9
    return
    // Так как мы не получаем второе значение и `out`,
    // то рутина зависает при попытке отправки чего либо.
}
</code></pre>

<p>Налицо утечка ресурсов. Go-рутина потребляет память и ресурсы. Все это хранится на стеке самой рутины и не будет подчищено сборщиком мусора, пока go-рутина не завершится.</p>

<p>Нам нужно так организовать передачу по конвейеру, чтобы обеспечить выход из функции, даже когда на более нижних уровнях значения не забираются. Один из способов реализации - это создание буфера. Буфер содержит фиксированное количество значений. Если в буфере есть место, то операция отправки завершается не ожидая получения:</p>

<pre><code class="go">c := make(chan int, 2) // размер буфера 2
c &lt;- 1  // успешная передача
c &lt;- 2  // успешная передача
c &lt;- 3  // блокируется пока другая go-рутина не прочитает из канала &lt;-c
</code></pre>

<p>Когда количество отправляемых данных известно, использование буфера может упростить код. Например, можем переписать функцию <code>gen</code> так, чтобы список значений передавался в буферизированный канал. В таком случае, нам не нужно создавать новую go-рутину:</p>

<pre><code class="go">func gen(nums ...int) &lt;-chan int {
    out := make(chan int, len(nums))
    for _, n := range nums {
        out &lt;- n
    }
    close(out)
    return out
}
</code></pre>

<p>Вернемся к нашим заблокированным go-рутинам в нашем конвейере. Можем использовать буфер для нашего канала, полученного после мультиплексирования:</p>

<pre><code class="go">func merge(cs ...&lt;-chan int) &lt;-chan int {
    var wg sync.WaitGroup
    out := make(chan int, 1) // добавляем место
    // ... остальное без изменений ...
</code></pre>

<p>Хотя это и решает проблему с заблокированными go-рутинами в конкретном примере, в целом это плохой код. Выбор размера буфера сильно зависит от количества значений в каналах которые мы получаем в <code>merge</code> и количества значений которые будут получены из результирующего канала. Это довольно костыльная схема. Если на вход передать больше значений или прочитать меньше значений, то go-рутины снова будут заблокированы.</p>

<p>Вместо этого, нам нужен механизм, который будет сигнализировать на верхний уровень, что мы прекратили получать значения на текущем уровне.</p>

<h3>Явная отмена</h3>

<p>Когда в <code>main</code> решено прекратить получать значения из канала <code>out</code>, то нужно как то сообщить go-рутине на верхнем уровне что нужно перестать посылать сообщения. Это возможно с помощью отправки сообщений в специальный канал <code>done</code>. В нашем случае отправляется два значения, так как есть вероятность двух блокировок:</p>

<pre><code class="go">func main() {
    in := gen(2, 3)

    // Распределяем работу `sq` между двумя go-рутинами 
    // которые считывают данные из `in`.
    c1 := sq(in)
    c2 := sq(in)

    // Получаем первое значение из выходного канала.
    done := make(chan struct{}, 2)
    out := merge(done, c1, c2)
    fmt.Println(&lt;-out) // 4 or 9

    // Сообщаем отправителям, что мы закончили.
    done &lt;- struct{}{}
    done &lt;- struct{}{}
}
</code></pre>

<p>Отправка в go-рутине в <code>merge</code> немного изменена, туда добавилась конструкция <code>select</code>. Такая структура будет работать, если можно отправить данные в канал <code>out</code> или можно получить данные из <code>done</code>. Тип, который используются для отправки/получения в канале <code>done</code>, пустая структура. Это самый "легкий" тип, который работает как индикатор, что отправка должна прерваться. В <code>output</code> go-рутине продолжает выполняться цикл по входящим каналам <code>c</code> и верхние этапы не блокируются. Позже мы рассмотрим как совсем остановить этот цикл.</p>

<pre><code class="go">func merge(done &lt;-chan struct{}, cs ...&lt;-chan int) &lt;-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Готовим go-рутину для каждого входного канала из `cs`. В этой
    // рутине копируются значения из `c` пока он не закроется. Или
    // принимаются значение из `done`. Затем вызывается `wg.Done()`
    output := func(c &lt;-chan int) {
        for n := range c {
            select {
            case out &lt;- n:
            case &lt;-done:
            }
        }
        wg.Done()
    }
    // ... остальное не меняется ...
</code></pre>

<p>Такой подход тоже имеет некоторые недостатки. Каждый приемник, который находится на нижнем уровне, должен знать, сколько потенциально заблокированных отправителей находятся на верхнем уровне и передавать сигнал завершения для конкретных отправителей. Однако, подсчет количества отправителей это утомительная задача. К тому же, такой подсчет подвержен ошибкам.</p>

<p>Нам нужен способ сообщить неизвестному числу go-рутин о прекращении передачи значений на более низкий уровень. В Go это можно реализовать путем закрытия канала. <a href="http://golang.org/ref/spec#Receive_operator">Операция получения значения на закрытом канале выполняется немедленно и всегда возвращает нулевое значение</a>.</p>

<p>Это означает, что <code>main</code> может разблокировать всех отправителей просто закрыв канал <code>done</code>. Это напоминает отправку широковещательного сообщения. Мы <em>расширим</em> наши конвейерные функции так, чтобы они принимали канал <code>done</code> как параметр и организуем закрытие этого канала с помощью <code>defer</code> выражение, которое сработает как только завершится <code>main</code>. Закрытие этого канала будет сигналом для остановки конвейера.</p>

<pre><code class="go">func main() {
    // Подготавливаем канал `done`, который будет общим для всего
    // конвейера и закрытие этого канала с помощью `defer` будет 
    // сигналом завершения для всех go-рутин.
    done := make(chan struct{})
    defer close(done)

    in := gen(done, 2, 3)

    // Распределяем `sq` между двумя go-рутинами, 
    // которые считывают данные из `in`.
    c1 := sq(done, in)
    c2 := sq(done, in)

    // Забираем первое значение из `output`.
    out := merge(done, c1, c2)
    fmt.Println(&lt;-out) // 4 или 9

    // Будет вызвано отложенное закрытие канала.
}
</code></pre>

<p>Теперь каждый шаг нашего конвейера может быть завершенным, как только <code>done</code> буде закрыт. Go-рутина <code>output</code> в функции <code>merge</code> может завершится без полной выборки данных из входного канала так как, она "знает", что отправитель на более верхнем уровне(<code>sq</code>) прекратил посылать данные как только <code>done</code> закрылся. <code>output</code> обеспечивает вызов <code>wg.Done</code> с помощью <code>defer</code>.</p>

<pre><code class="go">func merge(done &lt;-chan struct{}, cs ...&lt;-chan int) &lt;-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Готовим go-рутину для каждого входного канала из `cs`. В этой
    // рутине копируются значения из `c` пока он или `done` не закроются.
    // Затем вызывается `wg.Done()`
    output := func(c &lt;-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out &lt;- n:
            case &lt;-done:
                return
            }
        }
    }
    // ... остальной код без изменений ...
</code></pre>

<p>Кроме того, даже функция <code>sq</code> может выйти, как только <code>done</code> будет закрыт. <code>sq</code> реализует закрытие  канала <code>out</code> с помощью все того же <code>defer</code>:</p>

<pre><code class="go">func sq(done &lt;-chan struct{}, in &lt;-chan int) &lt;-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out &lt;- n * n:
            case &lt;-done:
                return
            }
        }
    }()
    return out
}
</code></pre>

<p>Сформулируем основные принципы построения конвейеров:</p>

<ul>
<li>Исходящие каналы закрываются на своем этапе когда вся отправка завершена.</li>
<li>На всех этапах происходит получение из каналов пока эти каналы не закрыты и отправители не заблокированы.</li>
</ul>

<p>В конвейер поддерживается неблокируемость или через использование буферизированного канала, или с помощью отправки сообщения, что нам больше не нужно получать сообщения.</p>

<h3>Хеширование файлов</h3>

<p>Давайте рассмотрим более реалистичный пример конвейера.</p>

<p>MD5 это алгоритм для создания "отпечатков" или <a href="https://ru.wikipedia.org/wiki/%D0%94%D0%B0%D0%B9%D0%B4%D0%B6%D0%B5%D1%81%D1%82_%D1%81%D0%BE%D0%BE%D0%B1%D1%89%D0%B5%D0%BD%D0%B8%D1%8F">дайджестов сообщений(message digest)</a>. Этот алгоритм можно использовать  для создания проверочной суммы файлов. Консольная программа <code>md5sum</code> выводит хешированные значения для списка файлов.</p>

<pre><code class="sh">% md5sum *.go
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
</code></pre>

<p>Наша программа будет работать аналогично <code>md5sum</code>, только в качестве аргумента будет принимать директорию и выводить хеши для всех файлов в директории с сортировкой по имени.</p>

<pre><code class="bash">% go run serial.go .
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
</code></pre>

<p>Функция <code>main</code> в нашем приложении использует вспомогательную функцию <code>MD5All</code>, которая возвращает map в котором имена файла это ключи, а хеши это значения. Затем этот map сортируется и в консоль выводится результат:</p>

<pre><code class="go">func main() {
    // Считаем MD5 хеш для всех файлов в указанной директории,
    // затем отображение отсортированных результатов.
    m, err := MD5All(os.Args[1])
    if err != nil {
        fmt.Println(err)
        return
    }
    var paths []string
    for path := range m {
        paths = append(paths, path)
    }
    sort.Strings(paths)
    for _, path := range paths {
        fmt.Printf("%x  %s\n", m[path], path)
    }
}
</code></pre>

<p>Функция <code>MD5All</code> этот самое интересное в нашем примере. В файле <a href="http://blog.golang.org/pipelines/serial.go">serial.go</a> эта функция реализована без использования конкурентности и просто получает хеш для каждого файла из дерева.</p>

<pre><code class="go"><br />// MD5All читает все файлы в дереве с помощью `filepath.Walk` начиная с `root`
// и возвращает `map` в котором ключи это путь к файлу, а значения - 
// хеш содержимого. Если при обходе директории или чтении файла 
// возникает ошибка, то она возвращается из функции.
func MD5All(root string) (map[string][md5.Size]byte, error) {
    m := make(map[string][md5.Size]byte)
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if !info.Mode().IsRegular() {
            return nil
        }
        data, err := ioutil.ReadFile(path)
        if err != nil {
            return err
        }
        m[path] = md5.Sum(data)
        return nil
    })
    if err != nil {
        return nil, err
    }
    return m, nil
}
</code></pre>

<h3>Параллельный обход</h3>

<p>В <a href="http://blog.golang.org/pipelines/parallel.go">parallel.go</a>, переделанная функция <code>MD5All</code> которая работает как двухуровневый конвейер. Первый этап это функция <code>sumFiles</code> в которой реализован обход по дереву, хеширование каждого файла в отдельной go-рутине и отправка результата в канал в виде значения типа <code>result</code>:</p>

<pre><code class="go">type result struct {
    path string
    sum  [md5.Size]byte
    err  error
}
</code></pre>

<p><code>sumFiles</code> возвращает два канала. Первый это канал для результатов <code>results</code>. И второй - для ошибок работы <code>filepath.Walk</code>. Функция обхода запускает новую функцию для обработки файла, затем проверяет <code>done</code>. Если канал <code>done</code> закрыт функция обхода завершается немедленно:</p>

<pre><code class="go">func sumFiles(done &lt;-chan struct{}, root string) (&lt;-chan result, &lt;-chan error) {
    // Для каждого файла запускается новая go-рутина, которая подсчитывает
    // хеш и отправляет результат в `c`. Отправляем результат 
    // `filepath.Walk` в `errc`.
    c := make(chan result)
    errc := make(chan error, 1)
    go func() {
        var wg sync.WaitGroup
        err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            wg.Add(1)
            go func() {
                data, err := ioutil.ReadFile(path)
                select {
                case c &lt;- result{path, md5.Sum(data), err}:
                case &lt;-done:
                }
                wg.Done()
            }()
            // Завершаем обход, если канал `done` закрыт.
            select {
            case &lt;-done:
                return errors.New("walk canceled")
            default:
                return nil
            }
        })
        // Обход закончен. Это значит все вызовы `wg.Add` завершены. 
        // Запускаем go-рутину для закрытия канала `c` как только 
        // все результаты отправлены.
        go func() {
            wg.Wait()
            close(c)
        }()
        // `errc` буферизированный канал.
        errc &lt;- err
    }()
    return c, errc
}
</code></pre>

<p><code>MD5All</code> получает хеши из <code>c</code>. И <code>MD5All</code> завершается немедленно в случае получения ошибки и закрывает канал <code>done</code> с помощью <code>defer</code>:</p>

<pre><code class="go">func MD5All(root string) (map[string][md5.Size]byte, error) {
    // `MD5All` закрывает канал `done` когда завершается. 
    // Это может произойти до получения 
    // всех значений из `c` и `errc`.
    done := make(chan struct{})
    defer close(done)

    c, errc := sumFiles(done, root)

    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }
    if err := &lt;-errc; err != nil {
        return nil, err
    }
    return m, nil
}
</code></pre>

<h3>Ограниченный параллелизм</h3>

<p>Функция <code>MD5All</code> в из файла <a href="http://blog.golang.org/pipelines/parallel.go">parallel.go</a> запускает новую go-рутину для хеширования каждого файла. В папке с большим количеством файлов это может вызвать проблемы с потреблением памяти.</p>

<p>Мы можем сократить выделение памяти с помощью ограничения количества параллельно обрабатываемых файлов. В <a href="http://blog.golang.org/pipelines/bounded.go">bounded.go</a> мы реализуем этот подход, создавая фиксированное количество go-рутин для чтения файлов. Наш конвейер теперь будет трехэтапный: обходим дерево, считаем хеши файлов, и собираем эти хеши.</p>

<p>Первый этап - функция <code>walkFiles</code> которая собирает пути файлов в дереве:</p>

<pre><code class="go">func walkFiles(done &lt;-chan struct{}, root string) (&lt;-chan string, &lt;-chan error) {
    paths := make(chan string)
    errc := make(chan error, 1)
    go func() {
        // Закрываем канал для путей после обхода дерева.
        defer close(paths)
        // Канал `errc` буферизированный.
        errc &lt;- filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            select {
            case paths &lt;- path:
            case &lt;-done:
                return errors.New("walk canceled")
            }
            return nil
        })
    }()
    return paths, errc
}
</code></pre>

<p>На втором шаге стартует функция <code>digester</code> запускает фиксированное число хеширующих go-рутин, которые получают имена файлов из <code>paths</code> и отправляют результаты <code>results</code> в канал <code>c</code>:</p>

<pre><code class="go">func digester(done &lt;-chan struct{}, paths &lt;-chan string, c chan&lt;- result) {
    for path := range paths {
        data, err := ioutil.ReadFile(path)
        select {
        case c &lt;- result{path, md5.Sum(data), err}:
        case &lt;-done:
            return
        }
    }
}
</code></pre>

<p>В отличии от предыдущего примера, функция <code>digester</code> не закрывает свой выходной канал после того как go-рутины отправят всю информацию. Вместо этого, функция <code>MD5All</code> обеспечивает закрытие всех каналов после завершения всех запущенных <code>digester</code>:</p>

<pre><code class="go">    // Запускаем фиксированное количество go-рутин 
    // для чтения и хеширования файлов
    c := make(chan result)
    var wg sync.WaitGroup
    const numDigesters = 20
    wg.Add(numDigesters)
    for i := 0; i &lt; numDigesters; i++ {
        go func() {
            digester(done, paths, c)
            wg.Done()
        }()
    }
    go func() {
        wg.Wait()
        close(c)
    }()
</code></pre>

<p>Мы могли бы делать отдельный канал для каждого вызова <code>digester</code>. Но тогда нам понадобилась бы еще одна go-рутина для <em>fan-in</em>.</p>

<p>На последнем этапе мы собираем все <code>results</code> из канала <code>c</code>, затем проверяем наличие ошибок в канале <code>errc</code>. Мы не можем проверить канал <code>errc</code> раньше, до этого места, потому что <code>walkFiles</code> просто заблокирует отправку сообщений:</p>

<pre><code class="go">    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }
    // Check whether the Walk failed.
    if err := &lt;-errc; err != nil {
        return nil, err
    }
    return m, nil
}
</code></pre>

<h3>Заключения</h3>

<p>В этой статье показаны технологии обработки данных с использованием конвейеров в Go. Отмена работы может оказаться нетривиальной задачей, так как на каждом этапе могут возникнуть блокировки и следующие этапы не смогут получить данные. В статье был показан пример как закрытие канала может транслировать сигнал "done" для всех go-рутин запущенных в потоке.</p>

<p>Что почитать:</p>

<ul>
<li><a href="http://talks.golang.org/2012/concurrency.slide#1">Go Concurrency Patterns</a> (<a href="https://www.youtube.com/watch?v=f6kdp27TYZs">видео</a>) презентация базовых примитивов конкурентного программирования на Go и несколько способов их применения.</li>
<li>Роб Пайк. <a href="http://blog.golang.org/advanced-go-concurrency-patterns">Advanced Go Concurrency Patterns</a> (<a href="http://www.youtube.com/watch?v=QDDwwePbDtw">видео</a>) рассматриваются более комплексные примитивы.</li>
<li>Статья <a href="http://swtch.com/~rsc/thread/squint.pdf">Squinting at Power Series</a> от Douglas McIlroy's  в которой показано, как элегпнтно можно использовать Go конкурентность для комплексных вычислений.</li>
</ul>