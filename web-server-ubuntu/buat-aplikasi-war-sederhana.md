# Buat Aplikasi War Sederhana

mywebapp/\
├── index.jsp\
└── WEB-INF/\
└── web.xml



Buat file index.jsp

```
index.jsp
```

isi file tersebut dengan kodingan dibawah ini

```
<%@ page language="java" contentType="text/html; charset=UTF-8" %>
<html>
<head>
    <title>Hello WebLogic!</title>
</head>
<body>
    <h1>Selamat datang di Web App sederhana</h1>
    <p>Ini aplikasi JSP sederhana yang dibungkus menjadi file WAR.</p>
</body>
</html>

```



Buat file didalam folder WEB-INF/web.xml

```
web.xml
```

isi file tersebut dengan kodingan dibawah ini

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <display-name>My Simple Web App</display-name>
    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```



Tetap di dalam folder mywebapp dan jalan kan perintah dibawh ini

```
jar cvf mywebapp.war *
```

