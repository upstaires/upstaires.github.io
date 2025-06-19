---
title: C++解析Xlsx文件
abbrlink: d87f7e0c  # 建议使用语义化缩写
tags:
  - Qt  # 具体技术标签
  - C++  # 应用场景标签
categories:
  - Qt开发  # 必须指定有效分类
author: upstairs
date: 2024-11-12 11:15:00  # 推荐ISO格式：2024-11-12T11:15:00+08:00
---
# C++解析Xlsx文件

在做一个项目的时候，需要将一些数据导出到xlsx文件中，但是Qt本身并没有提供直接导出xlsx的方法，需要使用第三方库来实现。

## 获取能够解析xlsx的库
在网上寻找能够解析Xlsx的库，发现有QXlsx、QtXlsxWriter，以及一个商业库libXL(还能解析xls格式)，除此之外类似的三方库库在此不做赘述，本文选择QtXlsxWriter库。
库地址：```https://github.com/dbzhang800/QtXlsxWriter```

## 库的使用

这里演示一个第三方库如何加到自己的VS-Qt项目中。

首先我们下载到三方库可能有构建好的二进制文件或类似lib、dll的库文件，也有那种只有源码压缩包需要我们自己去构建的，QtXlsxWriter需要自己编译，但是非常快，类似的这种项目配置基本一致。

当我们解压完下载的压缩包后可以看到有一个显眼的.pro文件，有Qt的可以直接打开->配置项目类型(Release还是Debug)和输出目录(大部分都在项目根目录下的build文件夹内)->配置完成后运行编译即可。过程很简单，没有任何错误，编译也非常迅速。

编译完成后在对应的输出目录下就找到了bin、include、lib三个文件夹，其中bin下的为dll库，include下为头文件，lib下为lib库。
将这三个文件拷贝到自己的项目中，在项目中新建一个三方库文件夹可以是3dparty，或者别的什么，之后就可以把各种三方库放到这里方便管理，这里只放头文件和静态库文件，一般给一个库一个文件夹，里面放include和lib文件夹，将这include和lib文件拷贝到三方库文件夹中，dll文件拷贝到项目程序使用的目录(也就是项目生成exe所在的文件夹)。

在VS项目中找到项目右键点击**属性**
点击C++选项中->常规->附加包含目录，如果没有C++选项那就是你这个项目没有C++文件，随便新建一个cpp文件就可以了。
将include文件夹的路径，写到附加包含库目录中。这里有个技巧，这个vs项目可能不只你在用，所以如果你写绝对路径，在别的电脑上就没办法运行编译了，所以可以使用
```
$(ProjectDir)
```
这个变量的作用就像名字写的翻译过来就是项目的文件夹，然后在后面写剩下的路径，就像这样
```
$(ProjectDir)3dparty\QtXlsxWriter\include
```

点击链接器->常规->附加库目录，将lib文件夹的路径写到这里。
点击链接器->输入->附加依赖项，将lib库的名称写到这里，注意要写到依赖项中，否则编译的时候会报错。

到这一步这个三方库就导入完成了，可以直接使用三方库中的方法(函数)了。

## 代码调用
引入头文件:`#include <QtXlsx>`
简单使用：

创建xlsx:
`QXlsx::Document xlsx;`
导出为excel文件:
`xlsx.saveAs("test.xlsx");`

理论上你可以操作xlsx表格的任何一个单元格，包括但不限于合并单元格、设置单元格的格式、设置单元格的背景颜色等等，具体可以查看QXlsx库的文档。
比如
```C++
	QXlsx::Document xlsx;                                             //新建一个xlsx对象

	// 创建格式对象
	QXlsx::Format cellFormat;
	cellFormat.setHorizontalAlignment(QXlsx::Format::AlignHCenter);   // 设置水平居中
	cellFormat.setVerticalAlignment(QXlsx::Format::AlignVCenter);     // 设置垂直居中
	cellFormat.setFontBold(true);                                     // 设置字体加粗
	cellFormat.setFontSize(12);                                       // 设置字体大小
	cellFormat.setBorderStyle(QXlsx::Format::BorderThin);             // 设置边框
	cellFormat.setFillPattern(QXlsx::Format::PatternSolid);           // 设置填充模式
	cellFormat.setFillForeground(QColor("lightblue"));                // 设置填充颜色
	// 合并单元格
	xlsx.mergeCells("A1:C2");

  // 填充合并单元格的内容（示例，可以根据需要修改）
	xlsx.write("A1", "单元格内容", cellFormat);
	xlsx.write("A2", "单元格内容", cellFormat);
	xlsx.write("B1", "单元格内容", cellFormat);
	xlsx.write("B2", "单元格内容", cellFormat);

	// 设置列宽，设定最小宽度
	int minWidth = 8; // 最小宽度
	for (int col = 1; col <= 12; ++col) 
  {
		xlsx.setColumnWidth(col, std::max(minWidth, 10)); // 设置列宽，确保不小于最小值
	}

  // 保存文件
	if (xlsx.saveAs(filePath)) {
		QMessageBox::information(nullptr, "成功", QString("表格已成功导出到 %1").arg(filePath));
	}
	else {
		QMessageBox::warning(nullptr, "错误", "无法保存Excel文件");
	}
```

加载一个xlsx文件，获取其中的所用工作簿。这个需要一个qtabwidget来显示
```C++
void loadAllSheetsToTabs(QString xlsxPath)
{
    QXlsx::Document xls(xlsxPath);
    auto sheets = xls.sheetNames();
    ui.tabWidget->clear();

    for (auto& name : sheets) {
        xls.selectSheet(name);
        auto dim = xls.dimension();
        int rows = dim.rowCount(), cols = dim.columnCount();
        if (rows < 1) continue;

        // 1. 读第一行作为 header 列名
        QStringList headers;
        for (int c = 1; c <= cols; ++c)
            headers << xls.read(1, c).toString();

        // 2. 创建 model：行数减 1，列数不变
        auto* model = new QStandardItemModel(rows - 1, cols, this);
        model->setHorizontalHeaderLabels(headers);

        // 3. 从第 2 行开始填数据
        for (int r = 2; r <= rows; ++r) {
            for (int c = 1; c <= cols; ++c) {
                auto* item = new QStandardItem(xls.read(r, c).toString());
                model->setItem(r - 2, c - 1, item);
            }
        }

        // 4. 用 QTableView 显示
        auto* view = new QTableView(this);
        view->setModel(model);
        view->resizeColumnsToContents();

        ui.tabWidget->addTab(view, name);
    }
    if (!sheets.isEmpty())
        ui.tabWidget->setCurrentIndex(0);
}
```

还有别的用法具体查看下载库中的example文件夹。