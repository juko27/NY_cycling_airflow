#### Пайплайн выгрузки ежедневных отчётов по количеству поездок на велосипедах в городе Нью-Йорк, который:

1. Отслеживает появление новых файлов в бакете на AWS S3. 
   (Представим, что пользователь или провайдер данных будет загружать 
   новые исторические данные по поездкам в наш бакет);
2. При появлении нового файла запускается оператор импорта данных в 
   созданную таблицу базы данных Clickhouse;
3. Далее сформируются таблицы с ежедневными отчётами по следующим 
   критериям:
   – количество поездок в день
   – средняя продолжительность поездок в день
   – распределение поездок пользователей, разбитых по категории «gender»
4. Данные статистики загружаются на специальный S3 бакет (reports) с 
   хранящимися отчётами по загруженным файлам.



#### В AWS создаем и конфигурируем:

1. Публичный бакет Amazon S3, куда будут загружаться файлы от пользователей (tripdata) и отчеты с обработанными данными (reports);
2. Amazon MWAA, куда загружаем наш даг
3. ClickHouse Cluster on AWS для обработки данных, в котором создаем таблицу report

#### Даг состоит из 6 Задач:
1. *new_file_onto_S3_trigger*:
Задача представляет собой сенсор, который проверяет содержимое бакета tripdata на появление файла 
"{{ ds.format('%Y%m') }}-citibike-tripdata.csv.zip", где ds.format('%Y%m') - дата запуска 
2. *new_JC_file_onto_S3_trigger*:
Аналогичная задача, но настроенная на файл "JC-{{ ds.format('%Y%m') }}-citibike-tripdata.csv.zip"
3. *print_message*:
PythonOperator, который после срабатывания сенсоров на оба файла выводит сообщение "Both files have arrived in s3 bucket"
4. *unziped_files_to_s3*:
Задача, которая извлекает csv файлы из zip архива, который приходит в бакет от пользователя/провайдера данных. По завершении задачи в бакете tripdata появляются одноименные архиву csv файлы.
5. *ClickHouse_transform*:
Задача на транформарцию данных в ClickHouse с помощью плагина airflow_clickhouse_plugin.
Результатом запросов являются 3 файла, которые загружается в s3 бакет reports:

#####  – количество поездок в день (daytrips.csv)

![image](https://github.com/VivSRD/NY_cycling_airflow/blob/main/screens/1.jpg)
   
##### – средняя продолжительность поездок в день (avg_duration.csv)

![image](https://github.com/VivSRD/NY_cycling_airflow/blob/main/screens/2.jpg)
   
##### – распределение поездок пользователей, разбитых по категории «gender» (daytrips_per_gender.csv)

![image](https://github.com/VivSRD/NY_cycling_airflow/blob/main/screens/3.jpg)


6. *report_aggregate*:
PythonOperator, который выводит в результат обработки данных из п.5 на печать.


#### Используемые функции:

##### unzip():

Args:
   bucket: tripdata
   prefix: {{ ds.format('%Y%m') }}-citibike-tripdata.csv.zip
   output: {{ ds.format('%Y%m') }}-citibike-tripdata.csv

##### new_file_detection():

Args:
   output: "Both files have arrived in s3 bucket"
