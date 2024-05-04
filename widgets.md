# Widgets

## Widgets vs QML (Описание недостатков и преимуществ QWidget и qml)

~~QWidget есть виджет выбора даты ([QCalendarWidget](https://doc.qt.io/qt-6/qcalendarwidget.html#details)) , в QML нет~~. Также есть поля [QDateEdit](https://doc.qt.io/qt-6/qdateedit.html#details), [QTimeEdit](https://doc.qt.io/qt-6/qtimeedit.html) и [QDateTimeEdit](https://doc.qt.io/qt-6/qdatetimeedit.html#details) которые можно использовать совместно с ранее указанным QCalendarWidget, через включение setCalendarWidget. При этом ранее, в QtQuick.Controls 1.4 был [Calendar QML Type](https://doc.qt.io/qt-5/qml-qtquick-controls-calendar.html)

В QML табличное? (Или древовидное) представление появилось недавно.

Через QWidget нет виджета для фотографирования в андроид (для Windows вроде есть).
