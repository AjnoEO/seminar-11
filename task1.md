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
   ```
   "Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.376..0.376 rows=0 loops=1)"
   "  Recheck Cond: (category IS NULL)"
   "  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.038..0.038 rows=0 loops=1)"
   "        Index Cond: (category IS NULL)"
   "Planning Time: 2.739 ms"
   "Execution Time: 0.916 ms"
   ```
   
   *Объясните результат:*   
   СУБД находит страницы, в которых есть нужные строчки таблицы (Index Cond), используя битмап-индекс (Bitmap Index Scan). Затем она находит необходимые (Recheck Cond) строки только в этих страницах (Bitmap Heap Scan)

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
   ```
   "Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=32.178..32.179 rows=0 loops=1)"
   "  Recheck Cond: ((category)::text = 'INDEX'::text)"
   "  Rows Removed by Index Recheck: 150000"
   "  Filter: ((author)::text = 'SYSTEM'::text)"
   "  Heap Blocks: lossy=1280"
   "  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.589..0.590 rows=12800 loops=1)"
   "        Index Cond: ((category)::text = 'INDEX'::text)"
   "Planning Time: 9.415 ms"
   "Execution Time: 32.894 ms"
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*   
   СУБД находит страницы, в которых есть строчки таблицы с нужной категорией (Index Cond), используя битмап-индекс (Bitmap Index Scan). Затем она находит строки с нужной категорией (Recheck Cond) только в этих страницах (Bitmap Heap Scan), и среди них находит строчки с нужным автором (Filter). Индекс по авторам в итоге не пригодился, потому что он позволяет быстро находить нужного автора только во всей таблице, а не в её подмножестве

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   "Sort  (cost=3163.11..3163.12 rows=5 width=7) (actual time=50.758..50.761 rows=6 loops=1)"
   "  Sort Key: category"
   "  Sort Method: quicksort  Memory: 25kB"
   "  ->  HashAggregate  (cost=3163.00..3163.05 rows=5 width=7) (actual time=50.670..50.680 rows=6 loops=1)"
   "        Group Key: category"
   "        Batches: 1  Memory Usage: 24kB"
   "        ->  Seq Scan on t_books  (cost=0.00..2788.00 rows=150000 width=7) (actual time=0.026..12.016 rows=150000 loops=1)"
   "Planning Time: 7.222 ms"
   "Execution Time: 51.553 ms"
   ```
   
   *Объясните результат:*   
   СУБД считывает все строки таблицы (Seq Scan), объединяет (HashAggregate) их там, где категории совпадают (Group Key), а затем сортирует (Sort) по категории (Sort Key). Индекс по категориям в итоге не используется, поскольку битмапы не позволяют с большей эффективностью отсеивать повторяющиеся значения ― они позволяют находить только строки с конкретным заданным значением

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   "Aggregate  (cost=3163.04..3163.05 rows=1 width=8) (actual time=21.334..21.336 rows=1 loops=1)"
   "  ->  Seq Scan on t_books  (cost=0.00..3163.00 rows=15 width=0) (actual time=21.328..21.329 rows=0 loops=1)"
   "        Filter: ((author)::text ~~ 'S%'::text)"
   "        Rows Removed by Filter: 150000"
   "Planning Time: 2.516 ms"
   "Execution Time: 21.369 ms"
   ```
   
   *Объясните результат:*   
   СУБД считывает все строки таблицы (Seq Scan), среди них находит строчки с нужным автором (Filter), и собирает их в одну строчку (Aggregate), подсчитывая. Индекс по авторам в итоге не пригодился, поскольку битмапы не позволяют с большей эффективностью выделять строчки, начинающиеся с заданного префикса ― они позволяют находить только строки с конкретным заданным значением

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
    ```
    "Aggregate  (cost=3539.88..3539.89 rows=1 width=8) (actual time=184.625..184.626 rows=1 loops=1)"
    "  ->  Seq Scan on t_books  (cost=0.00..3538.00 rows=750 width=0) (actual time=184.577..184.617 rows=1 loops=1)"
    "        Filter: (lower((title)::text) ~~ 'o%'::text)"
    "        Rows Removed by Filter: 149999"
    "Planning Time: 1.871 ms"
    "Execution Time: 184.663 ms"
    ```
   
    *Объясните результат:*   
    СУБД считывает все строки таблицы (Seq Scan), среди них находит строчки с нужным названием (Filter), и собирает их в одну строчку (Aggregate), подсчитывая. Регистронезависимый индекс по названиям в итоге не пригодился, поскольку индексы не позволяют с большей эффективностью выделять строчки, начинающиеся с заданного префикса ― они позволяют находить только строки с конкретным заданным значением

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
    ```
    "Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=2.046..2.047 rows=0 loops=1)"
    "  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
    "  Rows Removed by Index Recheck: 8846"
    "  Heap Blocks: lossy=128"
    "  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.128..0.129 rows=1280 loops=1)"
    "        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
    "Planning Time: 2.552 ms"
    "Execution Time: 2.094 ms"
    ```
   
    *Объясните результат:*   
    СУБД находит страницы, в которых есть строчки таблицы с нужными категорией и автором (Index Cond), используя битмап-индекс (Bitmap Index Scan). Затем она находит строки с нужными категорией и автором (Recheck Cond) только в этих страницах (Bitmap Heap Scan). В этот раз и категории, и авторы сортируются индексом, потому что он один составной и работает сразу с обоими значениями