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
    "Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=23.446..224.594 rows=499564 loops=1)"
    "  Recheck Cond: (category = 'A'::text)"
    "  Heap Blocks: exact=8334"
    "  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=21.774..21.775 rows=499564 loops=1)"
    "        Index Cond: (category = 'A'::text)"
    "Planning Time: 1.900 ms"
    "Execution Time: 248.901 ms"
    
    *Объясните результат:*
    [Ваше объяснение]

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on test_cluster  (cost=5544.95..20092.70 rows=497100 width=39) (actual time=21.960..116.714 rows=499564 loops=1)"
    "  Recheck Cond: (category = 'A'::text)"
    "  Heap Blocks: exact=4164"
    "  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5420.68 rows=497100 width=0) (actual time=21.107..21.108 rows=499564 loops=1)"
    "        Index Cond: (category = 'A'::text)"
    "Planning Time: 2.538 ms"
    "Execution Time: 139.711 ms"
    
    *Объясните результат:*
    [Ваше объяснение]

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    [Ваше сравнение]