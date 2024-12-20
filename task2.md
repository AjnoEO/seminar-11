## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*   
    ```
    "Bitmap Heap Scan on t_books  (cost=21.03..1371.25 rows=750 width=33) (actual time=0.316..0.317 rows=1 loops=1)"
     "  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
     "  Heap Blocks: exact=1"
     "  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.287..0.288 rows=1 loops=1)"
     "        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
     "Planning Time: 2.176 ms"
     "Execution Time: 0.360 ms"
    ``` 
    
    *Объясните результат:*    
    СУБД находит все страницы (Bitmap Index Scan), в которых есть подходящие строчки (Index Cond), и затем находит в них (Bitmap Heap Scan) те строчки, которые нам подходят (Recheck Cond)

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     "Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.051..0.052 rows=1 loops=1)"
     "  Index Cond: ((item_key)::text = '0000000455'::text)"
     "Planning Time: 0.209 ms"
     "Execution Time: 0.504 ms"
     ```
     
     *Объясните результат:*   
     СУБД при помощи индекса первичного ключа (Index Scan) находит подходящую строчку (Index Cond). На это тратится 0.5 мс

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     "Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.131..0.132 rows=1 loops=1)"
     "  Index Cond: ((item_key)::text = '0000000455'::text)"
     "Planning Time: 0.158 ms"
     "Execution Time: 0.151 ms"
     ```
     
     *Объясните результат:*   
     СУБД при помощи индекса первичного ключа (Index Scan) находит подходящую строчку (Index Cond). На это тратится 0.15 мс

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     "Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.098..0.098 rows=0 loops=1)"
     "  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
     "Planning Time: 2.084 ms"
     "Execution Time: 0.121 ms"
     ```
     
     *Объясните результат:*   
     СУБД при помощи индекса по значению (Index Scan) находит подходящую строчку (Index Cond). На это тратится 0.12 мс

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     "Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.091..0.091 rows=0 loops=1)"
     "  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
     "Planning Time: 2.966 ms"
     "Execution Time: 0.113 ms"
     ```
     
     *Объясните результат:*   
     СУБД при помощи индекса по значению (Index Scan) находит подходящую строчку (Index Cond). На это тратится 0.11 мс

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Поиск по ключу в кластеризованной таблице заметно быстрее, чем в обычной. А при поиске по значению тратится примерно одно и то же количество времени. Это связано с тем, что кластеризацию провели по первичному ключу, и строки отсортировались в памяти именно по нему. Поэтому кластеризация помогает эффективнее находить ключ, но бесполезна при поиске значения