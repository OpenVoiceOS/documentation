##### Adding Translations To QML GUI

1. All qml files will need to do text messages with the qsTr method call like so:

```
Label {
    text: qsTr("Text to be translated")
}
```

2. Use lupdate to generate TS file for target QML in the UI Folder

``` lupdate test.qml -ts test-french.ts ```

3. use Qt Linguist GUI Tool to add translations to the test-french.ts file
