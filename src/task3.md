## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster
        Recheck Cond: (category = 'A')
        Heap Blocks: exact=8334
        -> Bitmap Index Scan on test_cluster_cat_idx
            Index Cond: (category = 'A')
    Planning Time: 2.676 ms
    Execution Time: 149.345 ms

    
    *Объясните результат:*
    PostgreSQL использует Bitmap Index Scan по индексу test_cluster_cat_idx, так как условие category = 'A' возвращает очень большое количество строк (≈ 500 000).
    Bitmap Scan сначала формирует bitmap страниц, содержащих подходящие строки, а затем выполняет Bitmap Heap Scan, читая множество блоков таблицы.
    Так как таблица не кластеризована, строки с категорией 'A' распределены по большому количеству страниц (Heap Blocks: exact=8334), что приводит к значительному числу операций ввода-вывода и увеличенному времени выполнения запроса.   

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    Команда CLUSTER физически перестроила таблицу test_cluster в соответствии с индексом test_cluster_cat_idx.
    Теперь строки с одинаковым значением category расположены рядом друг с другом на диске, что снижает количество случайных чтений при выборках по этому столбцу.

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster
        Recheck Cond: (category = 'A')
        Heap Blocks: exact=4172
        -> Bitmap Index Scan on test_cluster_cat_idx
            Index Cond: (category = 'A')
    Planning Time: 1.829 ms
    Execution Time: 107.516 ms

    
    *Объясните результат:*
    После кластеризации таблицы строки с категорией 'A' были физически сгруппированы на диске.
    Это позволило PostgreSQL читать меньшее количество блоков таблицы (Heap Blocks: exact=4172 вместо 8334 до кластеризации).
    Несмотря на то, что план выполнения остался тем же (Bitmap Index Scan + Bitmap Heap Scan), сократилось число обращений к диску, что привело к снижению времени выполнения запроса.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    Кластеризация таблицы по индексу test_cluster_cat_idx существенно улучшила производительность запроса.
    Физическая группировка строк с одинаковым значением category уменьшила количество считываемых блоков и снизила время выполнения запроса примерно на 28%.
    Кластеризация особенно эффективна для запросов, возвращающих большие объёмы данных по индексируемому столбцу.