+++
date = "2020-04-23T12:00:00+03:00"
draft = true
title = "Core Data"
tags = ["ios", "swift"]
+++

Core Data - фреймворк для работы с базой дынных в приложениях. С его помощью можно хранить и управлять данными. Я не часто его использовал и у меня никак не было времени, чтобы разобраться с ним. Но на этих выходных время пришло.

Чтобы разобраться в принципах работы с Core Data я хочу написать небольшое туду приложение. Звучит банально, но в этом приложении список дел можно будет сохранять ка изображение и делать его заставкой на экране.

<!--more-->

Начнем с создания нового проекта. Желательно отметить галочку как указано на картинке. В этом случае в файле `AppDelegate.swift` сгенерируется дополнительный код и добавится специальный файл `Memo.xcdatamodeld`. Memo - это название моего проекта.

![](/img/core-data/create.png)
