# Как отображать и скрывать виджеты, окна, диалоги, меню

## Окна (отображение окон на рабочем столе)

Основной виджет в основной программе, который представляет собой главное окно отображают методом [`void QWidget::show`](https://doc.qt.io/qt-6/qwidget.html#show)

```cpp
#include <QApplication>
#include <QPlainTextEdit>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    QPlainTextEdit log;
    log.show();
    return a.exec();
}
```

```note
Примечание:
1. Окном являются все виджеты, у которых нет родительского виджета
2. Если у виджета нет родительского виджета и не вызван один из методов для отображения на рабочем столе, то он создается, но не отображается
```

```note
Примечание:
Можно также использовать [setVisible(true)](https://doc.qt.io/qt-6/qwidget.html#visible-prop)
Отображение в полноэкранном режиме [showFullScreen()](https://doc.qt.io/qt-6/qwidget.html#showFullScreen)
Отображение во весь экран [showMaximized()](https://doc.qt.io/qt-6/qwidget.html#showMaximized)
Отображение свернуто [void QWidget::showMinimized](https://doc.qt.io/qt-6/qwidget.html#showMinimized)
```

Что-бы скрыть окно можно вызвать [`void QWidget::hide`](https://doc.qt.io/qt-6/qwidget.html#hide) или `setVisible(false)` или `setHidden(true)`

Смотрим [исходники](https://code.qt.io/cgit/qt/qtbase.git/tree/src/widgets/kernel/qwidget.cpp):

```cpp
void QWidget::hide()
{
    setVisible(false);
}
```

Что-бы закрыть окно [`bool QWidget::close()`](https://doc.qt.io/qt-6/qwidget.html#close). Когда закрывается последнее отображаемое окно приложения, приложение завершает свою работу.

Смотрим [исходники](https://code.qt.io/cgit/qt/qtbase.git/tree/src/widgets/kernel/qwidget.cpp):

```cpp
bool QWidget::close()
{
    return d_func()->close();
}

bool QWidgetPrivate::close()
{
    // FIXME: We're not setting is_closing here, even though that would
    // make sense, as the code below will not end up in handleClose to
    // reset is_closing when there's a QWindow, but no QPlatformWindow,
    // and we can't assume close is synchronous so we can't reset it here.

    // Close native widgets via QWindow::close() in order to run QWindow
    // close code. The QWidget-specific close code in handleClose() will
    // in this case be called from the Close event handler in QWidgetWindow.
    if (QWindow *widgetWindow = windowHandle()) {
        if (widgetWindow->isTopLevel())
            return widgetWindow->close();
    }

    return handleClose(QWidgetPrivate::CloseWithEvent);
}
```

## Виджеты (Дочерние виджеты в окне - родительском виджете)

Дочерние виджеты по умолчанию отображаются, если отображается родительский виджет.

Скрыть виджет можно методом [`void QWidget::hide`](https://doc.qt.io/qt-6/qwidget.html#hide) или `setVisible(false)`. Так-же скрыть можно методом [`bool QWidget::close()`](https://doc.qt.io/qt-6/qwidget.html#close), который если вызван у виджета-окна, закрывает его, если вызван без установленного атрибута виджета  Qt::WA_DeleteOnClose, является оберткой над hide(), если вызван с установленным атрибутом виджета  Qt::WA_DeleteOnClose, удаляет объект этого виджета. Также модет дополнительно вызывать событие. См. исходники

```cpp
bool QWidgetPrivate::handleClose(CloseMode mode)
{
    Q_Q(QWidget);
    qCDebug(lcWidgetShowHide) << "Handling close event for" << q;

    if (data.is_closing)
        return true;

    // We might not have initiated the close, so update the state now that we know
    data.is_closing = true;

    QPointer<QWidget> that = q;

    if (data.in_destructor)
        mode = CloseNoEvent;

    if (mode != CloseNoEvent) {
        QCloseEvent e;
        if (mode == CloseWithSpontaneousEvent)
            QApplication::sendSpontaneousEvent(q, &e);
        else
            QCoreApplication::sendEvent(q, &e);
        if (!that.isNull() && !e.isAccepted()) {
            data.is_closing = false;
            return false;
        }
    }

    // even for windows, make sure we deliver a hide event and that all children get hidden
    if (!that.isNull() && !q->isHidden())
        q->hide();

    if (!that.isNull()) {
        data.is_closing = false;
        if (q->testAttribute(Qt::WA_DeleteOnClose)) {
            q->setAttribute(Qt::WA_DeleteOnClose, false);
            q->deleteLater();
        }
    }
    return true;
}
```

После этого, отобразить обратно можно методом  [`void QWidget::show`](https://doc.qt.io/qt-6/qwidget.html#show) или [`setVisible(true)`](https://doc.qt.io/qt-6/qwidget.html#visible-prop) или или `setHidden(false)`.

Сделать виджет активным или неактивным можно методом [`void QWidget::setEnabled(bool)`](https://doc.qt.io/qt-6/qwidget.html#enabled-prop) или `void QWidget::setDisabled(bool disable)`. Если родительский виджет неактивные, дочерние также неактивны. Неактивный виджет не воспринимает события, поступающие от мыши и клавиатуры.

## Диалоги

### Виды диалогов

Диалог это по сути обычное окно-виджет для выполнения какой-либо непродолжительной задачи или краткого взаимодействия с пользователем. По сути это обычный виджет, но со специальными характерными возможностями:

- можно сделать модальный диалог (особенности см. далее);
- предусмотрено закрытие окна - отмена выполнения диалога по клавише Esc.

Не модальный диалог - это отдельное окно-виджет, который существует независимо и, соответственно, **не** блокирует выполнение основной программы.

Модальный диалог - это отдельное окно-виджет, который блокирует доступ к одному окну или ко всему остальному приложению (при этом может блокировать, а может и не блокировать выполнение основной программы. Блокирование в данном случае означает создание своего цикла обработки событий).

### Не модальный диалог

#### Показать не модальный диалог

Для отображения не модального диалога (как говорит [документация](https://doc.qt.io/qt-6/qdialog.html#modeless-dialogs)) следует вызвать метод [`void QWidget::show() [slot]`](https://doc.qt.io/qt-6/qwidget.html#show) или [`setVisible(true)`](https://doc.qt.io/qt-6/qwidget.html#visible-prop) или `setHidden(true)`.

[`void QWidget::show() [slot]`](https://doc.qt.io/qt-6/qwidget.html#show) и [`setVisible(true)`](https://doc.qt.io/qt-6/qwidget.html#visible-prop) сразу возвращает выполнение в основной цикл.

При этом свойство [`modal:bool`](https://doc.qt.io/qt-6/qdialog.html#modal-prop), которое устанавливается методом [`void setModal(bool modal)`](https://doc.qt.io/qt-6/qdialog.html#modal-prop) должно быть false (каким и является по умолчанию  false, т.е. в коде устанавливать не надо). Также являет аналогом значения [`Qt::NonModal`](https://doc.qt.io/qt-6/qt.html#WindowModality-enum) свойства [`QWidget::windowModality`](https://doc.qt.io/qt-6/qwidget.html#windowModality-prop).

#### Скрыть и закрыть

Скрыть диалог можно также, как и обычное окно методами [`void QWidget::hide`](https://doc.qt.io/qt-6/qwidget.html#hide) или `setVisible(false)` или `setHidden(true)`

Активное диалоговое окно закроется если нажать на клавиатуре Esc. Нажатие клавиши Esc вызовет слот [`void QDialog::reject()`](https://doc.qt.io/qt-6/qdialog.html#reject) который закрывает диалог и устанавливает результат диалога `Rejected` (см. далее).

Получать результат показа диалога можно несколькими способами:

- обработка сигналов и событий взаимодействия с дочерними виджетами диалога (ничем не отличается от любого другого взаимодействия с виджетами, аналогичное поведение можно создать и без использования диалога, поэтому здесь не будет рассматриваться);
- обработка сигналов, создаваемых диалогом при закрытии специальными слотами диалога (можно использовать и при модальном диалоге вызванном методом `exec()`, но больше имеет смысла для остальных модальных диалогов без своего цикла обработки событий и для не модального диалога);
- обработка возвращаемого результата диалога (подходит только для модального диалога вызванного методом `exec()` - см. следующий подраздел).

##### Обработка сигналов, создаваемых диалогом при закрытии специальными слотами диалога

Такие слоты и их соответствующие сигналы:

- Слот [`void QDialog::accept()`](https://doc.qt.io/qt-6/qdialog.html#accept) или void QDialog::done(int r) со значением r равным QDialog::Accepted = 1 закрывает виджет (диалог, окно) как QWidget::close() и выпускает сигнал void QDialog::accepted() и void QDialog::finished(int result) со значением result равным QDialog::Accepted = 1
- слот void QDialog::rejected() или void QDialog::done(int r) со значением r равным QDialog::Rejected = 0 закрывает виджет (диалог, окно) как QWidget::close() и выпускает сигнал void QDialog::rejected() и void QDialog::finished(int result) со значением result равным QDialog::Rejected = 0
- как видим в слот void QDialog::done(int r) можно передать и другие значения, которые будут переданы в сигнал void QDialog::finished(int result)



### Модальный диалог

Отобразить модальный диалог можно разными способами, которые будут перечислены далее. В зависимости от выбранного способа, будет получен разный результат. Все указанные способы по сути вызов в той или иной форме с различными дополнительными настройками метода `QWidget::show` (или [`setVisible(true)`](https://doc.qt.io/qt-6/qwidget.html#visible-prop)). поэтому начнем с прямого вызова метода `QWidget::show` с различными настройками.

Возможны следующие комбинации (способы реализации указанных комбинаций см. далее):

- выполнение в основном цикле событий с блокированием доступа к одному (родительскому) окну;
- выполнение в основном цикле событий с блокированием доступа ко всем окнам (т.е. к приложению);
- выполнение в собственном цикле событий (основной цикл приостанавливается) с блокированием доступа ко всем окнам (т.е. к приложению).

#### Выполнение в основном цикле событий с блокированием доступа к одному (родительскому) окну

Можно сделать двумя способами. В обоих случаях QDialog создается с указанием родителя (без указания родителя диалог будет не модальным, т.к. не будет знать по отношению к какому окну он должен быть модальным). Родителем может быть виджет, не обязательно виджет окно. Диалог создается по центру родительского виджета. Блокируется окно, которому принадлежит родительский виджет диалога.

Первый способ:

- установить свойству `QWidget::windowModality`:[Qt::WindowModality](https://doc.qt.io/qt-6/qt.html#WindowModality-enum) методом `void setWindowModality(Qt::WindowModality windowModality)` значение `Qt::WindowModal`
- вызов метода `QWidget::show` или [`setVisible(true)`](https://doc.qt.io/qt-6/qwidget.html#visible-prop)

Второй способ:

- вызов метода void [`QDialog::open() [virtual slot]`](https://doc.qt.io/qt-6/qdialog.html#open)

Если посмотреть исходники, open является аналогом первого способа:

```cpp
void QDialog::open()
{
    Q_D(QDialog);

    Qt::WindowModality modality = windowModality();
    if (modality != Qt::WindowModal) {
        d->resetModalityTo = modality;
        d->wasModalitySet = testAttribute(Qt::WA_SetWindowModality);
        setWindowModality(Qt::WindowModal);
        setAttribute(Qt::WA_SetWindowModality, false);
#ifdef Q_OS_MAC
        setParent(parentWidget(), Qt::Sheet);
#endif
    }

    setResult(0);
    show();
}
```

#### Выполнение в основном цикле событий с блокированием доступа ко всем окнам (т.е. к приложению)

Вызов метода `QWidget::show` после установки свойства [`modal:bool`](https://doc.qt.io/qt-6/qdialog.html#modal-prop) равным `true` методом [`void setModal(bool modal)`](https://doc.qt.io/qt-6/qdialog.html#modal-prop), окно станет модальным, при этом выполнение программы продолжается в основном цикле. Также являет аналогом значения [`Qt::ApplicationModal`](https://doc.qt.io/qt-6/qt.html#WindowModality-enum) свойства [`QWidget::windowModality`](https://doc.qt.io/qt-6/qwidget.html#windowModality-prop).

#### Выполнение в собственном цикле событий (основной цикл приостанавливается) с блокированием доступа ко всем окнам (т.е. к приложению)

Вызов метода [`int QDialog::exec()`](https://doc.qt.io/qt-6/qdialog.html#exec). 

Стоит обратить внимание на наличие возвращаемого значения. Возвращаемое значение зависит от способа закрытия окна-диалога (см. далее).

Указанный метод [блокирует доступ ко всем окнам приложения](#выполнение-в-основном-цикле-событий-с-блокированием-доступа-ко-всем-окнам-т.е.-к-приложению), создает собственный цикл обработки событий и показывает диалог. Думаю, можно аналогично самостоятельно реализовать такое поведение, но зачем, ведь это уже сделано до нас

Смотрим исходники:

```cpp
int QDialog::exec()
{
    Q_D(QDialog);

    if (Q_UNLIKELY(d->eventLoop)) {
        qWarning("QDialog::exec: Recursive call detected");
        return -1;
    }

    bool deleteOnClose = testAttribute(Qt::WA_DeleteOnClose);
    setAttribute(Qt::WA_DeleteOnClose, false);

    d->resetModalitySetByOpen();

    bool wasShowModal = testAttribute(Qt::WA_ShowModal);
    setAttribute(Qt::WA_ShowModal, true);
    setResult(0);

    show();

    QPointer<QDialog> guard = this;
    if (d->nativeDialogInUse) {
        d->platformHelper()->exec();
    } else {
        QEventLoop eventLoop;
        d->eventLoop = &eventLoop;
        (void) eventLoop.exec(QEventLoop::DialogExec);
    }
    if (guard.isNull())
        return QDialog::Rejected;
    d->eventLoop = nullptr;

    setAttribute(Qt::WA_ShowModal, wasShowModal);

    int res = result();
    if (d->nativeDialogInUse)
        d->helperDone(static_cast<QDialog::DialogCode>(res), d->platformHelper());
    if (deleteOnClose)
        delete this;
    return res;
}
```

#### Скрыть закрыть

##### Обработка возвращаемого результата диалога exec()

Указанные слоты (см. открытие и закрытие не модального диалога) void QDialog::accept(), QDialog::done(int r), void QDialog::rejected() не только вызывают соответствующие сигналы и закрывают диалог, но и передают возвращаемое значение в exec().

## Контекстное меню

popup

exec

TODO: Посмотреть как работает просто вызов show(), т.к. в popup используется show(), но много дополнительного кода.
