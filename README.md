# SQL-data-cleaning

This is an educational project on data cleaning and preparation using SQL. The original database in CSV format is located in the file club_member_info.csv. Here, we will explore the steps that need to be applied to obtain a cleansed version of the dataset.

## Data introduction
```sql
SELECT * FROM club_member_info cmi LIMIT 10;
```
The result:
|full_name|age|martial_status|email|phone|full_address|job_title|membership_date|
|---------|---|--------------|-----|-----|------------|---------|---------------|
|addie lush|40|married|alush0@shutterfly.com|254-389-8708|3226 Eastlawn Pass,Temple,Texas|Assistant Professor|7/31/2013|
|      ROCK CRADICK|46|married|rcradick1@newsvine.com|910-566-2007|4 Harbort Avenue,Fayetteville,North Carolina|Programmer III|5/27/2018|
|Sydel Sharvell|46|divorced|ssharvell2@amazon.co.jp|702-187-8715|4 School Place,Las Vegas,Nevada|Budget/Accounting Analyst I|10/6/2017|
|Constantin de la cruz|35||co3@bloglines.com|402-688-7162|6 Monument Crossing,Omaha,Nebraska|Desktop Support Technician|10/20/2015|
|  Gaylor Redhole|38|married|gredhole4@japanpost.jp|917-394-6001|88 Cherokee Pass,New York City,New York|Legal Assistant|5/29/2019|
|Wanda del mar       |44|single|wkunzel5@slideshare.net|937-467-6942|10864 Buhler Plaza,Hamilton,Ohio|Human Resources Assistant IV|3/24/2015|
|Joann Kenealy|41|married|jkenealy6@bloomberg.com|513-726-9885|733 Hagan Parkway,Cincinnati,Ohio|Accountant IV|4/17/2013|
|   Joete Cudiff|51|divorced|jcudiff7@ycombinator.com|616-617-0965|975 Dwight Plaza,Grand Rapids,Michigan|Research Nurse|11/16/2014|
|mendie alexandrescu|46|single|malexandrescu8@state.gov|504-918-4753|34 Delladonna Terrace,New Orleans,Louisiana|Systems Administrator III|3/12/1921|
| fey kloss|52|married|fkloss9@godaddy.com|808-177-0318|8976 Jackson Park,Honolulu,Hawaii|Chemical Engineer|11/5/2014|

## Copy Table
### Create new table for cleaning
```sql
-- club_member_info definition

CREATE TABLE club_member_info_cleaned (
	full_name VARCHAR(50),
	age INTEGER,
	martial_status VARCHAR(50),
	email VARCHAR(50),
	phone VARCHAR(50),
	full_address VARCHAR(50),
	job_title VARCHAR(50),
	membership_date VARCHAR(50)
);
```
### Copy all values from original table
```sql
INSERT INTO club_member_info_cleaned 
SELECT * FROM club_member_info;
```

## Cleanse table 
### full_name column
#### Remove space
```sql
UPDATE club_member_info_cleaned SET full_name = TRIM(full_name);
```
#### Uppercase name
```sql
UPDATE club_member_info_cleaned SET full_name = UPPER(full_name);
```

### age column
```sql
--check unreasonable age
SELECT * FROM club_member_info_cleaned cmic
WHERE age <18 OR age >90;
```
#### Correct to the first 2 digits if age over 100
```sql
UPDATE club_member_info_cleaned
SET age = CAST(SUBSTR(CAST(age AS TEXT), 1, 2) AS INTEGER)
WHERE age IS NOT NULL;
```
#### Fill in spaces (age=0) using mode
```sql
UPDATE club_member_info_cleaned
SET age = (SELECT age
           FROM club_member_info_cleaned
           GROUP BY age
           ORDER BY COUNT(age) DESC
           LIMIT 1)
WHERE age = 0;
```

### martial_status column
```sql
--check martial_status
SELECT DISTINCT martial_status FROM club_member_info_cleaned cmic
```
#### Correct spelling mistakes and adjust spaces to unknown
```sql
UPDATE club_member_info_cleaned
SET marital_status = CASE
    WHEN TRIM(martial_status) = 'divored' THEN 'divorced'
    WHEN TRIM(martial_status) = '' THEN 'unknown'
    ELSE marital_status
END;
```

### email and phone column
#### delete duplicate
```sql
--check duplicate email and phone
SELECT email, phone, COUNT(*) AS count
FROM club_member_info_cleaned
GROUP BY email, phone
HAVING COUNT(*) > 1;

--delete duplicate email and phone
DELETE FROM club_member_info_cleaned
WHERE ROWID NOT IN (
    SELECT MIN(ROWID)
    FROM club_member_info_cleaned
    GROUP BY email, phone
);
```
#### Replace blank phone number to NULL
```sql
UPDATE club_member_info_cleaned
SET phone = NULL
WHERE TRIM(phone) = '';
```

### job_title column
#### Replace blank to NULL
```sql
UPDATE club_member_info_cleaned
SET job_title = NULL
WHERE TRIM(job_title) = '';
```
### membership_date column
#### Replace years 1900s to 2000s
```sql
UPDATE club_member_info_cleaned
SET membership_date = REPLACE(membership_date, '/19', '/20')
WHERE membership_date LIKE '%/19__';
```

#### Format dd/mm/yyyy
```sql
--add new column: membership_date_cleaned
ALTER TABLE club_member_info_cleaned
ADD COLUMN membership_date_cleaned TEXT;

--add 0 in day and month value
UPDATE club_member_info_cleaned
SET membership_date_cleaned = 
    printf('%02d/%02d/%04d',
        CAST(substr(membership_date, 1, instr(membership_date, '/') - 1) AS INTEGER), -- day
        CAST(substr(
            membership_date,
            instr(membership_date, '/') + 1,
            instr(substr(membership_date, instr(membership_date, '/') + 1), '/') - 1
        ) AS INTEGER), -- month
        CAST(substr(membership_date, -4) AS INTEGER) -- year
    );

-- If the month > 12, swap the position of day and month
UPDATE club_member_info_cleaned
SET membership_date_cleaned = 
    CASE 
        -- If the month > 12, swap the position of day and month
        WHEN CAST(SUBSTR(membership_date_cleaned, 4, 2) AS INTEGER) > 12
        THEN SUBSTR(membership_date_cleaned, 4, 2) || '/' || SUBSTR(membership_date_cleaned, 1, 2) || '/' || SUBSTR(membership_date_cleaned, 7, 4)
        -- If no change is needed, keep the original date
        ELSE membership_date_cleaned
    END;
```

## Cleaned result
```sql
SELECT * FROM club_member_info_cleaned LIMIT 10;
```
The result:
|full_name|age|martial_status|email|phone|full_address|job_title|membership_date|membership_date_cleaned|
|---------|---|--------------|-----|-----|------------|---------|---------------|-----------------------|
|club_member_info_cleaned|40|married|alush0@shutterfly.com|254-389-8708|3226 Eastlawn Pass,Temple,Texas|Assistant Professor|7/31/2013|31/07/2013|
|ROCK CRADICK|46|married|rcradick1@newsvine.com|910-566-2007|4 Harbort Avenue,Fayetteville,North Carolina|Programmer III|5/27/2018|27/05/2018|
|SYDEL SHARVELL|46|divorced|ssharvell2@amazon.co.jp|702-187-8715|4 School Place,Las Vegas,Nevada|Budget/Accounting Analyst I|10/6/2017|10/06/2017|
|CONSTANTIN DE LA CRUZ|35|unknown|co3@bloglines.com|402-688-7162|6 Monument Crossing,Omaha,Nebraska|Desktop Support Technician|10/20/2015|20/10/2015|
|GAYLOR REDHOLE|38|married|gredhole4@japanpost.jp|917-394-6001|88 Cherokee Pass,New York City,New York|Legal Assistant|5/29/2019|29/05/2019|
|WANDA DEL MAR|44|single|wkunzel5@slideshare.net|937-467-6942|10864 Buhler Plaza,Hamilton,Ohio|Human Resources Assistant IV|3/24/2015|24/03/2015|
|JOANN KENEALY|41|married|jkenealy6@bloomberg.com|513-726-9885|733 Hagan Parkway,Cincinnati,Ohio|Accountant IV|4/17/2013|17/04/2013|
|JOETE CUDIFF|51|divorced|jcudiff7@ycombinator.com|616-617-0965|975 Dwight Plaza,Grand Rapids,Michigan|Research Nurse|11/16/2014|16/11/2014|
|MENDIE ALEXANDRESCU|46|single|malexandrescu8@state.gov|504-918-4753|34 Delladonna Terrace,New Orleans,Louisiana|Systems Administrator III|3/12/2021|03/12/2021|
|FEY KLOSS|52|married|fkloss9@godaddy.com|808-177-0318|8976 Jackson Park,Honolulu,Hawaii|Chemical Engineer|11/5/2014|11/05/2014|
