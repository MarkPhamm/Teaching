# 1. Null
## 1.1 Checkout the null:
Check where's null coming frome? Why that's the case

## 1.2 Handle the null
## 1.2.1 Drop 
* **Option 1:**: Drop rows - If too many column values are missing -> Drop the whole rows (Set threshold)
* **Option 2:**: Drop columns - If too many rows values are mising -> Drop the whole column

## 1.2.2 Fill
### 1.2.2.1: Numerical  
* **Option 1:** Fill Mean - symmetrical distribution
* **Option 2:** Fill Median - skewed distribituion 

### 1.2.2.2: Categorical
* **Option 1**: Fill Mode

## 1.2.3 Flag (Ignore)
* Nulls are important
* Filled nulls might change the distribution

# 2. Duplicates
## 2.1 Identify duplications
* What's the unique combination?
* Check PK (primary key) -> Suppose to be not null and unique
* How to gen PK -> Generate PK from a group of columns

## 2.2 Handle duplication 
* rank/dense_rank/row_number() to remove duplicate, partition by unique combination, order by timestamp

# 3. Formatting

