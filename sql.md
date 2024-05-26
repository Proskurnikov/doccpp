# Работа с базами данных

## Как подключиться

Источники, которые читал, преимущественно говорят о трех уровнях, но это не добавляет понимания (по крайней мере для меня) как со всем этим работать (тем более что первый уровень для обращения к базе данных не понадобится).

Основной класс для подключения это [QSqlDatabase](https://doc.qt.io/qt-6/qsqldatabase.html). Вся информация по использованию этого класса есть в [документации](https://doc.qt.io/qt-6/qsqldatabase.html).

Что бы начать его использовать надо знать к какой базе данных подключаемся и как (имя, пароль и т.п. при необходимости).

Например, для подключения к файлу Access, надо знать тип [драйвера](https://doc.qt.io/qt-6/sql-driver.html#supported-databases) который следует указать в `SqlDatabase db = QSqlDatabase::addDatabase("QODBC");` и название базы данных (и это не просто путь к файлу)

```attention
Внимание:

Компилятор С++ должен быть той-же битности (32 бита или 64 бита), что и установленный Access, иначе драйвер не будет работать. Также, это означает что должен быть установлен Access. Также это означает, что на Linux этот драйвер работать не будет.
```

```cpp
QSqlDatabase db = QSqlDatabase::addDatabase("QODBC");
db.setDatabaseName("Driver={Microsoft Access Driver (*.mdb, *.accdb)};DSN='';DBQ=C:\\path\\to\\myDB.mdb");
```

После настройки базу данных следует открыть

```cpp
bool ok = db.open();
```

Здесь в случае, если открыть не удалось, можно запросить актуальное положение базы данных или предусмотреть создание пустой базы данных.

## База данных по умолчанию, подключение к нескольким базам данных

В случае, если создается одно подключение к базе данных, далее можно сформировать запрос к базе данных без указания, какая это база, т.к. она становится базой данных по умолчанию. В случае если используется несколько баз данных, следует задать имя базы данны и при формирования запроса указывать это имя.

TODO: Нужен пример

## База данных как член класса

В примерах приводят такие варианты, где используют базу данных как член класса, но документация рекомендует этого не делать, а получать, при необходимости, вызовом метода [database()](https://doc.qt.io/qt-6/qsqldatabase.html#database), в который можно передать имя базы данных (в случае если оно задано и используется несколько баз данных с именем).

## Проверки при запуске базы

Нашел несколько проверок, попытаемся разобраться для чего они.

В проекте [Books](https://doc.qt.io/qt-5/qtsql-books-example.html) в [bookwindow.cpp](https://code.qt.io/cgit/qt/qtbase.git/tree/examples/sql/books/bookwindow.cpp?h=5.15) первым делом идет проверка наличия драйвера. Мне нужна аналогичная, т.к., например, драйвер Access работает только той же битности, что и компилятор (64 бита не подойдет, если компилируем на 32 бита)

```cpp
    if (!QSqlDatabase::drivers().contains("QSQLITE"))
        QMessageBox::critical(
                    this,
                    "Unable to load database",
                    "This demo needs the SQLITE driver"
                    );
```

Далее происходит инициализация БД. Это сделано в отдельной функции в файле [initdb.h](https://code.qt.io/cgit/qt/qtbase.git/tree/examples/sql/books/initdb.h?h=5.15).

TODO: Надо разобраться что делает каждая проверка

```cpp
QSqlError initDb()
{
    QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");
    db.setDatabaseName(":memory:");

    if (!db.open())
        return db.lastError();

    QStringList tables = db.tables();
    if (tables.contains("books", Qt::CaseInsensitive)
        && tables.contains("authors", Qt::CaseInsensitive))
        return QSqlError();

    QSqlQuery q;
    if (!q.exec(BOOKS_SQL))
        return q.lastError();
    if (!q.exec(AUTHORS_SQL))
        return q.lastError();
    if (!q.exec(GENRES_SQL))
        return q.lastError();

    if (!q.prepare(INSERT_AUTHOR_SQL))
        return q.lastError();
    QVariant asimovId = addAuthor(q, QLatin1String("Isaac Asimov"), QDate(1920, 2, 1));
    QVariant greeneId = addAuthor(q, QLatin1String("Graham Greene"), QDate(1904, 10, 2));
    QVariant pratchettId = addAuthor(q, QLatin1String("Terry Pratchett"), QDate(1948, 4, 28));

    if (!q.prepare(INSERT_GENRE_SQL))
        return q.lastError();
    QVariant sfiction = addGenre(q, QLatin1String("Science Fiction"));
    QVariant fiction = addGenre(q, QLatin1String("Fiction"));
    QVariant fantasy = addGenre(q, QLatin1String("Fantasy"));

    if (!q.prepare(INSERT_BOOK_SQL))
        return q.lastError();
    addBook(q, QLatin1String("Foundation"), 1951, asimovId, sfiction, 3);
    addBook(q, QLatin1String("Foundation and Empire"), 1952, asimovId, sfiction, 4);
    addBook(q, QLatin1String("Second Foundation"), 1953, asimovId, sfiction, 3);
    addBook(q, QLatin1String("Foundation's Edge"), 1982, asimovId, sfiction, 3);
    addBook(q, QLatin1String("Foundation and Earth"), 1986, asimovId, sfiction, 4);
    addBook(q, QLatin1String("Prelude to Foundation"), 1988, asimovId, sfiction, 3);
    addBook(q, QLatin1String("Forward the Foundation"), 1993, asimovId, sfiction, 3);
    addBook(q, QLatin1String("The Power and the Glory"), 1940, greeneId, fiction, 4);
    addBook(q, QLatin1String("The Third Man"), 1950, greeneId, fiction, 5);
    addBook(q, QLatin1String("Our Man in Havana"), 1958, greeneId, fiction, 4);
    addBook(q, QLatin1String("Guards! Guards!"), 1989, pratchettId, fantasy, 3);
    addBook(q, QLatin1String("Night Watch"), 2002, pratchettId, fantasy, 3);
    addBook(q, QLatin1String("Going Postal"), 2004, pratchettId, fantasy, 3);

    return QSqlError();
}
```

Также стоит обратить внимание на проверку, перед тем как показывать ошибку пользователю

```cpp
    // Initialize the database:
    QSqlError err = initDb();
    if (err.type() != QSqlError::NoError) {
        showError(err);
        return;
    }
```

Еще нашел в примере для QML [приложение Chat](https://doc.qt.io/qt-6/qtquickcontrols-chattutorial-example.html)

```cpp
static void connectToDatabase()
{
    QSqlDatabase database = QSqlDatabase::database();
    if (!database.isValid()) {
        database = QSqlDatabase::addDatabase("QSQLITE");
        if (!database.isValid())
            qFatal("Cannot add database: %s", qPrintable(database.lastError().text()));
    }

    const QDir writeDir = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
    if (!writeDir.mkpath("."))
        qFatal("Failed to create writable directory at %s", qPrintable(writeDir.absolutePath()));

    // Ensure that we have a writable location on all devices.
    const QString fileName = writeDir.absolutePath() + "/chat-database.sqlite3";
    // When using the SQLite driver, open() will create the SQLite database if it doesn't exist.
    database.setDatabaseName(fileName);
    if (!database.open()) {
        qFatal("Cannot open database: %s", qPrintable(database.lastError().text()));
        QFile::remove(fileName);
    }
}
```

Также стоит разобрать создание таблицы

```cpp
#include "sqlcontactmodel.h"

#include <QDebug>
#include <QSqlError>
#include <QSqlQuery>

static void createTable()
{
    if (QSqlDatabase::database().tables().contains(QStringLiteral("Contacts"))) {
        // The table already exists; we don't need to do anything.
        return;
    }

    QSqlQuery query;
    if (!query.exec(
        "CREATE TABLE IF NOT EXISTS 'Contacts' ("
        "   'name' TEXT NOT NULL,"
        "   PRIMARY KEY(name)"
        ")")) {
        qFatal("Failed to query database: %s", qPrintable(query.lastError().text()));
    }

    query.exec("INSERT INTO Contacts VALUES('Albert Einstein')");
    query.exec("INSERT INTO Contacts VALUES('Ernest Hemingway')");
    query.exec("INSERT INTO Contacts VALUES('Hans Gude')");
}
```
