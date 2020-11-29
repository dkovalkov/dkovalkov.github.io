#Реактивная разработка через тестирование (Swift, RxSwift)

В данной статье приведу небольшой флоу разработки с использованием тестов. Применительно к Swift, RxSwift. Наличие тестов позволит вам еще больше увериться в работоспособности своего кода. Написание реактивных тестов может быть на первый взгляд замысловатым. Но нам нужно привести их к виду обычных юнит-тестов.

Для примера выбрано простое приложение с архитектурой MVVM. Тестироваться будет вью-модель. В ней обычно концентрируется наибольшая часть логики экрана. Приложение будет имитировать пин-доску для важных заметок и картинок.

Начнем с подготовки теста.

```
var viewModel: PinBoardViewModel!
var testScheduler: TestScheduler!
var disposeBag: DisposeBag!
    
override func setUpWithError() throws {
    testScheduler = TestScheduler(initialClock: 0)
    disposeBag = DisposeBag()
    viewModel = PinBoardViewModel()
}
```

Перед каждым тестом выполнится несколько шагов: создастся новая вью-модель, инициализируется планировщик (скедулер), сбросятся все предыдущие подписки.

```
func testAddNote() {
    editText()
    addNote()
    let dismissObserver = observeDismissing()

    testScheduler.start()
        
    XCTAssertEqual(dismissObserver.events.count, 1)
}
```

Первый тест. Изменяется текст заметки, инициируется ее добавление. Для сохранения ясности и краткости теста низкоуровневые операции нужно спрятать в функциях. Создается мини DSL для данного класса с тестами или для нескольких тест-кейсов. Вот упомянутые функции.

```
private func editText() {
    let editText = testScheduler.createHotObservable([.next(100, "Some note details.")])
    editText
        .bind(to: viewModel.noteText)
        .disposed(by: disposeBag)
}
    
private func addNote() {
    let addNote = testScheduler.createHotObservable([.next(200, ())])
    addNote
        .bind(to: viewModel.addNote)
        .disposed(by: disposeBag)
}
    
private func observeDismissing() -> TestableObserver<Void> {
    let dismissObserver = testScheduler.createObserver(Void.self)
    viewModel.dismiss
        .emit(to: dismissObserver)
        .disposed(by: disposeBag)
        
    return dismissObserver
}
```

Довольно просто понять, что делает каждая функция. Но если бы они были встроены в тело теста, то разобраться было бы сложнее.

Тест есть, и он не проходит. Реализация довольно простая. Во вью-модели добавляется код:

```
private func setUp() {
    addNote
        .withLatestFrom(noteText)
        .map { _ in }
        .bind(to: dismissRelay)
        .disposed(by: disposeBag)
}
```

Добавляем еще один тест.

```
func testAddEmptyNote() {
    addNote()
    let dismissObserver = observeDismissing()

    testScheduler.start()
        
    XCTAssertTrue(dismissObserver.events.isEmpty)
}
```

Это проверка, что пустой текст не добавится. Пара функций у нас переиспользуется. Для того, чтобы прошел тест нужно добавить одну строку во вью-модель.

```
private func setUp() {
    addNote
        .withLatestFrom(noteText)
        .filter { !$0.isEmpty }
        .map { _ in }
        .bind(to: dismissRelay)
        .disposed(by: disposeBag)
}
```

Итеративная разработка тестами происходит подобным образом: добавляется тест и затем реализация для прохождения теста. Небольшими шагами добавляется необходимая функциональность. В проекте можно самостоятельно найти следующий тест выбора картинки и реализацию для его прохождения.

##Особенности реактивного тестирования.

В реактивных тестах по сравнению с обычными будет больше кода. Больше механического шаблонного кода. Установить входные биндинги, выходные. Запустить планировщик. Больше кода нужно для проверки выходных событий. Многое из этого можно спрятать за короткими функциями. Принципы TDD будут применимы и для реактивных тестов. Минимально короткие шаги для прохождения теста, рефакторинг, переход к следующему тесту.



