Исполнители: Мартюшев, Маров, Пономарева

Настройки производились в файле postgresql.conf
Параметр synchronous_commit влияет на фиксирование ответов от ведомых серверов и фиксации их в хранилище, также такой вариант защищает от сбоев как ОС так и самой PG.

Таким образом лучший вариант параметра

```
synchronous_commit = remote_apply
```

При отключении данного параметра (переведение значения на off) отключается локальная фииксация и защита от сбоев таким образом демонстрируя худший вариант репликации.

```
synchronous_commit = off
```

Скрипт использованный в работе:
```
import psycopg2
import os
from psycopg2 import Error

def close_connection(Primary_ip):
    os.system(f"ssh -i p0 postgres@{Primary_ip} 'ifconfig eth0 down'")
    os.system(f"ssh -i p0 postgres@{Primary_ip} 'sleep 10'")
    os.system(f"ssh -i p0 postgres@{Primary_ip} 'ifconfig eth0 down'") 

def check_ping(hostname):
    response = os.system("ping " + hostname)
    return response

def compare(Primary_ip,Standby_ip):
    while check_ping(Primary_ip) == 0:
        connection = psycopg2.connect(dbname='table_test', user='postgres', password='123', host = Primary_ip )
        cur = connection.cursor()
        cur.execute('SELECT COUNT(*) FROM table_test;')
        rec_count = cur.fetchone()

        connection2 = psycopg2.connect(dbname='table_test', user='postgres', password='123', host = Standby_ip  )
        cur2 = connection2.cursor()
        cur2.execute('SELECT COUNT(*) FROM table_test;')
        rec_count_repl = cur2.fetchone()

        print(f"Количество записей table_test основной БД: {rec_count[0]}\n Количество записей table_test реплики БД: {rec_count_repl[0]}")
        if rec_count[0] == rec_count_repl[0]:
            print("Идентичное количество записей")
            exit(0)
        if rec_count[0] != rec_count_repl[0]:
            print(f"Несовпадение количества запсей: {(rec_count[0]-rec_count_repl[0])}")
            exit(0)

async def start_script(Primary_ip,Standby_ip):
    connection = psycopg2.connect(dbname='table_test', user='postgres', password='123', host = Primary_ip )
    try:
        i = 0
        while i < 100000:
            async with connection.transaction():
                await connection.execute(f"INSERT INTO table_test (Name, Phone) VALUES ('{i}','{i}');")
            if i == 1000:
                close_connection()
            i = i + 1
    except:
        pass

def main():

    Primary_ip = '192.168.23.140'
    Standby_ip = '192.168.23.141'
    
    while( True ):
        start_script(Primary_ip,Standby_ip)

if __name__ == "__main__":
    main()
```