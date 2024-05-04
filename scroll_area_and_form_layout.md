# Использование QFormLayout и [размещение в QScrollArea](https://stackoverflow.com/questions/49099606/how-can-i-put-a-qformlayout-in-a-scrollarea)

[QFormLayout](https://doc.qt.io/qt-6/qformlayout.html) представляет сетку с двумя столбцами. В первом столбце обычно выводят наименование, а в правом редактируемое поле (текстовое или поле выбора значения или даты).

Появилась необходимость разместить так QFormLayout, чтобы, если он не помещается в родительский виджет полностью, появлялись полосы прокрутки, а когда увеличивался размер родительского виджета - увеличивался по ширине. См. рисунки.

<img src="q_form_layout_img/small.png" alt="drawing" height="200"/>
<img src="q_form_layout_img/middle.png" alt="drawing" height="200"/>
<img src="q_form_layout_img/big.png" alt="drawing" height="200"/>

Для этого необходимо родительскому виджету присвоить какой-либо layout (например, QVBoxLayout) - если это не сделать, то у дочернего виджета установится размер по умолчанию, и не будет изменять размер при изменении размеров родительского виджета.

Далее необходимо создать [QScrollArea](https://doc.qt.io/qt-6/qscrollarea.html), которая:

- размещается в родительском виджете;
- устанавливается значение setWidgetResizable(true), что бы [дочерний виджет занимал всю ширину](https://ru.stackoverflow.com/questions/847717/Как-работает-qscrollarea) QScrollArea
- при необходимости также можно установить setMinimumWidth(200), для удобства использования, что бы горизонтальная полоса прокрутки не появлялась из-за сильного уменьшения размера.

После этого следует создать виджет для формы:

- который размещается в QScrollArea;
- которому назначается layout с.

Далее создаем QFormLayout, заполняем его и назначаем виджету для формы.

Объяснение идет в примой последовательности, в коде же сделаем все с конца:

```cpp
#include <QApplication>
#include <QWidget>
#include <QPushButton>
#include <QFormLayout>
#include <QLineEdit>
#include <QVBoxLayout>
#include <QLabel>
#include <QComboBox>
#include <QScrollArea>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    // Создаем родительский виджет
    QWidget w;

    // Создаем layout для формы и заполняем его
    QFormLayout *formLayout = new QFormLayout();
    for (int i = 0;i < 10; i++) {
        formLayout->addRow("&Name:", new QLineEdit());
    }

    // Создаем виджет которому устанавливаем layout формы
    QWidget* formWidget = new QWidget();
    formWidget->setLayout(formLayout);

    //Создаем область с полосами прокрутки, добавляем возможность изменения размеров дочерних элементов, устанавливаем минимальную ширину и добавляем дочерним элементов виджет с формой
    QScrollArea* scrollArea = new QScrollArea();
    scrollArea->setWidgetResizable(true);
    scrollArea->setMinimumWidth(200);
    scrollArea->setWidget(formWidget);

    // Создаем layout для родительского виджета для автоматического регулирования размеров дочерних виджетов
    QVBoxLayout* layout = new QVBoxLayout();
    w.setLayout(layout);
    w.layout()->addWidget(scrollArea);

    w.show();
    return a.exec();
}
```
