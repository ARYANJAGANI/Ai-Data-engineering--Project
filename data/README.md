# Local source datasets

The CSV source datasets are intentionally not included in this repository. The original files are large (the order and order-item files exceed GitHub's regular per-file upload limit), and should be obtained and stored locally.

Expected layout:

```text
data/
  food/food.csv
  menu/menu.csv
  order_items/order_items.csv
  orders/orders.csv
  restaurants/restaurant.csv
  reviews/reviews.csv
  users/users.csv
```

The Airflow DAG expects matching files to have been uploaded to folders under the Snowflake stage `ZOMATO.RAW.ZOMATO_RAW_STAGE`. Configure the stage, file formats, target raw tables, and access permissions in your Snowflake account before running the pipeline. Do not place credentials or other secrets in these data files.
