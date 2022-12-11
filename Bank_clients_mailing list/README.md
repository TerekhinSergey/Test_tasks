# Формирование сегмента клиентов для маркетинговой рассылки с помощью SQL на сгенерированных данных в Python

В данном проекте работа идёт с базой данных банка на сгенерированных в Python данных, которая хранит информацию о клиентах и их картах. Для проведения маркетинговой кампании необходимо выделить нужный сегмент клиентов с помощью SQL-запроса. ER-диаграмма приложена.

CREATE TABLE results (
    client_id serial PRIMARY KEY,
    hash_id int,
    rest_rub real,
    tarif_name text,
    tel_num text,
    month_activated int, 
    rest_thousands int, 
    phone_number text,
    groups int
); 
INSERT INTO results (client_id, hash_id, rest_rub, tarif_name, tel_num, month_activated,   rest_thousands, phone_number, groups) 
       SELECT tel.client_id,
       cards.hash_id,
       acc.rest_rub,
       cat.tarif_name,
       tel.tel_num,
       EXTRACT (MONTH FROM cards.date_activated::date) AS month_activated,
       TRUNC(acc.rest_rub, -3) AS rest_thousands,
       SUBSTR(tel.tel_num, 2, 3) AS phone_number, 
       CASE 
   	   WHEN (RIGHT(tel.tel_num, 1)::int)%2=0 THEN 1 
           ELSE 2 
       END AS group
        FROM tel_catalog AS tel
   LEFT JOIN cards ON tel.client_id=cards.client_id
   LEFT JOIN accounts AS acc ON acc.rest_rub=cards.acc_num
  INNER JOIN catalog AS cat ON cards.tarif_id=cat.tarif_id
  INNER JOIN card_status AS cs ON cards.hash_id=cs.hash_id
   LEFT JOIN black_list AS bl ON tel.client_id=bl.client_id
       WHERE cat.tarif_name=’амурский тигр’ 
  	     AND cards.date_activated::date BETWEEN ‘2016-04-01’ AND ‘2016-06-30’
	     AND tel.client_id IN (SELECT tel.client_id
			             FROM tel_catalog AS tel
  			        LEFT JOIN cards ON tel.client_id=cards.client_id
  			        LEFT JOIN accounts AS acc ON acc.rest_rub=cards.acc_num
        			          WHERE acc.report_date::date = ‘2016-07-01’ AND rest_rub >= 1000)
             AND cards.hash_id IN (SELECT hash_id
       				     FROM (SELECT DISTINCT hash_id, LAST_VALUE(status) OVER (PARTITION BY cs.hash_id ORDER BY cs.date_status ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS status 
					     FROM card_status AS cs) AS a
					    WHERE status=’open’)
       	     AND cards.hash_id IN (SELECT cards.hash_id
				     FROM (SELECT DISTINCT cards.hash_id, MAX(acc.rest_rub) OVER (PARTITION BY cards.client_id)
					     FROM cards
				       INNER JOIN accounts AS acc ON acc.rest_rub=cards.acc_num) AS b)
	     AND bl.client_id IS NULL

*Разметка контрольной группы*

WITH 
a AS (SELECT client_id,
		  groups,
    		  PERCENT_RANK() OVER (PARTITION BY groups ORDER BY client_id) AS percentage
        FROM results);
SELECT client_id,
       groups,
       percentage,
       CASE 
           WHEN groups=1 THEN concat ('Уважаемый', 'client_name,') 
           ELSE concat ('Ваш остаток по карте составляет', 'acc.rest_rub') 
       END AS sms_text,
       CASE
           WHEN percentage > 0.15 THEN 0
           ELSE 1
       END AS control_group
  FROM a
