# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books
      Recheck Cond: (category IS NULL)
      -> Bitmap Index Scan on t_books_brin_cat_idx
         Index Cond: (category IS NULL)
   Planning Time: 1.760 ms
   Execution Time: 0.334 ms

   
   *Объясните результат:*
   PostgreSQL использовал BRIN индекс по колонке category. Сначала был выполнен Bitmap Index Scan, который определил страницы таблицы, где потенциально могут находиться строки с NULL значением. Затем с помощью Bitmap Heap Scan были прочитаны соответствующие страницы и выполнена перепроверка условия (Recheck Cond). Несмотря на использование индекса, фактическое количество найденных строк равно нулю, что указывает на отсутствие записей с NULL в поле category.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books
      Recheck Cond: ((category = 'INDEX'))
      Filter: (author = 'SYSTEM')
      Rows Removed by Index Recheck: 150000
      -> Bitmap Index Scan on t_books_brin_cat_idx
         Index Cond: (category = 'INDEX')
   Planning Time: 1.044 ms
   Execution Time: 16.825 ms

   
   *Объясните результат (обратите внимание на bitmap scan):*
   PostgreSQL использовал bitmap-сканирование с BRIN индексом по колонке category. С помощью Bitmap Index Scan были определены страницы таблицы, которые потенциально содержат строки с категорией INDEX. Далее выполняется Bitmap Heap Scan, который читает эти страницы и перепроверяет условие (Recheck Cond). Условие по автору (author = 'SYSTEM') применено как фильтр, так как индекс по author в данном плане не был использован. Большое количество строк было отброшено на этапе перепроверки, что характерно для BRIN индексов из-за их грубой гранулярности (работа на уровне страниц, а не отдельных строк).

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   Sort
      Sort Key: category
      -> HashAggregate
         Group Key: category
         -> Seq Scan on t_books
   Planning Time: 1.424 ms
   Execution Time: 56.416 ms

   
   *Объясните результат:*
   Для выполнения запроса PostgreSQL использовал последовательное сканирование таблицы (Seq Scan), так как необходимо обработать все строки для получения уникальных значений. Далее применяется агрегатная операция HashAggregate, которая группирует строки по полю category. После этого выполняется сортировка (Sort) для соблюдения условия ORDER BY. BRIN индекс не используется, поскольку он не предназначен для точного извлечения уникальных значений и не может эффективно заменить полное сканирование таблицы.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   Aggregate
      -> Seq Scan on t_books
         Filter: (author LIKE 'S%')
         Rows Removed by Filter: 150000
   Planning Time: 1.823 ms
   Execution Time: 47.192 ms

   
   *Объясните результат:*
   PostgreSQL выполнил последовательное сканирование таблицы (Seq Scan), так как BRIN индекс по колонке author не подходит для строковых операций с шаблонами (LIKE). Каждая строка таблицы была проверена на соответствие условию, после чего применена агрегатная функция COUNT. Все строки были отброшены фильтром, что отражено в показателе Rows Removed by Filter.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   Aggregate
      -> Seq Scan on t_books
         Filter: (LOWER(title) LIKE 'o%')
            Rows Removed by Filter: 149999
   Planning Time: 3.252 ms
   Execution Time: 56.384 ms

   
   *Объясните результат:*
   Несмотря на создание функционального индекса по LOWER(title), PostgreSQL в данном случае использовал Seq Scan. Это может происходить, если планировщик решил, что стоимость последовательного сканирования ниже, чем использование индекса, например, когда количество подходящих строк очень мало. Фильтр LOWER(title) LIKE 'o%' применяется ко всем строкам таблицы, после чего агрегатная функция COUNT подсчитывает подходящие записи.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books
      Recheck Cond: ((category = 'INDEX') AND (author = 'SYSTEM'))
      Rows Removed by Index Recheck: 8798
      -> Bitmap Index Scan on t_books_brin_cat_auth_idx
         Index Cond: ((category = 'INDEX') AND (author = 'SYSTEM'))
   Planning Time: 0.859 ms
   Execution Time: 1.859 ms

   
   *Объясните результат:*
   После создания составного BRIN-индекса PostgreSQL использует его для одновременной фильтрации по category и author.
   Сначала выполняется Bitmap Index Scan, который находит страницы таблицы, содержащие подходящие записи. Затем Bitmap Heap Scan читает эти страницы и перепроверяет условия (Recheck Cond).
   В сравнении с отдельными индексами, количество удалённых на перепроверке строк (Rows Removed by Index Recheck) стало меньше, а время выполнения значительно снизилось (с ~16 мс до ~1,8 мс), что демонстрирует эффективность составного BRIN-индекса при комбинированных условиях.