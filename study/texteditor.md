# Qt Multiple-Tabs Text Editor

A step-by-step guide to building a multi-tab text editor application using the Qt framework (C++).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [Create the Main Window](#create-the-main-window)
4. [Add a Tab Widget](#add-a-tab-widget)
5. [Implement New Tab Functionality](#implement-new-tab-functionality)
6. [Implement Open File](#implement-open-file)
7. [Implement Save / Save As](#implement-save--save-as)
8. [Implement Close Tab](#implement-close-tab)
9. [Add a Menu Bar](#add-a-menu-bar)
10. [Add Keyboard Shortcuts](#add-keyboard-shortcuts)
11. [Handle Unsaved Changes](#handle-unsaved-changes)
12. [Optional Enhancements](#optional-enhancements)
13. [Build and Run](#build-and-run)

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Qt version  | Qt 5.x or Qt 6.x |
| Compiler    | GCC / Clang / MSVC (C++17 recommended) |
| Build tool  | CMake ≥ 3.16 **or** qmake |
| IDE         | Qt Creator (recommended) or any C++ IDE |

Install Qt from the [official Qt installer](https://www.qt.io/download) or your system package manager:

```bash
# Ubuntu / Debian
sudo apt install qt6-base-dev qt6-tools-dev cmake

# macOS (Homebrew)
brew install qt
```

---

## Project Setup

### Using CMake

Create the following project structure:

```
TabTextEditor/
├── CMakeLists.txt
├── main.cpp
├── mainwindow.h
├── mainwindow.cpp
└── codeeditor.h          # optional – syntax-aware editor widget
```

**`CMakeLists.txt`**

```cmake
cmake_minimum_required(VERSION 3.16)
project(TabTextEditor LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

find_package(Qt6 REQUIRED COMPONENTS Widgets)

add_executable(TabTextEditor
    main.cpp
    mainwindow.cpp
    mainwindow.h
)

target_link_libraries(TabTextEditor PRIVATE Qt6::Widgets)
```

### Using qmake

```pro
QT += widgets
CONFIG += c++17
TEMPLATE = app
TARGET = TabTextEditor

SOURCES += main.cpp mainwindow.cpp
HEADERS += mainwindow.h
```

---

## Create the Main Window

**`main.cpp`**

```cpp
#include <QApplication>
#include "mainwindow.h"

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);
    MainWindow window;
    window.setWindowTitle("Tab Text Editor");
    window.resize(900, 600);
    window.show();
    return app.exec();
}
```

**`mainwindow.h`**

```cpp
#pragma once
#include <QMainWindow>
#include <QTabWidget>
#include <QTextEdit>
#include <QMap>

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = nullptr);

protected:
    void closeEvent(QCloseEvent *event) override;

private slots:
    void newTab();
    void openFile();
    void saveFile();
    void saveFileAs();
    void closeTab(int index);
    void onTextChanged();

private:
    QTabWidget  *m_tabs;
    QMap<QTextEdit *, QString> m_filePaths;   // editor → file path

    QTextEdit  *currentEditor() const;
    QString    &currentFilePath();
    void        setTabModified(int index, bool modified);
    bool        maybeSave(int index);
    void        loadFile(const QString &path);
    bool        saveToFile(const QString &path, QTextEdit *editor);
    void        createMenuBar();
    int         newEditorTab(const QString &title = "Untitled");
};
```

---

## Add a Tab Widget

**`mainwindow.cpp`** – constructor

```cpp
#include "mainwindow.h"
#include <QMenuBar>
#include <QFileDialog>
#include <QMessageBox>
#include <QCloseEvent>
#include <QFileInfo>
#include <QTextStream>
#include <QFile>

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
{
    m_tabs = new QTabWidget(this);
    m_tabs->setTabsClosable(true);          // shows ✕ on every tab
    m_tabs->setMovable(true);               // tabs can be reordered by drag
    m_tabs->setDocumentMode(true);          // cleaner look on macOS

    connect(m_tabs, &QTabWidget::tabCloseRequested,
            this,   &MainWindow::closeTab);

    setCentralWidget(m_tabs);
    createMenuBar();

    newTab();   // open with one blank tab
}
```

---

## Implement New Tab Functionality

```cpp
int MainWindow::newEditorTab(const QString &title)
{
    auto *editor = new QTextEdit(this);
    editor->setAcceptRichText(false);       // plain-text only
    editor->setFont(QFont("Monospace", 11));

    // track modifications per editor
    connect(editor, &QTextEdit::textChanged,
            this,   &MainWindow::onTextChanged);

    m_filePaths[editor] = QString();        // no file yet

    int idx = m_tabs->addTab(editor, title);
    m_tabs->setCurrentIndex(idx);
    return idx;
}

void MainWindow::newTab()
{
    newEditorTab("Untitled");
}
```

---

## Implement Open File

```cpp
void MainWindow::openFile()
{
    const QString path = QFileDialog::getOpenFileName(
        this, "Open File", QString(),
        "Text Files (*.txt *.md *.cpp *.h *.py);;All Files (*)");

    if (path.isEmpty()) return;
    loadFile(path);
}

void MainWindow::loadFile(const QString &path)
{
    QFile file(path);
    if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
        QMessageBox::warning(this, "Open Error",
                             "Cannot open file:\n" + path);
        return;
    }

    QTextStream in(&file);
    const QString content = in.readAll();

    // reuse current tab if it is blank and unmodified
    QTextEdit *editor = currentEditor();
    bool reuseTab = editor && editor->document()->isEmpty()
                    && !editor->document()->isModified();

    if (!reuseTab) {
        newEditorTab();
        editor = currentEditor();
    }

    editor->setPlainText(content);
    editor->document()->setModified(false);

    m_filePaths[editor] = path;
    const int idx = m_tabs->currentIndex();
    m_tabs->setTabText(idx, QFileInfo(path).fileName());
    m_tabs->setTabToolTip(idx, path);
}
```

---

## Implement Save / Save As

```cpp
void MainWindow::saveFile()
{
    QTextEdit *editor = currentEditor();
    if (!editor) return;

    QString &path = m_filePaths[editor];
    if (path.isEmpty()) {
        saveFileAs();
        return;
    }
    saveToFile(path, editor);
}

void MainWindow::saveFileAs()
{
    QTextEdit *editor = currentEditor();
    if (!editor) return;

    const QString path = QFileDialog::getSaveFileName(
        this, "Save File As", QString(),
        "Text Files (*.txt *.md *.cpp *.h *.py);;All Files (*)");

    if (path.isEmpty()) return;

    m_filePaths[editor] = path;
    saveToFile(path, editor);

    const int idx = m_tabs->currentIndex();
    m_tabs->setTabText(idx, QFileInfo(path).fileName());
    m_tabs->setTabToolTip(idx, path);
}

bool MainWindow::saveToFile(const QString &path, QTextEdit *editor)
{
    QFile file(path);
    if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) {
        QMessageBox::warning(this, "Save Error",
                             "Cannot write file:\n" + path);
        return false;
    }

    QTextStream out(&file);
    out << editor->toPlainText();
    editor->document()->setModified(false);

    const int idx = m_tabs->indexOf(editor);
    setTabModified(idx, false);
    return true;
}
```

---

## Implement Close Tab

```cpp
void MainWindow::closeTab(int index)
{
    if (!maybeSave(index)) return;   // user cancelled

    QTextEdit *editor = qobject_cast<QTextEdit *>(m_tabs->widget(index));
    m_filePaths.remove(editor);
    m_tabs->removeTab(index);

    if (m_tabs->count() == 0)
        newTab();                    // always keep at least one tab
}
```

---

## Add a Menu Bar

```cpp
void MainWindow::createMenuBar()
{
    // ── File ──────────────────────────────────────────────
    QMenu *fileMenu = menuBar()->addMenu("&File");

    fileMenu->addAction("&New Tab",   this, &MainWindow::newTab,
                        QKeySequence::New);
    fileMenu->addAction("&Open…",     this, &MainWindow::openFile,
                        QKeySequence::Open);
    fileMenu->addSeparator();
    fileMenu->addAction("&Save",      this, &MainWindow::saveFile,
                        QKeySequence::Save);
    fileMenu->addAction("Save &As…",  this, &MainWindow::saveFileAs,
                        QKeySequence::SaveAs);
    fileMenu->addSeparator();
    fileMenu->addAction("&Close Tab", this,
                        [this]{ closeTab(m_tabs->currentIndex()); },
                        QKeySequence::Close);
    fileMenu->addAction("E&xit",      qApp, &QApplication::quit,
                        QKeySequence::Quit);

    // ── Edit ──────────────────────────────────────────────
    QMenu *editMenu = menuBar()->addMenu("&Edit");
    editMenu->addAction("&Undo",  this,
                        [this]{ if (auto *e = currentEditor()) e->undo(); },
                        QKeySequence::Undo);
    editMenu->addAction("&Redo",  this,
                        [this]{ if (auto *e = currentEditor()) e->redo(); },
                        QKeySequence::Redo);
    editMenu->addSeparator();
    editMenu->addAction("Cu&t",   this,
                        [this]{ if (auto *e = currentEditor()) e->cut(); },
                        QKeySequence::Cut);
    editMenu->addAction("&Copy",  this,
                        [this]{ if (auto *e = currentEditor()) e->copy(); },
                        QKeySequence::Copy);
    editMenu->addAction("&Paste", this,
                        [this]{ if (auto *e = currentEditor()) e->paste(); },
                        QKeySequence::Paste);
    editMenu->addSeparator();
    editMenu->addAction("Select &All", this,
                        [this]{ if (auto *e = currentEditor()) e->selectAll(); },
                        QKeySequence::SelectAll);
}
```

---

## Add Keyboard Shortcuts

The table below lists the shortcuts wired in `createMenuBar()`. They automatically adapt to the host OS via `QKeySequence` standard keys.

| Action      | Windows / Linux | macOS      |
|-------------|-----------------|------------|
| New Tab     | Ctrl+N          | ⌘N         |
| Open        | Ctrl+O          | ⌘O         |
| Save        | Ctrl+S          | ⌘S         |
| Save As     | Ctrl+Shift+S    | ⌘⇧S        |
| Close Tab   | Ctrl+W          | ⌘W         |
| Undo        | Ctrl+Z          | ⌘Z         |
| Redo        | Ctrl+Y / Ctrl+Shift+Z | ⌘⇧Z |
| Cut         | Ctrl+X          | ⌘X         |
| Copy        | Ctrl+C          | ⌘C         |
| Paste       | Ctrl+V          | ⌘V         |
| Select All  | Ctrl+A          | ⌘A         |
| Quit        | Alt+F4          | ⌘Q         |

---

## Handle Unsaved Changes

```cpp
void MainWindow::onTextChanged()
{
    const int idx = m_tabs->currentIndex();
    if (idx < 0) return;
    setTabModified(idx, true);
}

void MainWindow::setTabModified(int index, bool modified)
{
    QString title = m_tabs->tabText(index);
    const bool hasAsterisk = title.endsWith(" *");

    if (modified && !hasAsterisk)
        m_tabs->setTabText(index, title + " *");
    else if (!modified && hasAsterisk)
        m_tabs->setTabText(index, title.chopped(2));
}

bool MainWindow::maybeSave(int index)
{
    QTextEdit *editor = qobject_cast<QTextEdit *>(m_tabs->widget(index));
    if (!editor || !editor->document()->isModified())
        return true;

    const QString name = m_tabs->tabText(index).remove(" *");
    const auto reply = QMessageBox::question(
        this,
        "Unsaved Changes",
        QString("'%1' has unsaved changes.\nDo you want to save before closing?")
            .arg(name),
        QMessageBox::Save | QMessageBox::Discard | QMessageBox::Cancel);

    if (reply == QMessageBox::Save)
        return saveToFile(m_filePaths[editor], editor);
    if (reply == QMessageBox::Cancel)
        return false;
    return true;   // Discard
}

void MainWindow::closeEvent(QCloseEvent *event)
{
    for (int i = 0; i < m_tabs->count(); ++i) {
        if (!maybeSave(i)) {
            event->ignore();
            return;
        }
    }
    event->accept();
}
```

### Helper – current editor

```cpp
QTextEdit *MainWindow::currentEditor() const
{
    return qobject_cast<QTextEdit *>(m_tabs->currentWidget());
}
```

---

## Optional Enhancements

| Feature | Approach |
|---------|----------|
| **Syntax highlighting** | Subclass `QSyntaxHighlighter` and apply it to each `QTextEdit::document()` |
| **Line numbers** | Add a custom `QWidget` side-panel that paints line numbers synchronized with the editor's scroll position |
| **Find & Replace** | Add a `QDialog` or inline `QToolBar` using `QTextDocument::find()` |
| **Font chooser** | Use `QFontDialog` and apply the result with `editor->setFont()` |
| **Word wrap toggle** | Toggle `editor->setLineWrapMode(QTextEdit::NoWrap)` |
| **Recent files** | Store paths in `QSettings` and populate a *Recent Files* submenu |
| **Drag-and-drop open** | Override `dragEnterEvent` / `dropEvent` on the main window and call `loadFile()` |
| **Session restore** | Save open file paths to `QSettings` on close; reopen them on startup |
| **Status bar** | Use `QMainWindow::statusBar()` to show cursor position (line/column) |

---

## Build and Run

### CMake

```bash
cmake -S . -B build
cmake --build build
./build/TabTextEditor
```

### qmake

```bash
qmake TabTextEditor.pro
make
./TabTextEditor
```

---

## Summary

The editor is built from four main building blocks:

```
QMainWindow
 └── QTabWidget  (central widget)
      └── QTextEdit  (one per tab)
           └── QTextDocument  (tracks content + modification state)
```

Each `QTextEdit` is stored as a tab page.  A `QMap<QTextEdit*, QString>` maps every editor to its on-disk path, making it straightforward to support any number of simultaneously open files without global state.

> **Tip:** For a production-quality editor, consider replacing `QTextEdit` with `QPlainTextEdit`, which is optimised for plain text and handles large files more efficiently.
