---
title: 记一次QGIS数据图层保存问题
tags:
  - Qt
  - QGIS
  - C++
categories:
  - Qt开发
author: upstairs
abbrlink: 29b4bca
date: 2025-07-08 10:49:12
---

# 记一次QGIS数据图层保存问题
## 问题
#### 1.对数据源gpkg文件进行图层名修改
#### 2.对数据源gpkg文件进行图层字段名称修改并保存为新的gpkg文件

### 代码：
``` C++
    QString geomStr = QgsWkbTypes::displayString(lyr->wkbType());
    QString memUri = QString("%1?crs=%2").arg(geomStr, lyr->crs().authid());
    QgsVectorLayer mem(memUri, newLayerName, "memory");
    auto prov = mem.dataProvider();
    prov->addAttributes(newFields.toList());
    mem.updateFields();

    QgsFeatureIterator it = lyr->getFeatures();
    QgsFeature feat;
    QList<QgsFeature> feats;
    while (it.nextFeature(feat)) {
        QgsFeature nf(newFields);
        nf.setGeometry(feat.geometry());
        QgsAttributes atts;
        for (const QgsField& f : oldFields)
            atts.append(feat.attribute(f.name()));
        nf.setAttributes(atts);
        feats.append(nf);
    }
    prov->addFeatures(feats);

    QgsVectorFileWriter::SaveVectorOptions opts;
    opts.driverName = "GPKG";
    opts.layerName = newLayerName;
    opts.actionOnExistingFile = firstLayer
        ? QgsVectorFileWriter::CreateOrOverwriteFile
        : QgsVectorFileWriter::CreateOrOverwriteLayer;

    auto err = QgsVectorFileWriter::writeAsVectorFormatV3(
        &mem,
        dst,
        QgsProject::instance()->transformContext(),
        opts
    );
    if (err != QgsVectorFileWriter::NoError) {
        QString logErr = QStringLiteral("写入失败: %1|%2 错误码=%3")
            .arg(QFileInfo(src).fileName(), layerName)
            .arg(int(err));
    }
```
这段代码是在处理每个子图层时执行的。它的作用是将原始图层中的所有特征复制到新的内存图层中，并将新的内存图层保存到新的gpkg文件中。
在外部系统中，此段代码正常运行，且完成保存操作。
但是在另外一个项目中同样的代码出现字段缺失，只有图层名称被修改了。

## 解决过程
### 1.检查代码
经过和AI的斗智斗勇后，发现问题出在`addFeatures`方法上。这个东西是需要先设置开始编辑才能用的，所以需要在`addFeatures`方法之前加上`startEditing()`方法。(ps:期间还将图层的创建方式变为指针形式，直接new一个图层)
``` C++
mem.startEditing();
```
这就很抽象了，这是个内存图层啊。(最后不需要) 字段有了，但是没有值，没写进去，但是调试看到确实是有值的，现在只有第一列有值，还是序号列。然后开始网络闲逛，看到字段要检查类型，长度等。emmm，回头使用QGIS一看，哦，原来是长度没有。这里之前的创建的fields只保存了源图层的字段名和类型。好家伙，修改一看，没用，哈哈哈哈。
``` C++
QgsFields newFields;
newFields.append(QgsField("新的字段名称", field.type()));
```
这咋办，和AI探讨了一下，它说字段要和图层字段相匹配。嗯？Looking my Eyes！我难道不配吗？应该不是这个，回头望向内存图层的创建了，我总看着这个图层创建的不对，换了一种方式
``` C++
QgsFields newFields;
newFields.append(QgsField("新的字段名称", field.type(), field.length(), field.precision()));
// 使用QgsMemoryProviderUtils直接创建
QgsVectorLayer* memLayer = QgsMemoryProviderUtils::createMemoryLayer(
    newLayerName, newFields, lyr->wkbType(), lyr->crs());
```
这里的话，我直接拿源图层的修改字段去创建了，然后再去插入字段的值。然后保存成功。(所以可能确实是字段不匹配的问题)
ps:这里面还要注意一个就是保存数据，需要数据提供者去保存,是`memLayer->dataProvider()->addFeatures(feats);`，而不是`memLayer->addFeatures(feats);`。

## 总结
需要建虚拟图层的话，使用QgsMemoryProviderUtils去创建赋值的话就去用数据提供者去保存，而不是直接用图层去保存。