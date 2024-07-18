[Ссылка](https://onecompiler.com/sqlserver/42kfynrdg) на online server с работающим кодом


```
;WITH 
CTE1 AS -- ID счетов, содержащих "Контур-Экстерн"
(SELECT bID
FROM dbo.BillContent
WHERE Product = N'Контур-Экстерн' 
),

CTE2 AS  -- проверка условий
(SELECT  
c.cID, Name, Inn, Num, BDate, PayDate, Product, Cost, Paid,
CASE 
-- если открытая поставка с макс. датой окончания -> 1
WHEN (rp.Since <= "2021-01-01" AND rp.UpTo >= "2021-01-01")
AND  Product = N'Контур-Экстерн' AND UpTo = MAX(UpTo) OVER(PARTITION BY c.cID) THEN 1 
-- если счет с макс. датой оплаты и с подключением(продлением) -> 2
WHEN TypeID IN (1, 2)  AND Product = N'Контур-Экстерн'
AND PayDate = MAX(PayDate) OVER(PARTITION BY c.cID) THEN 2 END AS bill_rank
-- иначе -> Null
FROM 
(SELECT * FROM dbo.Bills WHERE bID IN (SELECT bID FROM CTE1)) AS b -- только счета с "Контур-Экстерн"
LEFT JOIN dbo.Clients AS c ON c.cID = b.cID
LEFT JOIN dbo.BillContent AS bc ON bc.bID = b.bID
LEFT JOIN dbo.RetailPacks AS rp ON rp.bcID = bc.bcID
),

CTE3 AS  -- группировка по счетам
(SELECT  
cID, Name, Inn, Num, BDate, PayDate,
SUM(Cost) AS BillSum, SUM(Paid) AS BillPay,
MIN(bill_rank) AS bill_rank
FROM CTE2
GROUP BY cID, Name, Num, Inn, BDate, PayDate
),

CTE4 AS -- удаление счетов, не удовлетворяющих условиям
(SELECT *, 
FIRST_VALUE(Num) OVER(PARTITION BY cID ORDER BY Num DESC) AS top_row -- маркер 1-го счета
FROM CTE3 AS t1
WHERE bill_rank = (SELECT MIN(bill_rank)
                  FROM CTE3 AS t2
                  WHERE t1.cID = t2.cID)
)

-- оставляю только 1 счет для каждого клиента
SELECT Name, Inn, Num, BDate, PayDate,
CAST(BillSum AS DECIMAL(10,2)) AS BillSum, -- 4 знака в 2 знака после запятой
CAST(BillPay AS DECIMAL(10,2)) AS BillPay
FROM CTE4
WHERE Num = top_row
```
