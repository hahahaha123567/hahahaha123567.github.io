---
title: JavaFX中的table相关
date: 2018-03-07 13:43:56
tags: 
- Java 
- JavaFX
description: 官方文档只介绍了固定属性的情况
---

# 概览

JavaFX中实现一个table需要实现两个类：`TableView` , `TableColumn` 

此外，还要将放入表格中的数据（每行是一个对象）用一个自定义的类表示。在JavaFX中，这个自定义类可以使用property属性绑定

为了将TableColumn的对象和数据对象进行关联，需要使用 [Cell Value Factory](https://docs.oracle.com/javafx/2/api/javafx/scene/control/TableColumn.html#cellValueFactoryProperty%28%29) 和 [Cell Factory](https://docs.oracle.com/javafx/2/api/javafx/scene/control/TableColumn.html#cellFactoryProperty%28%29) 

>setCellFactory的意图是在创建这列的时候要做的事情，你可以改变TableCell的任何内容，包括UI和Value，而setCellValueFactory呢，它的重点是关联属性，从你传递给它的Model中通过对应属性的getter来获取值。在setCellFactory中的TableCell有个回调，叫`updateItem`，它可以获取到你设置到此Cell的值，这个值是跟setCellValueFactory所关联的属性有关。

# 固定属性

```java
public class Data {
  private final SimpleStringProperty p1;
  private final SimpleStringProperty p2;
  Data (String s1, String s2) {
    //...
  }
}
```

```java
TableView table = new TableView();
TableColumn column1 = new TableColumn("c1");
TableColumn column2 = new TableColumn("c2");
table.getColumns().addAll(column1, column2);

column1.setCellValueFactory(new PropertyValueFactory<Data, String>("p1"));
column2.setCellValueFactory(new PropertyValueFactory<Data, String>("p2"));

ObservableList<Person> dataList = FXCollections.observableArrayList(
  new Data("Nico", "Maki"),
  new Data("haha", "hoho")
);
table.setItems(dataList);
```

# 动态属性

在写 [CSV editor](https://github.com/hahahaha123567/csv-editor) 时，显然table的行列数都是不确定的，因此不能按照上述步骤。TableColumn可以根据需要创建，但是绑定column和数据需要进行改动

```java
column.setCellValueFactory(param -> param.getValue().get(ii));
```

这里的参数不能继续使用PropertyValueFactory的对象，而应该用index去获取ObservableList的值

应用可以参考 [这段代码](https://github.com/hahahaha123567/csv-editor/blob/master/src/csv/Controller.java) 的showData()函数

---

参考资料

[JavaFx TableView疑难详解 | cmlanche](http://cmlanche.com/2017/06/08/JavaFx-TableView%E8%AF%A6%E8%A7%A3/)

[Creating columns dynamically | Oracle Community](https://community.oracle.com/thread/2474328)

[tableview - Cell factory in javafx - Stack Overflow](https://stackoverflow.com/questions/10519534/cell-factory-in-javafx)

